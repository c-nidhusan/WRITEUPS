# Relevant — TryHackMe — ROOT (SYSTEM)

**Date:** 2026-10-08 → 2026-10-11 (two sessions: 45m + 40m)
**Platform:** TryHackMe
**Outcome:** SYSTEM — **machine 18, Phase C room 3 of 5**
**Time:** ~1h25 total · Confidence 4 (day 1) → 3 (completion, tired)
**Approach:** day 1 fully solo; foothold pivot + privesc walkthrough-assisted (disclosed);
Google AI for smbclient `put` mechanics
**Tools:** nmap, gobuster, smbclient (anon), base64, ASPX webshell, metasploit handler,
PrintSpoofer64 (SeImpersonatePrivilege)

---

## 0. TL;DR chain

nmap (80 IIS 10 · 135 · 139/445 SMB · **3389 RDP**) → default IIS site = dead end →
**anonymous SMB** (`smbclient -L`, recalled from kenobi.md) → share **`nt4wrksv`** readable
*and writable* → `passwords.txt` = header says **"[Encoded]"** → `base64 -d` per line →
**Bob `!P@$$W0rD!123`** · **Bill `Juw4nnaM4n420696969!$$$`** → the share maps to a web
virtual directory served on **port 49663** → upload **ASPX reverse shell** via
`smbclient put` → trigger over HTTP → meterpreter → **PrintSpoofer64** (SeImpersonatePrivilege)
→ **SYSTEM**

## 1. Recon

| Port | Service | Means | Therefore |
|---|---|---|---|
| 80 | IIS 10 | default Windows site | dead — nothing but the stock page |
| 135/139/445 | RPC/SMB | file shares | **the real surface: anonymous nt4wrksv** |
| **3389** | RDP | interactive login | **an untested door — creds existed, never tried here** |
| (49663) | IIS virtual dir | found via walkthrough | **what `-p-` would have shown on day 1** |

The gap of the room: the top-1000 `-sV` scan was treated as full enumeration. It isn't —
Windows parks services on high ephemeral ports. **`-p-` first, `-sV` second, always.**

## 2. Vulnerability assessment (anonymous SMB)

```
smbclient -L //<ip> -N                      # share list, anonymous
smbclient //<ip>/nt4wrksv -N  →  ls         # passwords.txt
```

`passwords.txt` line one: **`[User Passwords - Encoded]`** — encoded, not hashed. john
loaded nothing because there was nothing to crack; `base64 -d` (one line per file — a
multi-line file breaks the decode) gave both creds. NT_STATUS codes were read and
self-corrected live (`ADMIN` → `ADMIN$`, per the Skynet lesson).

## 3. Foothold (writable share → webshell)

The share `nt4wrksv` is **writable anonymously and served by IIS as a virtual directory**.
That combination is the whole vulnerability: anyone can put a file where the web server
will execute it.

```
smb: \> lcd /home/Sanijdhu/HACKING/TOOLS/REVSHELL/    # local cwd — `put` is relative to it
smb: \> put shell.aspx                                # ASPX reverse shell
```

then `curl http://<ip>:49663/nt4wrksv/shell.aspx` (or browser) → handler catches →
meterpreter as the app-pool identity.

**Shell language must match the server:** a `.php` shell on stock IIS returns 404 — IIS
executes ASP/ASPX, not PHP. Self-caught from the 404 ("i used a php reverse shell on a
windows machine"). **Match the payload language to the server stack, or read the 404 as
"wrong language," not "wrong path."**

## 4. Privilege escalation (SeImpersonatePrivilege → PrintSpoofer)

The question the walkthrough answered but didn't explain — *"how did he know this machine
was vulnerable to this exe?"* — has a one-command answer:

```
whoami /priv
→ SeImpersonatePrivilege    Enabled
```

**SeImpersonatePrivilege** lets the account impersonate any client that authenticates to
it — service accounts (IIS app pools especially) hold it by design. **PrintSpoofer** abuses
the Printer Service: it makes SYSTEM connect to an attacker-controlled named pipe, then
*impersonates that token* → `CreateProcessAsUser()` → SYSTEM:

```
c:\inetpub\wwwroot\nt4wrksv> PrintSpoofer64.exe -i -c powershell.exe
[+] Found privilege: SeImpersonatePrivilege
PS C:\Windows\system32> whoami   →  nt authority\system
```

**The rule: any service-account shell on Windows → `whoami /priv` → if SeImpersonate is
enabled, PrintSpoofer/JuicyPotato/RoguePotato family.** It is the single most common
Windows privesc on CTF boxes and real IIS/MSSQL servers alike.

## 5. Misses and dead ends

- **`-p-` sweep skipped; the high port came from the walkthrough.** The one homework
  item from the last debrief was the one that got skipped — and it was exactly the
  missing door. Preflight and sweeps aren't ceremony; they're how the second path gets found.
- **Creds declared useless with 3389/RDP open in their own nmap output.** "No ftp, no ssh,
  no login page" — true, and beside the point: RDP is a login service, and two sets of
  credentials were in hand. Untested, so unproven either way — but the door was never
  knocked. **Every open port is a candidate for every credential you hold.**
- **PHP shell on IIS** (above) — self-caught from the 404. Credit.
- **`smbclient put` flail:** typo (`sell` vs `shell`) re-fired three times before being
  read — the identical-command retry pattern, sixth sighting, self-caught this time;
  then `put ~/...` (tilde not expanded inside smbclient) and `put /home/...` (leading `/`
  read as *remote* path). **Fix: `lcd <local dir>` then `put <filename>`.** Local paths
  don't exist to the remote server.
- **Privesc path unknown → walkthrough.** Honest cause: the sweep block in hackpark.md
  listed `whoami /groups` but **not `whoami /priv`** — mentor's gap, now fixed there.
  The token-impersonation instinct (Alfred's incognito) was the right family, wrong tool:
  incognito steals *existing* tokens; PrintSpoofer *manufactures* a SYSTEM one.

## 6. Lessons

1. **`-p-` before conclusions.** Top-1000 is a sample, not the port space — especially on
   Windows, where the interesting service is often at 49xxx.
2. **Anonymous writable share + web server = upload vector.** The share wasn't just loot
   storage; it *was* the webroot. When a share name appears in a URL path, the mapping is
   the vulnerability.
3. **`whoami /priv` is the Windows sudo -l.** SeImpersonate → potato family → SYSTEM.
   One command, the most repeated privesc in Windows CTFs.
4. **Payload language must match the stack:** ASPX for IIS, PHP for Apache/LAMP. A 404 on
   an uploaded shell means "server won't execute this language" as often as "wrong path."
5. **Credentials are keys; ports are doors.** Every key gets tried in every door — RDP
   included — before being called useless.
6. **`lcd` + `put`:** in smbclient, the local side needs its own cd. `~` and absolute local
   paths in `put` are read against the *remote* tree.
