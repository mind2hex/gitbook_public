# WifineticTwo

## 1. Machine General Information

* **Operative System:** Linux
* **Difficulty**: Medium
* **Keywords**:

## 2. Enumeration

### 2.1 Nmap Port Scanning

```
# Nmap 7.94SVN scan initiated Sat Mar 30 00:25:02 2024 as: nmap -sC -Pn -T5 -oN nmap/default-script-scan.nmap 10.10.11.7
Nmap scan report for 10.10.11.7
Host is up (0.35s latency).
Not shown: 998 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
| ssh-hostkey: 
|   3072 48:ad:d5:b8:3a:9f:bc:be:f7:e8:20:1e:f6:bf:de:ae (RSA)
|   256 b7:89:6c:0b:20:ed:49:b2:c1:86:7c:29:92:74:1c:1f (ECDSA)
|_  256 18:cd:9d:08:a6:21:a8:b8:b6:f7:9f:8d:40:51:54:fb (ED25519)
8080/tcp open  http-proxy
| http-title: Site doesn't have a title (text/html; charset=utf-8).
|_Requested resource was http://10.10.11.7:8080/login

# Nmap done at Sat Mar 30 00:25:07 2024 -- 1 IP address (1 host up) scanned in 5.47 seconds
```

The nmap scan shows only two ports open:

* `22/tcp open ssh OpenSSH 8.2p1`
* `8080/tcp open http-proxy Werkzeug/1.0.1 Python/2.7.18`

### 2.2 Web Server Enumeration

Accessing to the web server it show us a login page to what appears to be an OpenPLC web server.

<figure><img src="../../../../../.gitbook/assets/imagen (22).png" alt=""><figcaption></figcaption></figure>

OpenPLC is an open-source Programmable Logic Controller that is primarily designed to be compatible with industrial, home automation, and educational purposes. The OpenPLC web server provides a graphical user interface (GUI) that can be accessed via a web browser. This GUI allows users to configure and manage the OpenPLC device.&#x20;

We can log in using default credentials:

* `Username: openplc`
* `Password: openplc`



## 3. Exploitation

## 4. Privilege Escalation

## 5. References

