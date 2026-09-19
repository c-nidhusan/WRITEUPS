# Ice — TryHackMe — SYSTEM

**Date:** 2026-09-19
**Platform:** TryHackMe
**Outcome:** NT AUTHORITY\SYSTEM — **machine 11, first solo-run Windows privesc (Phase B)**
**Time:** 45m · Confidence self-rated 3/5 · zero mentor pings
**Tools:** nmap (-sS/-sV), Metasploit (icecast_header, local_exploit_suggester,
bypassuac_eventvwr), meterpreter (sysinfo/getprivs/migrate), kiwi (creds_all)

---

## 0. TL;DR chain

nmap -sS/-sV (Windows 7 x64 SP1, DARK-PC; 445 SMB · 3389 ms-wbt-server · **8000 Icecast**) →
room-named vuln: **CVE-2004-1561** (Icecast header overwrite) → msf
`exploit/windows/http/icecast_header` (rank: great) → meterpreter as **Dark-PC\Dark** (user —
not SYSTEM: a real privesc phase this time) → `local_exploit_suggester` →
**bypassuac_eventvwr** → elevated token (getprivs: SeTakeOwnership, SeDebug…) →
**`migrate -N spoolsv.exe`** → NT AUTHORITY\SYSTEM → `load kiwi` → `creds_all` →
**Dark:Password01!** (wdigest/kerberos plaintext)

## 1. Recon

| Port | Service | Means | Therefore |
|---|---|---|---|
| 135, 49152–49160 | msrpc | RPC plumbing | noise |
| 139 / 445 | SMB | file sharing | enumerated on Blue; not the door here |
| 3389 | ms-wbt-server | **RDP** (showed as `tcpwrapped` under -sV) | a door credentials walk through |
| **8000** | **Icecast streaming server** | niche app, 2004 vintage | **the attack surface** |

First scan used `-sn -sV` and printed no ports — `-sn` is ping-only, it skips the port scan
entirely. Self-caught in one step. The room's "service you can't see": 3389 under `-sV`
(`tcpwrapped`) = **ms-wbt-server**, RDP.

## 2. Vulnerability assessment

One non-Microsoft service on the box: **Icecast** on 8000, and old niche software is where
CVEs live. Room supplies the name: **CVE-2004-1561** — header overwrite (buffer overflow via
an overlong HTTP header). Rank in Metasploit: **great** — the reliable kind, unlike
EternalBlue's "average."

## 3. Foothold (user)

1. **Did:** `search icecast` → `exploit/windows/http/icecast_header` → `show options` →
   RHOSTS + **LHOST set deliberately to tun0** (Metasploit had auto-guessed a wrong
   interface, 192.168.254.213 — caught and overwritten without a nudge; Blue's LHOST lesson,
   retained).

2. **Result:** first try, `Meterpreter session 1 opened`. `getuid` → **Dark-PC\Dark** — a
   *regular user*. Unlike Blue's kernel exploit, the app-level bug lands low: this box has an
   actual privilege-escalation phase.

## 4. Privilege escalation (the new skill)

Windows privesc is not one move, it's a ladder — and each rung has its own tool:

1. **Did:** `sysinfo` (Windows 7 x64 SP1, meterpreter x86), then the Windows equivalent of
   running a suggester script:
   `run post/multi/recon/local_exploit_suggester` → recommends **bypassuac_eventvwr**.

2. **Did:** background session 1 → `use exploit/windows/local/bypassuac_eventvwr` →
   RHOSTS/LHOST/SESSION → `run` → session 2. Output explains the rung: Dark is *in the
   Administrators group* but runs with a **filtered token** (UAC); the exploit abuses
   `eventvwr.exe`'s auto-elevation to relaunch the payload with the full admin token.
   `getprivs` confirms the expansion — **SeTakeOwnershipPrivilege, SeDebugPrivilege**, etc.

3. **Did:** the last rung — admin ≠ SYSTEM. SYSTEM lives in *services*, so the shell
   **moves into one**:
   ```
   migrate -N spoolsv.exe     → Migration completed
   getuid                     → NT AUTHORITY\SYSTEM
   ```
   **Why it works:** the print spooler runs as SYSTEM; meterpreter injects into the process
   and inherits its identity. On Linux you escalate by running a binary differently
   (sudo/SUID); on Windows you escalate by **becoming a process that's already privileged**.

## 5. Loot

1. **Did:** `load kiwi` (mimikatz, built into meterpreter) → `creds_all`:
   - msv: NTLM hash for Dark
   - **wdigest/kerberos: `Dark : Password01!` in plaintext** — Windows keeps cleartext
     passwords in memory for legacy auth; SYSTEM can just read them out.

2. **Did:** connected the dots the room asks about: `Dark:Password01!` + port **3389** from
   recon = a full RDP login. Credentials from one door open the next — the Windows version of
   credential recycling.

## 6. Misses and dead ends

- **`-sn -sV` printed no ports** — `-sn` is a ping sweep, not a port scan. Self-caught in one
  step, no ping spent.
- A heavily guided room — the CVE, the suggester, kiwi, and the migration target were all
  room-supplied. Honestly logged; the understanding checks (why migration works, what UAC
  filtering is, what wdigest exposes) were done independently.
- Typo-level only otherwise (`geuid`, `use` without argument) — self-corrected from the
  error text both times.

## 7. Lessons

1. **Windows privesc ladder:** user shell → `local_exploit_suggester` → **UAC bypass**
   (filtered admin token → full admin token) → **migrate into a SYSTEM service**
   (`spoolsv.exe`). Three rungs, three different mechanisms.
2. **UAC ≠ permissions:** being in Administrators and *running* as administrator are
   different states; the token is the truth. `getprivs` reads it.
3. **Migration = identity theft by process injection** — you don't hack the service, you
   become it.
4. **kiwi/creds_all:** SYSTEM can pull plaintext credentials (wdigest) straight from memory.
   Looted creds point at the next door (RDP here).
5. **App-level exploit ⇒ low-privilege shell; kernel exploit ⇒ SYSTEM** (Ice vs Blue, the
   clean comparison). Vintage niche services (Icecast, 2004) are CVE gold.
