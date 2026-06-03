# HackTheBox — Fawn

**Platform:** HackTheBox (Starting Point — Tier 0) | **OS:** Linux | **Difficulty:** Very Easy | **Status:** Active (free)  
**CVEs Covered:** None — service misconfiguration (anonymous FTP access)  
**Tools Used:** Nmap, ftp

---

## Summary

Fawn is a Tier 0 Starting Point machine that teaches a single, foundational lesson: the danger of **anonymous FTP access**. The host exposes one service — an FTP server (vsftpd 3.0.3) on port 21 — configured to accept the built-in `anonymous` account with no password. Logging in anonymously grants read access to the FTP root, where the flag sits in plain view as `flag.txt`. There is no exploitation chain, no foothold pivot, and no privilege escalation; the entire box is a demonstration of why anonymous FTP is an immediate finding on any engagement. The takeaway scales well beyond the lab: misconfigured FTP servers still leak source code, backups, and credentials across the real internet.

---

## Attack Chain Overview

```
Nmap → 21/FTP (vsftpd 3.0.3, anonymous login allowed)
→ ftp anonymous : <blank password>
→ ls → flag.txt
→ get flag.txt → cat flag.txt → flag
```

---

## 1. Reconnaissance

### Port Scan

```bash
sudo nmap -p- --min-rate 10000 10.129.31.172
```

```text
PORT   STATE SERVICE
21/tcp open  ftp
```

Service/version detection with default scripts against the single open port:

```bash
sudo nmap -p 21 -sCV 10.129.31.172
```

```text
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--    1 0        0              32 Jun 04  2021 flag.txt
| ftp-syst:
|   STAT:
| FTP server status:
|_End of status
```

**Analysis:** Only one service is exposed, and the `ftp-anon` NSE script does the work for us — it confirms **anonymous login is allowed** (FTP response code `230 Login successful`) and even lists a world-readable `flag.txt` in the FTP root. The attack surface is a single misconfigured FTP server.

<img width="770" height="454" alt="image" src="https://github.com/user-attachments/assets/4123756d-7fad-4255-a707-dfd9113dbbf4" />


---

## 2. Exploitation — Anonymous FTP Access

### Vulnerability Explanation

FTP supports an "anonymous" mode intended for public file distribution, where anyone can authenticate with the username `anonymous` (or `ftp`) and any password — typically blank or an email address. When a server hosting sensitive files leaves this enabled, it becomes an unauthenticated read (and sometimes write) primitive. On Fawn it exposes the flag directly; in the real world the same misconfiguration routinely leaks backups, configuration files, and credentials.

### Logging In and Retrieving the Flag

Connect to the FTP service and authenticate as `anonymous` with a blank password:

```bash
ftp 10.129.31.172
```

```text
Connected to 10.129.31.172.
220 (vsFTPd 3.0.3)
Name (10.129.31.172:kali): anonymous
331 Please specify the password.
Password:
230 Login successful.
```

List the directory and pull down the flag:

```text
ftp> ls
-rw-r--r--    1 0        0              32 Jun 04  2021 flag.txt
ftp> get flag.txt
ftp> exit
```

Read the downloaded file:

```bash
cat flag.txt
```

<img width="528" height="459" alt="image" src="https://github.com/user-attachments/assets/bf9701a8-383a-40f6-8926-87fc2d43646b" />


**Flag captured.**

---

## Lessons Learned

**Anonymous FTP is an automatic finding.** It requires no exploit, no credentials, and no skill to abuse — a single `nmap` script or login attempt confirms it. Any FTP server that doesn't explicitly need public anonymous access should disable it (`anonymous_enable=NO` in vsftpd). When it is required, the anonymous root must contain nothing sensitive and should be read-only.

**Let your scanner do the enumeration.** Nmap's `ftp-anon` NSE script (run automatically with `-sC`/`-sCV`) tests anonymous login and lists the directory in one step, surfacing `flag.txt` before a single manual command. Reading default-script output carefully often hands you the next move for free.

**Cleartext protocols compound the risk.** FTP transmits both credentials and data unencrypted. Even with authentication enabled, anyone on the path can capture logins — the exact weakness exploited on the next tier up (see the Cap machine). Prefer SFTP/FTPS, and treat any plaintext service as a liability worth flagging.

---

## References

- [HackTheBox — Fawn (Starting Point)](https://app.hackthebox.com/starting-point)
- [vsftpd Documentation — anonymous_enable](https://security.appspot.com/vsftpd/vsftpd_conf.html)
- [Nmap NSE — ftp-anon script](https://nmap.org/nsedoc/scripts/ftp-anon.html)
- [Fawn — HackTheBox Write-up (DEV Community)](https://dev.to/shiahalan/fawn-hackthebox-write-up-2bm9)
- [HackTheBox — Fawn Walkthrough (olxxi)](https://olxxi.medium.com/hackthebox-fawn-walkthrough-c4995ee65232)
