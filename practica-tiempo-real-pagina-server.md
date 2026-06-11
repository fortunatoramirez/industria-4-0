# Práctica 3: Envío de eventos en tiempo real desde una página web hacia un servidor

## Objetivo

Implementar una página web capaz de enviar y recibir eventos en tiempo real utilizando `socket.io@1.7.2`, Node.js y Express.

En esta práctica el estudiante comprenderá cómo una página web puede comunicarse con el servidor sin necesidad de recargar la página.

En esta práctica todavía no se recibirá información del ESP32.
La integración con el contador del ESP32 se realizará en la siguiente práctica.

---

# 1. Introducción

En las prácticas anteriores se desarrolló una página web publicada desde un servidor Node.js y posteriormente se le agregó diseño visual utilizando CSS.

Ahora se agregará comunicación en tiempo real mediante Socket.IO.

En una página web tradicional, el navegador solicita una página y el servidor responde. Sin embargo, en sistemas de Industria 4.0 se necesita que la comunicación ocurra de forma inmediata.

Por ejemplo:

* Enviar un comando de inicio.
* Enviar un comando de paro.
* Reiniciar un contador.
* Mostrar mensajes del servidor.
* Actualizar estados sin recargar la página.

Para lograrlo se utilizará `socket.io@1.7.2`.

---

# 2. Descripción general del sistema

El sistema tendrá la siguiente estructura:

```text
Página web → Socket.IO → Servidor Node.js
Página web ← Socket.IO ← Servidor Node.js
```

La página podrá enviar eventos al servidor.

El servidor recibirá esos eventos, los mostrará en consola y enviará una respuesta de regreso a la página.

---

# 3. Material necesario

* Computadora con Node.js instalado.
* Visual Studio Code.
* Navegador web.
* Terminal de comandos.
* Conexión a red local, opcional.

---

# 4. Estructura del proyecto

Crear una carpeta llamada:

```bash
practica3_eventos_websocket
```

Dentro de esa carpeta se tendrá la siguiente estructura:

```text
practica3_eventos_websocket
│
├── server.js
│
└── public
    ├── index.html
    ├── styles.css
    └── cliente.js
```

En esta práctica se mantendrán separados:

```text
HTML  → estructura de la página
CSS   → diseño visual
JS    → lógica del cliente web
```

---

# 5. Instalación del proyecto

Abrir una terminal dentro de la carpeta del proyecto y ejecutar:

```bash
npm init -y
```

Después instalar Express y Socket.IO:

```bash
npm install express socket.io@1.7.2
```

Se utiliza:

```text
socket.io@1.7.2
```

porque es la versión que se ha venido utilizando para mantener compatibilidad con el entorno de trabajo del curso.

---

# 6. Bloque 1: Servidor base con Express y Socket.IO

## Archivo `server.js`

```js
var express = require('express');
var app = express();

var server = require('http').Server(app);
var io = require('socket.io')(server);

var puerto = 5001;

app.use(express.static('public'));

io.on('connection', function(socket) {

    console.log('Cliente web conectado:', socket.id);

    socket.emit('DESDE_SERVER_MENSAJE', 'Conexión establecida con el servidor');

    socket.on('DESDE_WEB_MENSAJE', function(data) {

        console.log('Mensaje recibido desde la página web:', data);

        socket.emit('DESDE_SERVER_MENSAJE', 'Servidor recibió el mensaje: ' + data);

    });

    socket.on('disconnect', function() {

        console.log('Cliente web desconectado:', socket.id);

    });

});

server.listen(puerto, function() {

    console.log('======================================');
    console.log(' Servidor con Socket.IO iniciado');
    console.log(' Dirección: http://localhost:' + puerto);
    console.log(' Versión utilizada: socket.io@1.7.2');
    console.log('======================================');

});
```

---

# 7. Explicación del servidor

Esta línea importa Express:

```js
var express = require('express');
```

Esta línea crea la aplicación web:

```js
var app = express();
```

Esta línea crea un servidor HTTP:

```js
var server = require('http').Server(app);
```

Socket.IO necesita trabajar sobre un servidor HTTP.

Esta línea agrega Socket.IO al servidor:

```js
var io = require('socket.io')(server);
```

Esta línea publica la carpeta `public`:

```js
app.use(express.static('public'));
```

Esta parte detecta cuando un cliente web se conecta:

```js
io.on('connection', function(socket) {
```

Cuando el navegador se conecta, el servidor imprime en consola:

```text
Cliente web conectado: id-del-cliente
```

El servidor también envía un primer mensaje a la página:

```js
socket.emit('DESDE_SERVER_MENSAJE', 'Conexión establecida con el servidor');
```

Esta parte recibe mensajes enviados desde la página web:

```js
socket.on('DESDE_WEB_MENSAJE', function(data) {
```

El nombre del evento debe coincidir exactamente con el que será usado en el archivo `cliente.js`.

---

# 8. Bloque 2: Página HTML

## Archivo `public/index.html`

```html
<!DOCTYPE html>
<html lang="es">

<head>

    <meta charset="UTF-8">

    <title>Industria 4.0 - Socket.IO</title>

    <link rel="stylesheet" href="styles.css">

</head>

<body>

    <header>

        <h1>Industria 4.0</h1>

        <p>Práctica 3: Eventos en tiempo real con Socket.IO</p>

    </header>

    <main>

        <section class="tarjeta">

            <h2>Estado de conexión</h2>

            <p id="estadoConexion" class="estado desconectado">
                Desconectado del servidor
            </p>

        </section>

        <section class="tarjeta">

            <h2>Enviar mensaje al servidor</h2>

            <p>
                Escriba un mensaje y envíelo mediante Socket.IO.
            </p>

            <input type="text" id="mensajeInput" placeholder="Escriba un mensaje">

            <button onclick="enviarMensaje()">
                Enviar mensaje
            </button>

        </section>

        <section class="tarjeta">

            <h2>Comandos simulados</h2>

            <p>
                Estos botones simulan comandos que más adelante podrían enviarse a un ESP32.
            </p>

            <button onclick="enviarComando('INICIAR_BANDA')">
                Iniciar banda
            </button>

            <button onclick="enviarComando('DETENER_BANDA')">
                Detener banda
            </button>

            <button onclick="enviarComando('REINICIAR_CONTADOR')">
                Reiniciar contador
            </button>

        </section>

        <section class="tarjeta">

            <h2>Respuesta del servidor</h2>

            <p id="respuestaServidor">
                Esperando mensajes del servidor...
            </p>

        </section>

        <section class="tarjeta">

            <h2>Registro de eventos</h2>

            <div id="registroEventos" class="registro">
                No hay eventos registrados.
            </div>

        </section>

    </main>

    <script src="/socket.io/socket.io.js"></script>
    <script src="cliente.js"></script>

</body>

</html>
```

---

# 9. Explicación del HTML

Esta línea conecta el archivo CSS:

```html
<link rel="stylesheet" href="styles.css">
```

Esta sección muestra el estado de conexión:

```html
<p id="estadoConexion" class="estado desconectado">
```

Este campo permite escribir un mensaje:

```html
<input type="text" id="mensajeInput" placeholder="Escriba un mensaje">
```

Este botón llama a una función de JavaScript:

```html
<button onclick="enviarMensaje()">
```

Esta línea carga la librería Socket.IO desde el servidor:

```html
<script src="/socket.io/socket.io.js"></script>
```

Esta línea carga el archivo JavaScript propio del cliente:

```html
<script src="cliente.js"></script>
```

---

# 10. Bloque 3: Estilos CSS

## Archivo `public/styles.css`

```css
body {

    margin: 0;

    font-family: Arial, sans-serif;

    background: #eef2f5;

}

header {

    background: #1f3c88;

    color: white;

    text-align: center;

    padding: 25px;

}

main {

    width: 85%;

    max-width: 1000px;

    margin: 30px auto;

}

.tarjeta {

    background: white;

    padding: 22px;

    margin-bottom: 20px;

    border-radius: 14px;

    box-shadow: 0px 4px 12px rgba(0, 0, 0, 0.15);

}

.tarjeta h2 {

    color: #1f3c88;

}

.estado {

    padding: 12px;

    border-radius: 8px;

    font-weight: bold;

}

.conectado {

    background: #e8f8ef;

    color: #087a36;

    border-left: 6px solid #087a36;

}

.desconectado {

    background: #fdecea;

    color: #a12622;

    border-left: 6px solid #a12622;

}

input {

    width: 70%;

    padding: 12px;

    font-size: 16px;

    border: 1px solid #ccc;

    border-radius: 8px;

    margin-right: 10px;

}

button {

    background: #1f3c88;

    color: white;

    border: none;

    padding: 12px 18px;

    margin: 5px;

    border-radius: 8px;

    cursor: pointer;

    font-size: 15px;

}

button:hover {

    background: #162b63;

}

.registro {

    background: #f4f6fb;

    padding: 15px;

    border-radius: 8px;

    min-height: 80px;

    font-family: Consolas, monospace;

}
```

---

# 11. Bloque 4: JavaScript del cliente web

## Archivo `public/cliente.js`

```js
var socket = io();

var estadoConexion = document.getElementById('estadoConexion');
var respuestaServidor = document.getElementById('respuestaServidor');
var registroEventos = document.getElementById('registroEventos');

socket.on('connect', function() {

    estadoConexion.innerHTML = 'Conectado al servidor';

    estadoConexion.className = 'estado conectado';

    agregarEvento('Conexión establecida con el servidor');

});

socket.on('disconnect', function() {

    estadoConexion.innerHTML = 'Desconectado del servidor';

    estadoConexion.className = 'estado desconectado';

    agregarEvento('Conexión perdida con el servidor');

});

socket.on('DESDE_SERVER_MENSAJE', function(data) {

    respuestaServidor.innerHTML = data;

    agregarEvento('Servidor: ' + data);

});

function enviarMensaje() {

    var mensaje = document.getElementById('mensajeInput').value;

    if (mensaje.trim() === '') {

        alert('Escriba un mensaje antes de enviar');

        return;

    }

    socket.emit('DESDE_WEB_MENSAJE', mensaje);

    agregarEvento('Página web envió: ' + mensaje);

    document.getElementById('mensajeInput').value = '';

}

function enviarComando(comando) {

    socket.emit('DESDE_WEB_MENSAJE', comando);

    agregarEvento('Comando enviado desde la página: ' + comando);

}

function agregarEvento(texto) {

    var fecha = new Date();

    var hora = fecha.toLocaleTimeString();

    if (registroEventos.innerHTML === 'No hay eventos registrados.') {

        registroEventos.innerHTML = '';

    }

    registroEventos.innerHTML += '[' + hora + '] ' + texto + '<br>';

}
```

---

# 12. Explicación del JavaScript

Esta línea crea la conexión con el servidor:

```js
var socket = io();
```

Esta parte detecta cuando la página se conecta al servidor:

```js
socket.on('connect', function() {
```

Esta parte detecta cuando la página pierde conexión:

```js
socket.on('disconnect', function() {
```

Esta parte recibe mensajes enviados desde el servidor:

```js
socket.on('DESDE_SERVER_MENSAJE', function(data) {
```

Esta función envía un mensaje al servidor:

```js
function enviarMensaje() {
```

El envío ocurre con:

```js
socket.emit('DESDE_WEB_MENSAJE', mensaje);
```

Esta función envía comandos simulados:

```js
function enviarComando(comando) {
```

Por ahora estos comandos solo se envían al servidor.

En una práctica posterior podrán utilizarse para controlar un ESP32.

---

# 13. Ejecución del sistema

Ejecutar el servidor con:

```bash
node server.js
```

Si todo está correcto, deberá aparecer:

```text
======================================
 Servidor con Socket.IO iniciado
 Dirección: http://localhost:5001
 Versión utilizada: socket.io@1.7.2
======================================
```

Abrir el navegador en:

```text
http://localhost:5001
```

---

# 14. Prueba del sistema

## Prueba 1

Abrir la página.

En la página deberá aparecer:

```text
Conectado al servidor
```

En la consola del servidor deberá aparecer:

```text
Cliente web conectado: id-del-cliente
```

---

## Prueba 2

Escribir un mensaje en la caja de texto y presionar:

```text
Enviar mensaje
```

En la consola del servidor deberá aparecer:

```text
Mensaje recibido desde la página web: texto-enviado
```

En la página deberá aparecer una respuesta del servidor:

```text
Servidor recibió el mensaje: texto-enviado
```

---

## Prueba 3

Presionar el botón:

```text
Iniciar banda
```

En la consola del servidor deberá aparecer:

```text
Mensaje recibido desde la página web: INICIAR_BANDA
```

En la página deberá registrarse el evento enviado.

---

# 15. Errores comunes

## La página no abre

Verificar que el servidor esté corriendo:

```bash
node server.js
```

También verificar que la dirección sea:

```text
http://localhost:5001
```

---

## Socket.IO no conecta

Verificar que en el HTML exista la línea:

```html
<script src="/socket.io/socket.io.js"></script>
```

También verificar que el servidor esté utilizando:

```js
var io = require('socket.io')(server);
```

---

## El botón no hace nada

Verificar que el archivo `cliente.js` esté dentro de la carpeta `public`.

También verificar que el HTML tenga la línea:

```html
<script src="cliente.js"></script>
```

---

## El servidor no recibe mensajes

Verificar que el nombre del evento coincida exactamente.

En el cliente:

```js
socket.emit('DESDE_WEB_MENSAJE', mensaje);
```

En el servidor:

```js
socket.on('DESDE_WEB_MENSAJE', function(data) {
```

---

# 16. Actividad de comprobación

El estudiante deberá modificar la página para agregar un nuevo botón llamado:

```text
Activar alarma
```

Al presionarlo, deberá enviar al servidor el comando:

```text
ACTIVAR_ALARMA
```

Ejemplo:

```html
<button onclick="enviarComando('ACTIVAR_ALARMA')">
    Activar alarma
</button>
```

El servidor deberá mostrar en consola:

```text
Mensaje recibido desde la página web: ACTIVAR_ALARMA
```

---

# 17. Reto adicional

Modificar el servidor para que, cuando reciba el mensaje:

```text
ACTIVAR_ALARMA
```

responda con:

```text
Alarma activada desde la página web
```

Sugerencia:

```js
if (data === 'ACTIVAR_ALARMA') {
    socket.emit('DESDE_SERVER_MENSAJE', 'Alarma activada desde la página web');
}
```

---

# 18. Conclusión

En esta práctica se implementó comunicación en tiempo real entre una página web y un servidor Node.js utilizando `socket.io@1.7.2`.

El estudiante aprendió a separar la estructura HTML, el diseño CSS y la lógica JavaScript del cliente.

También comprendió cómo una página web puede enviar eventos al servidor y recibir respuestas sin necesidad de recargar la página.

Esta práctica servirá como base para la siguiente actividad, donde el servidor recibirá eventos del contador enviado por el ESP32 y los mostrará en la página web.
