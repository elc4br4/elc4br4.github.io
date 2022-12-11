---
layout      : post
title       : "Valentine - HackTheBox"
author      : elc4br4
image       : assets/images/HTB/Valentine-HackTheBox/valentine.webp
category    : [ htb ]
tags        : [ Linux ]
---

❤️En esta máquina Linux de nivel easy explotaremos una vulnerabilidad conocida como Heartbleed y acabremos escalando privilegios a través de tmux❤️.

🎥Canal Writeups Youtube🎬 --> [https://www.youtube.com/channel/UCllewdxU0OQudNp9-1IVJYQ](https://www.youtube.com/channel/UCllewdxU0OQudNp9-1IVJYQ)

![](/assets/images/HTB/Valentine-HackTheBox/valentine2.webp)

![](/assets/images/HTB/Valentine-HackTheBox/valentine-rating.webp)

[![HTBadge](https://www.hackthebox.eu/badge/image/533771)](https://www.hackthebox.com/home/users/profile/533771)


***


**Un pequeño INDICE**

1. [Reconocimiento](#reconocimiento).
    * [Reconocimiento de Puertos](#recon-nmap).
2. [Puerto 443](#puerto443)
3. [Explotación](#explotación).
4. [Escalada de Privilegios](#privesc).


***

# Reconocimiento [#](reconocimiento) {#reconocimiento}

Como de costumbre comienzo con el reconocimiento, pero antes de lanzar la utilidad nmap, lanzo el <span style="color:red"> script Whichsystem.py </span> que me sirve para identificar el sistema operativo de la máquina a la que me voy a enfrentar.

Esta heramienta creada por s4vitar se basa en el ttl (time to live) para identificar si es una máquina que corre un sistema operativo Linux o Windows.

| TTL    | Sistema Operativo | 
| :----- | :------- | 
| 64     | Linux    | 
| 128    | Windows  |

![](/assets/images/HTB/Valentine-HackTheBox/whichsystem.webp)

## Reconocimiento de Puertos [🔍](#recon-nmap) {#recon-nmap}

Una vez que ya se que me enfrento a una máquina Linux procedo a lanzar un escaneo simple de puertos con el fin de detectar los puertos abiertos en la máquina víctima.

Este escaneo de puertos lo realizo con la herramienta nmap.

![](/assets/images/HTB/Valentine-HackTheBox/nmap1.webp)

Y como se puede ver solo hay dos puertos abiertos, el puerto 80, el puerto 443 y el puerto 22.

Pero necesito algo más de información acerca de estos puertos, por lo que lanzo un escaneo un poco más avanzado a estos puertos abiertos encontrados.

![](/assets/images/HTB/Valentine-HackTheBox/nmap2.webp)

> Encuentro el host valentine.htb que añado al archivo /etc/hosts

# Puerto 443 [🔢](#puerto443) {#puerto443}

Accedo al servidor web para ver que tenemos

![](/assets/images/HTB/Valentine-HackTheBox/web1.webp)

A simple vista no hay nada más que una imágen, por lo que decido fuzzear a ver que encuentro.

![](/assets/images/HTB/Valentine-HackTheBox/fuzz.webp)

Existen varias rutas, en la ruta dev encontré un texto codificado en hexadecimal que al decodificarlo se convertía en una clave rsa privada, pero al intentar usarla se me pedía la passphrase de la clave rsa.

Por lo que sigo buscando por otro lado y al quedarme sin ideas ni ningún hilo del que tirar, decidí mirar la imágen, a ver si hay algo oculto en ella.

Lo primero que hice es una búsqueda inversa usando la imágen.

![](/assets/images/HTB/Valentine-HackTheBox/imagen.webp)

Veo que se repite mucho la palabra heartbleed, que es el nombre de la imágen, por lo que decidí ponerme a buscar algo de información al respecto, encontrando lo siguiente:

![](/assets/images/HTB/Valentine-HackTheBox/heartbleed.webp)

# Explotación [💣](#explotación) {#explotación}

Es una vulnerabilidad existente en OpenSSL, por lo que buscando encuentro un módulo de metasploit, asique lo pruebo.

![](/assets/images/HTB/Valentine-HackTheBox/msf1.webp)

Al ejecutarlo me arroja cierta información pero un poco más abajo me filtra información un poco más valiosa, entre la cual encuentro una cadena que podría estar codificada en base64.

![](/assets/images/HTB/Valentine-HackTheBox/msf2.webp)

Decodifico la cadena y resulta ser una contraseña.

![](/assets/images/HTB/Valentine-HackTheBox/password.webp)

Ahora vuelvo hacia atrás, cuando encontré aquella clave privada.

![](/assets/images/HTB/Valentine-HackTheBox/dev.webp)

Paso la cadena de caracteres de hexadecimal a texto y ahí tenemos la clave.

![](/assets/images/HTB/Valentine-HackTheBox/clave.webp)

Pero aún existe un problema, la clave está encriptada y debemos desencriptarla para poder usarla con la contraseña que hemos encontrado.

Asique me copio la clave encriptada y la meto un archivo para proceder a desencriptarla.

![](/assets/images/HTB/Valentine-HackTheBox/dec1.webp)

Y como vemos ya está desencriptada y puedo usarla para conectarme a través de ssh con el usuario hype.

![](/assets/images/HTB/Valentine-HackTheBox/ssh.webp)


# Escalada de Privilegios [💊](#privesc) {#privesc}

Ahora como siempre toca escalar privilegios, y antes de nada voy a mirar el archivo /etc/passw en busca de algún otro usuario al que deba migrarme antes de escalar a root.

![](/assets/images/HTB/Valentine-HackTheBox/passwd.webp)

Pero en esta máquina solo tenemos el usuario hype y root, por lo que he de migrarme directamente al usuario root.

Buscando en el directorio de inicio del usuario hype veo que puedo leer el archivo .bash_history para ver el historial de comandos ejecutados, por lo que voy a ver que hay.

Antes de nada he de añadir que también puedo ver un archivo de configuración de tmux, por lo que podría ser una pista también y en caso de poder ejecutar tmux podríamos intentar algo.

![](/assets/images/HTB/Valentine-HackTheBox/bash.webp)

Y como se puede ver, se ha ejecutado recientemente el comando `tmux -S /.devs/dev_sess` que sirve para hacer un attach hacia una sesión ya creada.

Por lo que decido replicar el mismo comando en la máquina para ver que pasa...

![](/assets/images/HTB/Valentine-HackTheBox/tmux.webp)

Pero me arroja un error, aunque es fácil de solucionar... usando el comando `export TERM=xterm`

Y vuelvo a lanzar el comando

![](/assets/images/HTB/Valentine-HackTheBox/root.webp)

Y ya soy root y puedo leer las flag de user y root.




