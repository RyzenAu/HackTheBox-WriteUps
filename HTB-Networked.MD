# HTB: Networked Detailed Exploitation

## Quick Summary

Networked is an Easy difficulty Linux box with vulnerabilities in file upload, leading to code execution, and command injection via a user script, which can be leveraged to execute commands as root.

## Phase 1: Reconnaissance

**Nmap Scan:** The initial scan identified open ports and services.

```plaintext
nmap -sV -sT -sC -o nmapinitial networked.htb

Output:

Port 22: SSH (OpenSSH 7.4)
Port 80: HTTP (Apache 2.4.6 with PHP 5.4.16)
```
Phase 2: Web Enumeration
Gobuster Scan: We used Gobuster to identify directories.

gobuster -u http://networked.htb/ -w /usr/share/wordlists/dirb/common.txt

Interesting Directories:

/uploads
/backup
Backup Directory: Downloaded a tar archive from /backup containing the website's files.

wget http://networked.htb/backup/backup.tar
mkdir backup
cd backup
tar xvf ../backup.tar
