---
title: Unit42
description: En este Sherlock, te familiarizarás con los registros de Sysmon y varios EventIDs útiles para identificar y analizar actividades maliciosas en un sistema Windows. Unit42 de Palo Alto realizó recientemente una investigación sobre una campaña de UltraVNC, en la que los atacantes utilizaron una versión con backdoor de UltraVNC para mantener el acceso a los sistemas. Este laboratorio está inspirado en esa campaña y guía a los participantes a través de la etapa de acceso inicial de la campaña.
date: 2024-08-03
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-unit42/unit42_logo.png
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

```
PS C:\Users\litio7\Documents\htb\> 7z x -phacktheblue unit42.zip
Microsoft-Windows-Sysmon-Operational.evtx
```
![](/assets/img/htb-writeup-unit42/unit421_1.png)

---
### Q1. How many Event logs are there with Event ID 11?

Se utiliza la función 'Filter Current Log' para buscar exclusivamente eventos con Event ID 11.

El Event ID 11 incluye detalles como los siguientes:
*	La ubicación del archivo creado.
*	El proceso o imagen que lo creó.

<https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90011>

![](/assets/img/htb-writeup-unit42/unit421_2.png)

En la parte superior del visor de eventos, vemos que hay 56 eventos registrados con el Event ID 11.

![](/assets/img/htb-writeup-unit42/unit421_3.png)

##### A1. 56

### Q2. Whenever a process is created in memory, an event with Event ID 1 is recorded with details such as command line, hashes, process path, parent process path, etc. This information is very useful for an analyst because it allows us to see all programs executed on a system, which means we can spot any malicious processes being executed. What is the malicious process that infected the victim’s system?

Filtro únicamente los eventos con Event ID 1.

Este evento contiene información clave como:
*	Línea de comandos.
*	Hashes.
*	Ruta del proceso.
*	Proceso padre.

<https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90001>

Al aplicar el filtro, encuentro 6 eventos registrados con este ID. Esto reduce el ruido y facilita el análisis de procesos específicos.

![](/assets/img/htb-writeup-unit42/unit422_1.png)

Revisando los detalles de los eventos filtrados, busco procesos ejecutados desde rutas inusuales o con nombres sospechosos.

![](/assets/img/htb-writeup-unit42/unit422_2.png)

Esta ruta contiene varios indicadores de comportamiento malicioso:
*	Se ejecuta desde el directorio Downloads, que es comúnmente utilizado por malware descargado.
*	El nombre del archivo tiene una extensión repetida '.exe.exe', lo que podría ser un intento de evadir detecciones automáticas.

El archivo malicioso, 'Preventivo24.02.14.exe.exe', puede ser evaluado con herramientas como VirusTotal.

<https://www.virustotal.com/gui/home/search>

En los detalles del evento con Event ID 1 en Sysmon, se incluye información sobre el hash del archivo ejecutado.

![](/assets/img/htb-writeup-unit42/unit422_3.png)

![](/assets/img/htb-writeup-unit42/unit422_4.png)


##### A2. C:\Users\CyberJunkie\Downloads\Preventivo24.02.14.exe.exe

### Q3. Which Cloud drive was used to distribute the malware?

Utilizo Event ID 22 en los registros de Sysmon, que documenta las consultas DNS realizadas por el sistema, para buscar correlaciones entre las descargas y la creación del archivo malicioso identificado previamente.

Los eventos con Event ID 22 en Sysmon registran consultas DNS realizadas por el sistema. Estos eventos incluyen detalles como:
* El dominio consultado.
* El proceso o imagen que realizó la consulta.

<https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90022>

Filtro los eventos de Event ID 22 en el visor de eventos para identificar las solicitudes DNS realizadas alrededor del tiempo de descarga del archivo malicioso.

Una consulta a Dropbox.com justo antes de la creación del archivo malicioso indica que Dropbox fue utilizado como mecanismo de entrega del archivo.

![](/assets/img/htb-writeup-unit42/unit423_1.png)

Al combinar los eventos de Event ID 11 y Event ID 22, se encuentra un vínculo directo.

La creación de un archivo temporal en la ruta de Firefox '.part' ocurrió inmediatamente después de la consulta a Dropbox. Esto es consistente con el comportamiento de un archivo descargado desde un navegador, como parte del proceso de descarga.

![](/assets/img/htb-writeup-unit42/unit423_2.png)

##### A3.dropbox
  
### Q4. The initial malicious file time-stamped (a defense evasion technique, where the file creation date is changed to make it appear old) many files it created on disk. What was the timestamp changed to for a PDF file?

Filtro el visor de eventos para mostrar únicamente los eventos con Event ID 2, que registra modificaciones en las marcas de tiempo de creación de los archivos. 
En estos eventos, se detallan:
*	Ruta del archivo afectado.
*	Hora original de creación.
*	Nueva hora de creación.

<https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90002>

![](/assets/img/htb-writeup-unit42/unit424_1.png)

Tras aplicar el filtro, encuentro un evento relacionado con un archivo PDF.

Este archivo tenía su marca de tiempo original alterada.

Esto sugiere que el archivo fue manipulado para parecer más antiguo y evitar generar sospechas durante el análisis.

##### A.4 2024-01-14 08:10:06

### Q5. The malicious file dropped a few files on disk. Where was “once.cmd” created on disk? Please answer with the full path along with the filename.

Filtro el visor de eventos para mostrar únicamente los registros de Event ID 11.

![](/assets/img/htb-writeup-unit42/unit425_1.png)

Utilizando la función de búsqueda del visor de eventos, localizo las entradas relacionadas con la creación del archivo 'once.cmd'.

Encuentro un resultado donde el archivo fue creado por el proceso malicioso 'Preventivo24.02.14.exe'.

![](/assets/img/htb-writeup-unit42/unit425_2.png)

##### A5. C:\Users\CyberJunkie\AppData\Roaming\Photo and Fax Vn\Photo and vn 1.1.2\install\F97891C\WindowsVolume\Games\once.cmd

### Q6. The malicious file attempted to reach a dummy domain, most likely to check the internet connection status. What domain name did it try to connect to?

Filtro los registros para mostrar únicamente los eventos relacionados con Event ID 22, que corresponde a las consultas DNS realizadas por el sistema.

Dentro de estos registros, encuentro un evento que muestra una consulta DNS realizada.

![](/assets/img/htb-writeup-unit42/unit426_1.png)

En este caso, el archivo malicioso probablemente intentó usarlo como una forma de verificar la conexión a Internet, un paso típico en las primeras fases de ejecución de malware.

##### A6. www.example.com

### Q7. Which IP address did the malicious process try to reach out to?

Aplico un filtro para Event ID 3, que corresponde a conexiones de red iniciadas o recibidas por el sistema.
Este evento incluye detalles como:
*	Dirección IP y puerto de origen.
*	Dirección IP y puerto de destino.
*	Nombre del proceso que inició la conexión.

<https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90003>

![](/assets/img/htb-writeup-unit42/unit427_1.png)

Al filtrar los eventos, encuentro un registro que muestra una conexión iniciada por el proceso 'Preventivo24.02.14.exe'. Esto indica que el proceso intentó comunicarse con una dirección externa.

Detalles de la conexión:
* Protocolo: TCP.
* Dirección IP de destino: 93.184.216.34.
* Puerto de destino: 80.


##### A7. 93.184.216.34

### Q8. The malicious process terminated itself after infecting the PC with a backdoored variant of UltraVNC. When did the process terminate itself?

Filtro los eventos con Event ID 5, que registra la terminación de procesos en los registros de Sysmon.

<https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90005>

![](/assets/img/htb-writeup-unit42/unit428_1.png)

El proceso se auto terminó después de completar la infección con UltraVNC backdooreado.

##### A8. 2024-02-14 03:41:58

---
### Timeline

| Time (UTC)            | Description                           | Reference               |
| :-------------------- | :------------------------------------ | :---------------------: |
| 2024-02-14T03:41:26.4 | Firefox DNS query for Dropbox         | DNS (22)                |
| 2024-02-14T03:41:26.5 | Firefox malware download              | File Creation (11)      |
| 2024-02-14T03:41:30.4 | Windows tags malware as downloaded    | File Creation (11)      |
| 2024-02-14T03:41:45.8 | Firefox DNS query for Dropbox         | DNS (22)                |
| 2024-02-14T03:41:56.6 | Preventivo24.02.14.exe.exe launched   | Process Creation (1)    |
| 2024-02-14T03:41:57.9 | Malware starts msiexec                | Process Creation (1)    |
| 2024-02-14T03:41:58.4 | Malware writes files to disk          | File Creation (11)      |
| 2024-02-14T03:41:58.4 | Malware timestomps 15 files.          | Time Modification (2)   |
| 2024-02-14T03:41:58.6 | Malware connects to 93.184.216.34     | Network (3)             |
| 2024-02-14T03:41:58.8 | Malware DNS query for www.example.com | DNS (22)                |
| 2024-02-14 03:41:58.8 | Malware terminates itself             | Process Termination (5) |


> <https://labs.hackthebox.com/achievement/sherlock/1521382/632>
{: .prompt-tip }