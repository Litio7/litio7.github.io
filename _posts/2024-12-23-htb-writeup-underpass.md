---
title: UnderPass &#x1F512;
description: La maquina UnderPass esta activa. Este artículo se publicará para acceso público una vez que la maquina se retire, según la política de HackTheBox.
date: 2024-12-23
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-underpass/underpass_logo.png
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
/home/kali/Documents/htb/machines/administrator:-$ ping -c 1 10.10.11.48
PING 10.10.11.48 (10.10.11.48) 56(84) bytes of data.
64 bytes from 10.10.11.48: icmp_seq=1 ttl=127 time=334 ms

--- 10.10.11.48 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 334.216/334.216/334.216/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/htb/machines/administrator:-$ sudo nmap -p- --open -sS --min-rate 5000 -n -Pn -vvv 10.10.11.48 -oG nmap1
Host: 10.10.11.48 ()	Status: Up
Host: 10.10.11.48 ()	Ports: 22/open/tcp//ssh///, 80/open/tcp//http///
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/htb/machines/administrator:-$ sudo nmap -sCV -p22,80 -vvv -oN nmap2 10.10.11.48
Nmap scan report for 10.10.11.48
Host is up, received echo-reply ttl 63 (0.39s latency).
Scanned at 2024-12-21 18:17:11 -03 for 30s

PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 48:b0:d2:c7:29:26:ae:3d:fb:b7:6b:0f:f5:4d:2a:ea (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBK+kvbyNUglQLkP2Bp7QVhfp7EnRWMHVtM7xtxk34WU5s+lYksJ07/lmMpJN/bwey1SVpG0FAgL0C/+2r71XUEo=
|   256 cb:61:64:b8:1b:1b:b5:ba:b8:45:86:c5:16:bb:e2:a2 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIJ8XNCLFSIxMNibmm+q7mFtNDYzoGAJ/vDNa6MUjfU91
80/tcp open  http    syn-ack ttl 63 Apache httpd 2.4.52 ((Ubuntu))
|_http-server-header: Apache/2.4.52 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
| http-methods: 
|_  Supported Methods: GET POST OPTIONS HEAD
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

> <a href="https://www.hackthebox.com/achievement/machine/1521382/641" target="_blank">Underpass Machine from Hack The Box has been Pwned</a>
{: .prompt-tip }

> Una máquina puede estar activa o retirada. Retirada, significa que la máquina no cuenta para los puntos de temporada.
{: .prompt-tip }