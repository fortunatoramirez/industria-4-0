# Práctica: Publicación de una página web con Node.js y Express

## Objetivo

Implementar un servidor web básico utilizando Node.js y Express para publicar una página HTML desde una carpeta pública. El estudiante comprenderá qué es una página web, cómo un navegador solicita información a un servidor y cómo Express permite ofrecer archivos HTML como parte de un servicio web.

---

# 1. Introducción

En los sistemas de Industria 4.0 es común utilizar páginas web para visualizar información de sensores, monitorear procesos industriales, controlar dispositivos y presentar datos en tiempo real.

Una página web puede funcionar como una interfaz visual para sistemas como:

* Bandas transportadoras.
* Sensores infrarrojos.
* Sistemas de conteo de piezas.
* Monitoreo de temperatura.
* Paneles de producción.
* Sistemas IoT con ESP32.

En prácticas anteriores se enviaron datos desde el ESP32 hacia un servidor Node.js usando `socket.io@1.7.2`.

En esta práctica se estudiará una parte fundamental del sistema: cómo el servidor puede ofrecer una página web al navegador.

---

# 2. Material necesario

* Computadora con Node.js instalado.
* Visual Studio Code.
* Navegador web.
* Terminal de comandos.
* Conexión a red local, opcional.

---

# 3. Descripción general del sistema

El sistema tendrá la siguiente estructura:

```text
Navegador web → Servidor Node.js con Express → Página HTML
```

Cuando el usuario escriba la dirección del servidor en el navegador:

```text
http://localhost:3000
```

ocurrirá el siguiente proceso:

```text
El navegador solicita una página
El servidor recibe la solicitud
Express busca el archivo en la carpeta public
El servidor envía la página HTML
El navegador muestra la página
```

---

# 4. Preparación del proyecto

Crear una carpeta llamada:

```bash
practica1_pagina_web
```

Abrir esa carpeta en Visual Studio Code.

Dentro de la carpeta, abrir una terminal y ejecutar:

```bash
npm init -y
```

Después instalar Express:

```bash
npm install express
```

---

# 5. Estructura del proyecto

Crear la siguiente estructura de archivos:

```text
practica1_pagina_web
│
├── server.js
│
└── public
    └── index.html
```

La carpeta `public` será la carpeta pública del servidor.

Todo archivo colocado dentro de esa carpeta podrá ser enviado al navegador.

---

# 6. Código del servidor

Crear un archivo llamado:

```text
server.js
```

Colocar el siguiente código:

```js
var express = require('express');
var app = express();

var puerto = 3000;

// Esta línea permite publicar los archivos de la carpeta public
app.use(express.static('public'));

app.listen(puerto, function() {
    console.log('======================================');
    console.log(' Servidor web iniciado correctamente');
    console.log(' Carpeta pública: public');
    console.log(' Dirección: http://localhost:' + puerto);
    console.log('======================================');
});
```

---

# 7. Explicación del servidor

Esta línea importa Express:

```js
var express = require('express');
```

Express permite crear un servidor web de forma sencilla.

Esta línea crea la aplicación principal:

```js
var app = express();
```

Esta variable define el puerto donde funcionará el servidor:

```js
var puerto = 3000;
```

El puerto es el número que identifica el servicio dentro de la computadora.

Esta línea publica la carpeta `public`:

```js
app.use(express.static('public'));
```

Esto significa que Express buscará automáticamente los archivos que el navegador solicite dentro de esa carpeta.

Esta parte inicia el servidor:

```js
app.listen(puerto, function() {
```

Cuando el servidor inicia correctamente, se muestra un mensaje en la terminal.

---

# 8. Código de la página web

Dentro de la carpeta `public`, crear un archivo llamado:

```text
index.html
```

Colocar el siguiente código:

```html
<!DOCTYPE html>
<html lang="es">

<head>
    <meta charset="UTF-8">
    <title>Industria 4.0</title>
</head>

<body>

    <h1>Industria 4.0</h1>

    <h2>Primera página publicada desde Node.js</h2>

    <p>
        Esta página está siendo enviada desde un servidor creado con Node.js y Express.
    </p>

    <p>
        En futuras prácticas esta página permitirá visualizar información recibida desde un ESP32.
    </p>

    <h3>Aplicaciones relacionadas</h3>

    <ul>
        <li>Monitoreo de una banda transportadora.</li>
        <li>Conteo de piezas con sensor infrarrojo.</li>
        <li>Visualización de datos industriales.</li>
        <li>Comunicación entre dispositivos IoT.</li>
    </ul>

</body>

</html>
```

---

# 9. Ejecución del servidor

Para ejecutar el servidor, usar:

```bash
node server.js
```

Si todo está correcto, deberá aparecer en la terminal:

```text
======================================
 Servidor web iniciado correctamente
 Carpeta pública: public
 Dirección: http://localhost:3000
======================================
```

---

# 10. Prueba en el navegador

Abrir un navegador web y escribir:

```text
http://localhost:3000
```

Deberá mostrarse una página con el siguiente contenido:

```text
Industria 4.0

Primera página publicada desde Node.js

Esta página está siendo enviada desde un servidor creado con Node.js y Express.

En futuras prácticas esta página permitirá visualizar información recibida desde un ESP32.

Aplicaciones relacionadas
- Monitoreo de una banda transportadora.
- Conteo de piezas con sensor infrarrojo.
- Visualización de datos industriales.
- Comunicación entre dispositivos IoT.
```

---

# 11. Salida agradable en consola

El mensaje mostrado en consola permite verificar que el servidor está funcionando correctamente:

```text
======================================
 Servidor web iniciado correctamente
 Carpeta pública: public
 Dirección: http://localhost:3000
======================================
```

Esta salida indica tres aspectos importantes:

1. El servidor se inició sin errores.
2. La carpeta `public` está siendo compartida.
3. La página puede abrirse desde el navegador usando `localhost`.

---

# 12. Modificación de la página

El estudiante deberá modificar el archivo `index.html` para agregar sus datos personales.

Agregar debajo del título principal:

```html
<h3>Datos del estudiante</h3>

<p>Nombre: Escribir nombre completo</p>
<p>Grupo: Escribir grupo</p>
<p>Carrera: Escribir carrera</p>
```

Después de guardar los cambios, actualizar el navegador.

No es necesario detener el servidor si solamente se modificó el archivo HTML.

---

# 13. Prueba usando otra computadora de la red

Si se desea abrir la página desde otra computadora conectada a la misma red, primero se debe identificar la IP de la computadora donde corre el servidor.

En Windows, abrir una terminal y ejecutar:

```bash
ipconfig
```

Buscar la dirección IPv4.

Ejemplo:

```text
Dirección IPv4 . . . . . . . . . . : 192.168.1.75
```

Desde otra computadora, abrir el navegador y escribir:

```text
http://192.168.1.75:3000
```

La dirección debe cambiarse por la IP real de la computadora del servidor.

---

# 14. Relación con Industria 4.0

En esta práctica todavía no se reciben datos del ESP32.

Sin embargo, la página creada será la base para prácticas posteriores donde se podrá mostrar:

```text
Conteo de piezas detectadas
Estado de sensores
Mensajes enviados desde el ESP32
Alertas del proceso
Estado de una banda transportadora
```

El servidor Node.js será el punto central de comunicación entre los dispositivos y la interfaz web.

---

# 15. Errores comunes

## El comando `node server.js` no funciona

Verificar que la terminal esté abierta dentro de la carpeta correcta.

También verificar que Node.js esté instalado usando:

```bash
node -v
```

---

## Express no está instalado

Si aparece un error relacionado con Express, ejecutar:

```bash
npm install express
```

---

## La página no abre en el navegador

Verificar que el servidor esté ejecutándose.

Debe aparecer en la terminal:

```text
Servidor web iniciado correctamente
```

También verificar que la dirección sea:

```text
http://localhost:3000
```

---

## La página aparece en blanco

Verificar que el archivo se llame exactamente:

```text
index.html
```

También revisar que esté dentro de la carpeta:

```text
public
```

---

## Los cambios no aparecen

Actualizar el navegador con:

```text
Ctrl + R
```

o cerrar y abrir nuevamente la página.

---

# 16. Actividad de comprobación

El estudiante deberá modificar la página para agregar una sección llamada:

```text
Mi primer panel industrial
```

Dentro de esa sección deberá incluir:

1. Nombre del estudiante.
2. Grupo.
3. Carrera.
4. Una lista con tres sensores que podrían conectarse a un ESP32.
5. Una lista con tres datos que podrían mostrarse en una página web industrial.

Ejemplo:

```html
<h2>Mi primer panel industrial</h2>

<h3>Sensores posibles</h3>
<ul>
    <li>Sensor infrarrojo</li>
    <li>Sensor de temperatura</li>
    <li>Sensor de proximidad</li>
</ul>

<h3>Datos que puede mostrar la página</h3>
<ul>
    <li>Conteo de piezas</li>
    <li>Estado de la banda</li>
    <li>Última detección del sensor</li>
</ul>
```

---

# 17. Conclusión

En esta práctica se creó un servidor web básico utilizando Node.js y Express. El servidor publicó una página HTML almacenada en la carpeta `public`, permitiendo que el navegador la solicitara y la mostrara al usuario.

El estudiante identificó la función del navegador como cliente, la función de Node.js como servidor y el papel de Express para publicar archivos web.

Esta práctica sirve como base para desarrollar interfaces industriales más completas, donde posteriormente se integrarán estilos CSS, comunicación en tiempo real con `socket.io` y datos enviados desde el ESP32.
