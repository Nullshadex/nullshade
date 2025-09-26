---
layout: writeup
title: "HTB - Scrambled"
date: 2022-07-8 
difficulty: "Medium"
platform: "Windows"
description: "Writeup completo de la máquina Scrambled..."
image: "/assets/images/Scrambled.png"
---


  Hola, hoy vamos a tocar una máquina Windows de dificultad Medium en Hack the Box, explotáremos básicamente Active Directory y deserialización.

**********************
# Enumeración

Como siempre usaremos nmap y descubrimos los siguientes puertos:

![Nmap]({{site.baseurl}}/assets/images/Scrambled/2022-10-03_164755.png)

Hacemos una enumeración con mas profundidad con nmap y vemos un dominio y un nombre de dc. Scrm.local y DC1.scrm.local. Los metemos en el etc/hosts.


![Nmap]({{site.baseurl}}/assets/images/Scrambled/2022-10-03_171633.png)

Echamos un vistazo a la pagina web y no dice que tiene el NTLM desabilidado por problemas de seguridad, tambien podemos ver un leak de nombre de usuario ksimpson

![Leak]({{site.baseurl}}/assets/images/Scrambled/2022-10-04_191008.png)

Seguimos mirando y vemos que existe una opcion de debuging para envio de problemas.

![Debug]({{site.baseurl}}/assets/images/Scrambled/2022-10-04_190924.png)

Vemos que se envía al dc1.scrm.local por el puerto 4411.

******************
# User Enum y Kerberoasting

Vamos a intentar enumerar mas usuarios con userenum de kerbrute

![Userenum]({{site.baseurl}}/assets/images/Scrambled/2022-10-03_163728.png)

Nos aparecen unos cuantos nombres y entre ellos ksimpson, vamos a intentar hacer un password spraying para intentar adivinar la contraseña, antes de lanzar rockyou o algún diccionario de contraseñas vamso a usar los mismos nombres de usuario como contraseña, si no probaríamos con cewl.

![spraying]({{site.baseurl}}/assets/images/Scrambled/2022-10-03_163827.png)

Y bingo! como no podia ser de otra manera ksimpson esta usando como contraseña su propio nombre.

Probamos crackmapexec smb y winrm con ksimpson pero no nos da nada. Vamos a intentar conseguir un TGT (Ticket Granting Ticket). Informacion sobre TGT [aquí](https://doubleoctopus.com/security-wiki/authentication/ticket-granting-tickets/)

![TGT]({{site.baseurl}}/assets/images/Scrambled/2022-10-03_164859.png)

Nos genera un ticket y lo guarda en ksimpson.ccache

Metemos el ticket en KRB5CCNAME

>export KRB5CCNAME=ksimpson.ccache

Y lanzamos un kerberoasting attack:

![Kerberoasting]({{site.baseurl}}/assets/images/Scrambled/2022-10-03_171547.png)

Info sobre [Kerberos](https://medium.com/r3d-buck3t/attacking-service-accounts-with-kerberoasting-with-spns-de9894ca243f)

Esto nos reporta el hash del usuario sqlsvc, usamos john para intentar romperlo y podemos ver la contraseña.

![john]({{site.baseurl}}/assets/images/Scrambled/2022-10-03_171814.png)

Con Impacket-getPac vamos a enumerar el SID.

>impacket-getPac -targetUser ksimpson scrm.local/ksimpson:ksimpson

![SID]({{site.baseurl}}/assets/images/Scrambled/2022-10-03_172548.png)

Con [CodeBeauty](https://codebeautify.org/ntlm-hash-generator) codificamos la contraseña de sqlsvc y generamos un silver ticket.

![sql]({{site.baseurl}}/assets/images/Scrambled/2022-10-03_172835.png)

>export KRB5CCNAME=Administrator.ccache

Y nos conectamos a sql con imapcket-mssqlclient

>impacket-mssqlclient dc1.scrm.local -k -no-pass

Estamos dentro de SQL, ahora vamos a enumerar las bases de datos.

![sql2]({{site.baseurl}}/assets/images/Scrambled/2022-10-03_173034.png)

![sql3]({{site.baseurl}}/assets/images/Scrambled/2022-10-03_173119.png)

![sql4]({{site.baseurl}}/assets/images/Scrambled/2022-10-03_173302.png)

![sql5]({{site.baseurl}}/assets/images/Scrambled/2022-10-03_173438.png)

Ya tenemos otro usuario y contraseña aunque de momento no podemos hacer gran cosa. Vamos a intentar una ejecución remota de comandos para intentar colar una reverse shell.

Activamos el xp_cmdshell.

>enable_xp_cmdshell

Por otro lado descargamos netcat.exe para windows y creamos un servidor http para poder descargarlo.

>python3 -m http.server 80

Usamos curl para poder descargar netcat de nuestro equipo

![ncsql]({{site.baseurl}}/assets/images/Scrambled/2022-10-03_174353.png)

Por otro lado, nos ponemos en escucha con netcat

>rlwrap nc -lnvp 443

Y lanzamos la Shell:

![ncsql]({{site.baseurl}}/assets/images/Scrambled/2022-10-03_174158.png)

Ya estamos dentro como el usuario sqlsvc

![ncsql]({{site.baseurl}}/assets/images/Scrambled/2022-10-03_174442.png)

Con este usuario no podemos hacer gran cosa, pero tenemos la contraseña del usuario miscsvc, vamos a pivotar.

Nos ponemos en escucha con netcat y ejecutamos:

![misc]({{site.baseurl}}/assets/images/Scrambled/2022-10-03_175430.png)

----------------------------------
# Deserialización

Ya estamos comp el usuario miscsvc, después de buscar por el equipo, vemos unos archivos en c:\Shares\IT\Apps\Sales Order Client. ScrambleClient.exe y ScrambleLib.dll. Vamos a traerlos a nuestra maquina para observarlos mas de cerca, para ello podemos usar [powercat](https://github.com/besimorhino/powercat)

![Powercat]({{site.baseurl}}/assets/images/Scrambled/2022-10-03_191507.png)

Parece que es el binario que vimos en la web. Usamos el comando file y nos dice que es un binario .NET. Vamos a desensamblarlo e intentar ver el codigo fuente, esto lo podemos hacer pasando los binarios a un windows y usar [dnSpy](https://github.com/dnSpy/dnSpy) o [DotPeek](https://www.jetbrains.com/es-es/decompiler/) etc...

![uploadorder]({{site.baseurl}}/assets/images/Scrambled/2022-10-06_160627.png)

En el codigo fuente podemos ver una función que se llama UploadOrder que probablemente sea vulnerable a deserialización. En la maquina virtual de Windows descargamos el [Ysoserial](https://github.com/pwntester/ysoserial.net) y creamos una reverse shell serializada para windows.

![ysoserial]({{site.baseurl}}/assets/images/Scrambled/2022-10-06_160649.png)

la pasamos a nuestro equipo con linux y lanzamos un netcat a la máquina Scrambled.

>netcat 10.10.11.168 4411

La maquina nos responde con un input. Escribimos UPLOAD_ORDER; y nuestra shell serializada. por otro lado debemos estar en escucha con netcat para que nos llegue la shell.

![ysoserial]({{site.baseurl}}/assets/images/Scrambled/2022-10-03_200542.png)

Y bingo ya tenemos shell como Nt Authority System. Ahora solo queda copiar la root flag.

![root]({{site.baseurl}}/assets/images/Scrambled/2022-10-03_200605.png)

Espero que os haya gustado.

*************************























