---
title: Campfire-2
description: La red de Forela está bajo constante ataque. El sistema de seguridad emitió una alerta sobre una antigua cuenta de administrador que solicita un ticket de KDC en un controlador de dominio. El inventario muestra que esta cuenta de usuario no se utiliza en este momento, por lo que se le solicita que la revise. Esto puede ser un ataque AsREPRoast, ya que cualquiera puede solicitar el ticket de cualquier usuario que tenga la autenticación previa deshabilitada.
date: 2024-08-03
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-campfire-2/campfire-2_logo.png
categories:
  - Hack_The_Box
  - Sherlocks
tags:
  - windows
  - hack_the_box
  - forensics
  - dfir

---
### Initial Analysis

ASREPRoast es un ataque dirigido a cuentas de Active Directory que no requieren preautenticación Kerberos. Esta configuración permite a los atacantes solicitar un AS-REP del controlador de dominio (DC) sin proporcionar una contraseña.

```powershell
PS C:\Users\litio7\Documents\htb\campfire-2> 7z x -phacktheblue .\campfire-2.zip
Security.evtx
```

![](assets/img/htb-writeup-campfire-2/campfire-21.png)

---
### Q1. When did the ASREP Roasting attack occur and the Kerberos ticket was requested by attacker for the vulnerable user?

El ataque se registra en eventos con el ID 4768, que corresponde a solicitudes de tickets de autenticación de Kerberos (AS-REQ).
El evento debe cumplir con las siguientes condiciones.
* Pre-authentication type = 0: Indica que la preautenticación de Kerberos está deshabilitada, una configuración vulnerable.
* Ticket encryption type = 0x17: El cifrado RC4 es utilizado, lo cual es común en ataques ASREPRoast.
* Service name = krbtgt: El servicio solicita el ticket en nombre del usuario objetivo.

<https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4768>

Filtro todos los eventos con ID 4768 para aislar las solicitudes de tickets Kerberos.

![](assets/img/htb-writeup-campfire-2/campfire-22.png)

Realizo una búsqueda por la palabra clave "0x17" en el contenido de los eventos.

![](assets/img/htb-writeup-campfire-2/campfire-23.png)
![](assets/img/htb-writeup-campfire-2/campfire-24.png)

La pestaña de detalles del evento contiene un campo de tiempo específico (Sistem Time) en formato UTC. Este valor representa el momento exacto en que ocurrió el ataque.

![](assets/img/htb-writeup-campfire-2/campfire-25.png)

##### A1. 2024-05-29T06:36:40

### Q2. Please confirm the User Account that was targeted by the attacker.

El campo "Account Name" indica la cuenta de usuario para la cual se solicitó el ticket, lo que permite identificar la cuenta de usuario atacada.

![](assets/img/htb-writeup-campfire-2/campfire-24_1.png)

##### A2. arthur.kyle

### Q3. What was the SID of the account?

El SID de la cuenta de usuario se encuentra en el campo "User ID". Este campo contiene el SID asociado a la cuenta objetivo.

![](assets/img/htb-writeup-campfire-2/campfire-24_2.png)

##### A3. S-1-5-21-3239415629-1862073780-2394361899-1601

### Q4. It is crucial to identify the compromised user account and the workstation responsible for this attack. Please list the internal IP address of the compromised asset to assist our threat-hunting team.

El campo "Client Address" almacena la dirección IP del dispositivo desde el cual se originó la solicitud del ticket Kerberos.

![](assets/img/htb-writeup-campfire-2/campfire-24_3.png)

##### A4. 172.17.79.129

### Q5. We do not have any artifacts from the source machine yet. Using the same DC Security logs, can you confirm the user account used to perform the ASREP Roasting attack so we can contain the compromised account/s?

Para identificar la cuenta utilizada, se debe filtrar el evento 4769, que corresponde a una solicitud de ticket de servicio en el sistema. Este tipo de evento es una operación normal en un entorno Kerberos, pero también puede ser útil para identificar las cuentas que participan en el ataque.

<https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4769>

![](assets/img/htb-writeup-campfire-2/campfire-26.png)

Un evento 4769 ocurre inmediatamente después del evento 4768 y se refiere a la solicitud de un ticket de servicio para un usuario en particular.

Al analizar el evento 4769, se observa que la cuenta de usuario que realizó la solicitud del ticket de servicio fue happy.grunwald y no la cuenta de la víctima.

![](assets/img/htb-writeup-campfire-2/campfire-27.png)

Aunque el evento 4769 no muestra explícitamente una acción maliciosa, ya que es simplemente una solicitud de ticket, la secuencia temporal y la coincidencia de la IP indican que happy.grunwald es la cuenta que está utilizando privilegios para continuar el ataque en la red, lo que apunta a que esta cuenta está involucrada en la explotación del ataque ASREP Roasting.

##### A5. happy.grunwald

> <https://labs.hackthebox.com/achievement/sherlock/1521382/736>
{: .prompt-tip }
