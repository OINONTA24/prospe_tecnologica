---
layout: default
title: Página Web
parent: Servicio Web
nav_order: 1
permalink: /servicio-web/pagina-web/
---

# Práctica 1: Página Web 

## 1) Creación de la página (index.html)

Al momento de elaborar la página web, el primer paso es crear un archivo index.html, el cual funcionará como la portada del sitio. En la sección 1.1 se muestra el código utilizado. Ahí puedes ir construyendo la estructura básica de la página, agregando el encabezado <head> (título) y el cuerpo <body> (contenido visible como textos, imágenes y secciones).

### 1.1 Código index.html

```html
<!doctype html>
<html>
<head>
    <title>Luis CM</title>
</head>
<body>
    <h1>Ing. Luis Cortes</h1>
    <p>Este es mi primer proyecto web con HTML + CSS + JS.</p>
</body>
</html>
```

## 2) Visualización local con Live Server

Posteriormente, se recomienda trabajar en Visual Studio Code (escritorio) e instalar la extensión Live Server, la cual permite visualizar la página en un servidor local (localhost) y actualizarla automáticamente al guardar cambios. Una vez instalada la extensión, abre el archivo index.html, haz clic derecho y selecciona “Open with Live Server”. Si todo está configurado correctamente, se mostrará una pantalla como la presentada en la Figura 1.

![Figura 1 — LiveServer](assets/img/01-publicar/paginaWeb1.png)

*Figura 1: Visualización de la página web en localhost mediante la extensión Live Server en Visual Studio Code.*

En la Figura 1 se observa que, en la parte superior del navegador, la pestaña muestra el título Luis CM, y en la página se visualiza el contenido definido dentro del <body> del archivo index.html.


## 3) Configuración para acceso desde otros dispositivos (settings.json)

Como se observa en la Figura 1, la URL muestra localhost:5500 porque la página se está ejecutando de forma local mediante Live Server. Para poder acceder desde cualquier dispositivo utilizando la IP del equipo servidor, se debe crear un archivo settings.json y copiar el código mostrado en la sección 3.2. Con esta configuración, Live Server permitirá abrir el sitio usando la IP del equipo en lugar de únicamente localhost.

Es importante escribir el código con la sintaxis correcta. Además, el puerto puede configurarse con cualquiera de los valores comunes, como 5000, 5500, entre otros.

### 3.1 Estructura de carpetas

Es fundamental mantener la estructura de carpetas como se muestra en la Figura 2: crear una carpeta llamada .vscode donde se guardará el archivo settings.json, y dejar el archivo index.html fuera de esa carpeta, en el directorio principal del proyecto.

![Figura 2 — paginaWebIP](assets/img/01-publicar/paginaWeb2.png)

*Figura 2: Estructura de carpetas del proyecto (carpeta .vscode y archivo index.html).*

### 3.2 Código settings.json
```json
{
    "liveServer.settings.port":5500,
    "liveServer.settings.host":"localhost"
}
```

### 3.3 Obtención de la IP del equipo para acceder desde otros dispositivos

Una vez completados los pasos anteriores, se debe identificar la dirección IP del equipo que está funcionando como servidor. Para ello, abre la terminal del dispositivo, con Windows + R, escribe cmd y presiona Enter. En la terminal, ejecuta el comando ipconfig y presiona Enter.

Se mostrarán varias líneas, pero la más importante es “Dirección IPv4”, ya que esa es la IP que permitirá abrir la página desde otro dispositivo conectado a la misma red. En el ejemplo, la IP es 192.168.0.182.

Finalmente, para acceder a la página desde otro dispositivo, se utiliza la siguiente sintaxis:
192.168.0.182:5500, donde 5500 es el puerto definido previamente en el archivo settings.json. Si todo está configurado correctamente, se visualizará la misma página, pero con la diferencia de que la URL mostrará la IP del equipo, como se indica en el recuadro rojo de la Figura 3.

![Figura 3 — IP](assets/img/01-publicar/paginaWeb3.png)

*Figura 3: Acceso a la página mediante IP y puerto desde otro dispositivo*
