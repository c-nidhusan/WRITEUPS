# Agent Sudo — TryHackMe — ROOT

**Date:** 2026-09-27 (completed over two 45m timeboxes)
**Platform:** TryHackMe
**Outcome:** root — **machine 14, Phase B 5/6**
**Time:** ~1h45 total (45m partial + 1h completion) · Confidence self-rated 3/5
**Tools:** nmap, curl (-A), hydra, ftp, binwalk, foremost, zip2john/john, base64,
steghide, ssh, sudo -l, CVE-2019-14287

---

## 0. TL;DR chain

nmap (21 vsftpd 3.0.3 · 22 OpenSSH 7.6p1 · 80 Apache, "Annoucement" page) → the page
*is* the puzzle: *"use your codename as user-agent"* → `curl -A R` → hidden page ("25
employees") → `curl -A C` → **"Attention chris... change your password, it's weak"** →
hydra ftp `chris:crystal` → ftp loot: `To_agentJ.txt`, `cute-alien.jpg`, `cutie.png` →
**binwalk** `cutie.png` → embedded **encrypted zip** (`To_agentR.txt`) → binwalk -e failed →
**foremost** carved it → `zip2john` + rockyou → **alien** → `7z x -palien` → base64
`QXJlYTUx` → **Area51** (steg passphrase) → `steghide extract -sf cute-alien.jpg -p Area51`
→ `message.txt` = **james:hackerrules!** → ssh → `sudo -l` → `(ALL, !root) /bin/bash` →
**CVE-2019-14287** → `sudo -u#-1 /bin/bash` → root

## 1. Recon

| Port | Service | Means | Therefore |
|---|---|---|---|
| 21 | vsftpd 3.0.3 | FTP | anonymous refused — needs creds (they come later) |
| 22 | OpenSSH 7.6p1 | login | the door the final creds walk through |
| **80** | Apache 2.4.29 | "Annoucement" page | **the puzzle box: user-agent gate** |

Day-one tax, filed for the record: 30 minutes of firewall-evasion theory (`-f`, `-g 53`)
against an all-filtered scan — the target was a **stale IP** from a dead VPN session.
**When a scan makes no sense, verify the target against the room page first; infrastructure
before evasion theory.**

## 2. Vulnerability assessment (the page was the lock)

"Use your own codename as user-agent to access the site." — the server serves different
content per `User-Agent` header:

```
curl -A R -L <ip>   → "Are you one of the 25 employees?..."   (25 = alphabet of agents)
curl -A C -L <ip>   → "Attention chris ... change your password, it's weak!"
```

One header, two secrets: a **username** (chris) and a **promise** (the password is in
rockyou). Directory enumeration found nothing because the content was never hidden by
*path* — it was hidden by *identity*.

## 3. Foothold (a five-stage scavenger hunt)

1. **Did:** `hydra -l chris -P rockyou.txt <ip> ftp` → `chris:crystal` — Bounty Hacker's
   workflow, retained.
2. **Did:** ftp in → `To_agentJ.txt` ("the real picture hides the password") + two images.
3. **Did:** `binwalk cutie.png` → **encrypted zip appended inside the PNG**
   (`To_agentR.txt`). `binwalk -e` failed to extract ("no utility found") → **`foremost -i`**
   carved the zip out instead. Tool A failing is a reason to try tool B, not to stop.
4. **Did:** the `*2john` family, third deployment: `zip2john 00000067.zip > zip.hash` →
   john + rockyou → **alien** → `7z x -palien` → `To_agentR.txt`: send the picture to
   `QXJlYTUx` → recognized as encoded → `base64 -d` → **Area51**.
5. **Did:** `steghide extract -sf cute-alien.jpg -p Area51` → `message.txt` →
   **james:hackerrules!** → `ssh james@<ip>` → user flag.

The whole box is one lesson: **every artifact is a container until proven otherwise** —
PNG held a zip, zip held base64, base64 held a passphrase, jpg held the final creds.

## 4. Privilege escalation

`sudo -l` immediately on landing — the opening triple, automatic now:

```
User james may run: (ALL, !root) /bin/bash
```

**CVE-2019-14287:** sudo < 1.8.28 mishandles user-ID specification — `-u#-1` is read as
uid 0 *despite* the `!root` restriction:

```
sudo -u#-1 /bin/bash    → root
```

`!root` was a paper fence; the CVE walks through it. Linux privesc homecoming after four
Windows boxes — and the sudo reflex fired without being asked.

## 5. Misses and dead ends

- **Unicode quotes from the walkthrough** (`curl -A “R”`) — curl warned, self-corrected to
  plain `-A R` in one step. The paste rule applies to walkthroughs and videos too: web
  pages mangle payloads, code blocks are the only trustworthy source.
- **`zip2john` recalled from the walkthrough, not from notes** ("checking notes would take
  too long"). Countermeasure: the writeups are the searchable record — `*2john` is in
  basic-pentesting.md and tomghost.md; grep beats re-reading, and beats a walkthrough.
- **Stale-target detour** (day 1, above) — preflight line 1 exists for exactly this.
- Walkthrough + AI used for the user-agent command and CVE lookup — disclosed; the concept
  (identity-based access, the room's own hint) was called the day before from the page text.

## 6. Lessons

1. **Content can hide behind identity, not path** — user-agent gating beats gobuster;
   when dir enum is empty, re-read what the page *says*.
2. **Stego toolchain:** binwalk (see inside) → foremost (carve out when -e fails) →
   zip2john/john (crack) → steghide (extract with passphrase). Every file is a container.
3. **base64 is a disguise, not encryption** — `A-Z a-z 0-9 +/` and `=` padding are the
   fingerprints; `base64 -d` is the whole key.
4. **CVE-2019-14287:** `(ALL, !root)` + old sudo = root via `-u#-1`. Restrictions in
   sudoers are only as strong as the sudo binary enforcing them.
5. **Verify the target before theorizing about the target.** Stale IP ≠ firewall.
