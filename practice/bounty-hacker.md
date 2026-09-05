# Bounty Hacker — TryHackMe — ROOT

**Date:** 2026-09-05
**Platform:** TryHackMe
**Outcome:** user + root
**Time:** 1h00 (20 min to user, ~40 min on privesc)
**Tools:** nmap, ftp (anonymous), hydra (targeted wordlist), scp, sudo -l, tar (GTFOBins escape)

---

## 0. TL;DR chain

nmap (21 ftp / 22 ssh / 80 http) → **anonymous FTP** → task.txt (names lin) + locks.txt
(26 candidate passwords) → hydra with the targeted list → lin/RedDr4gonSynd1cat3 → ssh →
`sudo -l` → `(root) /bin/tar` → GTFOBins tar checkpoint escape → root

## 1. Recon

`nmap -sV` (967 filtered ports — THM firewall noise, ignore it):

| Port | Service | Means | Therefore |
|---|---|---|---|
| 21 | vsftpd 3.0.5 | FTP | anonymous check — the 30-second door, first move |
| 22 | OpenSSH 8.2p1 | login surface | brute force once loot names a user |
| 80 | Apache 2.4.41 | web | never needed — FTP paid first |

Anonymous FTP (`anonymous`/`anonymous`) → `ls` → two files, both pulled with `get`:
- `task.txt` — a to-do list signed **-lin** → a username.
- `locks.txt` — 26 candidate passwords → a wordlist built by the target itself.

## 2. Foothold (user)

1. **Did:** `hydra -l lin -P locks.txt IP ssh`
   **Why:** loot supplied both halves — a named user and a 26-line password list. No rockyou
   needed; 26 tries is seconds. **Targeted wordlists from loot beat giant lists every time.**
   **Result:** `lin / RedDr4gonSynd1cat3`.

2. **Did:** `ssh lin@IP` → user shell. 20 minutes from boot.

## 3. Privilege escalation (root)

1. **Did (the slow part):** ran the manual enumeration checklist, then transferred
   linuxprivchecker (`scp` — first attempt to `/home/` failed with Permission denied, corrected
   to `/home/lin`), ran it, and read its "Config files containing keyword 'password'" section.
   **Verdict:** that entire section was grep noise — comments inside stock Ubuntu config files.
   In any privesc tool's output the sections that matter are **sudo, SUID, writable files, cron**.
   Keyword matches across `/etc` are decoration.

2. **Did:** `sudo -l` — which the checklist itself calls "the single highest-value command" —
   only after all of the above. It asked for a password; that's the **caller's own password**
   (hydra's find — one password, two doors).
   **Result:** `User lin may run: (root) /bin/tar` — root-authorized tar.

3. **Did (first attempt, wrong):** grabbed the GTFOBins *file-write* snippet and ran
   `echo DATA >/bin/tar` → `Permission denied`. Two errors: wrong page section (the Sudo
   section is the one when sudo -l named the binary), and the paste was HTML-corrupted
   (`&gt;`) — in bash a bare `&` splits the line, so bash tried to redirect *into* root's file.

4. **Did (correct):** Sudo-section payload:
   ```
   sudo tar cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
   ```
   **Why it works, flag by flag:** `sudo tar` runs tar as root (sudo -l authorizes it);
   `-cf /dev/null /dev/null` is a deliberately pointless archive — a pretext to keep tar
   running; `--checkpoint=1` makes tar's built-in progress feature fire immediately;
   `--checkpoint-action=exec=/bin/sh` replaces the progress message with *execute this
   command*. The shell inherits tar's root privileges. Same inheritance rule as the `less`
   escape — every GTFOBins sudo trick is this rule wearing a different program.

5. **Result:** `id` → uid=0. Root flag read.

## 4. Misses and dead ends

- `sudo -l` sat at position seven of my own checklist while I ran six slower things first —
  then a whole automated script whose output I couldn't triage. The 30 lost minutes live here.
- Treated linuxprivchecker's config-keyword section as leads and `cat`-ed four stock config
  files. Noise-reading: sudo/SUID/writable/cron are the sections; the rest is decoration.
- Picked the wrong GTFOBins section (file write) and misread the goal as "write over
  /bin/tar". The escape never touches the binary — it abuses a feature *inside* it.
- Copied a payload with HTML entities (`&gt;`); bash split the line on `&` and the error
  ("Permission denied" on /bin/tar) described my corrupted paste, not a failed technique.
- Almost gave up on `sudo -l` because it "needed a password I didn't have" — I had it; the
  cracked SSH password *is* the sudo password.

## 5. Lessons

1. Privesc opens with a fixed triple, before any script: `id` → `sudo -l` → SUID find.
   Two minutes, and it has now won three boxes in a row.
2. One password, two doors: every credential found gets tried at every prompt met.
3. Triage tool output by section (sudo/SUID/writable/cron), and read GTFOBins by *how you got
   the binary* — sudo -l named it → Sudo section. Copy from code blocks; a weird error usually
   means a corrupted paste, not a dead technique.
