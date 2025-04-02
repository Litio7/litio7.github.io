---
title: Administrator &#x1F512;
description: La maquina Administrator esta activa. Este artículo se publicará para acceso público una vez que la maquina se retire, según la política de HackTheBox.
date: 2025-02-10
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-administrator/administrator_logo.png
categories:
  - Hack The Box
  - Machines
tags:
  - windows
  - hack the box
  - tcp
  - dns
  - smb
  - kerberos
  - ldap
  - winrm

---
### Machine Information

As is common in real life Windows pentests, you will start the Administrator box with credentials for the following account: Username: Olivia Password: ichliebedich

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/htb/machines/administrator:-$ ping -c 1 10.10.11.42
PING 10.10.11.42 (10.10.11.42) 56(84) bytes of data.
64 bytes from 10.10.11.42: icmp_seq=1 ttl=127 time=334 ms

--- 10.10.11.42 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 334.216/334.216/334.216/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/htb/machines/administrator:-$ sudo nmap -p- --open -sS --min-rate 5000 -vvv 10.10.11.42 -n -Pn -oG nmap1
Host: 10.10.11.42 ()    Status: Up
Host: 10.10.11.42 ()    Ports: 21/open/tcp//ftp///, 53/open/tcp//domain///, 88/open/tcp//kerberos-sec///, 135/open/tcp//msrpc///, 139/open/tcp//netbios-ssn///, 389/open/tcp//ldap///, 445/open/tcp//microsoft-ds///, 464/open/tcp//kpasswd5///, 593/open/tcp//http-rpc-epmap///, 636/open/tcp//ldapssl///, 3268/open/tcp//globalcatLDAP///, 3269/open/tcp//globalcatLDAPssl///, 5985/open/tcp//wsman///, 9389/open/tcp//adws///, 47001/open/tcp//winrm///, 49664/open/tcp/////, 49665/open/tcp/////, 49666/open/tcp/////, 49667/open/tcp/////, 49668/open/tcp/////, 51402/open/tcp/////, 58205/open/tcp/////, 58210/open/tcp/////, 58213/open/tcp/////, 58230/open/tcp/////, 58263/open/tcp/////
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/htb/machines/administrator:-$ sudo nmap -sCV -p21,53,88,135,139,389,445,464,593,636,3268,3269,5985,9389,47001,49664,49665,49666,49667,49668,51402,58205,58210,58213,58230,58263 -vvv 10.10.11.42 -oN nmap2
PORT      STATE SERVICE       REASON          VERSION
21/tcp    open  ftp           syn-ack ttl 127 Microsoft ftpd
| ftp-syst: 
|_  SYST: Windows_NT
53/tcp    open  domain        syn-ack ttl 127 Simple DNS Plus
88/tcp    open  kerberos-sec  syn-ack ttl 127 Microsoft Windows Kerberos (server time: 2025-02-10 06:26:02Z)
135/tcp   open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
139/tcp   open  netbios-ssn   syn-ack ttl 127 Microsoft Windows netbios-ssn
389/tcp   open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: administrator.htb0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds? syn-ack ttl 127
464/tcp   open  kpasswd5?     syn-ack ttl 127
593/tcp   open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped    syn-ack ttl 127
3268/tcp  open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: administrator.htb0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped    syn-ack ttl 127
5985/tcp  open  http          syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
9389/tcp  open  mc-nmf        syn-ack ttl 127 .NET Message Framing
47001/tcp open  http          syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49664/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49665/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49666/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49667/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49668/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
51402/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
58205/tcp open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
58210/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
58213/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
58230/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
58263/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| p2p-conficker: 
|   Checking for Conficker.C or higher...
|   Check 1 (port 35406/tcp): CLEAN (Couldn't connect)
|   Check 2 (port 40321/tcp): CLEAN (Couldn't connect)
|   Check 3 (port 52617/udp): CLEAN (Timeout)
|   Check 4 (port 32087/udp): CLEAN (Failed to receive data)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2025-02-10T06:27:07
|_  start_date: N/A
|_clock-skew: 7h00m01s
```

> <a href="https://www.hackthebox.com/achievement/machine/1521382/634" target="_blank">Administrator Machine from Hack The Box has been Pwned</a>
{: .prompt-tip }

> Una máquina puede estar activa o retirada. Retirada, significa que la máquina no cuenta para los puntos de temporada.
{: .prompt-tip }
