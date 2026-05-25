# HackMyVM - Todd (Easy) Writeup

This machine focused on enumeration, shell access, and privilege escalation through a vulnerable bash script with misconfigured sudo permissions.

## Overview

- Performed full port scanning and service enumeration using Nmap
- Identified open HTTP service and investigated available content
- Conducted directory fuzzing with Gobuster
- Discovered an exposed service on port 7066 leading to shell access
- Stabilized reverse shell for better interaction
- Retrieved user flag from local system
- Enumerated sudo permissions for privilege escalation opportunities
- Analyzed vulnerable script `/srv/guess_and_check.sh`
- Exploited improper input handling to create a SUID bash binary
- Escalated privileges to root and obtained root flag

## Skills Practiced

- Network Enumeration
- Web Enumeration
- Directory Bruteforcing
- Reverse Shell Handling
- Linux Privilege Escalation
- Sudo Misconfiguration Exploitation
- Bash Exploitation

## Tools Used

- Nmap
- Gobuster
- Netcat
- Bash
- Python3
- Linux utilities

**Machine:** Todd  
**Platform:** HackMyVM  
**Difficulty:** Easy  
**Status:** Rooted ✅
