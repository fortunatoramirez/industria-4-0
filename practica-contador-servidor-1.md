# Práctica: Conteo de piezas con ESP32 y envío de datos a un servidor Node.js

## Objetivo

Implementar un sistema básico de conteo de eventos utilizando un ESP32 y un push button, simulando el paso de piezas por una banda transportadora. El ESP32 deberá detectar cada activación del botón, incrementar un contador y enviar el valor actualizado a un servidor desarrollado en Node.js mediante comunicación en tiempo real con `socket.io`.

---

# 1. Introducción

En los sistemas de Industria 4.0 es común utilizar sensores para monitorear procesos industriales. Un ejemplo sencillo es el conteo de piezas que pasan por una banda transportadora.

En una aplicación real, este conteo podría realizarse con sensores como:

* Sensor infrarrojo.
* Sensor inductivo.
* Sensor capacitivo.
* Sensor fotoeléctrico.
* Encoder.
* Sensor magnético.

En esta práctica, como aún no se cuenta con el sensor físico, se usará un push button para simular la activación del sensor. Cada vez que el botón sea presionado, el ESP32 aumentará un contador y enviará ese dato al servidor.

El servidor recibirá el dato y lo mostrará en la consola.

---

# 2. Material necesario

* 1 ESP32.
* 1 push button.
* 1 resistencia de 10 kΩ, opcional.
* Cables Dupont.
* Protoboard.
* Computadora con Node.js instalado.
* Arduino IDE.
* Librería `SocketIoClient` instalada en Arduino IDE.
* Red WiFi local.

---

# 3. Descripción general del sistema

El sistema tendrá la siguiente estructura:

```text
Push button → ESP32 → WiFi → Servidor Node.js → Consola
```

El push button simula un sensor colocado en una banda transportadora.

Cada vez que se presiona el botón:

```text
Se detecta una activación
Se incrementa el contador
Se envía el conteo al servidor
El servidor imprime el dato en consola
```

---

# 4. Conexión del push button al ESP32

Se utilizará la resistencia interna `INPUT_PULLUP` del ESP32.

Por eso, la conexión será:

```text
Push button:
Un lado  → GPIO 18
Otro lado → GND
```

No es necesario conectar una resistencia externa, ya que el ESP32 activará una resistencia interna de pull-up.

## Funcionamiento lógico

Cuando el botón no está presionado:

```text
GPIO 18 = HIGH
```

Cuando el botón está presionado:

```text
GPIO 18 = LOW
```

Por eso, en el código se contará una activación cuando el pin cambie a `LOW`.

---

# 5. Preparación del servidor Node.js

Primero se debe crear una carpeta para el servidor.

Ejemplo:

```bash
servidor_contador_esp32
```

Dentro de esa carpeta, abrir una terminal y ejecutar:

```bash
npm init -y
```

Después instalar las librerías necesarias:

```bash
npm install express socket.io@1.7.2
```

Se utiliza `socket.io@1.7.2` porque es compatible con la librería utilizada en el ESP32.

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
var server = require('http').Server(app);
var io = require('socket.io')(server);

var puerto = 5001;

// Esta línea deja preparado el servidor para usar después una carpeta public
app.use(express.static('public'));

io.on('connection', function(socket) {

    console.log('Cliente conectado:', socket.id);

    socket.on('DESDE_ESP32_CONTADOR', function(data) {
        console.log('Conteo recibido desde ESP32:', data);
    });

    socket.on('disconnect', function() {
        console.log('Cliente desconectado:', socket.id);
    });

});

server.listen(puerto, function() {
    console.log('Servidor corriendo en el puerto ' + puerto);
});
```

---

# 7. Explicación del servidor

Esta línea importa Express:

```js
var express = require('express');
```

Express permitirá que más adelante el servidor pueda entregar páginas HTML, archivos CSS, JavaScript, imágenes y otros recursos.

Esta línea crea el servidor HTTP:

```js
var server = require('http').Server(app);
```

Esta línea integra `socket.io` al servidor:

```js
var io = require('socket.io')(server);
```

Esta línea deja preparada la carpeta `public`:

```js
app.use(express.static('public'));
```

Aunque en esta práctica todavía no se usará una página web, más adelante se podrá crear una carpeta `public` para colocar vistas HTML.

Esta parte detecta cuando un cliente se conecta:

```js
io.on('connection', function(socket) {
```

En este caso, el cliente será el ESP32.

Esta parte escucha los datos enviados desde el ESP32:

```js
socket.on('DESDE_ESP32_CONTADOR', function(data) {
    console.log('Conteo recibido desde ESP32:', data);
});
```

El nombre del evento debe coincidir exactamente con el nombre usado en el ESP32:

```text
DESDE_ESP32_CONTADOR
```

---

# 8. Ejecución del servidor

Para ejecutar el servidor, usar:

```bash
node server.js
```

Si todo está correcto, deberá aparecer:

```text
Servidor corriendo en el puerto 5001
```

Cuando el ESP32 se conecte, deberá aparecer algo similar a:

```text
Cliente conectado: I7GPWm8T2GxL5QAAAA
```

---

# 9. Identificar la IP del servidor

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

# 10. Código del ESP32

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
#define PUSH_BUTTON  18

/******************/

SocketIoClient socketIO;

String mensaje;

int contador = 0;

int estadoActual = HIGH;
int estadoAnterior = HIGH;

uint64_t tiempoUltimoCambio = 0;
uint64_t tiempoAntirrebote = 50;

/******************/

void setup() {
  Serial.begin(115200);

  connectWiFi_STA();

  socketIO.begin(server, port);

  socketIO.on("DESDE_SERVER_COMANDO", procesar_comando_recibido);

  pinMode(ONBOARD_LED, OUTPUT);
  pinMode(PUSH_BUTTON, INPUT_PULLUP);

  Serial.println("Sistema listo para contar pulsaciones.");
}

void loop() {
  now = millis();

  leerPushButton();

  socketIO.loop();
}

void leerPushButton() {
  int lectura = digitalRead(PUSH_BUTTON);

  if (lectura != estadoAnterior) {
    tiempoUltimoCambio = millis();
  }

  if ((millis() - tiempoUltimoCambio) > tiempoAntirrebote) {
    if (lectura != estadoActual) {
      estadoActual = lectura;

      if (estadoActual == LOW) {
        contador++;

        Serial.print("Conteo local: ");
        Serial.println(contador);

        mensaje = "\"" + String(contador) + "\"";
        socketIO.emit("DESDE_ESP32_CONTADOR", mensaje.c_str());
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

  while (WiFi.status() != WL_CONNECTED) {
    delay(100);
    Serial.print('.');
  }

  Serial.println("");
  Serial.print("Iniciado STA:\t");
  Serial.println(ssid);
  Serial.print("IP address:\t");
  Serial.println(WiFi.localIP());
}

void procesar_comando_recibido(const char * payload, size_t length) {
  Serial.printf("Mensaje recibido: %s\n", payload);

  String paystring = String(payload);

  if (paystring == "ON") {
    digitalWrite(ONBOARD_LED, HIGH);
  }
  else if (paystring == "OFF") {
    digitalWrite(ONBOARD_LED, LOW);
  }
}
```

---

# 11. Explicación del código del ESP32

Esta línea incluye la librería WiFi:

```cpp
#include <WiFi.h>
```

Esta línea incluye la librería para conectarse al servidor mediante socket.io:

```cpp
#include <SocketIoClient.h>
```

Estos datos deben modificarse de acuerdo con la red y la computadora del alumno:

```cpp
const char* ssid      = "Nombre-red";
const char* password  = "password-red";
const char* server    = "ip-del-server";
const uint16_t port   = 5001;
```

El pin del LED integrado se define así:

```cpp
#define ONBOARD_LED  2
```

El push button se conecta al GPIO 18:

```cpp
#define PUSH_BUTTON  18
```

El contador inicia en cero:

```cpp
int contador = 0;
```

Las variables siguientes ayudan a detectar correctamente el cambio de estado del botón:

```cpp
int estadoActual = HIGH;
int estadoAnterior = HIGH;
```

Esta variable define el tiempo de antirrebote:

```cpp
uint64_t tiempoAntirrebote = 50;
```

El antirrebote evita que una sola pulsación se cuente varias veces por el ruido mecánico del botón.

---

# 12. Función `leerPushButton()`

La función principal para contar las pulsaciones es:

```cpp
void leerPushButton() {
  int lectura = digitalRead(PUSH_BUTTON);

  if (lectura != estadoAnterior) {
    tiempoUltimoCambio = millis();
  }

  if ((millis() - tiempoUltimoCambio) > tiempoAntirrebote) {
    if (lectura != estadoActual) {
      estadoActual = lectura;

      if (estadoActual == LOW) {
        contador++;

        Serial.print("Conteo local: ");
        Serial.println(contador);

        mensaje = "\"" + String(contador) + "\"";
        socketIO.emit("DESDE_ESP32_CONTADOR", mensaje.c_str());
      }
    }
  }

  estadoAnterior = lectura;
}
```

Esta función hace lo siguiente:

1. Lee el estado del botón.
2. Detecta si hubo un cambio.
3. Espera un pequeño tiempo para evitar rebotes.
4. Verifica si el botón fue presionado.
5. Incrementa el contador.
6. Imprime el conteo en el monitor serial.
7. Envía el dato al servidor.

---

# 13. Prueba del sistema

Para probar la práctica:

1. Conectar el push button al ESP32.
2. Encender el ESP32.
3. Ejecutar el servidor con:

```bash
node server.js
```

4. Abrir el monitor serial del Arduino IDE.
5. Presionar el botón varias veces.
6. Observar el conteo en el monitor serial.
7. Observar el conteo en la consola del servidor.

En el monitor serial del ESP32 deberá aparecer algo similar a:

```text
Sistema listo para contar pulsaciones.
Conteo local: 1
Conteo local: 2
Conteo local: 3
```

En la consola del servidor deberá aparecer:

```text
Conteo recibido desde ESP32: 1
Conteo recibido desde ESP32: 2
Conteo recibido desde ESP32: 3
```

---

# 14. Errores comunes

## El ESP32 no se conecta al WiFi

Verificar:

```cpp
const char* ssid = "Nombre-red";
const char* password = "password-red";
```

También revisar que la red sea compatible con el ESP32. El ESP32 normalmente se conecta a redes WiFi de 2.4 GHz.

---

## El servidor no recibe datos

Verificar que el servidor esté corriendo:

```bash
node server.js
```

También verificar que la IP sea correcta:

```cpp
const char* server = "192.168.1.75";
```

La computadora y el ESP32 deben estar en la misma red.

---

## El botón cuenta varias veces con una sola pulsación

Aumentar el tiempo de antirrebote:

```cpp
uint64_t tiempoAntirrebote = 80;
```

o incluso:

```cpp
uint64_t tiempoAntirrebote = 100;
```

---

## El botón no cuenta

Verificar la conexión:

```text
GPIO 18 → Push button → GND
```

También verificar que se esté usando:

```cpp
pinMode(PUSH_BUTTON, INPUT_PULLUP);
```

---

# 15. Actividad de comprobación

El estudiante deberá modificar el sistema para agregar una de las siguientes mejoras:

## Opción A

Agregar un segundo botón conectado a otro pin del ESP32 para reiniciar el contador a cero.

Cuando se presione ese botón, el ESP32 deberá enviar al servidor:

```text
0
```

## Opción B

Modificar el código para que el ESP32 también encienda el LED integrado cada vez que detecte una pieza.

El LED deberá encender brevemente y luego apagarse.

## Opción C

Modificar el servidor para que muestre un mensaje especial cuando el contador llegue a 10.

Ejemplo:

```text
Meta alcanzada: 10 piezas detectadas
```

---

# 16. Conclusión

En esta práctica se implementó un sistema básico de adquisición y transmisión de datos usando un ESP32, una entrada digital y un servidor Node.js. El push button permitió simular un sensor industrial encargado de detectar el paso de piezas por una banda transportadora.

El ESP32 realizó el conteo localmente y envió el valor al servidor mediante `socket.io`. El servidor recibió los datos en tiempo real y los mostró en la consola.

Esta práctica sirve como base para desarrollar sistemas más completos de monitoreo industrial, donde posteriormente se podrán integrar sensores reales, interfaces web, almacenamiento en base de datos y tableros de visualización para aplicaciones de Industria 4.0.
