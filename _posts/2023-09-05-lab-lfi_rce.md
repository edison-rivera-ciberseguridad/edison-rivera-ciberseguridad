---
title: Laboratorio - LFI to RCE
author: cotes
date: 2023-06-24 17:51:00 +0800
categories: [Laboratorio, 'LFI to RCE']
tags: ['LFI to RCE']
math: true
mermaid: true
image:
  path: /assets/images/Laboratorios/lfi-rce/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: LFI to RCE Logo
---

Logramos un RCE (Remote Command Execution) al leer registros (logs) del servidor y modificando cabeceras en la petición (En estas podemos inyectar código)

Algunas rutas de archivos de **logs** son
+ [x] **Apache:** `/var/log/apache2/access.log`
+ [x] **SSH:** Debian y Ubuntu: `/var/log/auth.log` \ Red Hat y Centos `/var/log/btmp`
+ [x] **Otro archivos que contemplan User-Agent:** `/proc/self/environ` \ `/proc/self/fd/N`


## Explicación 🤯
Como de ante mano sabemos que podemos leer archivos del servidor (LFI), existen archivos de **registros**, en los cuales se almacenan las peticiones realizadas al servidor

![](/assets/images/Laboratorios/lfi-rce/log.png)

> Del anterior ejemplo de un **archivo de registro** la parte más importante es `[User-Agent]`, ya que es la única que podemos modificar en una petición que enviemos al servidor.
{: .prompt-info }

### **¿Cómo podemos cambiar el valor de User-Agent?**

Usaremos BurpSuite para cargar nuestro código **PHP** a ejecutar.

![](/assets/images/Laboratorios/lfi-rce/burp.png)

Y si ahora realizamos una petición con **cURL** veremos el output de nuestro comando ejecutado

![](/assets/images/Laboratorios/lfi-rce/command.png)

> Debemos tener mucho cuidado al momento de inyectar nuestro código PHP en la cabecera **`User-Agent`**, ya que, Si no escribimos la sintaxis correcta, el archivo **access.log** se corromperá y ya no será posible ejecutar comandos.
{: .prompt-warning }

## SSH: Session Poisoning 💉
Si la máquina que auditamos tiene el **puerto 22 (SSH) abierto** podemos tratar de ejecutar este ataque.

* Las rutas de los **registros** de SSH son: 
  * [x] **Debian y Ubuntu:** `/var/log/auth.log`
  * [x] **Red Hat y Centos:** `/var/log/btmp`

![](/assets/images/Laboratorios/lfi-rce/ssh.png)
> Archivo **`btmp`**

La inyección de código **PHP** la haremos en el **nombre del usuario**
```bash
ssh "<?php system('whoami'); ?>"@<Dirección IP>
```