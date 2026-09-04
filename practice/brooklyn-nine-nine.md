# Brooklyn Nine Nine — TryHackMe — ROOT

**Date:** 2026-09-04
**Platform:** TryHackMe
**Outcome:** user + root
**Time:** 1h23
**Tools:** nmap, ftp (anonymous), hydra, ssh, sudo -l, less (GTFOBins escape)

---

## 0. TL;DR chain

nmap (21 ftp / 22 ssh / 80 http) → gobuster empty → **anonymous FTP** → note_to_jake.txt
("password too weak", names jake) → hydra vs SSH → jake/987654321 → `sudo -l` →
NOPASSWD `/usr/bin/less /etc/profile` → pager shell escape `!/bin/sh` → root

## 1. Recon

`nmap -sV`:

| Port | Service | Means | Therefore |
|---|---|---|---|
| 21 | vsftpd 3.0.3 | FTP | **anonymous check — the 30-second door** |
| 22 | OpenSSH 7.6p1 | login surface | brute force once a username exists |
| 80 | Apache 2.4.29 | web | directory enumeration |

`gobuster dir` on port 80 → nothing but `server-status (403)`. Dead surface, correctly abandoned.

`searchsploit vsftpd 3.0.3` → DoS only. Correctly discarded: a DoS is not a way in, and this box
is 25-minute easy — the exploit is a convention, not a CVE.

The FTP login prompt looked like a credentials wall. It wasn't: FTP's universal convention is
username `anonymous`, password `anonymous`. **The cheapest door gets checked first.**

## 2. Foothold (user)

1. **Did:** `ftp IP` → `anonymous` / `anonymous` → `ls` → `get note_to_jake.txt`
   **Why:** anonymous FTP is free loot if it's open; it was.
   **Result:** the note: *"Jake please change your password. It is too weak…"* — a username AND a
   confession that the password is weak. That is a brute-force invitation with the target pre-named.

2. **Did:** `hydra -l jake -P rockyou.txt IP -t 64 ssh`
   **Why:** one named user + "weak password" + rockyou = minutes, not hours.
   **Result:** `jake / 987654321` in 12 seconds.

3. **Did:** `ssh jake@IP` → user shell.

## 3. Privilege escalation (root)

1. **Did:** manual enumeration first. `/etc/crontab` looked interesting but is the **stock Ubuntu
   default** — all four lines are system maintenance. Nothing custom, nothing writable, discarded.

2. **Did:** `sudo -l` — the first move of Linux privesc, the one that asks "what may I run as root?"
   **Result:** jake may run **`/usr/bin/less /etc/profile`** as root with **NOPASSWD**.

3. **Did:** the GTFOBins pager escape:
   ```
   sudo less /etc/profile     # exactly the permitted command
   (inside the pager, press Enter, then type:)
   !/bin/sh
   ```
   **Why it works:** a shell spawned from inside a program inherits that program's privileges.
   `sudo` launched `less` as root, so `!/bin/sh` inside it is a **root** shell. The
   misconfiguration isn't a bug in less — it's letting a user run an interactive program as root.

4. **Result:** `id` → uid=0. Root flag read.

## 4. Misses and dead ends

- Treated the FTP login as "I don't know the credentials" instead of trying the anonymous
  convention. Cost: a full detour through web enumeration and a considered hydra run before FTP
  ever got a fair shot. Lesson: when you see FTP, the anonymous check is 30 seconds — do it first.
- Read the **default** Ubuntu crontab as a privesc candidate. The eye for cron is "what here is
  not standard?" — nothing was.
- Ran the less escape as one command line, without sudo: `less /etc/hosts!/bin/sh` → a jake
  shell. Two errors in one line: the root power comes from `sudo`, and `!/bin/sh` is typed
  *inside* the pager, not appended to the filename.
- Skipped `sudo -l` on the first enumeration pass — it was the answer and it's the first thing
  to run every time.

## 5. Lessons

1. Check the cheapest door first. Anonymous FTP takes 30 seconds; do it before any brute force.
2. `sudo -l` is move one of Linux privesc. It wins more boxes than any exploit.
3. Escaped shells inherit the launcher's privileges: interactive program running as root
   (less, nano, vim, man…) = root shell waiting to happen — GTFOBins is the lookup.
