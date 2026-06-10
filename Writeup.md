# MR Robot CTF - TryHackMe

**Date:** 10 June 2026

## Overview
Completed the **MR Robot CTF** room on TryHackMe, inspired by the TV series *Mr. Robot*. This challenge focused on web enumeration, credential discovery, exploitation, and Linux privilege escalation.

## Skills Practiced
- Web Enumeration
- Directory Bruteforcing
- WordPress Enumeration
- Credential Discovery
- Linux Privilege Escalation
- CTF Methodology
- Post-Exploitation Enumeration

## Tools Used
- Nmap
- Gobuster
- Burp Suite
- Linux Enumeration Commands
- Python (basic scripting)
- GTFOBins

## Key Things I Learned

### Reconnaissance & Enumeration
- Performed service discovery and web enumeration.
- Identified hidden directories and files exposed by the web application.
- Learned the importance of checking `robots.txt` and other publicly accessible resources during reconnaissance.

### Web Application Analysis
- Investigated a WordPress installation.
- Enumerated users and potential attack vectors within the CMS.
- Explored how misconfigurations and exposed information can aid attackers.

### Credential Discovery
- Located credentials through enumeration and file analysis.
- Understood how poor credential management can lead to unauthorized access.

### Initial Access
- Used discovered credentials to gain access to the target.
- Practiced working with Linux systems after obtaining a shell.

### Linux Privilege Escalation
- Performed local enumeration to identify privilege escalation opportunities.
- Investigated SUID binaries and system misconfigurations.
- Leveraged privilege escalation techniques to gain elevated access.

### Post-Exploitation
- Navigated the system to locate important files and flags.
- Reinforced the importance of thorough enumeration after gaining access.

## Flags Captured
- Flag 1 ✅
- Flag 2 ✅
- Root Flag ✅

## Key Takeaways
- Enumeration is often the most critical phase of an assessment.
- Small pieces of exposed information can lead to full compromise.
- WordPress installations should be regularly updated and secured.
- Linux privilege escalation frequently depends on identifying overlooked misconfigurations.
- Methodical enumeration significantly increases the chances of success.

## Conclusion
The MR Robot CTF provided a realistic attack path from initial reconnaissance to full system compromise. It was an excellent room for practicing web enumeration, credential discovery, and Linux privilege escalation while reinforcing the importance of systematic investigation throughout an engagement.
