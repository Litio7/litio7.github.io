---
title: Eureka &#x1F512;
description: La maquina Eureka esta activa. Este artículo se publicará para acceso público una vez que la maquina se retire, según la política de HackTheBox.
date: 2025-05-01
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-eureka/eureka_logo.png
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
/home/kali/Documents/htb/machines/eureka:-$ ping -c 1 10.10.11.66
PING 10.10.11.66 (10.10.11.66) 56(84) bytes of data.
64 bytes from 10.10.11.66: icmp_seq=1 ttl=63 time=181 ms

--- 10.10.11.66 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 180.608/180.608/180.608/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/htb/machines/eureka:-$ sudo nmap -p- --open -sS --min-rate 5000 -vvv 10.10.11.66 -n -Pn -oG nmap1
Host: 10.10.11.66 ()    Status: Up
Host: 10.10.11.66 ()    Ports: 22/open/tcp//ssh///, 80/open/tcp//http///, 8761/open/tcp/////
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/htb/machines/eureka:-$ sudo nmap -sCV -p22,80,8761 -vvv 10.10.11.66 -oN nmap2
PORT     STATE SERVICE REASON         VERSION
22/tcp   open  ssh     syn-ack ttl 63 OpenSSH 8.2p1 Ubuntu 4ubuntu0.12 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 d6:b2:10:42:32:35:4d:c9:ae:bd:3f:1f:58:65:ce:49 (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCpa5HH8lfpsh11cCkEoqcNXWPj6wh8GaDrnXst/q7zd1PlBzzwnhzez+7mhwfv1PuPf5fZ7KtZLMfVPuUzkUHVEwF0gSN0GrFcKl/D34HmZPZAsSpsWzgrE2sayZa3xZuXKgrm5O4wyY+LHNPuHDUo0aUqZp/f7SBPqdwDdBVtcE8ME/AyTeJiJrOhgQWEYxSiHMzsm3zX40ehWg2vNjFHDRZWCj3kJQi0c6Eh0T+hnuuK8A3Aq2Ik+L2aITjTy0fNqd9ry7i6JMumO6HjnSrvxAicyjmFUJPdw1QNOXm+m+p37fQ+6mClAh15juBhzXWUYU22q2q9O/Dc/SAqlIjn1lLbhpZNengZWpJiwwIxXyDGeJU7VyNCIIYU8J07BtoE4fELI26T8u2BzMEJI5uK3UToWKsriimSYUeKA6xczMV+rBRhdbGe39LI5AKXmVM1NELtqIyt7ktmTOkRQ024ZoSS/c+ulR4Ci7DIiZEyM2uhVfe0Ah7KnhiyxdMSlb0=
|   256 90:11:9d:67:b6:f6:64:d4:df:7f:ed:4a:90:2e:6d:7b (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBNqI0DxtJG3vy9f8AZM8MAmyCh1aCSACD/EKI7solsSlJ937k5Z4QregepNPXHjE+w6d8OkSInNehxtHYIR5nKk=
|   256 94:37:d3:42:95:5d:ad:f7:79:73:a6:37:94:45:ad:47 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIHNmmTon1qbQUXQdI6Ov49enFe6SgC40ECUXhF0agNVn
80/tcp   open  http    syn-ack ttl 63 nginx 1.18.0 (Ubuntu)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Did not follow redirect to http://furni.htb/
8761/tcp open  http    syn-ack ttl 63 Apache Tomcat (language: en)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
| http-auth: 
| HTTP/1.1 401 \x0D
|_  Basic realm=Realm
|_http-title: Site doesn't have a title.
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

> <a href="https://www.hackthebox.com/achievement/machine/1521382/658" target="_blank">Eureka Machine from Hack The Box has been Pwned</a>
{: .prompt-tip }

> Una máquina puede estar activa o retirada. Retirada, significa que la máquina no cuenta para los puntos de temporada.
{: .prompt-tip }
