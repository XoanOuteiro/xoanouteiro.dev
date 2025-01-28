---
title: "[ES] Reconocimiento Web en sitios reales"
date: 2025-01-27T00:00:00+00:00
tags: ["hacking","spanish","cybersec"]
author: "XoanOuteiro"
showToc: true
TocOpen: false
draft: false
description: "Tutorial sobre enumeracion de los protocolo HTTP y HTTPS en el mundo real"
---

Buena parte del reconocimiento web en sistemas reales (no en CTFs) se basa en descubrir URLs y subdominios usando enumeración pasiva. Aunque los fuzzers son útiles en ciertos casos para ampliar la superficie de ataque, el modelo principal para encontrar endpoints web de forma activa en entornos reales sigue siendo el crawling o spidering.

En cuanto a herramientas, es recomendable emplear aquellas que tomen fuentes distintas, las que se mencionarán aquí son:

- Para enumeración de subdominios:
	- Subfinder
	- Assetfinder
	- Httpx
- Para descubrimiento de endpoints:
	- Waybackurls
	- Waymore
	- Hakrawler
	- Katana
- Para el parseo de información
	- Gf
	- Qsreplace

## Enumeración de subdominios
 
Es simple, una buena práctica es mantener siempre los ficheros generados por las herramientas.

Nuestro flujo de trabajo se basará en crear dos listados, uno generado por subfinder, otro por assetfinder, mezclarlos, eliminar duplicados y después quedarnos con los subdominios que httpx detecte como activos:

1- Se realiza el escaneo de subfinder, guardando sus resultados. Si se van a realizar más escaneos en un futuro (lo cual es recomendable periodicamente) entonces es una buena idea guardar los escaneos con la fecha y la hora de inicio:

``` bash
subfinder -d [dominio] -o subfinder_scan_[fecha_hora].txt
```

2- Se realiza el mismo proceso, pero con assetfinder

``` bash
assetfinder -d [dominio] -o assetfinder_scan_[fecha_hora].txt
```

3- Se mezclan ambos ficheros en uno que tendrá la salida de ambos escaneos, este fichero se puede eliminar tras el paso 4 ya que no es particularmente útil

``` bash
cat subfinder_scan_YYYYMMDD_HHMM.txt assetfinder_scan_YYYYMMDD_HHMM.txt > all_subs_duped.txt 
```

4- Se deduplican las entradas del archivo, quedando así con solo una copia de cada subdominio, como se mencionó antes, ahora podríamos eliminar el archivo all_subs_duped.txt

``` bash
sort all_subs_YYYYMMDD_HHMM.txt | uniq > all_subs_unduped_[fecha_hora].txt
```

5- Se pasa la lista de subdominios deduplicados por httpx, herramienta que devolverá solo aquellos que respondan a una petición HTTP con éxito, lo cual quiere decir que están activos y son alcanzables en ese instante

``` bash
httpx -l all_subs_unduped_YYYYMMDD_HHMM.txt -o subs_httpx_output_[fecha_hora].txt
```

Es importante tener en cuenta que el paso 5 no es de carácter pasivo, ya que httpx realiza peticiones directamente a los sistemas.

Este proceso se puede abstraer mediante un script de bash.

## Descubrimiento de endpoints

En este campo, las metodologías activas se emplean principalmente para identificar URLs que se sabe con certeza estaban activas en un momento específico. Sin embargo, este tipo de exploraciones tienden a ser más lentas y estan frecuentemente condicionadas por las implicaciones éticas derivadas de las normativas del contrato bajo el cual se están realizando las pruebas. En otras palabras, herramientas como los crawlers y arañas (por ejemplo, Katana y Hakrawler) deben adherirse a las políticas de cabeceras HTTP y a los límites de solicitudes por segundo establecidos por el cliente.

Por esta razón, también emplearemos herramientas de recolección de datos a partir de fuentes abiertas, como Waybackurls. Otras herramientas relevantes incluyen Waymore y Gau, aunque no se profundizará en su funcionamiento en este contexto.

1- Comenzaremos por waybackurls, se trata de una herramienta creada por @tomnomnom, un contribuidor de projectdiscovery en GitHub. Emplea la fuente WayBackMachine del InternetArchive para retornar las URLs que existieron de un sistema web en el pasado, esto también implica que dichas URLs podrían no llevar a recursos existentes en el presente. Su uso es sencillo:

``` bash
cat subs_httpx_output_[fecha_hora].txt | waybackurls > waybackurls_output.txt
```

La ejecución promedio de waybackurls en un sitio grande con una lista de varios cientos de subdominios puede tardar alrededor de media hora.

2- En este paso, podríamos utilizar GAU (GetAllUrls) y WayMore, ambas funcionan como waybackurls pero empleando distintas fuentes. Su ejecución es trivial.

3- En el caso de crawling y spidering, la herramienta más destacada es Katana. Es importante tener en cuenta que Katana es una herramienta extremadamente poderosa y rápida, capaz de alcanzar velocidades de hasta 150 peticiones por segundo, lo que puede sobrecargar un sistema web. Por esta razón, es crucial establecer un límite en las peticiones por segundo para evitar problemas. Un escaneo con Katana puede durar desde varias horas hasta semanas, dependiendo de la complejidad del objetivo. Un escaneo estándar suele tardar al menos 3 horas y genera entre 7,000 y 20,000 URLs visitables.

La ejecución de katana no es trivial, la herramienta cuenta con decenas de opciones diferentes, me limitare a explicar el escaneo que tiendo a ejecutar yo regularmente:

``` bash
katana -u subs_httpx_output.txt -duc -silent -nc -jc -kf -fx -xhr -ef woff,css,png,svg,jpg,woff2,jpeg,gif --rate-limit 10 | tee -a katana_[fecha_hora].txt 
```

- duc : desactiva la comprobación de actualizaciones de la herramienta katana
- silent: muestra por pantalla únicamente la salida de la herramiente
- nc: evita que katana utilice colores en la salida
- jc: Parseo de endpoints, permite que katana analice archivos .js
- kf: Indica a katana que analice aquellos archivos conocidos, como el sitemap.xml o el rebots.txt
- fx: Indica a katana que extraiga los formularios 
- xhr: Indica a katana que extraiga las URLS y métodos HTTP de las solicitudes XHR realizadas por la página
- ef: Permite filtrar URLs con ciertas extensiones
- rate-limit: Indica a katana que realice solo una cantidad de peticiones por segundo

Si se busca otra herramienta menos potente pero regularmente más sencilla y rápida, una muy buena opción es Hakrawler.

## Parseo de información

Este paso es muy subjetivo y en algunos casos puede considerarse no necesario, no obstante poder separar las URLs obtenidas en categorías y poder modificarlas para prepararlas para fuzzing es muy útil. Para esto se emplean dos herramientas de Tomnomnom: Qsreplace y Gf

Qsreplace funciona de una forma similar al comando tr, lo cual nos permite preparar URLs para fuzzing y asi realizar pruebas como LFI, XSS etc.

Gf es esencialmente un comando grep con funciones de regex bajo palabras clave, lo cual es extremadamente util para por ejemplo extraer todas las URLs que podrian ser potencialmente vulnerables a XSS, Open Redirect, SQLi...
