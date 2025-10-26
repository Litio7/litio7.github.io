---
title: Noxious
description: El IDS, nos alertó sobre un posible dispositivo no autorizado en la red interna de Active Directory. El sistema de detección de intrusiones también indicó señales de tráfico LLMNR, lo cual es inusual. Se sospecha que se produjo un ataque de envenenamiento de LLMNR. El tráfico LLMNR se dirigió hacia Forela-WKstn002, que tiene la dirección IP 172.17.79.136. Se le proporciona a usted, nuestro experto en análisis forense de redes, una captura de paquetes limitada del tiempo circundante. Dado que esto ocurrió en la VLAN de Active Directory, se sugiere que realicemos una búsqueda de amenazas de red teniendo en cuenta el vector de ataque de Active Directory, centrándonos específicamente en el envenenamiento de LLMNR.
date: 2024-08-02
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-noxious/noxious_logo.png
categories:
  - Hack_The_Box
  - Sherlocks
tags:
  - hack_the_box
  - soc

---
### Initial Analysis

```terminal
/home/kali/Documents/htb/sherlocks/noxious:-$ 7z -phacktheblue x noxious.zip
capture.pcap
```

---
### **`Q1.`** **Its suspected by the security team that there was a rogue device in Forela’s internal network running responder tool to perform an LLMNR Poisoning attack. Please find the malicious IP Address of the machine.**

LLMNR usa el puerto 'UDP 5355'. Dado que estoy tratando con un posible ataque de envenenamiento de LLMNR, utilizo Wireshark para filtrar este puerto.

![](/assets/img/htb-writeup-noxious/noxious1.png)

> **`A1.`** **172.17.79.135**

### Q2.What is the hostname of the rogue machine?

El dispositivo usó 'DHCP' para obtener una dirección IP asignada a sí mismo y mapearla a su nombre de host. Puedo buscar tráfico 'DHCP' relacionado con la IP revelada anteriormente. Por tanto, filtro por 'dhcp'.

![](/assets/img/htb-writeup-noxious/noxious2.png)

> **`A2.`** **kali**

### **`Q3.`** **Now we need to confirm whether the attacker captured the user’s hash and it is crackable!! What is the username whose hash was captured?**

Filtro por smb2. El primer conjunto de autenticación NTLM comienza en el paquete 9290 y termina en el 9293

![](/assets/img/htb-writeup-noxious/noxious3.png)

> **`A3.`** **john.deacon**

### **`Q4.`** **In NTLM traffic we can see that the victim credentials were relayed multiple times to the attacker’s machine. When were the hashes captured the First time?**

El tiempo debe estar definido en UTC

La configuracion correcta es: View → Time Display Format → UTC Date and Time of Day.

> **`A4.`** **2024-06-24 11:18:30**

### Q5.What was the typo made by the victim when navigating to the file share that caused his credentials to be leaked?

Al observar el tráfico LLMNR, la máquina del atacante respondió a una consulta "DCC01", lo que significa que la víctima escribió DCC01 en lugar de DC01, lo que provocó que el DNS fallara y la máquina recurriera al protocolo LLMNR para resolver la consulta.

![](/assets/img/htb-writeup-noxious/noxious1.png)

Allí es donde la máquina maliciosa del atacante respondió a la consulta haciéndose pasar por un controlador de dominio.

> **`A5.`** **DCC01**

### Q6.To get the actual credentials of the victim user we need to stitch together multiple values from the ntlm negotiation packets. What is the NTLM server challenge value?

El paquete '9291' contiene el 'NTLM server challenge value'

![](/assets/img/htb-writeup-noxious/noxious4.png)

> **`A6.`** **601019d191f054f1**

### **`Q7.`** **Now doing something similar find the NTProofStr value.**

En este caso utilizo el paquete '9292' para encontrar el 'NTProofStr value'.

![](/assets/img/htb-writeup-noxious/noxious5.png)

> **`A7.`** **c0cc803a6d9fb5a9082253a04dbd4cd4**

### **`Q8.`** **To test the password complexity, try recovering the password from the information found from packet capture. This is a crucial step as this way we can find whether the attacker was able to crack this and how quickly.**

Para construir el hash 'NTLMv2', necesito un ultimo valor.

El 'NTLMv2 Response value' se puede encontrar junto a 'NTProofStr value'.

![](/assets/img/htb-writeup-noxious/noxious5.png)

```
c0cc803a6d9fb5a9082253a04dbd4cd4010100000000000080e4d59406c6da01cc3dcfc0de9b5f2600000000020008004e0042004600590001001e00570049004e002d00360036004100530035004c003100470052005700540004003400570049004e002d00360036004100530035004c00310047005200570054002e004e004200460059002e004c004f00430041004c00030014004e004200460059002e004c004f00430041004c00050014004e004200460059002e004c004f00430041004c000700080080e4d59406c6da0106000400020000000800300030000000000000000000000000200000eb2ecbc5200a40b89ad5831abf821f4f20a2c7f352283a35600377e1f294f1c90a001000000000000000000000000000000000000900140063006900660073002f00440043004300300031000000000000000000
```

El formato de 'NTLMv2' es: User::Domain:ServerChallenge:NTProofStr:NTLMveResponse(without first 16 bytes/32 characters).

Por lo que el hash seria el siguiente.

```
john.deacon::FORELA:601019d191f054f1:c0cc803a6d9fb5a9082253a04dbd4cd4:010100000000000080e4d59406c6da01cc3dcfc0de9b5f2600000000020008004e0042004600590001001e00570049004e002d00360036004100530035004c003100470052005700540004003400570049004e002d00360036004100530035004c00310047005200570054002e004e004200460059002e004c004f00430041004c00030014004e004200460059002e004c004f00430041004c00050014004e004200460059002e004c004f00430041004c000700080080e4d59406c6da0106000400020000000800300030000000000000000000000000200000eb2ecbc5200a40b89ad5831abf821f4f20a2c7f352283a35600377e1f294f1c90a001000000000000000000000000000000000000900140063006900660073002f00440043004300300031000000000000000000
```

```terminal
/home/kali/Documents/htb/sherlocks/noxious:-$ echo 'john.deacon::FORELA:601019d191f054f1:c0cc803a6d9fb5a9082253a04dbd4cd4:010100000000000080e4d59406c6da01cc3dcfc0de9b5f2600000000020008004e0042004600590001001e00570049004e002d00360036004100530035004c003100470052005700540004003400570049004e002d00360036004100530035004c00310047005200570054002e004e004200460059002e004c004f00430041004c00030014004e004200460059002e004c004f00430041004c00050014004e004200460059002e004c004f00430041004c000700080080e4d59406c6da0106000400020000000800300030000000000000000000000000200000eb2ecbc5200a40b89ad5831abf821f4f20a2c7f352283a35600377e1f294f1c90a001000000000000000000000000000000000000900140063006900660073002f00440043004300300031000000000000000000' > hash.txt

/home/kali/Documents/htb/sherlocks/noxious:-$ hashcat --show hash.txt
5600 | NetNTLMv2 | Network Protocol

/home/kali/Documents/htb/sherlocks/noxious:-$ hashcat -a 0 -m 5600 hash.txt /usr/share/wordlists/rockyou.txt 
NotMyPassword0k?
```

> **`A8.`** **NotMyPassword0k?**

### **`Q9.`** **Just to get more context surrounding the incident, what is the actual file share that the victim was trying to navigate to?**

Revisando algunos paquetes mas, encuentro el paquete numero '10214' con informacion de un recurso compartido.

![](/assets/img/htb-writeup-noxious/noxious6.png)

Aquí la víctima se conecta al recurso compartido en el controlador de dominio.

> **`A9.`** **\\DC01\DC-Confidential**

> <a href="https://labs.hackthebox.com/achievement/sherlock/1521382/747" target="_blank">***Litio7 has successfully solved Noxious from Hack The Box***</a>
{: .prompt-info style="text-align:center" }
{: .prompt-tip }
