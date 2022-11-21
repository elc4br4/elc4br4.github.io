---
layout      : post
title       : "Squashed - HackTheBox"
author      : elc4br4
image       : assets/images/HTB/Squashed-HackTheBox/Squashed.webp
category    : [ htb ]
tags        : [ Linux ]
---

⚔️En esta máquina Linux de nivel easy tendremos que montar directorios compartidos nfs, obtener reverse shell a través de la subida de un archivo malicioso y escalaremos privilegios a través del fichero .Xauthority⚔️.

🎥Canal Writeups Youtube🎬 --> [https://www.youtube.com/channel/UCllewdxU0OQudNp9-1IVJYQ](https://www.youtube.com/channel/UCllewdxU0OQudNp9-1IVJYQ)

![](/assets/images/HTB/Squashed-HackTheBox/squashed2.webp)

![](/assets/images/HTB/Squashed-HackTheBox/squashed-rating.webp)

[![HTBadge](https://www.hackthebox.eu/badge/image/533771)](https://www.hackthebox.com/home/users/profile/533771)


***


**Un pequeño INDICE**

1. [Reconocimiento](#reconocimiento).
    * [Reconocimiento de Puertos](#recon-nmap).
2. [Enumeración](#enumeración).
    * [Puerto 80](#puerto80).
    * [Puerto 111](#puerto111).
4. [Escalada de Privilegios](#privesc). 
    * [.Xauthority](#Xauthority).

    
***

# Reconocimiento [#](reconocimiento) {#reconocimiento}



## Reconocimiento de Puertos [🔍](#recon-nmap) {#recon-nmap}

Como de costumbre comienzo con el reconocimiento, pero antes de lanzar la utilidad nmap, lanzo el <span style="color:red"> script Whichsystem.py </span> que me sirve para identificar el sistema operativo de la máquina a la que me voy a enfrentar.

Esta heramienta creada por s4vitar se basa en el ttl (time to live) para identificar si es una máquina que corre un sistema operativo Linux o Windows.

| TTL    | Sistema Operativo | 
| :----- | :------- | 
| 64     | Linux    | 
| 128    | Windows  |

![](/assets/images/HTB/Squashed-HackTheBox/whichsystem.webp)

En este caso me arroja un TTL de 63, por lo que al acercarse más a 64 que a 128 ya sé que es una máquina Linux.

Una vez que ya se que es una máquina Linux puedo proceder al <span style="color:red">reconocimiento de puertos</span>.

```bash
# Primer escaneo para sacar los puertos abiertos de la máquina
--------------------------------------------------------------
nmap -p- -Pn -n --min-rate 5000 10.10.11.191 --open -vvv
```

![](/assets/images/HTB/Squashed-HackTheBox/nmap1.webp)

```bash
# Segundo escaneo para sacar la versión de lo que se ejecuta en cada puerto y lanzamiento de una serie de scripts básicos de nmap contra dichos puertos.
--------------------------------------------------------------
nmap -p80,22,111 -sCV -n 10.10.11.191
```

![](/assets/images/HTB/Squashed-HackTheBox/nmap2.webp)

Por el momento tengo la siguiente información:

| Puerto | Servicio | Versión |
| :----- | :------- | :------ |
| 22     | ssh      | OpenSSH 8.2p1 Ubuntu|
| 80     | http     | Apache httpd 2.4.41 |
| 111    | rpcbind  | rpcbind 2-4 |

> En el pueeto 111 (rpcbind) hay varias comparticiones nfs.

Iré por partes, comenzando por la enumeración del puerto 80 en el que reside un servidor web.

# Enumeración [#](enumeración) {#enumeración}



## Puerto 8️⃣0️⃣ [🔢](#puerto80) {#puerto80}

Accedo a la url desde el navegador para ver que me encuentro.

![](/assets/images/HTB/Squashed-HackTheBox/web1.webp)

Aparentemente no hay nada útil a primera vista, en el código fuente tampoco.

Procedo a fuzzear para ver que rutas existen en el servidor...

![](/assets/images/HTB/Squashed-HackTheBox/fuzz.webp)

Pero no encuentro nada útil... Busqué vhosts pero tampoco había nada por lo que decido pasar al puerto 111.

## Puerto 1️⃣1️⃣1️⃣ [🔢](#puerto111) {#puerto111}

En el escaneo nmap encontré varias comparticiones o <span style="color:red">montajes nfs</span> que podría intentar montar en mi máquina.

Os dejo un recurso muy útil para que puedan entenderlo bien y explotarlo.

[https://hacktricks.boitatech.com.br/pentesting/nfs-service-pentesting](https://hacktricks.boitatech.com.br/pentesting/nfs-service-pentesting)

Existen varios scripts de nmap para enumerar nfs así como un módulo de mestaploit.

Primeramente he de averiguar que rutas puedo montar.

```bash
# Primero he de averiguar que carpetas tiene el servidor disponibles para montar
--------------------------------------------------------------------------------
> showmount -e 10.10.11.191
Export list for 10.10.11.191:
/home/ross *
/var/www/html *
```

Puedo montar esas dos rutas, por lo que creo dos directorios en /tmp para montar cada directorio de servidor en cada carpeta creada en /tmp.

```bash
# Creo los directorios
----------------------
mkdir /tmp/ross
mkdir /tmp/www
# Monto los directorios en cada carpeta
sudo mount 10.10.11.191:/home/ross /tmp/ross
sudo mount 10.10.11.191:/var/www/html /tmp/www
```

Una vez montados accedo al directorio home del usuario ross y echo un vistazo a ver que encuentro...

![](/assets/images/HTB/Squashed-HackTheBox/ross-nfs.webp)

Veo que existe un archivo llamado <span style="color:red">Passwords.kdbx</span> que es básicamente el archivo de la base de datos que usa keepass para almacenar las contraseñas.

Intento obtener el hash de la contraseña para posteriormente intentar crackearlo con john o hashcat.

```bash
# Con la herramienta keepas2john intento sacar el hash con el siguiente comando.
-------------------------------------------------------------------------------
keepass2john Passwords.kdbx > /home/elc4br4/HTB/Squashed/hashkdbx
```

Pero al intentarlo me arroja un error.

![](/assets/images/HTB/Squashed-HackTheBox/error.webp)

Tras investigar durante un buen rato llego a la conclusión de que es un <span style="color:red">rabbit hole</span> y que he de ir por otro camino...

Aún tengo el directorio /var/www/html por investigar, asique una vez montado intento acceder pero no tengo permisos.

![](/assets/images/HTB/Squashed-HackTheBox/www-permisos.webp)

![](/assets/images/HTB/Squashed-HackTheBox/www-permisos2.webp)

Como vemos solo tiene acceso al directorio el usuario www-data y su uid es 2017, por lo que podría intentar algo...

Crearé un usuario en mi máquina llamado www-data con el uid 2017.

![](/assets/images/HTB/Squashed-HackTheBox/www-data.webp)

![](/assets/images/HTB/Squashed-HackTheBox/www-data2.webp)

Una vez creado ya consigo acceder al directorio.

Recordemos que estoy dentro de la ruta del servidor web, por lo que podría crearme una reverse shell y subirla al propio servidor.

De la siguiente forma:

1. Primero creo la reverse shell y la modifico con mi ip y el puerto (443 en mi caso).

Usaré la reverse shell de pentest monkey.

![](/assets/images/HTB/Squashed-HackTheBox/rev.webp)

2. La subo a la carpeta compartida descargándola con wget tras haber iniciado un servidor python3 en mi máquina local.

```bash
# Servidor python3 
------------------
python3 -m http.server 8081
```

![](/assets/images/HTB/Squashed-HackTheBox/rev2.webp)

3. Pongo un oyente de netcat en escucha y ejecuto la reverse shell desde el navegador.

![](/assets/images/HTB/Squashed-HackTheBox/alex.webp)

Y ya tengo acceso al sistema como el usuario alex.

Pero antes de continuar actualizo la tty para tener una shell estable e interactiva.

```bash
> script /dev/null -c bash
--------------------------
> Ctrl+z
--------------------------
> stty raw -echo; fg 
        reset
    terminal type? xterm
--------------------------
export TERM=xterm
export SHELL=bash
```

Ahora ya tengo una shell más estable e interactiva.

# Escalada de Privilegios [#](privesc) {#privesc}



## .Xauthority [☣️](#Xauthority) {#Xauthority}

Toca escalar privilegios, y para ello en primer lugar lanzo el script linpeas.sh

Peeeerooo no encuentro nada útil, asique recuerdo que en el directorio que montamos anteriormente /home/ross había archivos que no podíamos leer, por lo que podría realizar lo mismo que anteriormente para poder leerlos.

Creo el usuario ross y le asigno el uid 1001.

![](/assets/images/HTB/Squashed-HackTheBox/ross.webp)

Ahora veo que puedo leer aquellos archivos que antes no podía y veo que un posible vector de escalada puede ser a través de .Xauthority.

Buscando información al respecto encuentro lo siguiente.

[https://blog.stderror.net/post/hijack-a-xsession/](https://blog.stderror.net/post/hijack-a-xsession/)

Para <span style="color:red">conseguir la escalada de privilegios sigo estos pasos</span>:

1. Primero paso el archivo .Xauthority a la sesión del usuario alex (la de la reverse shell)

2. A continuación ejecuto los siguientes comandos en la sesión de alex (la de la reverse shell).

```bash
XAUTHORITY=/tmp/.Xauthority
---------------------------
export XAUTHORITY
---------------------------
# Realizo la captura de pantalla de la sesión
xwd -root -screen -silent -display :0 -out /tmp/captura.xwd
```

3. Pasamos el archivo screen.xwd a la máquina atacante a través de un servidor http.server de python

4. Abrimos la captura.

```bash
xwud -in captura.xwd
```
Y ya tenemos la password del usuario root para loguearnos por ssh.

![](/assets/images/HTB/Squashed-HackTheBox/ssh-pass.webp)

Me logueo y ya puedo leer la flag de usuario root.

![](/assets/images/HTB/Squashed-HackTheBox/root.webp)

![](/assets/images/HTB/Squashed-HackTheBox/share.webp)








