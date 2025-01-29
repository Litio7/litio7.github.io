---
title: Path Traversal
description: La vulnerabilidad Path Traversal permite a un atacante manipular las rutas de acceso a los archivos en un servidor, lo que le da la capacidad de leer archivos arbitrarios almacenados en el sistema, fuera de los directorios previstos por la aplicación. En algunos casos, si la aplicación no valida correctamente las rutas, el atacante podría incluso escribir en archivos del servidor. Esto podría permitirle modificar datos sensibles, alterar el comportamiento de la aplicación o, en escenarios más graves, obtener acceso total al servidor y comprometer la seguridad del sistema.
date: 2025-01-28
toc: true
pin: false
image:
 path: /assets/img/ps-writeup-pathtraversal/pathtraversal_logo.png
categories:
  - Port Swigger
tags:
  - port swigger
  - path traversal
  - lfi

---

### Path Traversal

#### Lab 1: Simple Case

This lab contains a path traversal vulnerability in the display of product images.

To solve the lab, retrieve the contents of the /etc/passwd file. 

La vulnerabilidad se presenta en la visualización de imágenes de productos. La aplicación permite a los usuarios solicitar imágenes a través de un parámetro en la URL, pero no filtra ni restringe correctamente los accesos a otros archivos del sistema.

![](/assets/img/ps-writeup-pathtraversal/lab1_1.png)

Para analizar el comportamiento de la aplicación, intercepto una solicitud en Burp Suite y observo que las imágenes se cargan con la siguiente petición:

```http
GET /image?filename=26.jpg HTTP/2
```

![](/assets/img/ps-writeup-pathtraversal/lab1_3.png)

Para comprobar si existe la vulnerabilidad, modifico el parámetro `filename`, agregando una secuencia de `../` que me permite retroceder directorios y acceder a archivos del sistema. Si la aplicación es vulnerable, en la respuesta obtendré el contenido del archivo `/etc/passwd`.

```http
GET /image?filename=../../../../../../../../etc/passwd HTTP/2
```

![](/assets/img/ps-writeup-pathtraversal/lab1_4.png)
![](/assets/img/ps-writeup-pathtraversal/lab1_5.png)

<https://portswigger.net/web-security/file-path-traversal#reading-arbitrary-files-via-path-traversal>

#### Lab 2: Traversal Sequences Blocked With Absolute Path Bypass

This lab contains a path traversal vulnerability in the display of product images.

The application blocks traversal sequences but treats the supplied filename as being relative to a default working directory.

To solve the lab, retrieve the contents of the /etc/passwd file. 

![](/assets/img/ps-writeup-pathtraversal/lab2_1.png)
![](/assets/img/ps-writeup-pathtraversal/lab2_3.png)

En este caso, la aplicación bloquea las secuencias de `../` para evitar el acceso a archivos arbitrarios. Sin embargo, trata el archivo solicitado como relativo a un directorio predeterminado, lo que permite bypassear la protección utilizando rutas absolutas.

![](/assets/img/ps-writeup-pathtraversal/lab2_4.png)

En lugar de usar `../`, intento acceder directamente a un archivo del sistema usando su ruta absoluta.

```http
GET /image?filename=/etc/passwd HTTP/2
```

![](/assets/img/ps-writeup-pathtraversal/lab2_5.png)
![](/assets/img/ps-writeup-pathtraversal/lab2_6.png)

<https://portswigger.net/web-security/file-path-traversal#common-obstacles-to-exploiting-path-traversal-vulnerabilities>

#### Lab 3: Traversal Sequences Stripped Non-recursively

This lab contains a path traversal vulnerability in the display of product images.

The application strips path traversal sequences from the user-supplied filename before using it.

To solve the lab, retrieve the contents of the /etc/passwd file. 

![](/assets/img/ps-writeup-pathtraversal/lab3_1.png)
![](/assets/img/ps-writeup-pathtraversal/lab3_3.png)

En este laboratorio, la aplicación implementa un filtro que elimina cada ocurrencia de `../` en el parámetro `filename`. Sin embargo, este filtro no es recursivo, lo que significa que solo elimina la secuencia una vez por cada aparición.

Para evadir este filtro, utilizo una técnica en la que anido los caracteres de traversal dentro de una cadena más larga, engañando al sistema.

```http
GET /image?filename=....//....//....//....//....//....//....//etc/passwd HTTP/2
```

![](/assets/img/ps-writeup-pathtraversal/lab3_4.png)
![](/assets/img/ps-writeup-pathtraversal/lab3_5.png)

<https://portswigger.net/web-security/file-path-traversal#common-obstacles-to-exploiting-path-traversal-vulnerabilities>

#### Lab 4: Traversal Sequences Stripped With Superfluous URL-decode

This lab contains a path traversal vulnerability in the display of product images.

The application blocks input containing path traversal sequences. It then performs a URL-decode of the input before using it.

To solve the lab, retrieve the contents of the /etc/passwd file. 

![](/assets/img/ps-writeup-pathtraversal/lab4_1.png)
![](/assets/img/ps-writeup-pathtraversal/lab4_3.png)

La aplicación intenta bloquear las secuencias `../` para evitar que se acceda a archivos fuera del directorio permitido. Sin embargo, realiza una doble decodificación de URL, lo que permite evadir la restricción y acceder a archivos sensibles.

```http
GET /image?filename=../../../../../../etc/passwd HTTP/2
```

La solicitud es bloqueada porque la aplicación detecta `../` y la filtra.

![](/assets/img/ps-writeup-pathtraversal/lab4_4.png)

La aplicación realiza una doble decodificación de URL, lo que significa que una entrada codificada puede convertirse en una secuencia válida tras ser procesada dos veces.

Codifico `../` en su forma hexadecimal (`%2f` representa `/` en URL encoding).

![](/assets/img/ps-writeup-pathtraversal/lab4_5.png)

```http
GET /image?filename=..%2f..%2f..%2f..%2f..%2f..%2fetc/passwd HTTP/2
```

La aplicación decodifica `%2f` en `/`, reconstruye `../`, lo detecta y lo filtra.

![](/assets/img/ps-writeup-pathtraversal/lab4_6.png)

Aplico una segunda capa de codificación. Codifico el carácter `%` de `%2f`, obteniendo `%25`.

![](/assets/img/ps-writeup-pathtraversal/lab4_7.png)

```http
GET /image?filename=..%252f..%252f..%252f..%252f..%252f..%252fetc/passwd HTTP/2
```

Al enviar una secuencia doblemente codificada `%252f`, logro que el filtro no detecte la primera conversión y, en la segunda, la cadena vuelve a su estado original, permitiéndome acceder a `/etc/passwd`.

![](/assets/img/ps-writeup-pathtraversal/lab4_8.png)
![](/assets/img/ps-writeup-pathtraversal/lab4_9.png)

<https://portswigger.net/web-security/file-path-traversal#common-obstacles-to-exploiting-path-traversal-vulnerabilities>

#### Lab 5: Validation Of Start Of Path

This lab contains a path traversal vulnerability in the display of product images.

The application transmits the full file path via a request parameter, and validates that the supplied path starts with the expected folder.

To solve the lab, retrieve the contents of the /etc/passwd file. 

![](/assets/img/ps-writeup-pathtraversal/lab5_1.png)
![](/assets/img/ps-writeup-pathtraversal/lab5_3.png)

La aplicación transmite la ruta completa del archivo mediante un parámetro en la solicitud y valida que la ruta comience con el directorio esperado `/var/www/images/`. Sin embargo, no restringe el uso de secuencias de path traversal `../` después de esta validación.

Intento modificar el parámetro `filename` para salir del directorio `/var/www/images/` y acceder a `/etc/passwd` usando `../`. La aplicación solo verifica que la ruta inicie con `/var/www/images/`, pero no valida la ruta completa después de la validación inicial.

```http
GET /image?filename=/var/www/images/../../../etc/passwd HTTP/2
```

![](/assets/img/ps-writeup-pathtraversal/lab5_4.png)
![](/assets/img/ps-writeup-pathtraversal/lab5_5.png)

<https://portswigger.net/web-security/file-path-traversal#common-obstacles-to-exploiting-path-traversal-vulnerabilities>

#### Lab 6: Validation Of File Extension With Null Byte Bypass

This lab contains a path traversal vulnerability in the display of product images.

The application validates that the supplied filename ends with the expected file extension.

To solve the lab, retrieve the contents of the /etc/passwd file. 

![](/assets/img/ps-writeup-pathtraversal/lab6_1.png)
![](/assets/img/ps-writeup-pathtraversal/lab6_3.png)

En este laboratorio, la aplicación valida que el nombre de archivo proporcionado termine con una extensión de archivo esperada, por ejemplo, .jpg o .png. Sin embargo, la aplicación no maneja adecuadamente los bytes nulos `%00`, que pueden ser utilizados para terminar prematuramente la ruta del archivo antes de que la extensión sea validada.

![](/assets/img/ps-writeup-pathtraversal/lab6_4.png)

Modifico el parámetro `filename` para inyectar un byte nulo, de manera que la ruta del archivo se termine antes de la extensión `.jpg`. Esto permite incluir la ruta de un archivo sensible, como `/etc/passwd`.

```http
GET /image?filename=../../../../../../../etc/passwd%00.jpg HTTP/2
```

El byte nulo `%00` termina la cadena de caracteres en la ubicación donde se inserta, lo que hace que el servidor vea solo la ruta de archivo hasta ese punto. La validación de la extensión de archivo solo ocurre después del byte nulo, lo que significa que la aplicación ya no considera la extensión `.jpg` al procesar la ruta del archivo.

![](/assets/img/ps-writeup-pathtraversal/lab6_5.png)
![](/assets/img/ps-writeup-pathtraversal/lab6_6.png)

<https://portswigger.net/web-security/file-path-traversal#common-obstacles-to-exploiting-path-traversal-vulnerabilities>
