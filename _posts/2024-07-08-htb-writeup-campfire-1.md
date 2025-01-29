---
title: Campfire-1
description: Alonzo detectó archivos extraños en su computadora e informó al equipo SOC recién formado. Al evaluar la situación, se cree que puede haberse producido un ataque de Kerberosting en la red. Es su trabajo confirmar los hallazgos mediante el análisis de la evidencia proporcionada. Se le proporcionan registros de seguridad del controlador de dominio, registros operativos de PowerShell de la estación de trabajo afectada y archivos de precarga de la estación de trabajo afectada.
date: 2024-07-08
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-campfire-1/campfire-1_logo.png
categories:
  - Hack The Box
  - Sherlocks
tags:
  - windows
  - hack the box
  - forensics
  - dfir

---
### Initial Analysis

```powershell
PS C:\Users\litio7\Documents\htb\campfire-1> 7z x -phacktheblue campfire-1.zip
```
```powershell
PS C:\Users\litio7\Documents\htb\campfire-1> tree .\Triage\ /f
C:\USERS\LITIO7\DOCUMENTS\HTB\CAMPFIRE-1\TRIAGE
|---Domain Controller
|       SECURITY-DC.evtx
|
└---Workstation
    |   Powershell-Operational.evtx
    |
    └---2024-05-21T033012_triage_asset
        └---C
            └---Windows
                └---prefetch
```

Para este análisis, se utilizaron dos herramientas de Eric Zimmerman's
* PECmd: Este programa es clave para procesar y analizar los archivos Prefetch, que proporcionan información sobre aplicaciones ejecutadas y su temporalidad.
* Timeline Explorer: Una herramienta visual que permite analizar líneas de tiempo generadas a partir de datos forenses.

<https://ericzimmerman.github.io/#!index.md>

```powershell
PS C:\Users\litio7\Documents\htb\campfire-1> C:\Users\litio7\Documents\tools\PECmd\PECmd.exe -d "C:\Users\litio7\Documents\htb\campfire-1\Triage\Workstation\2024-05-21T033012_triage_asset\C\Windows\prefetch" --csv . --csvf prefetch_output.csv
CSV output will be saved to .\prefetch_output.csv
CSV time line output will be saved to .\prefetch_output_Timeline.csv
```

Este comando generó dos archivos:
* prefetch_output.csv: Contiene un resumen de las aplicaciones ejecutadas.
* prefetch_output_Timeline.csv: Una línea de tiempo que detalla las ejecuciones.

El ataque de kerberoasting explota la funcionalidad del servicio Kerberos para obtener un ticket de servicio cifrado, que puede ser posteriormente descifrado offline por el atacante. Los registros de seguridad del controlador de dominio (DC) permiten identificar este comportamiento mediante el análisis de eventos específicos asociados con el servicio Kerberos.

---
### Q1. Analyzing Domain Controller Security Logs, can you confirm the date & time when the kerberoasting activity occurred?

![](assets/img/htb-writeup-campfire-1/campfire-11.png)
![](assets/img/htb-writeup-campfire-1/campfire-12.png)

Para detectar kerberoasting en los logs de seguridad de un controlador de dominio (DC).

* Filtro por Event ID 4769. Este evento se genera cuando un usuario solicita un Ticket Granting Service (TGS).
* Busco entradas con TicketEncryptionType = 0x17. Este tipo de cifrado corresponde a RC4-HMAC, comúnmente utilizado en ataques de kerberoasting.

<https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4769>

![](assets/img/htb-writeup-campfire-1/campfire-13.png)

El evento que cumple con todos los criterios indica el momento en que ocurrió la actividad de kerberoasting.

Los eventos muestran una marca de tiempo que indica las 00hs (mi hora local). Al convertir esta hora a UTC, obtengo la hora del evento 03:18:09. O tambien, se puede revisar los detalles del evento donde figura el tiempo en UTC.

![](assets/img/htb-writeup-campfire-1/campfire-14.png)

##### A1. 2024-05-21 03:18:09

### Q2. What is the Service Name that was targeted?

En el evento relacionado con kerberoasting, se identificó el servicio objetivo a través del atributo 'Service Name'.

![](assets/img/htb-writeup-campfire-1/campfire-15.png)

El servicio solicitado fue: MSSQLService.

##### A2. MSSQLService

### Q3. It is really important to identify the Workstation from which this activity occurred. What is the IP Address of the workstation?

El evento relevante contiene la información sobre el cliente que realizó la solicitud.

![](assets/img/htb-writeup-campfire-1/campfire-16.png)

Al analizar el registro, se identifica el valor correspondiente al campo 'Client Address' como 172.17.79.129.

##### A3. 172.17.79.129

### Q4. What is the name of the file used to Enumerate Active directory objects and possibly find Kerberoastable accounts in the network?

Análisis de los registros de PowerShell

![](assets/img/htb-writeup-campfire-1/campfire-17.png)

Específicamente filtrando por ID de evento 4104, me permite visualizar los comandos y scripts ejecutados en la estación de trabajo comprometida. Este ID está asociado con la ejecución de bloques de script en PowerShell.

Según los registros de seguridad del controlador de dominio, la actividad ocurrió a las 03:18:09 UTC. Filtrando los eventos 4104 alrededor de esta hora, se identifica una actividad sospechosa a las 03:16 UTC, dos minutos antes del ataque de Kerberoasting. En el análisis de los eventos cercanos a este horario, se identificó que el atacante ejecutó un script utilizando el parámetro '-ep bypass', lo que indica que la política de ejecución de PowerShell fue desactivada para ejecutar scripts sin restricciones.

![](assets/img/htb-writeup-campfire-1/campfire-18.png)

Al analizar el contenido del bloque de script registrado, se detectó que la herramienta utilizada era PowerView.ps1, una herramienta comúnmente utilizada en ataques para enumerar objetos de Active Directory. 

![](assets/img/htb-writeup-campfire-1/campfire-19.png)

##### A4. powerview.ps1

### Q5. When was this script executed?

El primer evento registrado tiene un timestamp de 2024-05-21 03:16:32 UTC. Este es el momento en el que se ejecutó el script powerview.ps1, justo dos minutos antes del ataque de Kerberoasting identificado en los registros de seguridad del controlador de dominio.

![](assets/img/htb-writeup-campfire-1/campfire-110.png)

##### A5. 2024-05-21 03:16:32

### Q6. What is the full path of the tool used to perform the actual kerberoasting attack?

Análisis de los archivos Prefetch.

El comando ```PECmd.exe -d "C:\Users\litio7\Documents\htb\campfire-1\Triage\Workstation\2024-05-21T033012_triage_asset\C\Windows\prefetch" --csv . --csvf prefetch_output.csv``` generó dos archivos CSV con los detalles de los programas ejecutados.
Estos archivos los carge en Timeline Explorer, donde se analize las entradas relevantes para identificar programas ejecutados alrededor de las 03:18:09 UTC, momento del ataque de Kerberoasting.

![](assets/img/htb-writeup-campfire-1/campfire-111.png)

Identifique un programa ejecutable llamado Rubeus.exe, conocido por ser una herramienta ofensiva para ataques basados en Kerberos. Su ejecución ocurrió 1 segundo antes del evento malicioso registrado en los logs del controlador de dominio.

![](assets/img/htb-writeup-campfire-1/campfire-112.png)

##### A6. C:\USERS\ALONZO.SPIRE\DOWNLOADS\RUBEUS.EXE

### Q7. When was the tool executed to dump creden

En el campo Last Run, la marca de tiempo registrada para Rubeus.exe fue (2024-05-21 03:18:08) UTC, 1 segundo antes del evento malicioso registrado en los logs del controlador de dominio.

![](assets/img/htb-writeup-campfire-1/campfire-112.png)

##### A7. 2024-05-21 03:18:08

---
### Timeline

| Time (UTC)          | Description                       | Reference         |
| :------------------ | :-------------------------------- | :---------------: |
| 2024-05-21 03:16:32 | PowerView.ps1 Loaded 	Workstation | PowerShell Logs   |
| 2024-05-21 03:18:08 | Rubeus.exe run                    | Prefetch          |
| 2024-05-21 03:18:09 | Kerberoasting auth attempt        | DC Security Logs  |

> <https://labs.hackthebox.com/achievement/sherlock/1521382/737>
{: .prompt-tip }