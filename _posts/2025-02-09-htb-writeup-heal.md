---
title: Heal &#x1F512;
description: La maquina Heal esta activa. Este artículo se publicará para acceso público una vez que la maquina se retire, según la política de HackTheBox.
date: 2025-02-09
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-heal/heal_logo.png
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
/home/kali/Documents/htb/machines/heal:-$ ping -c 1 10.10.11.46
PING 10.10.11.46 (10.10.11.46) 56(84) bytes of data.
64 bytes from 10.10.11.46: icmp_seq=1 ttl=63 time=308 ms

--- 10.10.11.46 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 307.576/307.576/307.576/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/htb/machines/heal:-$ sudo nmap -p- --open -sS --min-rate 5000 -vvv 10.10.11.46 -n -oG nmap1
Host: 10.10.11.46 ()    Status: Up
Host: 10.10.11.46 ()    Ports: 22/open/tcp//ssh///, 80/open/tcp//http///        Ignored State: closed (65533)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/htb/machines/heal:-$ sudo nmap -sCV -p22,80 -vvv 10.10.11.46 -oN nmap2
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 68:af:80:86:6e:61:7e:bf:0b:ea:10:52:d7:7a:94:3d (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBFWKy4neTpMZp5wFROezpCVZeStDXH5gI5zP4XB9UarPr/qBNNViyJsTTIzQkCwYb2GwaKqDZ3s60sEZw362L0o=
|   256 52:f4:8d:f1:c7:85:b6:6f:c6:5f:b2:db:a6:17:68:ae (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAILMCYbmj9e7GtvnDNH/PoXrtZbCxr49qUY8gUwHmvDKU
80/tcp open  http    syn-ack ttl 63 nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Did not follow redirect to http://heal.htb/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

> <a href="https://www.hackthebox.com/achievement/machine/1521382/640" target="_blank">Heal Machine from Hack The Box has been Pwned</a>
{: .prompt-tip }

> Una máquina puede estar activa o retirada. Retirada, significa que la máquina no cuenta para los puntos de temporada.
{: .prompt-tip }