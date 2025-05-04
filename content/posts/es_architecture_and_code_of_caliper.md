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

Un ejemplo de las comprobaciones realizadas para instanciar el módulo OHT:

``` python

    def parse_vector(self, args, request_item):
        match args.Vector:
            case "JDI":
                self.instance_JDI(args, request_item)
            case "OHT":
                self.instance_OHT(args, request_item)
            case "HVS":
                self.instance_HVS(args, request_item)
            case "RPC":
                self.instance_RPC(args, request_item)

    def instance_OHT(self, args, request_item):
        try:
            OHTHandler(request_item, str(args.segment), int(args.code), bool(args.match_content))
        except Exception as error:
            Utilities.print_error_msg(f"Couldn't instance OHTHandler: {error}")

    '''
    Check if CLI options are valid
    '''
    def parse_options(self, args):
        # VEC MODE CHECKS
        if args.Mode == "VEC":
            # Check Vector is set
            if not args.Vector:
                Utilities.print_error_msg("Vector is required on VEC mode")
            else:
                Utilities.print_success_msg(f"Caliper will instance {args.Vector} module")

            if not args.protocol:
                Utilities.print_error_msg("No HTTP/HTTPS protocol specified")

            if not args.segment:
                Utilities.print_error_msg("Segment is required on VEC mode")
            else:
                Utilities.print_success_msg(f"Set segment: {args.segment}")

            if not args.code:
                Utilities.print_error_msg("Code is required on VEC mode")
            else:
                Utilities.print_success_msg(f"Exploratory check will need to match response code: {args.code}")

            if args.match_content:
                Utilities.print_success_msg(f"Content matching is ON")
            else:
                Utilities.print_success_msg(f"Content matching is OFF")

            if not args.request_file:
                Utilities.print_error_msg("Request File is required on VEC mode")
            else:
                serialized_req = ReqHandler(args.request_file, args.protocol)

            # ALl general checks OK, begin module loading
            self.parse_vector(args, serialized_req)
```

Otro ejemplo de código interesante es la deserialización de peticiones HTTP de texto plano a objeto, realizado por la clase Reqhandler mediante el bloque:

``` python

    def _parse_file(self, file_path):
        """
        Parse the file content and separate it into HTTP verb, URL, headers, and body.
        """

        try:
            with open(file_path, 'r') as file:
                content = file.read()
        except Exception as error:
            Utilities.print_error_msg("File not found.")

        # Regex to match the HTTP verb and URL (first line in the request)
        request_line_match = re.match(r"(POST|GET|PUT|DELETE|PATCH)\s+([^\s]+)\s+HTTP/1.1", content)
        if request_line_match:
            self.http_verb = request_line_match.group(1)

            if self.http_verb == "GET":
                Utilities.print_error_msg("GET requests are not yet supported under VEC mode.")
            else:
                Utilities.print_success_msg(f"Successfully parsed HTTP Verb as: {str(self.http_verb)}")

            self.url = request_line_match.group(2)
            Utilities.print_success_msg(f"Successfully parsed PATH as: {str(self.url)}")

        # Regex to match headers (lines after the request line until an empty line)
        header_lines = re.findall(r"([^\r\n:]+):\s*([^\r\n]+)", content)
        self.headers = {key.strip(): value.strip() for key, value in header_lines}

        # Extract host from headers (use 'Host' header)
        self.host = self.headers.get('Host')
        if not self.host:
            Utilities.print_error_msg("No Host header found in the request.")
        else:
            Utilities.print_success_msg(f"Successfully parsed host as: {str(self.host)}")
        
        # Check if at least one 'Content-Type' header is present
        if 'Content-Type' not in self.headers:
            Utilities.print_error_msg("No Content-type header found.")

        # Construct the full URL (scheme + host + path)
        self.full_url = f"{self.scheme}://{self.host}{self.url}"
        Utilities.print_success_msg(f"Successfully built URL as: {str(self.full_url)}")

        # Body is everything after an empty line
        body_match = re.search(r"\r?\n\r?\n(.*)", content, re.DOTALL)
        if body_match:
            self.body = body_match.group(1).strip()
            Utilities.print_success_msg(f"Successfully parsed body")
```

Finalmente, todos los módulos implementan su propia comprobación de resultados ya que podría necesitar ser diferente en ciertos casos, pero todas siguen un esquema general:

![Diagram showing unified response handling](/images/analysis_flow.png)
