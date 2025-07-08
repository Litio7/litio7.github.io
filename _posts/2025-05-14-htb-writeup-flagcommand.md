---
title: Flag Command
description: Embark on the "Dimensional Escape Quest" where you wake up in a mysterious forest maze that's not quite of this world. Navigate singing squirrels, mischievous nymphs, and grumpy wizards in a whimsical labyrinth that may lead to otherworldly surprises. Will you conquer the enchanted maze or find yourself lost in a different dimension of magical challenges? The journey unfolds in this mystical escape!
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
  - misconfigurations
  - api
  - devtools
  - web analysis
  - misconfiguration exploitation

---
## Web Analysis

![](assets/img/htb-writeup-flagcommand/flagcommand1.png)

El sitio web presenta una interfaz de juego con múltiples caminos narrativos, donde cada conjunto de opciones conduce a diferentes finales.

![](assets/img/htb-writeup-flagcommand/flagcommand2.png)

---
## Misconfiguration Exploitation

Desde la pestaña Network en DevTools, identifico un endpoint `/api/options` que responde con datos en formato JSON.

![](assets/img/htb-writeup-flagcommand/flagcommand3.png)

Inspeccionando su contenido, se listan todas las opciones posibles del juego, incluyendo una cadena secreta no visible en la interfaz principal.

![](assets/img/htb-writeup-flagcommand/flagcommand4.png)

Al reiniciar el juego y enviar el valor oculto `Blip-blop, in a pickle with a hiccup! Shmiggity-shmack`, se revela la flag del reto.

![](assets/img/htb-writeup-flagcommand/flagcommand5.png)

> <a href="https://www.hackthebox.com/achievement/challenge/1521382/646" target="_blank">Flag Command Challenge from Hack The Box has been Pwned</a>
{: .prompt-tip }