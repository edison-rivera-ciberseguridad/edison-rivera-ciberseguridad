---
title: Laboratorio - Docker
author: cotes
date: 2023-09-07 17:51:00 +0800
categories: [Laboratorio, Docker]
tags: [Docker]
math: true
mermaid: true
image:
  path: /assets/images/Laboratorios/basic-docker/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Docker Machine Logo
---

Docker es una plataforma que nos sirve para empaquetar código (Contenedores). A nosotros como pentesters nos sirve para crear laboratorios, prácticar y explotar vulnerabilidades conocidas, de forma muy rápida y sencilla

## Cheatsheet 🔬

|   **Comando**                | **Descripción**      |
|:----------------------------:|:-----------------|
| `docker images`              | Listar imágenes de Docker |
| `docker ps`                  | Listar contenedores en ejecución |
| `docker volume ls`           | Listar volúmenes creados |
| `docker stop [Container ID]` | Detener la ejecución de un contenedor |
| `docker network ls`          | Ver interfaces de red creadas |
| `docker run -dit [Image ID]` | Ejecutamos un contenedor |
| `-p [Puerto Local]:[Puerto Contenedor]`     | Port Forwarding |
| `-v [Directorio Local]:[Puerto Contenedor]` | Usando Monturas |
| `docker exec -dit [Container ID] bash`      | Accedemos al contenedor a través de una bash |
| `docker rm [Container ID]`   | Eliminar un contenedor |
| `docker rmi [Image ID]`      | Eliminar una imagen |
| `docker volume rm [Volume ID]`              | Eliminar un volumen |
| `docker network rm [Network ID]`            | Eliminar una interfaz de red |
| `docker-compose up -d`       | Crear contenedores a partir de un archivo compose.yaml |
| `docker build -t [Nombre Imagen] .`        | Crear una imagen a partir de un archivo Dockerfile |

## **Instalando Docker** 🐳
* **Comando:** `apt install docker.io -y`

Después de ejecutar el comando anterior, comprobamos que todo haya salido correctamente
```s
─$ docker -v
Docker version 20.10.24+dfsg1, build 297e128
```
---

## **Desplegando Contenedores** 🛂
1. Podemos descargarnos una imagen de manera remota (Como un repositorio de **Github**).
* **Comando:** `docker pull ubuntu:latest`
    * **`pull`:** Con esto podemos descargarnos una imagen de manera remota.
    * **`ubuntu:latest`:** Indicamos que queremos un **ubuntu** en su última versión (**`:latest`**), si quisieramos una versión específica lo indicamos con **`:X.X`**

  * Para ver las imágenes que tenemos en nuestro equipos usamos **`docker images`**
  ```sh
  ─$ docker images
  REPOSITORY   TAG       IMAGE ID       CREATED       SIZE
  ubuntu       latest    99284ca6cea0   2 weeks ago   77.8MB
  ```
  > Tenemos la imagen de **ubuntu** en nuestro equipo.

2. Ya con nuestra imagen, lo que haremos es ejecutarla para tener un **contenedor**
* **Comando:** `docker run -dit --name <Nombre del contenedor> <Nombre de la imagen>`
    * **`-d`:** Ejecutará el contenedor en segundo plano.
    * **`-i`:** Indicamos que queremos el **modo interactivo**.
    * **`-t`:** Dispondremos de una consola para acceder al contenedor.

    * Verificamos los contenedores que se estén ejecutando **`docker ps`**
    ```sh
    -$ docker ps
    CONTAINER ID   IMAGE     COMMAND       CREATED         STATUS         PORTS     NAMES
    85628b181292   ubuntu    "/bin/bash"   2 minutes ago   Up 2 minutes             primercontenedor
    ```
    > Tenemos el contenedor `primercontenedor` ejecutándose, usando la imagen de **ubuntu**.

3. Ahora, ya que tenemos este contenedor, nosotros queremos acceder a este desde una **bash** (terminal)
* **Comando:** `docker exec -it <Container ID> bash`
    * Con esto indicamos que queremos un **bash**

    Después de ejecutar el comando anterior, ya dispondremos de una **bash** para interactuar con el contenedor
    ```
    root@85628b181292:/# hostname
    85628b181292
    ```

4. Podemos desplegar contenedores desde archivos **`compose.yaml`**
* Supongamos que queremos desplegar el siguiente proyecto de manera local: [Avatar](https://github.com/dockersamples/single-dev-env)

* Una vez descargado el proyecto, usamos **`docker-compose up -d`** para desplegar

* Si vemos los contenedores que se están ejecutando veremos que hay 2
    ```bash
    ─$ docker ps
    CONTAINER ID   IMAGE                                 COMMAND            CREATED         STATUS        PORTS          NAMES
    e57e19afbea4   docker/dev-environments-go:stable-1   "sleep infinity"   9 seconds ago   Up 5 seconds                                                                                                    single-dev-env_app_1
    da85451879d0   3d18fe6d6805                          "/portainer"       4 weeks ago     Up 6 minutes   0.0.0.0:8000->8000/tcp, :::8000->8000/tcp, 0.0.0.0:9000->9000/tcp, :::9000->9000/tcp, 9443/tcp   portainer
    ```

## **Creando nuestros contenedores** 🏭
1. Todas nuestra configuraciones se guardarán en un archivo con nombre **Dockerfile**, en este ejemplo crearemos un contenedor que ejecute un script en **Python**
   * **script.py**
   ```py
   print("Hello World!")
   ```

   * **Dockerfile**
   ```dockerfile
   FROM python:latest # Escogemos la imagen base
   COPY script.py ./ # Copiamos el recurso script.py de nuestra máquina en `./` del contenedor.
   CMD ["python", "./script.py"] # Separamos por argumentos los comandos que se ejecutarán al iniciar el contenedor.
   ```

   * **Comando:** `docker build -t <Nombre Imagen> .`
     * **`. `:** Referenciamos al directorio actual de trabajo.

   * Se nos creó una nueva imagen
   ```java
   ─$ docker images
   REPOSITORY     TAG       IMAGE ID       CREATED          SIZE
   pythondocker   latest    5768b5571bf0   12 seconds ago   1.01GB
   ```

   * Por último, ejecutamos el contenedor
    ```java
    -$ docker run a13ab9cb7b2d
    Hello World!
    ```

## **Port Forwarding** 🎣
Muchas de las veces, al desplegar un contenedor lo que buscamos es ejecutar una aplicación o programa para testear su funcionamiento, pero, al no contar con una interfaz gráfica se vuelve complicado, para esto tenemos **port forwarding**, lo que logramos con esto es que si una aplicación se ejecuta en el contenedor por el puerto **80**, podemos hacer que esa aplicación se vea desde nuestra máquina host.

![](/assets/images/Laboratorios/basic-docker/port-forwarding.png)

   * Supongamos que necesitamos ver el servicio **nginx** en nuestra máquina host.
   * **Comando:** `docker run -dit -p 4444:80 <Image ID>`
     * **`-p 4444:80`:** Con esto indicamos que el **puerto 4444** de nuestra máquina host se convertirá en el **puerto 80** del contenedor.

   * **Nginx ejecutándose en un contenedor**
   ```bash
    root@a413e5947d13:/# systemctl status nginx
    nginx.service - A high performance web server and a reverse proxy server
    Loaded: loaded (/usr/lib/systemd/system/nginx.service, enabled)
    Active: active (running)
   ```

   * **Nginx ejecutándose en nuestra máquina host**

![](/assets/images/Laboratorios/basic-docker/nginx.png)

## **Usando volúmenes (monturas)** 🌋
Al usar volúmenes podemos **montar** archivos de nuestra máquina local en el contenedor.

Supongamos que necesitamos pasar toda un directorio al contenedor

   * **Directorio Local 'Test'** 📁 
    ```bash
    -rw-r--r-- 1 root root      32 Jun 14 22:53 clave_simetrica_original.txt
    -rw-r--r-- 1 root root 1173440 Jun 14 22:29 ElPrincipito.enc
    -rw------- 1 root root    1704 Jun 14 22:36 private_key.pem
    -rw-r--r-- 1 root root     256 Jun 14 22:40 simetric_cypher_rsa.txt
    ```

   * **Comando:** `docker run -dit -v /home/kali/Desktop/Test:/home/ <Image ID>`

   * Contenido de **Test** en el contenedor
        ```
        root@814f65b2e65b:/home# ls -la
        <SNIP>
        -rw-r--r-- 1 root root 1173440 Jun 15 02:29 ElPrincipito.enc
        -rw-r--r-- 1 root root      32 Jun 15 02:53 clave_simetrica_original.txt
        -rw------- 1 root root    1704 Jun 15 02:36 private_key.pem
        -rw-r--r-- 1 root root     256 Jun 15 02:40 simetric_cypher_rsa.txt
        ```

        ❗ Hay que tener cuidado al usar **volúmenes** ya que al ser monturas, si eliminamos un archivo del contenedor, también se eliminará de nuestra máquina **host**.