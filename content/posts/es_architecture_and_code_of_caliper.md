---
title: "[ES] Arquitectura y Código de Caliper Suite"
date: 2025-05-03T00:00:00+00:00
tags: ["hacking", "spanish","cybersec","writeups","literature"]
author: "XoanOuteiro"
showToc: true
TocOpen: false
draft: false
description: "Descripción de la arquitectura y detalles de implementación de Caliper Suite"
---

Caliper Suite es una herramienta desarrollada en Python 3 que utiliza una serie de módulos diseñados para facilitar pruebas relacionadas con Web Application Firewalls (WAF).

Está centrada en un diseño modular y extensible, con un flujo de ejecución unificado para todos los casos de uso. La única diferencia entre ejecuciones radica en el modo operativo seleccionado y el módulo específico que se instancia.

A continuación podemos ver varios diagramas que ejemplifican el funcionamiento y diseño del programa:

Un diagrama de componentes:

![Component diagram](/images/component.png)

Un diagrama de flujo:

![Flow diagram](/images/flowchart.png)

Y finalmente un diagrama de secuencia:

![Sequence diagram](/images/sequence.png)

Como bien podemos ver la parte inicial de la ejecución del programa es identica para todos los casos de uso.

![Unified processing diagram](/images/unified_processing.png)

Esto se debe al funcionamiento extremadamente sencillo de la función main en caliper.py, que se limita a instanciar la clase encargada del parseo de argumentos:

``` python
def run():
    args = Argparser()

if __name__ == "__main__":

    Utilities.print_separator("=", 60)
    Utilities.print_logo()
    print(Utilities.get_random_quote())
    Utilities.print_separator("=", 60)

    run()
```

La clase Argparser define tres objetos argument parsers:

- Un parser principal genérico que valida el uso de argumentos posicionales obligatorios (VEC o EVAL) para indicar el modo de operación.
  
- Un parser específico para el modo VEC (Evasion Vector Mode).
  
- Un parser específico para el modo EVAL (Evaluation Mode).

Cada parser adicional gestiona los argumentos propios de su respectivo modo, permitiendo modularidad y claridad en la interfaz de línea de comandos.
La clase Argparser también cuenta con baterías de comprobaciones que se realizan previamente a la instanciación de cualquier módulo para asegurar, en la medida de lo posible, que podrán hacer su función cuando existan

Podemos ver también las opciones de cada uno de los subparsers:

El modo VEC cargara su módulo a partir del segundo argumento posicional:

![VEC Operating modes](/images/VEC_operating_mode.png)

Mientras que el modo EVAL cargará un diccionario u otro a partir del argumento ```--syntax-type```:

![EVAL Operating modes](/images/EVAL_operating_mode.png)
