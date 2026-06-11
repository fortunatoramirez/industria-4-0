# Práctica: Visualización en página web del contador enviado por ESP32

## Objetivo

Implementar un sistema de comunicación en tiempo real donde un ESP32 envía el conteo de activaciones de un sensor infrarrojo hacia un servidor Node.js utilizando `socket.io@1.7.2`, y posteriormente el servidor reenvía ese dato a una página web para mostrarlo visualmente.

---

# 1. Introducción

En prácticas anteriores se trabajó con:

* Publicación de páginas web usando Node.js y Express.
* Separación de archivos HTML, CSS y JavaScript.
* Envío de eventos desde una página web hacia el servidor utilizando Socket.IO.

En esta práctica se integrará nuevamente el ESP32.

El ESP32 detectará activaciones de un sensor infrarrojo, simulando el paso de piezas por una banda transportadora.

Cada vez que el sensor detecte una pieza:

```text
El ESP32 incrementa el contador
El ESP32 envía el conteo al servidor
El servidor recibe el dato
El servidor reenvía el dato a la página web
La página actualiza el contador visualmente
```

Esta práctica representa una base importante para construir un dashboard industrial de monitoreo en tiempo real.

---

# 2. Descripción general del sistema

El sistema tendrá la siguiente estructura:

```text
Sensor infrarrojo → ESP32 → WiFi → Servidor Node.js → Página web
```

La comunicación se realizará mediante eventos de Socket.IO.

```text
ESP32 envía evento:
DESDE_ESP32_CONTADOR

Servidor reenvía evento:
DESDE_SERVER_CONTADOR

Página web recibe evento:
DESDE_SERVER_CONTADOR
```

---

# 3. Material necesario

* 1 ESP32.
* 1 sensor infrarrojo digital.
* Cables Dupont.
* Protoboard, opcional.
* Computadora con Node.js instalado.
* Visual Studio Code.
* Arduino IDE.
* Librería `SocketIoClient` instalada en Arduino IDE.
* Red WiFi local.

---

# 4. Funcionamiento esperado

Cuando el sensor infrarrojo detecte una pieza, el ESP32 deberá mostrar en el monitor serial:

```text
Pieza detectada. Conteo local: 1
```

El servidor deberá mostrar en consola:

```text
Conteo recibido desde ESP32: 1
```

La página web deberá actualizar el contador principal:

```text
Piezas detectadas: 1
```

---

# 5. Conexión del sensor infrarrojo al ESP32

Para esta práctica se utilizará el GPIO 18.

Conexión sugerida:

```text
Sensor infrarrojo → ESP32

VCC  → 3.3V o 5V, según el módulo utilizado
GND  → GND
OUT  → GPIO 18
```

En muchos sensores infrarrojos digitales:

```text
Sin detección     → HIGH
Con detección     → LOW
```

Por esa razón, el código contará una pieza cuando el pin cambie a:

```text
LOW
```

Si el sensor del alumno trabaja al revés, se deberá modificar la condición en el código.

---

# 6. Estructura del proyecto web

Crear una carpeta llamada:

```bash
practica4_esp32_dashboard
```

Dentro de la carpeta crear la siguiente estructura:

```text
practica4_esp32_dashboard
│
├── server.js
│
└── public
    ├── index.html
    ├── styles.css
    └── cliente.js
```

---

# 7. Instalación del proyecto Node.js

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

para mantener compatibilidad con las prácticas anteriores y con la librería utilizada desde el ESP32.

---

# 8. Bloque 1: Código del servidor

Crear el archivo:

```text
server.js
```

Colocar el siguiente código:

```js
var express = require('express');
var app = express();

var server = require('http').Server(app);
var io = require('socket.io')(server);

var puerto = 5001;

var contadorActual = 0;
var ultimaHora = 'Sin eventos';

app.use(express.static('public'));

io.on('connection', function(socket) {

    console.log('Cliente conectado:', socket.id);

    socket.emit('DESDE_SERVER_MENSAJE', 'Conectado al servidor Node.js');

    socket.emit('DESDE_SERVER_CONTADOR', {
        contador: contadorActual,
        hora: ultimaHora,
        origen: 'Servidor'
    });

    socket.on('DESDE_ESP32_CONTADOR', function(data) {

        var contadorRecibido = limpiarDatoContador(data);

        if (isNaN(contadorRecibido)) {

            console.log('Dato no válido recibido desde ESP32:', data);

            return;

        }

        contadorActual = contadorRecibido;
        ultimaHora = obtenerHoraActual();

        console.log('Conteo recibido desde ESP32:', contadorActual);

        io.emit('DESDE_SERVER_CONTADOR', {
            contador: contadorActual,
            hora: ultimaHora,
            origen: 'ESP32'
        });

    });

    socket.on('disconnect', function() {

        console.log('Cliente desconectado:', socket.id);

    });

});

server.listen(puerto, function() {

    console.log('======================================');
    console.log(' Servidor Industria 4.0 iniciado');
    console.log(' Dirección: http://localhost:' + puerto);
    console.log(' Esperando datos del ESP32...');
    console.log('======================================');

});

function limpiarDatoContador(data) {

    var texto = String(data);

    texto = texto.replace(/"/g, '');
    texto = texto.trim();

    return parseInt(texto);

}

function obtenerHoraActual() {

    var fecha = new Date();

    return fecha.toLocaleTimeString();

}
```

---

# 9. Explicación del servidor

Esta línea crea el servidor HTTP:

```js
var server = require('http').Server(app);
```

Esta línea integra Socket.IO al servidor:

```js
var io = require('socket.io')(server);
```

Esta línea publica los archivos de la carpeta `public`:

```js
app.use(express.static('public'));
```

Esta parte detecta cuando un cliente se conecta:

```js
io.on('connection', function(socket) {
```

El cliente puede ser:

```text
Página web
ESP32
Otro cliente Socket.IO
```

Esta parte recibe el contador enviado desde el ESP32:

```js
socket.on('DESDE_ESP32_CONTADOR', function(data) {
```

El evento debe llamarse exactamente:

```text
DESDE_ESP32_CONTADOR
```

Esta línea reenvía el contador hacia todos los clientes conectados, incluida la página web:

```js
io.emit('DESDE_SERVER_CONTADOR', {
    contador: contadorActual,
    hora: ultimaHora,
    origen: 'ESP32'
});
```

La página web escuchará el evento:

```text
DESDE_SERVER_CONTADOR
```

---

# 10. Bloque 2: Página HTML

Crear el archivo:

```text
public/index.html
```

Colocar el siguiente código:

```html
<!DOCTYPE html>
<html lang="es">

<head>

    <meta charset="UTF-8">

    <title>Industria 4.0 - Contador ESP32</title>

    <link rel="stylesheet" href="styles.css">

</head>

<body>

    <header>

        <h1>Industria 4.0</h1>

        <p>Dashboard de conteo de piezas con ESP32</p>

    </header>

    <main>

        <section class="tarjeta">

            <h2>Estado de conexión</h2>

            <p id="estadoConexion" class="estado desconectado">
                Desconectado del servidor
            </p>

        </section>

        <section class="tarjeta contador-principal">

            <h2>Piezas detectadas</h2>

            <div id="contador" class="contador">
                0
            </div>

            <p>
                Conteo recibido desde el ESP32
            </p>

        </section>

        <section class="tarjeta">

            <h2>Último evento recibido</h2>

            <p>
                Origen:
                <strong id="origenEvento">Sin datos</strong>
            </p>

            <p>
                Hora:
                <strong id="horaEvento">Sin datos</strong>
            </p>

        </section>

        <section class="tarjeta">

            <h2>Estado del proceso</h2>

            <p id="estadoProceso" class="proceso normal">
                Esperando detecciones del sensor...
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

# 11. Explicación del HTML

Esta línea enlaza la hoja de estilos:

```html
<link rel="stylesheet" href="styles.css">
```

Esta sección muestra el estado de conexión:

```html
<p id="estadoConexion" class="estado desconectado">
```

Este elemento muestra el conteo principal:

```html
<div id="contador" class="contador">
```

Estos elementos mostrarán información del último evento recibido:

```html
<strong id="origenEvento">Sin datos</strong>
<strong id="horaEvento">Sin datos</strong>
```

Esta línea carga la librería Socket.IO desde el servidor:

```html
<script src="/socket.io/socket.io.js"></script>
```

Esta línea carga la lógica del cliente web:

```html
<script src="cliente.js"></script>
```

---

# 12. Bloque 3: Estilos CSS

Crear el archivo:

```text
public/styles.css
```

Colocar el siguiente código:

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

.contador-principal {

    text-align: center;

}

.contador {

    font-size: 90px;

    font-weight: bold;

    color: #087a36;

    margin: 20px 0;

}

.proceso {

    padding: 12px;

    border-radius: 8px;

    font-weight: bold;

}

.normal {

    background: #f4f6fb;

    color: #333;

    border-left: 6px solid #1f3c88;

}

.alerta {

    background: #fff3cd;

    color: #856404;

    border-left: 6px solid #f0ad4e;

}

.meta {

    background: #e8f8ef;

    color: #087a36;

    border-left: 6px solid #087a36;

}

.registro {

    background: #f4f6fb;

    padding: 15px;

    border-radius: 8px;

    min-height: 100px;

    font-family: Consolas, monospace;

    font-size: 14px;

}
```

---

# 13. Bloque 4: JavaScript del cliente web

Crear el archivo:

```text
public/cliente.js
```

Colocar el siguiente código:

```js
var socket = io();

var estadoConexion = document.getElementById('estadoConexion');
var contador = document.getElementById('contador');
var origenEvento = document.getElementById('origenEvento');
var horaEvento = document.getElementById('horaEvento');
var estadoProceso = document.getElementById('estadoProceso');
var registroEventos = document.getElementById('registroEventos');

socket.on('connect', function() {

    estadoConexion.innerHTML = 'Conectado al servidor';

    estadoConexion.className = 'estado conectado';

    agregarEvento('Página web conectada al servidor');

});

socket.on('disconnect', function() {

    estadoConexion.innerHTML = 'Desconectado del servidor';

    estadoConexion.className = 'estado desconectado';

    agregarEvento('Página web desconectada del servidor');

});

socket.on('DESDE_SERVER_MENSAJE', function(data) {

    agregarEvento('Servidor: ' + data);

});

socket.on('DESDE_SERVER_CONTADOR', function(data) {

    contador.innerHTML = data.contador;

    origenEvento.innerHTML = data.origen;

    horaEvento.innerHTML = data.hora;

    actualizarEstadoProceso(data.contador);

    agregarEvento('Conteo recibido: ' + data.contador + ' | Origen: ' + data.origen);

});

function actualizarEstadoProceso(valor) {

    if (valor == 0) {

        estadoProceso.innerHTML = 'Esperando detecciones del sensor...';

        estadoProceso.className = 'proceso normal';

    }
    else if (valor > 0 && valor < 10) {

        estadoProceso.innerHTML = 'Proceso en operación. Conteo activo.';

        estadoProceso.className = 'proceso normal';

    }
    else if (valor >= 10 && valor < 20) {

        estadoProceso.innerHTML = 'Meta inicial alcanzada: 10 piezas o más.';

        estadoProceso.className = 'proceso alerta';

    }
    else {

        estadoProceso.innerHTML = 'Producción alta detectada: 20 piezas o más.';

        estadoProceso.className = 'proceso meta';

    }

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

# 14. Explicación del JavaScript

Esta línea crea la conexión con el servidor:

```js
var socket = io();
```

Esta parte detecta la conexión:

```js
socket.on('connect', function() {
```

Esta parte detecta la desconexión:

```js
socket.on('disconnect', function() {
```

Esta parte recibe el contador enviado por el servidor:

```js
socket.on('DESDE_SERVER_CONTADOR', function(data) {
```

El servidor envía un objeto con la siguiente estructura:

```js
{
    contador: 5,
    hora: "10:25:30",
    origen: "ESP32"
}
```

La página actualiza el valor visual con:

```js
contador.innerHTML = data.contador;
```

También actualiza el estado del proceso con:

```js
actualizarEstadoProceso(data.contador);
```

---

# 15. Ejecución del servidor

Ejecutar el servidor con:

```bash
node server.js
```

Si todo está correcto, deberá aparecer:

```text
======================================
 Servidor Industria 4.0 iniciado
 Dirección: http://localhost:5001
 Esperando datos del ESP32...
======================================
```

Abrir el navegador en:

```text
http://localhost:5001
```

La página deberá mostrar:

```text
Conectado al servidor
Piezas detectadas: 0
Esperando detecciones del sensor...
```

---

# 16. Identificar la IP del servidor

El ESP32 necesita conocer la IP de la computadora donde corre el servidor.

En Windows, abrir una terminal y ejecutar:

```bash
ipconfig
```

Buscar la dirección IPv4 de la red WiFi.

Ejemplo:

```text
Dirección IPv4 . . . . . . . . . . : 192.168.1.75
```

Esa IP deberá colocarse en el código del ESP32:

```cpp
const char* server = "192.168.1.75";
```

La computadora y el ESP32 deben estar conectados a la misma red WiFi.

---

# 17. Bloque 5: Código del ESP32

Cargar el siguiente programa en el ESP32:

```cpp
#include <WiFi.h>
#include <SocketIoClient.h>

/******************/

const char*     ssid      = "Nombre-red";
const char*     password  = "password-red";
const char*     server    = "ip-del-server";
const uint16_t  port      = 5001;

uint64_t        now       = 0;

/******************/

#define ONBOARD_LED  2
#define SENSOR_IR    18

/******************/

SocketIoClient socketIO;

String mensaje;

int contador = 0;

int estadoActual = HIGH;
int estadoAnterior = HIGH;

uint64_t tiempoUltimoCambio = 0;
uint64_t tiempoAntirrebote = 80;

/******************/

void setup() {

  Serial.begin(115200);

  pinMode(ONBOARD_LED, OUTPUT);
  pinMode(SENSOR_IR, INPUT_PULLUP);

  connectWiFi_STA();

  socketIO.begin(server, port);

  socketIO.on("DESDE_SERVER_COMANDO", procesar_comando_recibido);

  Serial.println("Sistema listo para contar piezas con sensor infrarrojo.");

}

void loop() {

  now = millis();

  leerSensorIR();

  socketIO.loop();

}

void leerSensorIR() {

  int lectura = digitalRead(SENSOR_IR);

  if (lectura != estadoAnterior) {

    tiempoUltimoCambio = millis();

  }

  if ((millis() - tiempoUltimoCambio) > tiempoAntirrebote) {

    if (lectura != estadoActual) {

      estadoActual = lectura;

      if (estadoActual == LOW) {

        contador++;

        digitalWrite(ONBOARD_LED, HIGH);

        Serial.print("Pieza detectada. Conteo local: ");
        Serial.println(contador);

        mensaje = "\"" + String(contador) + "\"";

        socketIO.emit("DESDE_ESP32_CONTADOR", mensaje.c_str());

        delay(80);

        digitalWrite(ONBOARD_LED, LOW);

      }

    }

  }

  estadoAnterior = lectura;

}

void connectWiFi_STA() {

  delay(10);

  Serial.println("");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);

  Serial.print("Conectando a WiFi");

  while (WiFi.status() != WL_CONNECTED) {

    delay(500);
    Serial.print(".");

  }

  Serial.println("");
  Serial.print("WiFi conectado a: ");
  Serial.println(ssid);

  Serial.print("IP del ESP32: ");
  Serial.println(WiFi.localIP());

}

void procesar_comando_recibido(const char * payload, size_t length) {

  Serial.printf("Mensaje recibido desde servidor: %s\n", payload);

}
```

---

# 18. Explicación del código del ESP32

Esta línea incluye la conexión WiFi:

```cpp
#include <WiFi.h>
```

Esta línea incluye la librería para Socket.IO:

```cpp
#include <SocketIoClient.h>
```

Estos datos deben modificarse según la red del alumno:

```cpp
const char* ssid      = "Nombre-red";
const char* password  = "password-red";
const char* server    = "ip-del-server";
const uint16_t port   = 5001;
```

El sensor infrarrojo se conecta al GPIO 18:

```cpp
#define SENSOR_IR 18
```

El contador inicia en cero:

```cpp
int contador = 0;
```

El sensor se configura con resistencia interna pull-up:

```cpp
pinMode(SENSOR_IR, INPUT_PULLUP);
```

Cuando el sensor detecta una pieza, el contador aumenta:

```cpp
contador++;
```

El dato se prepara para enviarse al servidor:

```cpp
mensaje = "\"" + String(contador) + "\"";
```

El ESP32 envía el evento al servidor:

```cpp
socketIO.emit("DESDE_ESP32_CONTADOR", mensaje.c_str());
```

El nombre del evento debe coincidir exactamente con el del servidor:

```text
DESDE_ESP32_CONTADOR
```

---

# 19. Orden correcto de prueba

Para probar el sistema se recomienda seguir este orden:

## Paso 1

Conectar el sensor infrarrojo al ESP32.

## Paso 2

Ejecutar el servidor:

```bash
node server.js
```

## Paso 3

Abrir la página web:

```text
http://localhost:5001
```

## Paso 4

Cargar el código al ESP32.

## Paso 5

Abrir el monitor serial del Arduino IDE.

## Paso 6

Activar el sensor infrarrojo pasando un objeto frente a él.

## Paso 7

Observar el conteo en:

```text
Monitor serial del ESP32
Consola del servidor Node.js
Página web
```

---

# 20. Salidas esperadas

## Monitor serial del ESP32

```text
WiFi conectado a: Nombre-red
IP del ESP32: 192.168.1.90
Sistema listo para contar piezas con sensor infrarrojo.
Pieza detectada. Conteo local: 1
Pieza detectada. Conteo local: 2
Pieza detectada. Conteo local: 3
```

---

## Consola del servidor

```text
======================================
 Servidor Industria 4.0 iniciado
 Dirección: http://localhost:5001
 Esperando datos del ESP32...
======================================

Cliente conectado: 0gPZxN88A3GkffAAAA
Cliente conectado: 7fYkM9wPqK20sdBBBB
Conteo recibido desde ESP32: 1
Conteo recibido desde ESP32: 2
Conteo recibido desde ESP32: 3
```

---

## Página web

La página deberá mostrar:

```text
Piezas detectadas: 1
Origen: ESP32
Hora: hora-del-evento
Estado: Proceso en operación. Conteo activo.
```

Cuando el conteo llegue a 10:

```text
Meta inicial alcanzada: 10 piezas o más.
```

Cuando el conteo llegue a 20:

```text
Producción alta detectada: 20 piezas o más.
```

---

# 21. Errores comunes

## El ESP32 no se conecta al WiFi

Verificar:

```cpp
const char* ssid = "Nombre-red";
const char* password = "password-red";
```

También verificar que la red sea de 2.4 GHz, ya que muchos ESP32 no se conectan a redes de 5 GHz.

---

## El ESP32 no se conecta al servidor

Verificar que la IP sea correcta:

```cpp
const char* server = "192.168.1.75";
```

También verificar que el servidor esté corriendo:

```bash
node server.js
```

La computadora y el ESP32 deben estar en la misma red.

---

## La página abre, pero no cambia el contador

Verificar que el servidor esté recibiendo datos desde el ESP32.

En la consola de Node.js deberá aparecer:

```text
Conteo recibido desde ESP32
```

Si no aparece, el problema está entre el ESP32 y el servidor.

---

## El servidor recibe datos, pero la página no cambia

Verificar que en `cliente.js` exista:

```js
socket.on('DESDE_SERVER_CONTADOR', function(data) {
```

También verificar que en `server.js` exista:

```js
io.emit('DESDE_SERVER_CONTADOR', {
```

El nombre del evento debe coincidir exactamente.

---

## El sensor cuenta muchas veces con una sola detección

Aumentar el tiempo de antirrebote:

```cpp
uint64_t tiempoAntirrebote = 120;
```

También se puede agregar una pausa mayor después de cada detección.

---

## El sensor no cuenta

Verificar si el sensor entrega `LOW` o `HIGH` cuando detecta.

Actualmente el código cuenta cuando:

```cpp
if (estadoActual == LOW)
```

Si el sensor funciona al revés, cambiar por:

```cpp
if (estadoActual == HIGH)
```

---

# 22. Actividad de comprobación

El estudiante deberá modificar la página para agregar una nueva sección llamada:

```text
Información del proceso
```

La sección deberá mostrar:

* Nombre del estudiante.
* Nombre de la línea de producción.
* Meta de piezas.
* Estado actual del proceso.

Ejemplo:

```html
<section class="tarjeta">

    <h2>Información del proceso</h2>

    <p>Operador: Nombre del estudiante</p>

    <p>Línea: Banda transportadora 1</p>

    <p>Meta: 20 piezas</p>

    <p>Estado: En operación</p>

</section>
```

---

# 23. Reto adicional

Modificar la página para que el contador cambie de color según el valor recibido:

```text
0 a 9 piezas      → verde
10 a 19 piezas    → naranja
20 o más piezas   → azul o morado
```

Sugerencia:

Agregar clases CSS nuevas:

```css
.contador-verde {
    color: #087a36;
}

.contador-naranja {
    color: #f0ad4e;
}

.contador-alto {
    color: #5b2c83;
}
```

Después modificar `cliente.js` para cambiar la clase según el valor recibido.

---

# 24. Conclusión

En esta práctica se integró el ESP32 con una página web mediante un servidor Node.js y `socket.io@1.7.2`.

El ESP32 detectó activaciones de un sensor infrarrojo, incrementó un contador y envió el dato al servidor.

El servidor recibió el evento `DESDE_ESP32_CONTADOR` y lo reenvió a la página web mediante el evento `DESDE_SERVER_CONTADOR`.

La página web actualizó el contador de forma automática sin recargar el navegador.

Esta práctica representa la base de un sistema de monitoreo industrial en tiempo real, similar a un tablero básico utilizado en aplicaciones de Industria 4.0.
