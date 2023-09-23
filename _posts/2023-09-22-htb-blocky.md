---
title: HackTheBox - Blocky
author: cotes
date: 2023-09-22 17:51:00 +0800
categories: [Writeup, HackTheBox, Easy]
tags: [Linux, 'Information Leakage', 'Wordpress', 'Misallocated Privileges']
math: true
mermaid: true
image:
  path: /assets/images/HTB/Easy/Blocky/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Blocky Machine Logo
---

Máquina Linux de nivel **Easy** de HackTheBox.

Técnicas usadas: **Information Leakage, Wordpress, Misallocated Privileges**

### Fase de Reconocimiento 🧣

a. Enumeramos los puertos que están abiertos.

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

    ![](/assets/images/HTB/Easy/Blocky/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos abiertos

* **`nmap -p<Puertos> -sCV <IP> -oN versiones`**

    ![](/assets/images/HTB/Easy/Blocky/02-versions.png)


> Vemos un host `blocky.htb` el cual añadiremos al fichero /etc/hosts
{: .prompt-info }

c. El servicio web es un Wordpress

![](/assets/images/HTB/Easy/Blocky/03-web.png)

* Al investigar un poco la web encontramos un usuario `notch`

    ![](/assets/images/HTB/Easy/Blocky/06-user.png)


* Fuzzearemos por directorios o ficheros existentes en este servicio

    ```bash
    ❯ dirsearch -u http://blocky.htb/
     
    
    [10:07:00] 200 -  745B  - /plugins/
    [10:07:16] 301 -  311B  - /wp-admin
    [10:07:16] 301 -  314B  - /wp-includes
    <SNIP>
    ```

* Al visitar el directorio `/plugins` veremos dos archivos .jar los cuales descargaremos para inspeccionarlos

    ![](/assets/images/HTB/Easy/Blocky/04-jar.png)

d. Al abrir el fichero **Blocky-Core.jar** con `jd-gui` obtenemos credenciales para una base de datos

![](/assets/images/HTB/Easy/Blocky/05-leak.png)


* Nos conectamos por SSH con el usuario **notch** y la contraseña encontrada anteriormente

    ```bash
    ❯ ssh notch@10.10.10.37
    notch@10.10.10.37's password: 8YsqfCTnvxAUeduzjNSXe22
    ```

### Escalada de Privilegios 💹

a. Listamos nuestros privilegios a nivel de sudoers

```bash
notch@Blocky:~$ sudo -l

User notch may run the following commands on Blocky:
    (ALL : ALL) ALL
```

* Podemos ejecutar cualquier comando como cualquier usuario 🤡

    ```bash
    notch@Blocky:~$ sudo su
    root@Blocky:/home/notch#
    ```