---
title: BigBang &#x1F512;
description: La maquina BigBang esta activa. Este artículo se publicará para acceso público una vez que la maquina se retire, según la política de HackTheBox.
date: 2025-03-22
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-bigbang/bigbang_logo.png
categories:
  - Hack The Box
  - Machines
tags:
  - linux
  - hack the box
  - tcp
  - http
  - ssh

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/htb/machines/bigbang:-$ ping -c 1 10.10.11.52
PING 10.10.11.52 (10.10.11.52) 56(84) bytes of data.
64 bytes from 10.10.11.52: icmp_seq=1 ttl=63 time=1117 ms

--- 10.10.11.52 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 1116.788/1116.788/1116.788/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/htb/machines/bigbang:-$ sudo nmap -p- --open -sS --min-rate 5000 -vvv 10.10.11.52 -n -Pn -oG nmap1
Host: 10.10.11.52 ()	Status: Up
Host: 10.10.11.52 ()	Ports: 22/open/tcp//ssh///, 80/open/tcp//http///	Ignored State: closed (65533)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/htb/machines/bigbang:-$ sudo nmap -sCV -p22,80 -vvv 10.10.11.52 -oN nmap2
```

> <a href="https://www.hackthebox.com/achievement/machine/1521382/645" target="_blank">BigBang Machine from Hack The Box has been Pwned</a>
{: .prompt-tip }

> Una máquina puede estar activa o retirada. Retirada, significa que la máquina no cuenta para los puntos de temporada.
{: .prompt-tip }