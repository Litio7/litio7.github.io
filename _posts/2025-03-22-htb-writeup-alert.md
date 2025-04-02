---
title: Alert &#x1F512;
description: Alert es una máquina Linux de nivel fácil con un sitio web para subir, ver y compartir archivos Markdown. El sitio es vulnerable a XSS, que se explota para acceder a una página interna vulnerable a Arbitrary File Read y se utiliza para obtener acceso al hash de una contraseña. Posteriormente, se descifra el hash para revelar las credenciales utilizadas para obtener acceso SSH al objetivo. La enumeración de los procesos que se ejecutan en el sistema muestra un archivo PHP que se ejecuta regularmente, con privilegios excesivos para el grupo de administración al que pertenece nuestro usuario, lo que nos permite sobrescribir el archivo para la ejecución de código como root.
date: 2024-12-14
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-alert/alert_logo.png
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
/home/kali/Documents/htb/machines/alert:-$ ping -c 1 10.10.11.44
PING 10.10.11.44 (10.10.11.44) 56(84) bytes of data.
64 bytes from 10.10.11.44: icmp_seq=1 ttl=63 time=397 ms

--- 10.10.11.44 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 396.557/396.557/396.557/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/htb/machines/alert:-$ sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn -oG nmap1 10.10.11.44
Host: 10.10.11.44 ()    Status: Up
Host: 10.10.11.44 ()    Ports: 22/open/tcp//ssh///, 80/open/tcp//http///
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/htb/machines/alert:-$ sudo nmap -sCV -vvv -p22,80 -oN nmap2 10.10.11.44
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 7e:46:2c:46:6e:e6:d1:eb:2d:9d:34:25:e6:36:14:a7 (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDSrBVJEKTgtUohrzoK9i67CgzqLAxnhEsPmW8hS5CFFGYikUduAcNkKsmmgQI09Q+6pa+7YHsnxcerBnW0taI//IYB5TI/LSE3yUxyk/ROkKLXPNiNGUhC6QiCj3ZTvThyHrFD9ZTxWfZKEQTcOiPs15+HRPCZepPouRtREGwmJcvDal1ix8p/2/C8X57ekouEEpIk1wzDTG5AM2/D08gXXe0TP+KYEaZEzAKM/mQUAqNTxfjc9x5rlfPYW+50kTDwtyKta57tBkkRCnnns0YRnPNtt0AH374ZkYLcqpzxwN8iTNXaeVT/dGfF4mA1uW89hSMarmiRgRh20Y1KIaInHjv9YcvSlbWz+2sz3ev725d4IExQTvDR4sfUAdysIX/q1iNpleyRgM4cvDMjxD6lEKpvQYSWVlRoJwbUUnJqnmZXboRwzRl+V3XCUaABJrA/1K1gvJfsPcU5LX303CV6LDwvLJIcgXlEbtjhkcxz7b7CS78BEW9hPifCUDGKfUs=
|   256 45:7b:20:95:ec:17:c5:b4:d8:86:50:81:e0:8c:e8:b8 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBHYLF+puo27gFRX69GBeZJqCeHN3ps2BScsUhKoDV66yEPMOo/Sn588F/wqBnJxsPB3KSFH+kbYW2M6erFI3U5k=
|   256 cb:92:ad:6b:fc:c8:8e:5e:9f:8c:a2:69:1b:6d:d0:f7 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIG/QUl3gapBOWCGEHplsOKe2NlWjlrb5vTTLjg6gMuGl
80/tcp open  http    syn-ack ttl 63 Apache httpd 2.4.41 ((Ubuntu))
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Did not follow redirect to http://alert.htb/
|_http-server-header: Apache/2.4.41 (Ubuntu)
```

> <a href="https://www.hackthebox.com/achievement/machine/1521382/636" target="_blank">Alert Machine from Hack The Box has been Pwned</a>
{: .prompt-tip }

> Una máquina puede estar activa o retirada. Retirada, significa que la máquina no cuenta para los puntos de temporada.
{: .prompt-tip }