---
title: Vulnhub - Ripper
author: cotes
date: 2023-10-11 17:51:00 +0800
categories: [Writeup, Vulnhub, Medium]
tags: [Linux, 'Information Leakage']
math: true
mermaid: true
image:
  path: /assets/images/Vulnhub/Medium/Ripper/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Ripper Machine Logo
---

Máquina Linux de nivel **Medium** de Vulnhub.

Técnicas usadas: **Information Leakage**. Esta máquina se base en la enumeración tanto del sistema web como una vez dentro del sistema.

### Fase de Reconocimiento 🧣

a. Enumeramos los puertos que están abiertos.

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

    ![](/assets/images/Vulnhub/Medium/Ripper/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos

* **`nmap -p22,80 -sCV <IP> -oN versiones`**

    ![](/assets/images/Vulnhub/Medium/Ripper/02-versions.png)

c. Al visitar **MiniServ** veremos un dominio el cual añadiremos al fichero /etc/hosts

![](/assets/images/Vulnhub/Medium/Ripper/03-host.png)

* Al fuzzer por archivos comunes (**robots.txt**) veremos una cadena encodeada en base64

    ![](/assets/images/Vulnhub/Medium/Ripper/04-string.png)

* Si la decodeamos obtendremos el siguiente mensaje: **we scan php codes with rips**

d. Al investigar por algo relacionado con **Rips** y php llegamos a que es un servicio que analiza código estático. Este servicio existe en la ruta `http://ripper-min/rips`

![](/assets/images/Vulnhub/Medium/Ripper/05-rip.png)

* Si aquí analizamos la ruta **/var/www** encontraremos un fichero con credenciales válidas para el servicio **SSH**

    ![](/assets/images/Vulnhub/Medium/Ripper/06-leak.png)


### Escalada de Privilegios 💹

a. Al ver nuestro fichero **.bash_history** encontramos un archivo 'secret.file' en el directorio `/mnt`

```bash
ripper@ripper-min:~$ cat .bash_history
<SNIP> 
cd /mnt/
cat secret.file
```

* El fichero '/mnt/secret.file' contiene la contraseña del usuario `cubes`

    ```bash
    ripper@ripper-min:/mnt$ ls /home/
    cubes  ripper
    ripper@ripper-min:/mnt$ cat secret.file 
    -passwd : Il00tpeople
    ripper@ripper-min:/mnt$ su cubes
    Password: Il00tpeople
    cubes@ripper-min:/mnt$
    ```

b. Ahora, vemos el fichero **.bash_history** del usuario 'cubes'

```bash
cubes@ripper-min:~$ cat .bash_history
cd /var/
cd webmin/
cd backup/
cat miniserv.log
```

* El fichero 'miniserv.log' contiene credenciales válidas para el servicio `Webmin`

    ```bash
    cubes@ripper-min:/var/webmin/backup$ cat miniser.log
    <SNIP>
    [04/Jun/2021:11:33:16 -0400] [10.0.0.154] Authentication : session_login.cgi=username=admin&pass=tokiohotel
    ```

c. Si nos autenticamos y ejecutamos comandos veremos que somos el usuario **root**

![](/assets/images/Vulnhub/Medium/Ripper/07-rce.png)

d. Para autenticarnos por SSH como el usuario root, primero generamos un par de claves SSH en nuestra máquina y copiamos la clave **id_rsa.pub** en el fichero 'authorized_keys' en la máquina Ripper

```bash
❯ ssh-keygen
```

![](/assets/images/Vulnhub/Medium/Ripper/08-ssh.png)

* Ahora ya podemos conectarnos como el usuario root con nuestra clave id_rsa privada

    ```bash
    ❯ ssh root@192.168.100.39 -i id_rsa
    root@ripper-min:~#
    ```