---
title: "[ES] Adquisicion de Evidencias para analisis forense"
date: 2025-01-27T00:00:00+00:00
tags: ["forensics", "classnotes","spanish","cybersec"]
author: "XoanOuteiro"
showToc: true
TocOpen: false
draft: false
description: "Apuntes de clase de Analisis Forense Informatico sobre la adquisicion de evidencias"
---

Existen dos tipos principales de adquisicion de la evidencia:

- Estatico o "Dead": Consiste en apagar el ordenador tirando del cable, se recomienda mucho ya que permite una clonacion exacta, pero no se obtienen datos volatiles
- "Live" o en vivo: Se pueden obtener datos volatiles, pero se puede invalidar la evidencia ya que aun se podrian modificar los archivos. Tambien se pueden alterar los metadatos. A las imagenes obtenidas de este modo se les denomina **difuminadas**

Los pasos a seguir independientemente del metodo incluyen:
- Eliminar todos los agentes de cambio posibles
- Recolectar datos con herramientas especializadas
- Registrar la desviacion del reloj del sistema
- Preguntarse que mas podria ser una evidencia
- Documentar todos los pasos
- Tomar notas de quien estaba en el lugar la adquisicion y que estuvo haciendo
- Generar checksums y firmas criptograficas de la evidencia (utilizando herramientas que no modifiquen los metadatos)
- Explicar claramente como se encontro la evidencia, como se manejo y que paso con ella (donde se descubrio, quien la descubrio, quien la manejo, examino, recolecto... Cuanto tiempo la tuvo bajo su custodia, cuanto tiempo estuvo y donde, quien y como hizo la transferencia...)

En cuanto a herramientas a utilizar para la recoleccion:
- Debemos utilizar software en modo solo lectura
- NO Debemos utilizar las herramientas del sistema ya que pueden estar modificadas
- Deberiamos utilizar herramientas enlazadas de forma estatica
- Deberiamos utilizar herramientas ligeras
- Deberiamos utilizar herramientas que de tener que modificar datos, lo hagon lo minimo posible
- Deberiamos estar preparados para testificar sobre la autenticidad y fiabilidad de las herramientas

Existen asimismo unas series de directrices que debemos obedecer:
- Debemos tomar notas detalladas
- Debemos especificar siempre la hora (en formato UTC o hora local)
- Debemos estar preparados para testificar todo lo que hemos realizado
- En caso de dilema entre recoleccion y analisis, primara SIEMPRE la recoleccion
- Debemos comprobar los metodos de recoleccion implementables
- Debemos ser veloces, precisos y metodicos (trabajar en paralelo entre dispositivos)
- Debemos seguir el orden de volatilidad de mas a menos:
	- 1- Registros y cache
	- 2- Tablas de enrutamiento, procesos, estadisticas del kernel
	- 3- Memoria
	- 4- Sistemas de ficheros temporales
	- 5- Disco
	- 6- Logs remotos del sistema
	- 7- Configuracion fisica y topologia de red
	- 8- Copias de seguridad (Aunque esto puede variar(Cloud))

Debemos evitar a toda costa:
- Apagar/Encender los dispositivos innecesariamente
- Confiar en los programas del sistema
- Emplear programas que modifiquen o modifiquen metadatos
- No tener cuidado al eliminar agentes externos de cambio 
- Dar acceso a personas que no lo tendrian normalmente
- Entrometerse en la privacidad de otros salvo necesidad

Los factores que debemos tener en cuenta en cuanto a la recoleccion de evidencias:
- Tipos de Incidente
- Colaboración
	- Dependerá de Quien participa
	- Dependerá de a quien hay que mantener informado
	- Dependerá de las contraseñas(/autorización) necesarias
- Autorización (esencial, y escrita)
- Entrada y registro
	- Se debe hacer con intervención del juez o titular
	- Se debe ser aséptico para evitar tensiónes con el titular
	- Se debe registrar un acta descriptiva al terminar
- Se debe realizar un inventario (etiquetado y fotografia)
	- Marcas
	- Modelos
	- Especificaciones
	- Responsable
	- Dueño
	- Etc.
- Preparación de herramientas (de antemano)
	- Discos duros
	- USBs
	- USBs booteables
	- USBs con herramientas forenes
	- Destornilladores
	- Clonadora forense, bloqueadores de escritura
	- Bases para insertar discos duros
	- Bolsas para evidencias y jaulas de faraday


Finalmente debemos ver como se realizara la clonacion de discos:
- Las copias deben ser siempre bit a bit
- Podemos utilizar:
	- Hardware: Clonadoras y Bloqueadores de escritura
	- Software: dd, dc3dd, dcfldd, GuyMager, FTKImager, OSFClone y RATool

La clonacion por hardware es de tecnica sencilla y rapida, permite multiples copias en paralelo y se suele bloquear la escritura de forma automatica. Eso si: Se requiere acceso fisico al disco y las clonadoras suelen ser extremadamente caras.

Es preferible que el disco destino tenga la misma marca, modelo y capacidad que el original (y que este escrito con ceros).

La clonacion por software permite clonar bit a bit, existen soluciones gratuitas y de pago. Es necesario utilizas un bloqueador de escritura (ya sea de hardware o de software)

Vamos a generar imagenes de disco, tipicamente con formatos:
- Raw (dd,raw,img,ima)
- Smart (s01)
- Encase Evidence File (e01)
- Advanced Forensics Format (aff)
