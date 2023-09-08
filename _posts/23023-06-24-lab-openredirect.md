---
title: Laboratorio - Open Redirect
author: cotes
date: 2023-09-07 17:51:00 +0800
categories: [Laboratorio, 'Open Redirect']
tags: ['Open Redirect', OWASP]
math: true
mermaid: true
image:
  path: /assets/images/Laboratorios/open-redirect/logo.jpg
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Open Redirect Logo
---

Esta vulnerabilidad se aplica cuando la aplicación web (vulnerable) permite a un atacante manipular la **dirección URL de redireccionamiento** hacia una página web, en este caso, hacia un **sitio web malicioso** o un **sitio web al que se desee atacar (DoS)**

> **Laboratorio con Docker:** [skf-labs](https://github.com/blabla1337/skf-labs/tree/master/python/Url-redirection)
{: .prompt-info }

* Nos lo descargamos: **svn checkout https://github.com/blabla1337/skf-labs/trunk/python/Url-redirection**

* Agregamos el argumento **port=80** en la línea 25 del archivo **redirect.py**

```python
app.run(host='0.0.0.0', host=80)
```

* Desplegamos **docker build -t openredirect .**
* Por último ejecutamos el contenedor **docker run -p 80:80 <Image ID>**

Si visitamos la página web veremos lo siguiente

![](/assets/images/Laboratorios/open-redirect/web.png)

* Al dar clic en **`Go to new website`** nos saldrá el mensaje **`Welcome to the new website!`** y la **url** cambiará a **`http://localhost/newsite`**

Al capturar la petición con **`Burp Suite`** vemos que se tramita una petición por **POST** a **`/redirect`**

![](/assets/images/Laboratorios/open-redirect/burp.png)

* El parámetro **`/newsite`** es el vulnerable, ya que aquí enviamos la **url** del sitio al cuál queremos redireccionar.
* Si indicamos un **url** de cualquier otro sitio web (Ej. **https://github.com**), veremos que somos redireccionados hacia allí.

![](/assets/images/Laboratorios/open-redirect/redirect.png)


**Laboratorio ✌:** [skf-labs](https://github.com/blabla1337/skf-labs/tree/master/python/Url-redirection-harder)

Si usamos una **url** normal, veremos el siguiente mensaje al enviar la petición al servidor

![](/assets/images/Laboratorios/open-redirect/web2.png)

* **Bypass Filtros**: [PayloadAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Open%20Redirect#Using)

Capturamos la petición con Burp Suite y enviamos lo siguiente

![](/assets/images/Laboratorios/open-redirect/burp2.png)

* Reemplazamos **`.`** por **`%E3%80%82`**
  
Y seremos redirigidos a **https://github.com**


**Laboratorio 3️⃣:** [skf-labs](https://github.com/blabla1337/skf-labs/tree/master/python/Url-redirection-harder2)

Si enviamos la **url** `https://github%E3%80%82com` vemos el siguiente mensaje

![](/assets/images/Laboratorios/open-redirect/web3.png)

* Si leemos formas de **bypassear** este filtro en **`PayloadAllTheThings`** veremos que podemos usar: `https:google.com` sin los símbolos **`//`**

![](/assets/images/Laboratorios/open-redirect/burp3.png)

Y seremos redirigidos a **https://github.com**