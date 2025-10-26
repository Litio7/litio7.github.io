---
title: Bft
description: En este Sherlock, se familiarizará con la investigación forense de MFT (tabla maestra de archivos). Se le presentarán herramientas y metodologías conocidas para analizar artefactos de MFT a fin de identificar actividades maliciosas. Durante nuestro análisis, utilizará la herramienta MFTECmd para analizar el archivo MFT proporcionado, TimeLine Explorer para abrir y analizar los resultados del MFT analizado y un editor hexadecimal para recuperar el contenido de los archivos del MFT.
date: 2024-08-06
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-bft/bft_logo.png
categories:
  - Hack_The_Box
  - Sherlocks
tags:
  - hack_the_box
  - dfir

---
### Initial Analysis

```powershell
PS C:\Users\Escritorio\HtB> 7z x BFT.zip -phacktheblue
```
El archivo Zip contiene solo un archivo 'C/$MFT'. 

MFT (Master File Table) registra detalles sobre cada archivo y directorio en una unidad NTFS (New Technology File System). Cada archivo o directorio está representado por una entrada en la tabla, denominada entrada MFT. A estas entradas se les asigna un identificador único, conocido como número de registro MFT. 

Para analizar el archivo '$MFT' utilizo la siguiente herramienta.

<https://ericzimmerman.github.io/#!index.md>

MFTeCMD es una herramienta que se especializa en analizar la 'Master File Table' de los sistemas de archivos NTFS.

```powershell
PS C:\Users\Escritorio\HtB\BFT\C> MFTECmd.exe -f ".\$MFT" --csv . --csvf mft.csv
MFTECmd version 1.2.2.1
Author: Eric Zimmerman (saericzimmerman@gmail.com)
https://github.com/EricZimmerman/MFTECmd
Command line: -f .\$MFT --csv . --csvf mft.csv
Warning: Administrator privileges not found!
File type: Mft
Processed .\$MFT in 6,5221 seconds
.\$MFT: FILE records found: 171.927 (Free records: 142.905) File size: 307,5MB
CSV output will be saved to .\mft.csv
```
Para abrir el archivo 'mft.csv' es necesario la herramienta 'Timeline Explorer'.

<https://ericzimmerman.github.io/#!index.md>

Timeline Explorer es una herramienta diseñada para visualizar, filtrar y analizar líneas de tiempo durante investigaciones forenses digitales. Se suele utilizar para revisar grandes conjuntos de registros de eventos, registros del sistema de archivos y otros datos con marca de tiempo extraídos durante un análisis forense.

Timeline Explorer requiere '.Net'

<https://dotnet.microsoft.com/es-es/download/dotnet>

---
### **`Q1.`** **Simon Stark was targeted by attackers on February 13. He downloaded a ZIP file from a link received in an email. What was the name of the ZIP file he downloaded from the link?**

Filtro por '.zip'

![](/assets/img/htb-writeup-bft/bft1_1.png)

Y en la columna 'Parent Path' filtro por '.\Users\simon.stark\Downloads'.

Los resultados una vez filtrados muestran 3 archivos zip:
* Stage-20240213T093324Z-001.zip
* invoices.zip
* KAPE.zip

![](/assets/img/htb-writeup-bft/bft1_2.png)

Se identifica que 'Stage-20240213T093324Z-001.zip' es la descarga inicial debido a su referencia de ruta en otro archivo zip.

> **`A1.`** **Stage-20240213T093324Z-001.zip**

### **`Q2.`** **Examine the Zone Identifier contents for the initially downloaded ZIP file. This field reveals the HostUrl from where the file was downloaded, serving as a valuable Indicator of Compromise (IOC) in our investigation/analysis. What is the full Host URL from where this ZIP file was downloaded?**

Filtro por 'HostUrl='

![](/assets/img/htb-writeup-bft/bft1_3.png)

Esta es la URL del host desde donde se descargó este archivo. A partir de esto, podemos suponer que se utilizó Google Drive para alojar este archivo zip.

> **`A2.`** **https://storage.googleapis.com/drive-bulk-export-anonymous/20240213T093324.039Z/4133399871716478688/a40aecd0-1cf3-4f88-b55a-e188d5c1c04f/1/c277a8b4-afa9-4d34-b8ca-e1eb5e5f983c?authuser**

### **`Q3.`** **What is the full path and name of the malicious file that executed malicious code and connected to a C2 server?**

Para encontrar el archivo malicioso, utilizo la columna 'Parent Path' para filtrar cualquier referencia al directorio 'Stage'.

![](/assets/img/htb-writeup-bft/bft1_4.png)

El archivo 'invoice.bat' se destaca porque los archivos por lotes '.bat' se usan comúnmente para ejecutar comandos en sistemas Windows.

> **`A3.`** **C:\Users\simon.stark\Downloads\Stage-20240213T093324Z-001\Stage\invoice\invoices\invoice.bat**

### **`Q4.`** **Analyze the $Created0x30 timestamp for the previously identified file. When was this file created on disk?**

![](/assets/img/htb-writeup-bft/bft1_5.png)

> **`A4.`** **2024-02-13 16:38:39**

### **`Q5.`** **Finding the hex offset of an MFT record is beneficial in many investigative scenarios. Find the hex offset of the stager file from Question 3.⁵**

![](/assets/img/htb-writeup-bft/bft1_5.png)

El número de entrada MFT es 23436.

Como cada entrada MFT ocupa 1024 bytes, multiplico este número por 1024 para calcular el 'offset' en bytes.

```math
23436 * 1024 = 23998464
```

![](/assets/img/htb-writeup-bft/bft1_6.png)

> **`A5.`** **16E3000**

### **`Q6.`** **Each MFT record is 1024 bytes in size. If a file on disk has smaller size than 1024 bytes, they can be stored directly on MFT File itself. These are called MFT Resident files. During Windows File system Investigation, its crucial to look for any malicious/suspicious files that may be resident in MFT. This way we can find contents of malicious files/scripts. Find the contents of The malicious stager identified in Question3 and answer with the C2 IP and port.**

Abro el archivo $MFT con el programa 'Windows Hex Editor'.

<https://mh-nexus.de/en/hxd/>

![](/assets/img/htb-writeup-bft/bft1_8.png)

![](/assets/img/htb-writeup-bft/bft1_9.png)

Al analizar el script de PowerShell incrustado en el archivo '.bat', puedo extraer la dirección IP y puerto que utiliza el malware para comunicarse con su servidor de comando y control.

> **`A6.`** **43.204.110.203:6666**

---
### Timeline

| Time (UTC) | Description                 | Reference |
| :--------- | :-------------------------- | :-------: |
| 16:34:40   | Malicious Zip downloaded    | $MFT      |
| 16:35:15   | Initial Zip begin unzipping | $MFT      |
| 16:38:39   | invoice.bat unzipped        | $MFT      |


> <a href="https://labs.hackthebox.com/achievement/sherlock/1521382/633" target="_blank">***Litio7 has successfully solved Bft from Hack The Box***</a>
{: .prompt-info style="text-align:center" }
{: .prompt-tip }
