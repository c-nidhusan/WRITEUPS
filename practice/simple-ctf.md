# Simple CTF (EasyCTF) — TryHackMe — ROOT

**Date:** 2026-09-06
**Platform:** TryHackMe
**Outcome:** user + root
**Time:** ~1h45 (consolidation run — same skeleton as Bounty Hacker, new failure modes)
**Tools:** nmap, gobuster, curl (FTP active mode), hydra, ssh, sudo -l, vim (GTFOBins escape)

---

## 0. TL;DR chain

nmap (21 ftp / 80 http / **2222 ssh**) → gobuster → /simple = CMS Made Simple 2.2.8
(CVE-2019-9053, SQLi) → anonymous FTP: passive data channel blocked → **active-mode curl** →
pub/ForMitch.txt ("same pass for the system user", names mitch) → hydra on **port 2222** →
mitch/secret → ssh -p 2222 → `sudo -l` → `(root) NOPASSWD: /usr/bin/vim` →
`sudo vim -c ':!/bin/sh'` → root

## 1. Recon

`nmap -sV` (997 filtered ports — the firewall opens only three):

| Port | Service | Means | Therefore |
|---|---|---|---|
| 21 | vsftpd 3.0.3 | FTP | anonymous check |
| 80 | Apache 2.4.18, default page | decoy front page | directory enumeration |
| **2222** | OpenSSH 7.2p2 | **SSH on a non-standard port** | every SSH tool needs `-p 2222` / `ssh://IP:2222` |

A non-standard SSH port is easy to default back to 22 out of habit — re-check it before every
SSH-related tool. Room answers that fall straight out of this table: services under port 1000 =
**2** (21, 80); the "higher port" = **SSH on 2222**.

## 2. Foothold (user)

1. **Did:** anonymous FTP login — worked, but `ls` hung at `229 Entering Extended Passive Mode`.
   **Diagnosis:** curl `-v` showed the control channel fine and the **passive data port filtered**
   (timeout on 49601) — the same firewall that filtered 997 ports.
   **Fix:** active mode, `curl -s -P - ftp://anonymous:anonymous@IP/` — the server connects back
   from port 20, and the listing arrives: `pub/ForMitch.txt`.
   **Lesson:** FTP has *two* channels. When listing hangs after a successful login, it's the data
   channel, and passive/active is the toggle.

2. **Did:** read the loot — *"You set the same pass for the system user, and the password is so
   weak…"* → username **mitch**, target **SSH**, expectation **weak password**. The note says
   where the door is; believe the loot before inventing new attack surfaces.

3. **Did:** `hydra -l mitch -P rockyou.txt IP ssh` → **timeout on port 22**. Misread as "wrong
   credentials, must be the web login" — it wasn't: a *connection* error means wrong port or dead
   host, never wrong password. The scan had said 2222.
   **Fix:** `hydra -l mitch -P rockyou.txt ssh://IP:2222 -t 16` → `mitch / secret` in 9 seconds.

4. **Did:** `ssh mitch@IP -p 2222` (after `ssh mitch@IP:2222` failed — ssh takes the port as
   `-p`, which its own usage output shows). User flag in mitch's home.

5. **CVE question:** the app at /simple is **CMS Made Simple 2.2.8**; product + version searched
   → **CVE-2019-9053** (SQL injection, GHSA-rrqg-2h39-2567). Identify first, search second.

## 3. Privilege escalation (root)

1. **Did:** `cat /etc/passwd | grep /home` → users sunbath, mitch. Then the opening move:

2. **Did:** `sudo -l` → `(root) NOPASSWD: /usr/bin/vim`.

3. **Did (misses first):** reflexively tried the *tar* checkpoint payload — caught myself: wrong
   binary, sudo authorized vim only. Then pasted GTFOBins' "Python" variant as
   `vim python -c 'import os; os.execl(...)'` — "Python" is the *label of a variant*, not an
   argument; vim opened a file named "python" → `E492: Not an editor command`. And no `sudo` in
   the line — the third box running that mistake.

4. **Did (correct):** Sudo-section payload:
   ```
   sudo vim -c ':!/bin/sh'
   ```
   **Why it works:** `sudo vim` runs vim as root (sudo -l authorizes exactly this); `-c` makes vim
   execute an editor command immediately; `:!` is vim's "run this in an external shell" feature;
   the shell inherits vim's root privileges. **Same inheritance rule as less and tar — third
   costume, same actor.**

5. **Result:** `#` prompt, uid=0, root flag.

## 4. Misses and dead ends

A long day, and these were attention slips rather than missing knowledge — each one was already
answerable from something on screen or in the notes:

- Misread the room question: "services under port 1000" means ports *below* 1000, not port 1000.
- Hydra aimed at port 22 while the scan output said SSH was on 2222. Worth remembering the error
  class: a *timeout* is a connection problem (port/host), never a password problem.
- Tried `ssh user@host:2222` and then `-D/-L/-R`; the right flag is `-p`, printed in ssh's own
  usage output and already in my Linux Commands.md.
- Used an IBM i support page to fix a Kali ftp client (`?Invalid command` × 6) — check what
  platform a document is written for before applying it.
- The FTP notes holding the passive/epsv/curl fixes had been dropped in a vault merge; restored.
- GTFOBins: the section to read is chosen by how you got the binary — sudo -l named it →
  Sudo section → the snippet already starts with `sudo`.

## 5. Lessons

1. **One quoted fact before each tool call.** Port, path, username — copied from the scan, e.g.
   "scan said 2222 → command gets 2222." Cheap insurance against habit defaults.
2. **Error classes:** timeout/refused = wrong port or dead host; "login failed" = wrong
   credentials. Never diagnose across categories.
3. **FTP = control channel + data channel.** Login OK but listing hangs → data channel; passive
   blocked → go active (`curl -P -`). And read what platform a doc is for before using it.
