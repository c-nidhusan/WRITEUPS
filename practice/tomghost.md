# tomghost — TryHackMe — ROOT

**Date:** 2026-09-09
**Platform:** TryHackMe
**Outcome:** user + lateral + root — **Phase A complete (machine 9)**
**Time:** 1h45 · Confidence self-rated 2.5/5
**Tools:** nmap, gobuster, CVE-2020-1938 (Ghostcat), scp, gpg, gpg2john, john, sudo -l, zip (GTFOBins)

---

## 0. TL;DR chain

nmap (22 ssh / 53 tcpwrapped / **8009 ajp13** / 8080 Tomcat 9.0.30) → research: AJP +
**CVE-2020-1938 "Ghostcat"** (found independently) → exploit = file read → Python 2→3 fixes →
read `/WEB-INF/web.xml` → **skyfuck:8730281lkjlkjdqlksalks** → ssh → home: `credential.pgp` +
`tryhackme.asc` → scp key to Kali → `gpg2john` → john+rockyou → **alexandru** → decrypt →
**merlin:asuyus...** → ssh merlin → `sudo -l` → `(root) NOPASSWD: /usr/bin/zip` → GTFOBins
Sudo escape → root

## 1. Recon

| Port | Service | Means | Therefore |
|---|---|---|---|
| 22 | OpenSSH 7.2p2 | login surface | the door every credential walks through |
| 53 | tcpwrapped | handshake accepted, service hung up without talking | noted, not a path |
| **8009** | **ajp13** | Apache↔Tomcat connector protocol | **Ghostcat — CVE-2020-1938** |
| 8080 | Tomcat 9.0.30 | app server | manager page, directory enum |

The 8009 port was walked past on machine one (Basic Pentesting). Not this time: researched the
protocol, found the CVE **without AI and without a nudge** — recon → protocol → CVE, full
professional sequence, solo.

## 2. Vulnerability assessment

gobuster on 8080 → `/manager/` → the 401 page leaked the `context.xml` reference and the
tomcat/s3cret pair. Tried on SSH — failed, logged, moved on (credential recycling: right
reflex, wrong door this time).

The real vulnerability: **CVE-2020-1938 (Ghostcat)** — the AJP connector will read arbitrary
files inside the webapp for anyone who asks politely. A **file-read primitive**, not RCE.

## 3. Foothold (user)

1. **Did:** ran the public Ghostcat exploit; converted it Python 2→3:
   - `socket.makefile(bufsize=0)` → `buffering=0` — fix supplied by the mentor (via AI when the
     mentor was briefly unreachable; announced in the journal). Retrospect: solvable alone.
   - `"".join(bytes)` → `b"".join(...).decode("utf-8", errors="replace")` (py3 bytes vs str) —
     fix supplied by the mentor.
   Status: the py2→3 family is now *seen* twice but not yet retained solo — next vintage
     exploit, the first traceback gets five solo minutes before any ping.

2. **Did:** `-f WEB-INF/web.xml` — on a Tomcat box, that file is *the* prize. Output contained:
   `skyfuck:8730281lkjlkjdqlksalks` sitting in the app's own description tag.
   **The miss:** first instinct was "now upload a reverse shell" — while holding SSH credentials
   in the output. ForMitch.txt, again: **the door you're holding the key to beats the wall
   you're planning to climb.** Self-caught after the nudge.

3. **Did:** `ssh skyfuck@IP` → user shell.

## 4. Lateral movement (the new skill)

Home directory: `credential.pgp` (encrypted blob) + `tryhackme.asc` (a **GPG private key**).

1. **Did:** `gpg --import tryhackme.asc` on the target; `gpg --decrypt credential.pgp` →
   demands a **passphrase**. Tried known passwords (skyfuck's, s3cret) — guessing scales
   terribly.

2. **Did:** the Basic Pentesting workflow, second costume — **encrypted artifact = hash
   wearing a hat**:
   ```
   scp skyfuck@IP:/home/skyfuck/tryhackme.asc ~/HACKING/
   gpg2john tryhackme.asc > gpg.hash
   john gpg.hash -w=rockyou.txt        → alexandru
   ```

3. **Did:** decrypt where the keyring lives (import on Kali, or back on target) →
   `merlin:asuyusdoiuqoilkda312j31k2j123j1g23g12k3g12kj3gk12jg3k12j3kj123j` → ssh merlin.
   Note: "No secret key" ≠ "bad passphrase" — the key lives in a keyring, decryption happens
   wherever that keyring is.

**The `*2john` family, one entry forever:** ssh2john, gpg2john, zip2john… extract the hash on
the target's artifact, crack offline at home, walk back through the door.

## 5. Privilege escalation (root)

1. **Did:** `sudo -l` immediately → `(root : root) NOPASSWD: /usr/bin/zip`. The opening triple,
   firing on schedule for the third box running.

2. **Did:** GTFOBins — **Sudo** section (binary came from sudoers, so Sudo it is):
   ```
   sudo zip x.zip /etc/hosts -T -TT '/bin/sh #'
   ```
   **Why it works:** zip has a legitimate "test the archive" feature (`-T` runs `-TT`'s command
   on it). The archive itself is a pretext — the point is that `-TT` executes *your* string,
   and sudo launched zip as root. Same actor, fifth costume.
   (First attempt zipped `/usr/bin/zip` as the archive name — harmless, it still fired; the
   archive name is a dummy.)

3. **Result:** `id` → uid=0. Root flag.

## 6. Misses and dead ends

- **Read past the loot:** web.xml printed credentials and the plan went to "upload a shell."
  The recurring attention tax — output first, imagination second.
- **Slow to name the tool:** trying the two known passwords at the prompt was fair triage —
  the gap was not recognizing the encrypted artifact as a `*2john`-family target right away.
  Rule: a few manual tries are legal; a locked artifact is john's job from minute one.
- One brief AI touch on the py2 error while the mentor was unreachable — announced honestly in
  the journal; syntax category, same shelf as `--help`.

## 7. Lessons

1. **8009/ajp13 = Ghostcat**: file-read CVE; on Tomcat, `/WEB-INF/web.xml` is the first file to
   read. Protocol → CVE → the file that matters.
2. **The `*2john` workflow:** artifact home with you, hash out, crack offline, return through
   the door. Guessing is for prompts; rockyou is for hashes.
3. **Vintage exploits: py2→3 pass first** (print, raw_input, urllib.parse, bufsize→buffering,
   bytes vs str). The traceback always names the line.
