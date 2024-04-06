# Devvortex

## 1. Machine General Information

* **Operative System:** Linux
* **Difficulty**: Easy
* **Keywords**:
  * Hash Cracking
  * RCE

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

### 2.3 DNS Subdomain Enumeration

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
