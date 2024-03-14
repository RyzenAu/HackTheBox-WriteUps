
# HTB - Perfection 


### TL;DR
This is an Ubuntu 22.04 system hosting a website that is susceptible to Server-Side Template Injection (SSTI), a vulnerability that has been exploited to gain shell access to the system. In terms of privilege escalation, a hashed password found within the file system has been decrypted, providing the credentials needed for a user with sudo privileges.

### References 



### ENUMERATION 
Port Scan/ Eunenmerate open ports 
```markdown

> nmap -sC -sV -oA nmap $target    


Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-03-14 16:12 AEDT
Nmap scan report for 10.10.11.253
Host is up (0.0092s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 80:e4:79:e8:59:28:df:95:2d:ad:57:4a:46:04:ea:70 (ECDSA)
|_  256 e9:ea:0c:1d:86:13:ed:95:a9:d0:0b:c8:22:e4:cf:e9 (ED25519)
80/tcp open  http    nginx
|_http-title: Weighted Grade Calculator
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 13.09 seconds

```
Found port 80 and port 22 open

# 10.10.11.253 
added to /etc/hosts


![perfection](image.png)

Enter any input but need to make sure the weights 
