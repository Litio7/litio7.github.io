---
title: LinkVortex &#x1F512;
description: La maquina LinkVortex esta activa. Este artículo se publicará para acceso público una vez que la maquina se retire, según la política de HackTheBox.
date: 2024-12-11
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-linkvortex/linkvortex_logo.png
categories:
  - Hack The Box
  - Machines
tags:
  - linux
  - hack the box
  - tcp
  - ssh
  - http

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
-$ ping -c 1 10.10.11.47
PING 10.10.11.47 (10.10.11.47) 56(84) bytes of data.
64 bytes from 10.10.11.47: icmp_seq=1 ttl=63 time=243 ms

--- 10.10.11.47 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 243.448/243.448/243.448/0.000 ms
```
Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
-$ nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn -oG nmap1 10.10.11.47
Host: 10.10.11.47 ()	Status: Up
Host: 10.10.11.47 ()	Ports: 22/open/tcp//ssh///, 80/open/tcp//http///	Ignored State: closed (65533)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/htb/machines/titanic:-$ sudo nmap -sCV -p22,80 -vvv 10.10.11.55 -oN nmap2
-$ nmap -sCV -p22,80 -vvv -oN nmap2 10.10.11.47
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 3e:f8:b9:68:c8:eb:57:0f:cb:0b:47:b9:86:50:83:eb (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBMHm4UQPajtDjitK8Adg02NRYua67JghmS5m3E+yMq2gwZZJQ/3sIDezw2DVl9trh0gUedrzkqAAG1IMi17G/HA=
|   256 a2:ea:6e:e1:b6:d7:e7:c5:86:69:ce:ba:05:9e:38:13 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIKKLjX3ghPjmmBL2iV1RCQV9QELEU+NF06nbXTqqj4dz
80/tcp open  http    syn-ack ttl 63 Apache httpd
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Did not follow redirect to http://linkvortex.htb/
|_http-server-header: Apache
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

> <a href="https://www.hackthebox.com/achievement/machine/1521382/638" target="_blank">Linkvortex Machine from Hack The Box has been Pwned</a>
{: .prompt-tip }

> Una máquina puede estar activa o retirada. Retirada, significa que la máquina no cuenta para los puntos de temporada.
{: .prompt-tip }

