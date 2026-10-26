# Internal — TryHackMe — ROOT

**Date:** 2026-10-12 → 2026-10-26 (six sessions across two school-heavy weeks)
**Platform:** TryHackMe
**Outcome:** root — **machine 19, Phase C room 4 of 5**
**Time:** ~3h15 total · Confidence arc 4 → 3 → 3.5 → 4 → 3 → 3
**Approach:** room-guided recon; walkthrough for vhost fix, wpscan discovery, and the
container port-forward; AI for the Jenkins callback fix (all disclosed)
**Tools:** nmap, gobuster, wpscan, hydra (xmlrpc), WordPress theme editor, nc, ssh -L
(port forward), linpeas, scp, Jenkins script console

---

## 0. TL;DR chain

nmap (22 ssh · 80 Apache default page) → gobuster: `/blog` `/wordpress` `/phpmyadmin`
`/javascript` → phpMyAdmin 4.6.6 = locked door (authenticated exploit, no creds) →
**wpscan** `/blog` → WP 5.4.2 + xmlrpc + user `admin` → `-U admin -P rockyou.txt` →
**admin:my2boys** → login redirects to `internal.thm` → **vhost: /etc/hosts mapping** →
media upload blocks .php/.php3/.phtml → **theme editor** → PHP reverse shell → www-data →
`wp-save.txt` → **aubreanna:bubb13guM!@#123** → ssh → PE dead ends → `jenkins.txt` →
**`ssh -L 8080:172.17.0.2:8080`** → Jenkins container → hydra → **admin:spongebob** →
Jenkins console reverse shell → `/opt/note.txt` → **root:tr0ub13guM!@#123** → ssh → root

## 1. Recon

| Port | Service | Means | Therefore |
|---|---|---|---|
| 22 | OpenSSH 7.6p1 | login | takes the creds the web layer leaks |
| 80 | Apache 2.4.29, default page | four hidden dirs | the whole box lives under /blog and /phpmyadmin |

Dead end worth keeping: **phpMyAdmin 4.6.6** — the only nearby exploit is *authenticated*,
and the metasploit module tried with no creds (root/empty) answered `not-vulnerable`
(it targets 4.8.x anyway). **An exploit title is a contract; "Authenticated" means the
creds come first, from somewhere else.** The somewhere else was WordPress.

## 2. Vulnerability assessment (WordPress)

```
wpscan --url http://<ip>/blog -e u                      # WP 5.4.2, xmlrpc on, user: admin
wpscan --url http://<ip>/blog -U admin -P rockyou.txt   # xmlrpc password attack
→ admin:my2boys
```

Two mechanics banked: **xmlrpc gives wpscan a fast brute-force channel**, and the login
redirect to `internal.thm` is not a room bug — WordPress stores its site URL; the fix is
a **virtual-host mapping in /etc/hosts**. When a web app speaks in hostnames, resolve the
hostname to the target before calling anything broken.

## 3. Foothold (theme editor → www-data → user)

- Media library rejected `.php`/`.php3`/`.phtml` — intended hardening. **The pivot to the
  theme editor was self-derived**: existing theme files are already `.php`, so editing one
  into a reverse shell needs no upload filter to pass. Load the page that renders the
  edited template → nc 4444 → **www-data**.
- Looting the webroot: `wp-save.txt` → **aubreanna:bubb13guM!@#123** → ssh → user flag.
- Creds also tried on ssh immediately on the www-data side — every key tries every door.

## 4. Privilege escalation (the long way round — and the real lesson)

Dead ends, in order: `sudo -l` (needs a password nobody had) → linuxprivchecker (broken
on py3) → linpeas → kernel CVE scripts: 47166 died on `^M` (**CRLF line endings —
dos2unix**), a 2026 splice CVE simply missed. **Kernel exploits are the least reliable
vector on a curated lab box; the room planted its path elsewhere.**

The planted thread was a note: `jenkins.txt` — cross-referenced against the Alfred writeup
unprompted. Jenkins was running **inside a docker container on 172.17.0.2**, reachable
only from the host — so the user shell became a pivot:

```
ssh -L 8080:172.17.0.2:8080 aubreanna@<ip>    # host → container, through the user's ssh
→ http://localhost:8080 = Jenkins login
→ hydra (http-post-form) → admin:spongebob
→ Jenkins script console → Groovy reverse shell (Alfred's console, new costume)
```

The callback failed at first because the payload had to be adjusted for the container
context — diagnosed by **reading what the shell answered to the first command**, then
fixed. Inside the container: `/opt/note.txt` → **root:tr0ub13guM!@#123** → ssh → root.

**Note for the record (mentor miss):** the hypothesis offered mid-run — docker group →
container escape — was **disproved by the user's own `id`** (groups: adm, cdrom, dip,
plugdev; no docker). The check was right, the hypothesis wrong, and the disproof was
one command. That is how hypotheses are supposed to die.

## 5. Misses and dead ends

- **"This is a room mistake"** — the internal.thm redirect was the app's configuration
  talking; decode the environment before outsourcing it. The hostname had already
  appeared in the user's own wpscan command days earlier.
- **wpscan met via walkthrough** despite being named in the previous debrief — the record
  keeps holding answers; retrieval is the weak link, not knowledge.
- **Metasploit on phpMyAdmin with no creds** — the contract lesson re-broken on live fire,
  settled by the module's own `not-vulnerable`.
- **Kernel-CVE shotgun** — two scripts, zero hypotheses written first; CRLF and version
  mismatch killed both.
- **Port-forward concept came from the walkthrough** — the second tunnel in two rooms
  (Game Zone's `ssh -L` was the first), and Wreath is made of these. Next time the tool
  is the user's before the walkthrough opens.

## 6. Lessons

1. **Chains beat doors:** web → CMS creds → user → internal service → container → host.
   Every stage's loot was the next stage's key — journal the hand-offs and the chain
   assembles itself.
2. **WordPress shell without uploads:** the theme editor edits *existing* PHP — upload
   filters are irrelevant. Block on the media page ≠ block on the CMS.
3. **`ssh -L local:internal-host:port user@box`** turns any ssh foothold into a tunnel to
   services only the box can see (docker bridges, internal ports). The Game Zone lesson,
   re-fired — this is the Wreath primitive.
4. **A hostname redirect is information** — vhost config → /etc/hosts → continue.
5. **Hypotheses die by evidence, not argument:** one `id` buried a confident theory. Run
   the check before believing anyone — mentors included.
6. **Kernel CVEs last:** on curated boxes the planted vector is a config note
   (jenkins.txt), not a kernel bug.
