---
layout      : post
title       : "Shoppy - HackTheBox"
author      : elc4br4
image       : assets/images/HTB/Shoppy-HackTheBox/Shoppy.webp
category    : [ htb ]
tags        : [ Linux ]
---

🤖En esta máquina Linux de nivel Easy tendremos un login que debemos bypassear con una inyección sql, una vez dentro buscaremos usuarios válidos para conseguir su hash y poder loguearnos en otro panel web dentro de un vhost que debemos descubrir para conseguir credenciales ssh. Finalmente .

🎥Canal Writeups Youtube🎬 --> [https://www.youtube.com/channel/UCllewdxU0OQudNp9-1IVJYQ](https://www.youtube.com/channel/UCllewdxU0OQudNp9-1IVJYQ)

![](/assets/images/HTB/Shoppy-HackTheBox/shoppy2.webp)

![](/assets/images/HTB/Shoppy-HackTheBox/shoppy-rating.webp)

[![HTBadge](https://www.hackthebox.eu/badge/image/533771)](https://www.hackthebox.com/home/users/profile/533771)


...


**Un pequeño INDICE**

1. [Reconocimiento](#reconocimiento).
    * [Reconocimiento de Puertos](#recon-nmap).
2. [Enumeración](#enumeración).
    * [Enumeración Web](#enum-web).
3. [Explotación](#explotacion).    
    * [Burpsuite](#burpsuite).
    * [Hash Cracking](#cracking).
    * [VHOST Mattermost](#vhost).
4. [SSH](#ssh)
    * [ssh jaeger](#jaeger).
    * [ssh deploy](#deploy).
5. [Escalada de Privilegios](#privesc). 
 
...

# Reconocimiento [#](reconocimiento) {#reconocimiento}

***

## Reconocimiento de Puertos [🔍](#recon-nmap) {#recon-nmap}

Como siempre comenzamos con un reconocimiento de puertos a través de la herramienta nmap.

![](/assets/images/HTB/Shoppy-HackTheBox/nmap.webp)

Como vemos tenemos los puertos 22 y 80.

| Puerto | Servicio | Versión |
| :----- | :------- | :------ |
| 22     | SSH      | OpenSSH 8.4p1 |
| 80     | HTTP     | nginx 1.23.1 |

* Además encuentro el dominio `shoppy.htb`

Una vez tenemos estos datos, procedo a comprobar que tenemos en el servidor web (Puerto 80).

# Enumeración [#](enumeración) {#enumeración}

***

## Enumeración Web [🔍](#enum-web) {#enum-web}

Añado el dominio al archivo hosts y accedo al mismo a través del navegador.

![](/assets/images/HTB/Shoppy-HackTheBox/web1.webp)

Buscando a simple vista no veo nada que me pueda servir, tampoco veo nada en el código fuente, por lo que procedo a fuzzear rutas en el servidor web con la herramienta feroxbuster.

![](/assets/images/HTB/Shoppy-HackTheBox/feroxbuster.webp)

El fuzzeo me reporta la ruta login, por lo que voy a ver que puedo hacer.

![](/assets/images/HTB/Shoppy-HackTheBox/login.webp)

# Explotación [#](explotación) {#explotación}

***

## Burpsuite [🔍](#burpsuite) {#burpsuite}

Intento loguearme con credenciales aleatorias y capturar la petición con Burpsuite para probar diferentes cosas.

* Capturo la petición

![](/assets/images/HTB/Shoppy-HackTheBox/burp1.webp)

La envío al repeater y pruebo a editar el campo del usuario metiendo una comilla, al darle a enviar se queda cargando sin recibir respuesta, lo que me hace pensar que es vulnerable a sqli.

Asique pruebo a meter diferentes inyecciones hasta que doy con la adecuada y puedo saltarme el login.

![](/assets/images/HTB/Shoppy-HackTheBox/sqli.webp)

Y conseguimos bypassear el login sin conocer la contraseña.

Una vez estamos dentro vemos que solo podemos buscar por usuarios..., por lo que podría fuzzear en busca de usuarios válidos, pero esta vez lo haré a través de burpsuite usando el intruder y una lista de usuarios.

Capturamos la petición al insertar algo en el campo de búsqueda.

![](/assets/images/HTB/Shoppy-HackTheBox/search.webp)

![](/assets/images/HTB/Shoppy-HackTheBox/burp2.webp)

Lo enviamos al intruder, seleccionamos como tipo de ataque Sniper y en el aprtado de payloads añadimos una lista de usuarios y lanzamos el ataque.

Una vez terminal el ataque debemos fijarnos en la longitud de la respuesta, en este caso si el usaurio no existe tendremos una longitud de 2763 y si es exitoso tendremos 2922.

![](/assets/images/HTB/Shoppy-HackTheBox/burp3.webp)

Una vez veo que existe el usuario josh existe lo busco desde el panel de búsqueda de la web y descargo el archivo export.

![](/assets/images/HTB/Shoppy-HackTheBox/export.webp)

Lo abro y tenemos lo siguiente:

![](/assets/images/HTB/Shoppy-HackTheBox/creds-export.webp)


## Hash Cracking [🔍](#cracking) {#cracking}

Parece el hash del usuario josh... asique vamos a intentar crackearlo.

![](/assets/images/HTB/Shoppy-HackTheBox/hashcat.webp)

## VHOST Mattermost [🔍](#vhost) {#vhost}

Ya tengo el hash de josh crackeado, por lo que podría intentar loguearme a través de ssh.

![](/assets/images/HTB/Shoppy-HackTheBox/ssh-fail.webp)

Pero no ha habido suerte, por lo que pienso que he de haber algo más en algún vhost... asique procedo a fuzzear en busca de un vhost.

![](/assets/images/HTB/Shoppy-HackTheBox/vhost.webp)

> vhost `mattermost.shoppy.htb`.

Lo añado al archivo hosts y accedo al mismo para ver que hay.

Al acceder veo que hay un login donde podría probar con las credenciales de josh...

![](/assets/images/HTB/Shoppy-HackTheBox/web2.webp)

Y consigo acceder a un panel que parece que sirve para crear canales de comunicación... asique investigo un poco... y encuentro esto.

![](/assets/images/HTB/Shoppy-HackTheBox/web-ssh.webp)

# SSH [#](ssh) {#ssh}

## ssh jaeger [🔍](#jaeger) {#jaeger}

Con estas credenciales consigo loguearme a través de ssh como el usuario jaeger.

![](/assets/images/HTB/Shoppy-HackTheBox/ssh1.webp)

Una vez me he logueado intento leer la flag de usuario.

![](/assets/images/HTB/Shoppy-HackTheBox/user.webp)

A continuación lo último que toca es escalar priviegios para poder leer la flag root.

Pero antes de migrarme al usuario root he de migrarme al usaurio deploy.

![](/assets/images/HTB/Shoppy-HackTheBox/privesc1.webp)

Podemos ejecutar como root el script password-manager...

Pruebo a ejecutarlo y me pide una contraseña.

![](/assets/images/HTB/Shoppy-HackTheBox/password-manager.webp)

Lanzo un cat al script para intentar buscar la contraseña.

![](/assets/images/HTB/Shoppy-HackTheBox/pass.webp)

La contraseña parece ser `Sample` asique voy a probar.

![](/assets/images/HTB/Shoppy-HackTheBox/creds.webp)

## ssh deploy [🔍](#deploy) {#deploy}

Y ya tenemos credenciales del usuario deploy, por lo que a través de ssh puedo conectarme.

* username : deploy
* password : Deploying@pp!

![](/assets/images/HTB/Shoppy-HackTheBox/ssh2.webp)

Ahora ya si que si toca migrar al usuario root.

# Escalada de Privilegios [#](privesc) {#privesc}

Si nos fijamos bien podemos ver que existe el grupo docker y estamos dentro del mismo...

Podría intentar escalar privilegios con docker.

Buscando encuentro la forma de escalar privielgios abusando de docker.

[https://gtfobins.github.io/gtfobins/docker/](https://gtfobins.github.io/gtfobins/docker/)

![](/assets/images/HTB/Shoppy-HackTheBox/gtfo.webp)

Lanzamos el comando...

![](/assets/images/HTB/Shoppy-HackTheBox/root.webp)

Y ya nos hemos convertido en el usuario root :)

![](/assets/images/HTB/Shoppy-HackTheBox/share.webp)

















