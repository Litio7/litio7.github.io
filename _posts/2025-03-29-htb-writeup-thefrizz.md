---
title: TheFrizz &#x1F512;
description: La maquina TheFrizz esta activa. Este artículo se publicará para acceso público una vez que la maquina se retire, según la política de HackTheBox.
date: 2025-03-29
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-thefrizz/thefrizz_logo.png
categories:
  - Hack The Box
  - Machines
tags:
  - windows
  - hack the box
  - tcp
  - ssh
  - dns
  - http
  - rpc
  - smb
  - kerberos
  - ldap

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/htb/machines/thefrizz:-$ ping -c 1 10.10.11.60
PING 10.10.11.60 (10.10.11.60) 56(84) bytes of data.
64 bytes from 10.10.11.60: icmp_seq=1 ttl=127 time=348 ms

--- 10.10.11.60 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 347.597/347.597/347.597/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/htb/machines/thefrizz:-$ sudo nmap -p- --open -sS --min-rate 5000 -vvv 10.10.11.60 -n -Pn -oG nmap1
Host: 10.10.11.60 ()    Status: Up
Host: 10.10.11.60 ()    Ports: 22/open/tcp//ssh///, 53/open/tcp//domain///, 80/open/tcp//http///, 88/open/tcp//kerberos-sec///, 135/open/tcp//msrpc///, 139/open/tcp//netbios-ssn///, 389/open/tcp//ldap///, 445/open/tcp//microsoft-ds///, 464/open/tcp//kpasswd5///, 593/open/tcp//http-rpc-epmap///, 636/open/tcp//ldapssl///, 3268/open/tcp//globalcatLDAP///, 3269/open/tcp//globalcatLDAPssl///, 9389/open/tcp//adws///, 49664/open/tcp/////, 49668/open/tcp/////, 49670/open/tcp/////, 59973/open/tcp/////, 59977/open/tcp/////, 59987/open/tcp/////    Ignored State: filtered (65515)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/htb/machines/thefrizz:-$ sudo nmap -sCV -p22,53,80,88,135,139,389,445,464,593,636,3268,3269,9389,49664,49668,49670,59973,59977,59987 -vvv 10.10.11.60 -oN nmap2
PORT      STATE SERVICE       REASON          VERSION
22/tcp    open  ssh           syn-ack ttl 127 OpenSSH for_Windows_9.5 (protocol 2.0)
53/tcp    open  domain        syn-ack ttl 127 Simple DNS Plus
80/tcp    open  http          syn-ack ttl 127 Apache httpd 2.4.58 (OpenSSL/3.1.3 PHP/8.2.12)
|_http-title: Did not follow redirect to http://frizzdc.frizz.htb/home/
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.58 (Win64) OpenSSL/3.1.3 PHP/8.2.12
88/tcp    open  kerberos-sec  syn-ack ttl 127 Microsoft Windows Kerberos (server time: 2025-03-27 06:38:31Z)
135/tcp   open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
139/tcp   open  netbios-ssn   syn-ack ttl 127 Microsoft Windows netbios-ssn
389/tcp   open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: frizz.htb0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds? syn-ack ttl 127
464/tcp   open  kpasswd5?     syn-ack ttl 127
593/tcp   open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped    syn-ack ttl 127
3268/tcp  open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: frizz.htb0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped    syn-ack ttl 127
9389/tcp  open  mc-nmf        syn-ack ttl 127 .NET Message Framing
49664/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49668/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49670/tcp open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
59973/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
59977/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
59987/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
Service Info: Hosts: localhost, FRIZZDC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| p2p-conficker: 
|   Checking for Conficker.C or higher...
|   Check 1 (port 21203/tcp): CLEAN (Timeout)
|   Check 2 (port 40792/tcp): CLEAN (Timeout)
|   Check 3 (port 49509/udp): CLEAN (Timeout)
|   Check 4 (port 33366/udp): CLEAN (Timeout)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
|_clock-skew: 7h00m01s
| smb2-time: 
|   date: 2025-03-27T06:39:32
|_  start_date: N/A
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
```

> <a href="https://www.hackthebox.com/achievement/machine/1521382/652" target="_blank">TheFrizz Machine from Hack The Box has been Pwned</a>
{: .prompt-tip }

> Una máquina puede estar activa o retirada. Retirada, significa que la máquina no cuenta para los puntos de temporada.
{: .prompt-tip }