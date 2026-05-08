# Sorcery

**Platform:** HackTheBox | **Category:** Web | **Difficulty:** Insane

## Overview
(Pulled from the "Machine Info" section on HTB.) Sorcery is a Linux machine that starts with an HTTPS web application. The web application is open source and gives an attacker full access to the application's code, including the authentication flow, passkey enrollment, and internal debug functionality. After registering, source code review reveals a custom Rust Model derive macro that builds Neo4j Cypher queries using string formatting. However, it does not use proper validation, leading to Cypher injection that allows an attacker to register a seller account. With a seller account, the attacker can create a product that is automatically visited by an administrator. The product description is rendered with dangerouslySetInnerHTML, resulting in stored XSS. By abusing this attack vector, the attacker can register a passkey and log in as the admin user. As an administrator, additional features become available. Some blog posts reveal that the user tom_summers is susceptible to phishing attacks. The attacker needs to craft a highly intricate phishing chain to trick tom_summers into logging in so as to capture his credentials. To achieve this, the attacker exploits an SSRF primitive through the debug page to leak CA certificates from an ftp server, register a .sorcery.htb subdomain through Kafka, and send the final email. Privilege escalation begins by discovering an Xvfb display and a mousepad process running as tom_summers_admin. By capturing an image of that buffer, the attacker exfiltrates the password for tom_summers_admin. From there, Docker is configured to use a custom credential helper, and sudo -l permits running docker login and strace as rebecca_smith. By leveraging this combination, the attacker finds the password of the user rebecca_smith however, she is configured to use One Time Passwords (OTPs) as an added layer of security. By reversing the binary and extracting the OTP generation logic, the attacker is able to dump the Docker registry and find credentials for the user donna_adams. This user is a member on the main FreeIPA realm. Finally, the attacker can change the password of ash_winter over LDAP and grant the user extremely permissive sudo rules, thus allowing escalation to root.

Generally speaking, this is an extremely intricate macchine that truly deserves its "Insane" rating. I wasn't sure about tackling this one but difficult machines often provide the most quantity of practical experience. I wanted to truly understand what it is that sets apart merely good hacking skills and what the industry needed. This was precisely it, with a high balance of real-life application and complex enumeration, I'll be able to prove myself more easily this way.
<img width="299" height="153" alt="image" src="https://github.com/user-attachments/assets/a2c44e38-0968-44dc-95e0-657312367f62" />
<img width="499" height="778" alt="image" src="https://github.com/user-attachments/assets/ebf06f8d-379a-4b4d-ac36-04c4c2980834" />

## Reconnaissance
Nmap reveals 

## Vulnerability
What the flaw is and why it exists.

## Exploitation
Step-by-step with commands and code blocks:

```bash
nmap -sV -sC 10.10.10.40
```

## Flags / Proof
`HTB{your_flag_here}` or a screenshot of root.txt

## Lessons Learned
What you'd take into a real engagement.
