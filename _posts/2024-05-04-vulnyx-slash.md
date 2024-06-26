---
title: Vulnyx - Slash
author: estx
date: 2024-05-04 12:10:00 +0800
categories: [Writeup, Vulnyx, Easy]
tags: [Linux, 'File Upload', 'Binary Abusing', 'Nginx']
math: true
mermaid: true
image:
  path: /assets/images/Vulnyx/Easy/Air/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Hook Machine Logo
---

Máquina Linux de nivel **Easy** de Vulnyx.

Técnicas usadas: **File Upload, Binary Abusing**

### Fase de Reconocimiento 🧣

Como primer punto, identificamos la dirección IP de la **`Máquina Vulnyx`**

```bash
root@kali> arp-scan -I eth0 --local --ignoredups

<SNIP>
192.168.100.72	08:00:27:c6:4f:1d	PCS Systemtechnik GmbH
```

a. Enumeramos los puertos que están abiertos.

```bash
❯ nmap -p- -sS --min-rate 5000 -Pn -n 192.168.100.72 -oG ports

PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
8080/tcp open  http-proxy
```

b. Vemos las versiones de los servicios que se están ejecutando

```bash
❯ nmap -p22,80,8080 -sCV 192.168.100.72 -oN versions

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.2p1 Debian 2+deb12u1 (protocol 2.0)
| ssh-hostkey: 
|   256 0e:95:f2:88:f3:0f:ca:38:ec:da:3c:c0:cd:19:20:41 (ECDSA)
|_  256 53:21:e1:34:a6:f0:70:2b:87:e7:cf:3d:6b:85:9d:64 (ED25519)
80/tcp   open  http    nginx 1.22.1
|_http-title: Welcome to nginx!
|_http-server-header: nginx/1.22.1
8080/tcp open  http    nginx 1.22.1
|_http-server-header: nginx/1.22.1
|_http-title: Did not follow redirect to http://air.nyx:8080/
|_http-open-proxy: Proxy might be redirecting requests
```