---
title: Cat &#x1F4DC;
description: Cat es una máquina Linux de dificultad media que presenta una aplicación web en PHP vulnerable a Cross-Site Scripting (XSS), el cual puede ser desencadenado mediante un evento onerror para evadir los filtros de seguridad de la aplicación. Aprovechando esta vulnerabilidad XSS, es posible realizar un robo de cookies (cookie hijacking) y obtener la cookie de un administrador para escalar privilegios dentro de la aplicación. Una vez con privilegios elevados, se puede explotar una SQL Injection en una base de datos SQLite para lograr Remote Code Execution (RCE) al almacenar una webshell maliciosa directamente en la base de datos. Con acceso a la base de datos interna de la aplicación, se recupera una contraseña hasheada que, al ser crackeada, permite autenticarse como un usuario con pertenencia a un grupo que tiene permisos para leer los registros del servidor. Estos logs filtran una contraseña en texto claro de un usuario con acceso a una instancia interna de Gitea en su versión 1.22.0, la cual es vulnerable a una vulnerabilidad XSS identificada como CVE-2024-6886 debido a una sanitización incorrecta de entradas. Al explotar CVE-2024-6886, se logra leer un repositorio privado de Gitea que contiene credenciales del usuario root.
date: 2025-02-08
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-cat/cat_logo.png
categories:
  - Hack_The_Box
  - Machines
tags:
  - linux
  - hack_the_box
  - tcp
  - ssh
  - http

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/htb/machines/cat:-$ ping -c 1 10.10.11.53
PING 10.10.11.53 (10.10.11.53) 56(84) bytes of data.
64 bytes from 10.10.11.53: icmp_seq=1 ttl=63 time=252 ms

--- 10.10.11.53 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 252.212/252.212/252.212/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/htb/machines/cat:-$ sudo nmap -p- --open -sS --min-rate 5000 -vvv 10.10.11.53 -n -Pn -oG nmap1
Host: 10.10.11.53 ()    Status: Up
Host: 10.10.11.53 ()    Ports: 22/open/tcp//ssh///, 80/open/tcp//http///
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/htb/machines/cat:-$ sudo nmap -sCV -p22,80 -vvv 10.10.11.53 -oN nmap2
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 96:2d:f5:c6:f6:9f:59:60:e5:65:85:ab:49:e4:76:14 (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC/7/gBYFf93Ljst5b58XeNKd53hjhC57SgmM9qFvMACECVK0r/Z11ho0Z2xy6i9R5dX2G/HAlIfcu6i2QD9lILOnBmSaHZ22HCjjQKzSbbrnlcIcaEZiE011qtkVmtCd2e5zeVUltA9WCD69pco7BM29OU7FlnMN0iRlF8u962CaRnD4jni/zuiG5C2fcrTHWBxc/RIRELrfJpS3AjJCgEptaa7fsH/XfmOHEkNwOL0ZK0/tdbutmcwWf9dDjV6opyg4IK73UNIJSSak0UXHcCpv0GduF3fep3hmjEwkBgTg/EeZO1IekGssI7yCr0VxvJVz/Gav+snOZ/A1inA5EMqYHGK07B41+0rZo+EZZNbuxlNw/YLQAGuC5tOHt896wZ9tnFeqp3CpFdm2rPGUtFW0jogdda1pRmRy5CNQTPDd6kdtdrZYKqHIWfURmzqva7byzQ1YPjhI22cQ49M79A0yf4yOCPrGlNNzeNJkeZM/LU6p7rNJKxE9CuBAEoyh0=
|   256 9e:c4:a4:40:e9:da:cc:62:d1:d6:5a:2f:9e:7b:d4:aa (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBEmL+UFD1eC5+aMAOZGipV3cuvXzPFlhqtKj7yVlVwXFN92zXioVTMYVBaivGHf3xmPFInqiVmvsOy3w4TsRja4=
|   256 6e:22:2a:6a:6d:eb:de:19:b7:16:97:c2:7e:89:29:d5 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIEOCpb672fivSz3OLXzut3bkFzO4l6xH57aWuSu4RikE
80/tcp open  http    syn-ack ttl 63 Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Did not follow redirect to http://cat.htb/
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

> <a href="https://www.hackthebox.com/achievement/machine/1521382/646" target="_blank">Cat Machine from Hack The Box has been Pwned</a>
{: .prompt-tip }

