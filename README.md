# CTF Writeups

Detailed, engagement-style writeups of the machines and challenges I've solved — written to document not just *what* worked, but *why* each vulnerability exists and how it maps to real-world risk. Reporting clarity is the point: every writeup moves from reconnaissance through exploitation to remediation, the same shape as a professional penetration-test report.

**Author:** Syaiful Hadad Al-Hafidz — Certified Ethical Hacker (CEH, EC-Council)
**Focus:** Offensive security · web & network exploitation · clear technical reporting

---

## Index

| Machine | Platform | OS | Difficulty | Key techniques |
|---|---|---|---|---|
| [Snapped](machines/htb/Snapped.md) | HackTheBox | Linux | **Hard** | Virtual-host enumeration · **CVE-2026-27944** (Nginx-UI unauthenticated backup disclosure) · bcrypt cracking (Hashcat) · **CVE-2026-3888** (snapd TOCTOU race → root) |
| [Cap](machines/htb/Cap.md) | HackTheBox | Linux | Easy | **IDOR** on sequential capture IDs · pcap credential recovery (tshark/Wireshark) · SSH credential reuse · Linux capability abuse (`cap_setuid` on Python) |
| [Dancing](machines/htb/Dancing.md) | HackTheBox (Starting Point) | Windows | Very Easy | SMB null-session enumeration · anonymous share access (`smbclient`) |
| [Fawn](machines/htb/Fawn.md) | HackTheBox (Starting Point) | Linux | Very Easy | Anonymous FTP misconfiguration · Nmap NSE enumeration |

---

## What these demonstrate

- **Full attack chains, not just flags** — Cap and Snapped each chain an application-layer flaw into a full root compromise, with the reasoning at every pivot.
- **Current CVE work** — Snapped exploits two 2026 CVEs end to end, including a TOCTOU privilege-escalation race, decrypting an AES backup, and cracking the recovered bcrypt hash.
- **Remediation focus** — every writeup closes with a *Lessons Learned* section framing each finding as a defensive fix, the way a client-facing report would.
- **Fundamentals done properly** — the Starting Point boxes (Fawn, Dancing) document why anonymous FTP/SMB are automatic findings on real engagements.

---

## Repository structure

```
ctf-writeups/
├── README.md
└── machines/
    └── htb/
        ├── Fawn.md
        ├── Dancing.md
        ├── Cap.md
        └── Snapped.md
```

## Note on disclosure

Writeups for **retired** or free **Starting Point** machines only. Active-machine content is withheld until retirement, in line with HackTheBox policy.
