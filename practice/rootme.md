# RootMe — TryHackMe — ROOT

**Date:** 2026-09-08
**Platform:** TryHackMe
**Outcome:** user + root — first privesc executed solo (mentor nudge: "look for the unfamiliar")
**Time:** 1h20 · Confidence self-rated 3/5
**Tools:** nmap, gobuster, php-reverse-shell (pentestmonkey), nc, pty, SUID sweep, python2.7 (GTFOBins SUID)

---

## 0. TL;DR chain

nmap (22 ssh / 80 http) → gobuster → `/panel` (upload form) + `/uploads` → `.php` rejected →
manual extension bruteforce → **`.phtml` accepted** → trigger `/uploads/<shell>.phtml` →
nc 9001 → www-data → pty → sudo -l dead end → SUID sweep diffed against baseline →
**`/usr/bin/python2.7` is SUID** → GTFOBins **SUID** section → `os.execl(..., "-p")` →
euid=0 → root flag

## 1. Recon

| Port | Service | Means | Therefore |
|---|---|---|---|
| 22 | OpenSSH 8.2p1 | login surface | later, if creds appear |
| 80 | Apache 2.4.41 | web app | directory enumeration |

gobuster → `/panel/` (an upload form) and `/uploads/` (where uploads land). The app IS the
attack surface: two directories tell the whole story — a door and a destination.

## 2. Vulnerability assessment

`searchsploit Apache 2.4.41` × 3 → nothing relevant, correctly abandoned. The lesson:
**searchsploit is for named products; the server is just the delivery truck.** The vulnerable
component here is the upload form's extension filter — a misconfiguration, not a CVE. Those
are exploited by hand.

## 3. Foothold (user)

1. **Did:** uploaded pentestmonkey's PHP reverse shell (IP + port edited, listener `nc -lvnp 9001`).
   **Result:** "PHP is not permitted" — the filter works as advertised.

2. **Did:** manual extension bruteforce (12 candidates, typed list — N small, hands beat tools).
   `.php3` uploaded successfully but no connection.
   **The miss:** an uploaded payload is inert until it is **requested** — visiting
   `/uploads/<file>` is the trigger. Upload ≠ execution.

3. **Did:** `.phtml` accepted AND executed when visited → reverse shell on 9001 as www-data.
   **Result:** user shell. (Bonus find in /uploads: `shell.php5` pointing at a foreign IP —
   another player's shell on a shared instance. Noise, logged and ignored.)

## 4. Privilege escalation (root)

1. **Did:** `sudo -l` → demands www-data's password. Dead end, cleanly closed.
   (Needed the pty upgrade first — sudo can't prompt without a terminal.)

2. **Did:** full sweep: SGID (stock), cron (stock), webapp config (only the foreign shell).
   SUID sweep — **first pass failed by scanning for familiar names** ("no nano or vim").
   The fix: **diff against the stock baseline.** Ubuntu's standard SUID set is known furniture
   (sudo, su, mount, ping, chfn…). The line that ISN'T baseline: **`/usr/bin/python2.7`** —
   an interpreter, i.e. a program whose job is running other programs.

3. **Did:** GTFOBins — and the right section this time: the binary came from the **SUID bit**,
   not from sudoers, so the **SUID** section, not Sudo. Payload:
   ```
   python -c 'import os; os.execl("/bin/sh", "sh", "-p")'
   ```
   First attempt without `-p` stayed www-data; `-p` preserves privileges across the exec.

4. **Result:** `id` → `uid=33(www-data) ... euid=0(root)` — the **effective** uid is root, which
   is what file permissions check: /root is readable. Root flag taken.

## 5. Misses and dead ends

- Three searchsploit passes aimed at the web *server* instead of recognizing the *app's upload
  form* as the vulnerability class (no CVE exists for a bad filter — hand-exploit it).
- Uploaded the shell and waited — never triggered it. An uploaded script runs only when
  requested; the response then tells you which failure you have (source echoed back = wrong
  extension; blank = it ran).
- First SUID sweep read every line but through a familiarity filter (nano/vim). The vector was
  the one line that didn't match anything known — recognition is diff-against-baseline.

## 6. Lessons

1. **Upload ≠ execution.** Request the payload; read the response; it names the failure.
2. **Name the vulnerable component before searching:** server vs app vs config. searchsploit
   answers for products, not for misconfigurations.
3. **SUID sweeps are baseline diffs, and interpreters are gold.** SUID-sourced binary →
   GTFOBins SUID section → `-p` style privilege preservation. uid vs euid: permissions follow
   the effective id.
