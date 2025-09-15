---
layout: writeup
title: "HTB - Blackfield"
date: 2022-07-14
difficulty: "Hard"
platform: "Windows"
description: "Writeup completo de la máquina Blackfield..."
image: "/assets/images/Blackfield/blackfield.png"
---

  Hola, hoy vamos a tocar una máquina Windows de dificultad Hard en Hack the Box, explotáremos active directory y SeBackupPrivilege.

**********************
# Enumeración

En primer lugar realizamos una enumeración de la maquina con nmap y
descubrimos los siguientes puertos:

![Nmap]({{site.baseurl}}/assets/images/Blackfield/2022-07-04_201701.png)

Lanzamos crackmapexec y por un lado comprobamos el build de Windows 17763 (buscamos en Microsoft.com y vemos que pertenece a Windows server 2019)
y por otro vemos que el dominio es blackfield.local. Este ultimo lo metemos al etc/hosts.


![crackmapexec]({{site.baseurl}}/assets/images//Blackfield/2022-07-04_201727.png)


Tiene el puerto de smb abierto, vamos a ver si hay algo expuesto:
>smbmap -H 10.10.10.192 -u 'null'

y si, tiene información expuesta.

![smbmap]({{site.baseurl}}/assets/images/Blackfield/2022-07-04_201844.png)

Vemos unas cuantas cosas, después de enumerarlas vemos algo que nos llama la atención en la carpeta profiles$, unos cuantos nombres que pueden ser potenciales usuarios. Vamos a crear un diccionario con ellos.
**************************
#Active Directory

Metemos todo el contenido de la carpeta profiles$ en un archivo que llamaremos users.txt

>smbmap -H 10.10.10.192 -u 'null' -r 'profiles$' > users.txt \| awk 'NF{print$NF}

(el awk es para quedarnos con la ultima columna)

Arreglamos las primeras líneas con nano o vim ( o con el editor que queramos ;)) y ya tenemos nuestro archivo de usuarios,  
Bien, ya tenemos un listado potencial de usuarios, vamos a ver si hay algún usuario valido.(Información sobre kerbrute [aquí](https://infinitelogins.com/2020/11/16/enumerating-valid-active-directory-usernames-with-kerbrute/))

![Kerbrute]({{site.baseurl}}/assets/images/Blackfield/2022-07-04_204307.png)

vemos algunos usuarios validos, los metemos en otra lista valid_users.txt para ver si podemos hacer un Asreproast attack.
Información sobre asreproast [aquí](https://www.hackplayers.com/2020/11/asreproast-o-as-rep-roasting.html) y [aquí](https://book.hacktricks.xyz/windows-hardening/active-directory-methodology/asreproast)

Lanzamos GetNPUsers.py y bingo nos devuelve el hash de support

![Asreproast]({{site.baseurl}}/assets/images/Blackfield/2022-07-04_204842.png)

Copiamos el hash en un archivo y lo intentamos crackear con Johntheripper.

![John]({{site.baseurl}}/assets/images/Blackfield/2022-07-04_205010.png)

Bien, la contraseña esta en el rockyou y la hemos conseguido sin problema. Validamos la cuenta con crackmapexec y vemos que no tiene permisos de winrm.
Con ldapdomaindump, rpcclient, enum4linux etc.. podemos enumerar mas usuarios para ver cual es administrador o tiene permisos de winrm, pero de momento no podemos hacer mucho mas.

Vamos a seguir enumerando con bloodhound a ver si podemos hacer algo.
*******************************************
#BloodHound

Como tenemos usuario y contraseña vamos a usar [bloodhound-python](https://github.com/fox-it/BloodHound.py) con esta herramienta podemos hacer una enumeración de forma remota para luego verla en bloodhound.

>bloodhound-python -c all -u 'support' -p '#00^BlacKnight' -ns 10.10.10.192 -d blackfield.local

Una vez que tengamos la información la importamos en bloodhound y pasamos a analizarla.
después de mucho buscar vemos que tenemos ForceChangePassword con el usuario audit2020


![bloodhound]({{site.baseurl}}/assets/images/Blackfield/blood.png)

¡Podemos cambiarle la contraseña!, vamos a ver como hacerlo: [https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Active%20Directory%20Attack.md#forcechangepassword](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Active%20Directory%20Attack.md#forcechangepassword)
podemos hacerlo con rpc ya que tenemos un usuario valido vamos a ello:
>net rpc password Audit2020 -U 'support' -S 10.10.10.192

nos pide la nueva contraseña para audit2020, yo voy a poner password123$  y la contraseña del usuario support y listo, contraseña cambiada.

Volvemos a validar con crackmapexec y seguimos sin tener winrm, vamos a ver si en smb nos deja ver algo mas.. Y si, no deja ver la carpeta forensic, dentro de ella hay otra memory_analysis, esto tiene buena pinta.
Vemos que tiene un archivo [lsass]( https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-2000-server/cc961760(v=technet.10)?redirectedfrom=MSDN)

![smb]({{site.baseurl}}/assets/images/Blackfield/2022-07-05_173104.png)

Nos lo traemos a nuestra maquina:
>smbmap -H 10.10.10.192 -u 'audit2020' -p'password123$' --download forensic/memory_analysis/lsass.zip

lo descomprimimos y vemos un archivo lsass.DMP, pinta bien. Usamos pypykatz para ver el dumpeo.

>pypykatz lsa minidump lsass.dmp

![pypykatz]({{site.baseurl}}/assets/images/Blackfield/2022-07-05_173702.png)

vemos que tenemos el hash nt del usuario svc_backup y el hash nt podemos usarlo para hacer passthehash. Lo validamos y vemos que tenemos acceso winrm, por fin estamos dentro.

![evilwinrm]({{site.baseurl}}/assets/images/Blackfield/2022-07-05_174000.png)

ya podemos ver la flag de usuario.
**********************************
#Escalada

Necesitamos escalar privilegios para eso vamos a que privilegios tenemos y que podemos explotar.

![priv]({{site.baseurl}}/assets/images/Blackfield/2022-07-05_174603.png)

Como somos el usuario backup tenemos el privilegio [SeBackupPrivilege](https://www.hackingarticles.in/windows-privilege-escalation-sebackupprivilege/), podemos usarlo para escalar a Administrador.

Como lo que nos interesa es ser administrador del controlador de dominio vamos a intentar hacer una copia del ntds.dit. primero en nuestra maquina creamos un archivo  txt por ejemplo disk.txt

>nano disk.txt

que contenga lo siguiente:

>set context persistent nowriters 

>add volume c: alias nombre

>create

>expose %nombre% k:

Es muy importante que después de cada sentencia le metas un espacio al final ya que si no borrara la ultima letra.

Una vez que lo tengas lo subes a un directorio temp( desde evil winrm es muy fácil, con upload) y ejecutamos diskshadow.

![Diskshadow]({{site.baseurl}}/assets/images/Blackfield/2022-07-05_190600.png)

Nos vamos a la unidad z: o k: o donde lo tengáis copiado y probamos a copiarlo con copy, si no podemos usaremos robocopy:

![robocopy]({{site.baseurl}}/assets/images/Blackfield/2022-07-05_190902.png)


Ahora hacemos copia de la [sam](https://www.techtarget.com/searchenterprisedesktop/definition/Security-Accounts-Manager) 
 
![sam]({{site.baseurl}}/assets/images/Blackfield/2022-07-05_191454.png)

Ya lo tenemos todo, lo pasamos a nuestra maquina y con el [impacket-secretsdump](https://github.com/SecureAuthCorp/impacket/blob/master/examples/secretsdump.py) lo enumeramos.

![impacket]({{site.baseurl}}/assets/images/Blackfield/2022-07-05_192201.png)

ya podemos los ver los hashes y entre ellos el de administrador ;) Solo nos queda hacer passthehash con evil-winrm y listo.

![pass]({{site.baseurl}}/assets/images/Blackfield/2022-07-05_192243.png)

!Ya somos Admin! ya podemos ir a por la flag de root.

![root]({{site.baseurl}}/assets/images/Blackfield/2022-07-05_192323.png)

Eso es todo, una máquina muy divertida y útil para practicar Active directory.






















