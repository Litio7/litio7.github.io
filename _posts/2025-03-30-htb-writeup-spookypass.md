---
title: SpookyPass
description: All the coolest ghosts in town are going to a Haunted Houseparty - can you prove you deserve to get in?
date: 2025-03-30
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-challenges/reversing_logo.png
categories:
  - Hack The Box
  - Challenges
tags:
  - hack the box
  - reversing
  - reverse engineering
  - information gathering
  - misconfiguration exploitation

---
## Information Gathering

```terminal
/home/kali/Documents/htb/challenges/spookypass:-$ unzip SpookyPass.zip
inflating: rev_spookypass/pass
```

Extraigo el contenido del archivo comprimido y obtengo un ejecutable llamado `pass`.

```terminal
/home/kali/Documents/htb/challenges/spookypass:-$ tree .
.
├── SpookyPass.zip
└── rev_spookypass
    └── pass
```

Se trata de un ejecutable ELF de 64 bits para Linux.

```terminal
/home/kali/Documents/htb/challenges/spookypass:-$ file pass
pass: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=3008217772cc2426c643d69b80a96c715490dd91, for GNU/Linux 4.4.0, not stripped
```

Ejecuto el binario e identifico que solicita una contraseña para continuar.

```terminal
/home/kali/Documents/htb/challenges/spookypass:-$ ./pass           
Welcome to the SPOOKIEST party of the year.
Before we let you in, you'll need to give us the password: pass
You're not a real ghost; clear off!
```

Este comportamiento sugiere que el binario contiene algún mecanismo de validación de contraseñas.

---
## Misconfiguration Exploitation

Utilizo strings para extraer cadenas de texto del binario, y revela una posible contraseña en texto plano.

```terminal
/home/kali/Documents/htb/challenges/spookypass:-$ strings pass
```

![](assets/img/htb-writeup-spookypass/spookypass1.png)

Se ejecuta el binario con ltrace para observar las funciones que se llaman en tiempo de ejecución y analizar la comparación de la contraseña introducida.

```terminal
/home/kali/Documents/htb/challenges/spookypass:-$ ltrace ./pass
```

![](assets/img/htb-writeup-spookypass/spookypass2.png)

La salida de ltrace muestra que el programa utiliza `strcmp` para comparar la entrada con la cadena `s3cr3t_p455_f0r_gh05t5_4nd_gh0ul5`, lo que confirma que esta es la contraseña correcta.

Al introducirla en la ejecución del binario, se obtiene la flag.

```terminal
/home/kali/Documents/htb/challenges/spookypass:-$ ./pass
Welcome to the SPOOKIEST party of the year.
Before we let you in, you'll need to give us the password: s3cr3t_p455_f0r_gh05t5_4nd_gh0ul5
Welcome inside!
HTB{un0bfu5c4t3d_5tr1ng5}
```

> <a href="https://www.hackthebox.com/achievement/challenge/1521382/806" target="_blank">SpookyPass Challenge from Hack The Box has been Pwned</a>
{: .prompt-tip }