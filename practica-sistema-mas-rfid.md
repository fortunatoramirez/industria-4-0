# Práctica: Identificación automática de cajas mediante RFID

## Integración del lector RC522 con la estación de pesaje, ESP32, Node.js y MySQL

---

# 1. Introducción

En las prácticas anteriores se desarrolló un sistema capaz de:

* Administrar productos y usuarios.
* Detectar piezas sobre una banda transportadora.
* Medir la altura de cada pieza.
* Medir el peso individual mediante la diferencia del peso acumulado.
* Clasificar las piezas de acuerdo con sus especificaciones.
* Guardar las mediciones en MySQL.
* Mostrar resultados y estadísticas en páginas web.

En la práctica de peso, la caja receptora se identificó mediante un valor escrito directamente en el programa:

```cpp
const char* NOMBRE_CAJA =
    "CAJA_01";
```

Este método funciona para una sola caja, pero obliga a modificar y volver a cargar el programa cada vez que se cambia el recipiente.

En esta práctica se utilizará un lector RFID RC522 para identificar automáticamente la caja colocada en la estación de pesaje.

Cada caja tendrá asociado un llavero o una tarjeta RFID. Cuando el operador acerque el identificador al lector, el ESP32 obtendrá su UID y lo enviará al servidor.

El servidor determinará:

* Si la caja está registrada.
* El nombre de la caja.
* Si se encuentra activa.
* La capacidad asignada.
* Si puede comenzar una nueva sesión de pesaje.

El MFRC522 trabaja con comunicación sin contacto a 13.56 MHz y admite etiquetas compatibles con ISO/IEC 14443 A, incluyendo distintas familias MIFARE y NTAG.

---

# 2. Objetivo general

Desarrollar un sistema de identificación automática de cajas mediante RFID que permita asociar las mediciones de peso con la caja receptora correspondiente.

---

# 3. Objetivos específicos

Al finalizar la práctica, el estudiante será capaz de:

* Conectar un lector RFID RC522 al ESP32.
* Utilizar el protocolo SPI.
* Leer el UID de una tarjeta o llavero RFID.
* Convertir el UID en una cadena de texto.
* Enviar el identificador al servidor Node.js.
* Registrar cajas y sus UID en MySQL.
* Validar si una caja está autorizada.
* Realizar la tara solamente después de identificar una caja válida.
* Asociar cada medición de peso con una caja.
* Registrar intentos correctos e incorrectos de identificación.
* Mostrar las lecturas RFID en tiempo real.
* Cambiar de caja sin modificar el programa del ESP32.

---

# 4. Funcionamiento general

El proceso será el siguiente:

```text
El operador retira la caja llena
                │
                ▼
Coloca una caja vacía sobre la báscula
                │
                ▼
Acerca el llavero RFID de la caja
                │
                ▼
El RC522 obtiene el UID
                │
                ▼
El ESP32 envía el UID a Node.js
                │
                ▼
Node.js busca el UID en MySQL
                │
        ┌───────┴────────┐
        ▼                ▼
Caja registrada     Caja desconocida
        │                │
        ▼                ▼
Realizar tara       Rechazar identificación
        │
        ▼
Activar la estación
        │
        ▼
Las piezas comienzan a caer
        │
        ▼
Cada peso se relaciona con la caja
```

---

# 5. Resultado esperado

Al acercar un llavero registrado, el monitor serial deberá mostrar algo parecido a:

```text
UID leído: A4D38B21

Caja identificada:
Nombre: CAJA_03
Capacidad: 5000 g

Realizando tara...
Tara terminada.
Estación preparada para recibir piezas.
```

Si se acerca un llavero no registrado:

```text
UID leído: 7312B4C9
Caja no registrada.
La estación permanecerá detenida.
```

Cuando una pieza caiga:

```text
Caja: CAJA_03
UID: A4D38B21
Número de pieza: 1
Peso de la pieza: 102.40 g
Peso acumulado: 102.40 g
```

---

# 6. Material necesario

## Hardware

* Un ESP32.
* Un lector RFID RC522.
* Al menos un llavero o tarjeta RFID.
* Celda de carga.
* Módulo HX711.
* Caja receptora.
* Plataforma de pesaje.
* Protoboard.
* Cables Dupont.
* Banda transportadora.
* Piezas para realizar las pruebas.

## Software

* Arduino IDE.
* Node.js.
* MySQL Server.
* Visual Studio Code.
* Navegador web.
* Proyecto desarrollado en las prácticas anteriores.

---

# 7. Organización de la estación

En esta práctica se recomienda utilizar el mismo ESP32 de la estación de pesaje.

```text
                 ESP32
        ┌──────────┴──────────┐
        │                     │
        ▼                     ▼
      HX711                 RC522
        │                     │
        ▼                     ▼
Celda de carga        Llavero de la caja
```

La estación de altura puede continuar funcionando con otro ESP32.

Esto evita conflictos con los pines utilizados anteriormente por el HC-SR04.

---

# 8. Colocación del lector RFID

El lector deberá colocarse cerca de la posición de la caja, pero sin interferir con la celda de carga.

Una posible instalación es:

```text
Vista lateral

        Caja receptora
   ┌─────────────────────┐
   │                     │◄── Llavero RFID
   │                     │
   └──────────┬──────────┘
              │
       Plataforma de peso
   ───────────────────────
              │
       Celda de carga
              │
         Base rígida

                    ┌─────────┐
                    │  RC522  │
                    └─────────┘
```

## Recomendaciones

* Colocar el llavero en un costado de la caja.
* Mantener el lector a una distancia corta.
* Evitar colocar el RC522 directamente sobre metal.
* Mantenerlo alejado del motor de la banda.
* Sujetarlo para evitar movimientos.
* Comprobar que el llavero pueda leerse con la caja en su posición normal.
* Evitar que los cables del lector ejerzan fuerza sobre la báscula.

---

# 9. Identificación mediante UID

Cada tarjeta o llavero tiene un identificador denominado UID.

Ejemplo:

```text
A4 D3 8B 21
```

Para enviarlo al servidor se utilizará el siguiente formato:

```text
A4D38B21
```

El UID será tratado como una cadena y no como un número.

Esto permite conservar:

* Letras hexadecimales.
* Ceros iniciales.
* Diferentes longitudes de UID.

> En esta práctica el UID se utiliza para identificar una caja dentro de un sistema didáctico. No debe considerarse un mecanismo de autenticación de alta seguridad, porque algunos identificadores RFID pueden copiarse o emularse. La documentación de la biblioteca también advierte que el UID no es adecuado por sí solo para aplicaciones de seguridad.

---

# 10. Conexión del RC522 al ESP32

El RC522 utiliza comunicación SPI.

## Tabla de conexión

| RC522    |        ESP32 | Función                   |
| -------- | -----------: | ------------------------- |
| 3.3V     |        3.3 V | Alimentación              |
| GND      |          GND | Tierra                    |
| SDA o SS |       GPIO 5 | Selección del dispositivo |
| SCK      |      GPIO 18 | Reloj SPI                 |
| MOSI     |      GPIO 23 | Datos del ESP32 al RC522  |
| MISO     |      GPIO 19 | Datos del RC522 al ESP32  |
| RST      |      GPIO 22 | Reinicio                  |
| IRQ      | Sin conexión | No se utilizará           |

Los pines seleccionados corresponden al bus SPI comúnmente utilizado por el ESP32: SCK en GPIO 18, MISO en GPIO 19, MOSI en GPIO 23 y selección en GPIO 5.

## Importante

El pin marcado como:

```text
SDA
```

en muchos módulos RC522 funciona como:

```text
SS
CS
SDA/SS
```

cuando el módulo se utiliza mediante SPI.

No corresponde al pin SDA de una comunicación I²C.

## Alimentación

El RC522 deberá alimentarse con:

```text
3.3 V
```

No deberá conectarse a 5 V. El circuito MFRC522 está diseñado para trabajar con alimentación de bajo voltaje y la conexión de 3.3 V también mantiene compatibilidad con las señales del ESP32.

---

# 11. Conexión completa de la estación

## HX711

| HX711 |   ESP32 |
| ----- | ------: |
| VCC   |   3.3 V |
| GND   |     GND |
| DOUT  | GPIO 16 |
| SCK   | GPIO 17 |

## RC522

| RC522  |        ESP32 |
| ------ | -----------: |
| 3.3V   |        3.3 V |
| GND    |          GND |
| SDA/SS |       GPIO 5 |
| SCK    |      GPIO 18 |
| MOSI   |      GPIO 23 |
| MISO   |      GPIO 19 |
| RST    |      GPIO 22 |
| IRQ    | Sin conexión |

Todos los módulos deberán compartir:

```text
GND común
```

---

# Bloque 1. Instalar la biblioteca

## 12. Instalar la biblioteca MFRC522

Desde Arduino IDE:

1. Abrir el administrador de bibliotecas.
2. Buscar:

```text
MFRC522
```

3. Instalar:

```text
MFRC522
```

4. Comprobar que se reconozca:

```cpp
#include <MFRC522.h>
```

La biblioteca MFRC522 disponible en la documentación de Arduino utiliza comunicación SPI y proporciona ejemplos para leer tarjetas y obtener su UID.

También se utilizará:

```cpp
#include <SPI.h>
```

La biblioteca `SPI` ya forma parte del entorno del ESP32.

---

# Bloque 2. Probar solamente el RC522

## 13. Código para leer el UID

Antes de integrarlo con la báscula y el servidor, se comprobará únicamente el lector RFID.

```cpp
#include <SPI.h>
#include <MFRC522.h>

const int PIN_SS = 5;
const int PIN_RST = 22;

const int PIN_SCK = 18;
const int PIN_MISO = 19;
const int PIN_MOSI = 23;

MFRC522 lectorRFID(
    PIN_SS,
    PIN_RST
);

String convertirUidATexto()
{
    String uid = "";

    for(
        byte i = 0;
        i < lectorRFID.uid.size;
        i++
    )
    {
        if(
            lectorRFID.uid.uidByte[i] <
            0x10
        )
        {
            uid += "0";
        }

        uid += String(
            lectorRFID.uid.uidByte[i],
            HEX
        );
    }

    uid.toUpperCase();

    return uid;
}

void setup()
{
    Serial.begin(115200);

    SPI.begin(
        PIN_SCK,
        PIN_MISO,
        PIN_MOSI,
        PIN_SS
    );

    lectorRFID.PCD_Init();

    delay(100);

    Serial.println();
    Serial.println(
        "LECTOR RFID PREPARADO"
    );

    Serial.println(
        "Acerque una tarjeta o llavero."
    );
}

void loop()
{
    if(
        !lectorRFID.PICC_IsNewCardPresent()
    )
    {
        return;
    }

    if(
        !lectorRFID.PICC_ReadCardSerial()
    )
    {
        return;
    }

    String uid =
        convertirUidATexto();

    Serial.print(
        "UID: "
    );

    Serial.println(uid);

    lectorRFID.PICC_HaltA();

    lectorRFID.PCD_StopCrypto1();

    delay(500);
}
```

---

# 14. Probar los identificadores

Acercar cada llavero y completar la tabla:

| Llavero o tarjeta | UID obtenido | Caja asignada |
| ----------------- | ------------ | ------------- |
| Llavero 1         |              | CAJA_01       |
| Llavero 2         |              | CAJA_02       |
| Llavero 3         |              | CAJA_03       |
| Llavero 4         |              |               |

Ejemplo:

```text
Llavero 1
UID: A4D38B21
Caja: CAJA_01
```

No continuar hasta que el lector muestre correctamente los UID.

---

# Bloque 3. Preparar la base de datos

## 15. Entrar a MySQL

```bash
mysql -u root -p
```

Seleccionar la base de datos:

```sql
USE industria40_web;
```

---

# 16. Crear la tabla de cajas

Ejecutar:

```sql
CREATE TABLE cajas (
    id INT AUTO_INCREMENT PRIMARY KEY,

    nombre VARCHAR(100) NOT NULL,

    uid_rfid VARCHAR(32) NOT NULL UNIQUE,

    capacidad_max_g DECIMAL(10,2) NULL,

    estado VARCHAR(20) NOT NULL
        DEFAULT 'activa',

    fecha_registro TIMESTAMP
        DEFAULT CURRENT_TIMESTAMP
);
```

El campo `estado` podrá contener:

```text
activa
inactiva
```

---

# 17. Crear la tabla de eventos RFID

Ejecutar:

```sql
CREATE TABLE eventos_rfid (
    id INT AUTO_INCREMENT PRIMARY KEY,

    caja_id INT NULL,

    uid_rfid VARCHAR(32) NOT NULL,

    dispositivo VARCHAR(50) NOT NULL,

    resultado VARCHAR(30) NOT NULL,

    fecha TIMESTAMP
        DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY (caja_id)
        REFERENCES cajas(id)
        ON DELETE SET NULL
);
```

El campo `resultado` podrá contener:

```text
identificada
no_registrada
inactiva
```

---

# 18. Relacionar las mediciones de peso con las cajas

La tabla `mediciones_peso` ya contiene una columna de texto llamada:

```text
caja
```

Se conservará para no afectar los registros anteriores.

Agregar las nuevas columnas:

```sql
ALTER TABLE mediciones_peso
ADD caja_id INT NULL
    AFTER producto_id,

ADD uid_rfid VARCHAR(32) NULL
    AFTER caja_id;
```

Agregar la relación:

```sql
ALTER TABLE mediciones_peso
ADD CONSTRAINT fk_mediciones_peso_caja
FOREIGN KEY (caja_id)
REFERENCES cajas(id)
ON DELETE SET NULL;
```

Estos comandos deberán ejecutarse una sola vez.

---

# 19. Verificar las tablas

```sql
SHOW TABLES;
```

Comprobar:

```sql
DESCRIBE cajas;
```

```sql
DESCRIBE eventos_rfid;
```

```sql
DESCRIBE mediciones_peso;
```

Salir:

```sql
EXIT;
```

---

# Bloque 4. Modificar el servidor

## 20. Normalizar los UID

Agregar en `server.js`, cerca de las funciones auxiliares:

```javascript
function normalizarUid(valor)
{
    return String(valor || "")
        .trim()
        .toUpperCase()
        .replace(
            /[^0-9A-F]/g,
            ""
        );
}
```

Esta función permite que los siguientes valores sean interpretados de la misma manera:

```text
A4 D3 8B 21
A4:D3:8B:21
a4d38b21
```

Resultado:

```text
A4D38B21
```

---

# 21. Crear la ruta de la página de cajas

Agregar:

```javascript
app.get(
    "/cajas",
    requiereSesionPagina,
    (req, res) =>
    {
        enviarPagina(
            res,
            "cajas.html"
        );
    }
);
```

---

# 22. API para registrar una caja

Agregar:

```javascript
// ==================================================
// API: REGISTRAR UNA CAJA
// ==================================================

app.post(
    "/api/cajas",
    requiereSesionAPI,
    async (req, res) =>
    {
        try
        {
            const nombre =
                String(
                    req.body.nombre || ""
                ).trim();

            const uid =
                normalizarUid(
                    req.body.uid_rfid
                );

            const capacidadTexto =
                String(
                    req.body.capacidad_max_g ??
                    ""
                ).trim();

            let capacidad = null;

            if(capacidadTexto !== "")
            {
                capacidad =
                    Number(capacidadTexto);
            }

            if(
                nombre === "" ||
                nombre.includes("|")
            )
            {
                return res.status(400).json({
                    correcto: false,
                    mensaje:
                        "El nombre de la caja es inválido"
                });
            }

            if(
                uid.length < 4 ||
                uid.length > 32
            )
            {
                return res.status(400).json({
                    correcto: false,
                    mensaje:
                        "El UID es inválido"
                });
            }

            if(
                capacidad !== null &&
                (
                    !Number.isFinite(capacidad) ||
                    capacidad <= 0
                )
            )
            {
                return res.status(400).json({
                    correcto: false,
                    mensaje:
                        "La capacidad debe ser mayor que cero"
                });
            }

            await conexion.execute(
                `INSERT INTO cajas
                (
                    nombre,
                    uid_rfid,
                    capacidad_max_g,
                    estado
                )
                VALUES (?, ?, ?, ?)`,
                [
                    nombre,
                    uid,
                    capacidad,
                    "activa"
                ]
            );

            res.json({
                correcto: true,
                mensaje:
                    "Caja registrada correctamente"
            });
        }
        catch(error)
        {
            console.log(error);

            if(error.code === "ER_DUP_ENTRY")
            {
                return res.status(409).json({
                    correcto: false,
                    mensaje:
                        "El UID ya está relacionado con una caja"
                });
            }

            res.status(500).json({
                correcto: false,
                mensaje:
                    "No fue posible registrar la caja"
            });
        }
    }
);
```

---

# 23. API para consultar las cajas

Agregar:

```javascript
// ==================================================
// API: CONSULTAR CAJAS
// ==================================================

app.get(
    "/api/cajas",
    requiereSesionAPI,
    async (req, res) =>
    {
        try
        {
            const [cajas] =
                await conexion.execute(
                    `SELECT
                        id,
                        nombre,
                        uid_rfid,
                        capacidad_max_g,
                        estado,
                        fecha_registro
                    FROM cajas
                    ORDER BY nombre`
                );

            res.json(cajas);
        }
        catch(error)
        {
            console.log(error);

            res.status(500).json({
                mensaje:
                    "No fue posible consultar las cajas"
            });
        }
    }
);
```

---

# 24. API para identificar una caja

Agregar:

```javascript
// ==================================================
// API: IDENTIFICAR CAJA MEDIANTE RFID
// ==================================================

app.post(
    "/api/rfid/identificar",
    requiereClaveDispositivo,
    async (req, res) =>
    {
        try
        {
            const uid =
                normalizarUid(
                    req.body.uid_rfid
                );

            const dispositivo =
                String(
                    req.body.dispositivo || ""
                ).trim();

            if(
                uid.length < 4 ||
                uid.length > 32 ||
                dispositivo === ""
            )
            {
                return res.status(400)
                    .type("text/plain")
                    .send("DATOS_INVALIDOS");
            }

            const [cajas] =
                await conexion.execute(
                    `SELECT
                        id,
                        nombre,
                        uid_rfid,
                        capacidad_max_g,
                        estado
                    FROM cajas
                    WHERE uid_rfid = ?`,
                    [uid]
                );

            let cajaId = null;
            let resultado = "no_registrada";
            let caja = null;

            if(cajas.length > 0)
            {
                caja = cajas[0];
                cajaId = caja.id;

                if(caja.estado === "activa")
                {
                    resultado =
                        "identificada";
                }
                else
                {
                    resultado =
                        "inactiva";
                }
            }

            await conexion.execute(
                `INSERT INTO eventos_rfid
                (
                    caja_id,
                    uid_rfid,
                    dispositivo,
                    resultado
                )
                VALUES (?, ?, ?, ?)`,
                [
                    cajaId,
                    uid,
                    dispositivo,
                    resultado
                ]
            );

            const evento = {
                uid_rfid:
                    uid,

                dispositivo:
                    dispositivo,

                resultado:
                    resultado,

                caja:
                    caja
                    ? caja.nombre
                    : null,

                capacidad_max_g:
                    caja
                    ? caja.capacidad_max_g
                    : null,

                fecha:
                    new Date().toISOString()
            };

            io.emit(
                "lectura_rfid",
                evento
            );

            if(resultado === "no_registrada")
            {
                return res.status(404)
                    .type("text/plain")
                    .send("NO_REGISTRADA");
            }

            if(resultado === "inactiva")
            {
                return res.status(403)
                    .type("text/plain")
                    .send("CAJA_INACTIVA");
            }

            const capacidad =
                caja.capacidad_max_g === null
                ? 0
                : Number(
                    caja.capacidad_max_g
                );

            res.type("text/plain").send(
                `OK|${caja.id}|${caja.nombre}|${capacidad}`
            );
        }
        catch(error)
        {
            console.log(error);

            res.status(500)
                .type("text/plain")
                .send("ERROR_SERVIDOR");
        }
    }
);
```

La respuesta correcta tendrá el formato:

```text
OK|3|CAJA_03|5000
```

Donde:

```text
OK       → identificación aceptada
3        → ID de la caja
CAJA_03  → nombre
5000     → capacidad máxima en gramos
```

---

# 25. API para consultar eventos RFID

Agregar:

```javascript
// ==================================================
// API: CONSULTAR EVENTOS RFID
// ==================================================

app.get(
    "/api/eventos-rfid",
    requiereSesionAPI,
    async (req, res) =>
    {
        try
        {
            const [eventos] =
                await conexion.execute(
                    `SELECT
                        eventos_rfid.id,
                        eventos_rfid.uid_rfid,
                        eventos_rfid.dispositivo,
                        eventos_rfid.resultado,
                        eventos_rfid.fecha,

                        cajas.nombre AS caja

                    FROM eventos_rfid

                    LEFT JOIN cajas
                        ON eventos_rfid.caja_id =
                           cajas.id

                    ORDER BY
                        eventos_rfid.fecha DESC

                    LIMIT 100`
                );

            res.json(eventos);
        }
        catch(error)
        {
            console.log(error);

            res.status(500).json({
                mensaje:
                    "No fue posible consultar los eventos RFID"
            });
        }
    }
);
```

---

# 26. Actualizar la API de mediciones de peso

La ruta anterior:

```text
POST /api/mediciones-peso
```

recibía directamente el nombre de la caja.

Ahora deberá recibir:

```text
uid_rfid
```

Eliminar o reemplazar la versión anterior de esa ruta por la siguiente:

```javascript
// ==================================================
// API: RECIBIR MEDICIÓN DE PESO CON RFID
// ==================================================

app.post(
    "/api/mediciones-peso",
    requiereClaveDispositivo,
    async (req, res) =>
    {
        try
        {
            const productoId =
                Number(
                    req.body.producto_id
                );

            const uid =
                normalizarUid(
                    req.body.uid_rfid
                );

            const numeroPieza =
                Number(
                    req.body.numero_pieza
                );

            const pesoAnterior =
                Number(
                    req.body.peso_anterior_g
                );

            const pesoActual =
                Number(
                    req.body.peso_actual_g
                );

            const dispositivo =
                String(
                    req.body.dispositivo || ""
                ).trim();

            if(
                !Number.isInteger(productoId) ||
                productoId <= 0
            )
            {
                return res.status(400).json({
                    correcto: false,
                    mensaje:
                        "Producto inválido"
                });
            }

            if(
                uid.length < 4 ||
                uid.length > 32
            )
            {
                return res.status(400).json({
                    correcto: false,
                    mensaje:
                        "UID inválido"
                });
            }

            if(
                !Number.isInteger(numeroPieza) ||
                numeroPieza <= 0
            )
            {
                return res.status(400).json({
                    correcto: false,
                    mensaje:
                        "Número de pieza inválido"
                });
            }

            if(
                !Number.isFinite(pesoAnterior) ||
                !Number.isFinite(pesoActual)
            )
            {
                return res.status(400).json({
                    correcto: false,
                    mensaje:
                        "Lecturas de peso inválidas"
                });
            }

            if(dispositivo === "")
            {
                return res.status(400).json({
                    correcto: false,
                    mensaje:
                        "Dispositivo inválido"
                });
            }

            const [cajas] =
                await conexion.execute(
                    `SELECT
                        id,
                        nombre,
                        capacidad_max_g,
                        estado
                    FROM cajas
                    WHERE uid_rfid = ?`,
                    [uid]
                );

            if(cajas.length === 0)
            {
                return res.status(404).json({
                    correcto: false,
                    mensaje:
                        "La caja no está registrada"
                });
            }

            const caja =
                cajas[0];

            if(caja.estado !== "activa")
            {
                return res.status(403).json({
                    correcto: false,
                    mensaje:
                        "La caja está inactiva"
                });
            }

            const pesoPieza =
                Number(
                    (
                        pesoActual -
                        pesoAnterior
                    ).toFixed(2)
                );

            if(pesoPieza <= 0)
            {
                return res.status(400).json({
                    correcto: false,
                    mensaje:
                        "El peso de la pieza debe ser mayor que cero"
                });
            }

            const [productos] =
                await conexion.execute(
                    `SELECT
                        id,
                        nombre,
                        peso_min_g,
                        peso_max_g
                    FROM productos
                    WHERE id = ?`,
                    [productoId]
                );

            if(productos.length === 0)
            {
                return res.status(404).json({
                    correcto: false,
                    mensaje:
                        "El producto no existe"
                });
            }

            const producto =
                productos[0];

            let resultado =
                "sin_especificacion";

            if(
                producto.peso_min_g !== null &&
                producto.peso_max_g !== null
            )
            {
                const pesoMinimo =
                    Number(
                        producto.peso_min_g
                    );

                const pesoMaximo =
                    Number(
                        producto.peso_max_g
                    );

                if(pesoPieza < pesoMinimo)
                {
                    resultado =
                        "demasiado_ligero";
                }
                else if(
                    pesoPieza > pesoMaximo
                )
                {
                    resultado =
                        "demasiado_pesado";
                }
                else
                {
                    resultado =
                        "correcto";
                }
            }

            const capacidad =
                caja.capacidad_max_g === null
                ? 0
                : Number(
                    caja.capacidad_max_g
                );

            const cajaLlena =
                capacidad > 0 &&
                pesoActual >= capacidad;

            const [insercion] =
                await conexion.execute(
                    `INSERT INTO mediciones_peso
                    (
                        producto_id,
                        caja_id,
                        uid_rfid,
                        caja,
                        numero_pieza,
                        peso_anterior_g,
                        peso_actual_g,
                        peso_pieza_g,
                        resultado,
                        dispositivo
                    )
                    VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)`,
                    [
                        productoId,
                        caja.id,
                        uid,
                        caja.nombre,
                        numeroPieza,
                        pesoAnterior,
                        pesoActual,
                        pesoPieza,
                        resultado,
                        dispositivo
                    ]
                );

            const medicion = {
                id:
                    insercion.insertId,

                producto_id:
                    productoId,

                producto:
                    producto.nombre,

                caja_id:
                    caja.id,

                caja:
                    caja.nombre,

                uid_rfid:
                    uid,

                numero_pieza:
                    numeroPieza,

                peso_anterior_g:
                    pesoAnterior,

                peso_actual_g:
                    pesoActual,

                peso_pieza_g:
                    pesoPieza,

                resultado:
                    resultado,

                capacidad_max_g:
                    capacidad,

                caja_llena:
                    cajaLlena,

                dispositivo:
                    dispositivo,

                fecha:
                    new Date().toISOString()
            };

            io.emit(
                "nueva_medicion_peso",
                medicion
            );

            res.json({
                correcto: true,
                mensaje:
                    "Medición registrada",

                medicion:
                    medicion
            });
        }
        catch(error)
        {
            console.log(error);

            res.status(500).json({
                correcto: false,
                mensaje:
                    "No fue posible guardar la medición"
            });
        }
    }
);
```

---

# 27. Actualizar la consulta de mediciones

En:

```text
GET /api/mediciones-peso
```

utilizar la siguiente consulta:

```javascript
const [mediciones] =
    await conexion.execute(
        `SELECT
            mediciones_peso.id,
            mediciones_peso.uid_rfid,
            mediciones_peso.numero_pieza,
            mediciones_peso.peso_anterior_g,
            mediciones_peso.peso_actual_g,
            mediciones_peso.peso_pieza_g,
            mediciones_peso.resultado,
            mediciones_peso.dispositivo,
            mediciones_peso.fecha,

            COALESCE(
                cajas.nombre,
                mediciones_peso.caja
            ) AS caja,

            productos.id AS producto_id,
            productos.nombre AS producto,
            productos.peso_min_g,
            productos.peso_max_g

        FROM mediciones_peso

        INNER JOIN productos
            ON mediciones_peso.producto_id =
               productos.id

        LEFT JOIN cajas
            ON mediciones_peso.caja_id =
               cajas.id

        ORDER BY
            mediciones_peso.fecha DESC

        LIMIT 100`
    );
```

El resto de la ruta puede conservarse igual.

---

# Bloque 5. Crear la página de cajas

## 28. Crear `paginas/cajas.html`

```html
<!DOCTYPE html>
<html lang="es">

<head>
    <meta charset="UTF-8">

    <meta
        name="viewport"
        content="width=device-width, initial-scale=1.0"
    >

    <title>Administración de cajas</title>

    <link
        rel="stylesheet"
        href="/css/estilos.css"
    >
</head>

<body>

    <header>

        <div>
            <h1>Identificación RFID de cajas</h1>

            <p>
                Usuario:
                <strong id="nombreUsuario"></strong>
            </p>
        </div>

        <button
            id="botonCerrarSesion"
            class="boton-gris"
        >
            Cerrar sesión
        </button>

    </header>

    <nav>
        <a href="/panel">Panel</a>
        <a href="/inventario">Inventario</a>
        <a href="/mediciones-altura">Alturas</a>
        <a href="/mediciones-peso">Pesos</a>
        <a href="/estadisticas-peso">Estadísticas</a>
        <a href="/cajas">Cajas RFID</a>
    </nav>

    <main>

        <section class="tarjetas">

            <article class="tarjeta">
                <h3>Último UID</h3>

                <p
                    id="ultimoUid"
                    class="numero-rfid"
                >
                    Sin lectura
                </p>
            </article>

            <article class="tarjeta">
                <h3>Resultado</h3>

                <p id="ultimoResultado">
                    Sin lectura
                </p>
            </article>

            <article class="tarjeta">
                <h3>Caja identificada</h3>

                <p id="ultimaCaja">
                    Ninguna
                </p>
            </article>

            <article class="tarjeta">
                <h3>Dispositivo</h3>

                <p id="ultimoDispositivo">
                    Sin lectura
                </p>
            </article>

        </section>

        <section class="seccion">

            <h2>Registrar caja</h2>

            <div
                id="mensaje"
                class="mensaje"
            ></div>

            <form
                id="formularioCaja"
                class="formulario-horizontal"
            >

                <div>
                    <label for="nombre">
                        Nombre de la caja
                    </label>

                    <input
                        type="text"
                        id="nombre"
                        placeholder="CAJA_01"
                        required
                    >
                </div>

                <div>
                    <label for="uid">
                        UID RFID
                    </label>

                    <input
                        type="text"
                        id="uid"
                        placeholder="A4D38B21"
                        required
                    >
                </div>

                <div>
                    <label for="capacidad">
                        Capacidad máxima en gramos
                    </label>

                    <input
                        type="number"
                        id="capacidad"
                        min="1"
                        step="0.01"
                        placeholder="5000"
                    >
                </div>

                <button type="submit">
                    Registrar caja
                </button>

            </form>

        </section>

        <section class="seccion">

            <h2>Cajas registradas</h2>

            <div class="tabla-contenedor">

                <table>

                    <thead>
                        <tr>
                            <th>ID</th>
                            <th>Caja</th>
                            <th>UID</th>
                            <th>Capacidad</th>
                            <th>Estado</th>
                            <th>Fecha</th>
                        </tr>
                    </thead>

                    <tbody
                        id="tablaCajas"
                    ></tbody>

                </table>

            </div>

        </section>

        <section class="seccion">

            <h2>Historial de lecturas RFID</h2>

            <div class="tabla-contenedor">

                <table>

                    <thead>
                        <tr>
                            <th>ID</th>
                            <th>UID</th>
                            <th>Caja</th>
                            <th>Resultado</th>
                            <th>Dispositivo</th>
                            <th>Fecha</th>
                        </tr>
                    </thead>

                    <tbody
                        id="tablaEventos"
                    ></tbody>

                </table>

            </div>

        </section>

    </main>

    <script src="/socket.io/socket.io.js"></script>
    <script src="/js/comun.js"></script>
    <script src="/js/cajas.js"></script>

</body>
</html>
```

---

# 29. Crear `public/js/cajas.js`

```javascript
const socket = io();

const formulario =
    document.getElementById(
        "formularioCaja"
    );

const mensaje =
    document.getElementById(
        "mensaje"
    );

function textoResultado(resultado)
{
    if(resultado === "identificada")
    {
        return "Caja identificada";
    }

    if(resultado === "inactiva")
    {
        return "Caja inactiva";
    }

    return "UID no registrado";
}

async function cargarCajas()
{
    const respuesta = await fetch(
        "/api/cajas"
    );

    if(!respuesta.ok)
    {
        return;
    }

    const cajas =
        await respuesta.json();

    const tabla =
        document.getElementById(
            "tablaCajas"
        );

    tabla.innerHTML = "";

    cajas.forEach(
        caja =>
        {
            tabla.innerHTML += `
                <tr>
                    <td>${caja.id}</td>

                    <td>${caja.nombre}</td>

                    <td>
                        <code>
                            ${caja.uid_rfid}
                        </code>
                    </td>

                    <td>
                        ${
                            caja.capacidad_max_g === null
                            ? "Sin límite"
                            : `${caja.capacidad_max_g} g`
                        }
                    </td>

                    <td>${caja.estado}</td>

                    <td>
                        ${new Date(
                            caja.fecha_registro
                        ).toLocaleString()}
                    </td>
                </tr>
            `;
        }
    );
}

async function cargarEventos()
{
    const respuesta = await fetch(
        "/api/eventos-rfid"
    );

    if(!respuesta.ok)
    {
        return;
    }

    const eventos =
        await respuesta.json();

    const tabla =
        document.getElementById(
            "tablaEventos"
        );

    tabla.innerHTML = "";

    eventos.forEach(
        evento =>
        {
            tabla.innerHTML += `
                <tr>
                    <td>${evento.id}</td>

                    <td>
                        <code>
                            ${evento.uid_rfid}
                        </code>
                    </td>

                    <td>
                        ${evento.caja || "Desconocida"}
                    </td>

                    <td>
                        ${textoResultado(
                            evento.resultado
                        )}
                    </td>

                    <td>
                        ${evento.dispositivo}
                    </td>

                    <td>
                        ${new Date(
                            evento.fecha
                        ).toLocaleString()}
                    </td>
                </tr>
            `;
        }
    );
}

formulario.addEventListener(
    "submit",
    async function(evento)
    {
        evento.preventDefault();

        const capacidad =
            document.getElementById(
                "capacidad"
            ).value;

        const datos = {
            nombre:
                document.getElementById(
                    "nombre"
                ).value,

            uid_rfid:
                document.getElementById(
                    "uid"
                ).value,

            capacidad_max_g:
                capacidad === ""
                ? null
                : Number(capacidad)
        };

        const respuesta = await fetch(
            "/api/cajas",
            {
                method: "POST",

                headers: {
                    "Content-Type":
                        "application/json"
                },

                body:
                    JSON.stringify(datos)
            }
        );

        const resultado =
            await respuesta.json();

        mensaje.textContent =
            resultado.mensaje;

        if(respuesta.ok)
        {
            mensaje.className =
                "mensaje mensaje-correcto";

            formulario.reset();

            cargarCajas();
        }
        else
        {
            mensaje.className =
                "mensaje mensaje-error";
        }
    }
);

socket.on(
    "lectura_rfid",
    (evento) =>
    {
        document.getElementById(
            "ultimoUid"
        ).textContent =
            evento.uid_rfid;

        document.getElementById(
            "ultimoResultado"
        ).textContent =
            textoResultado(
                evento.resultado
            );

        document.getElementById(
            "ultimaCaja"
        ).textContent =
            evento.caja ||
            "Desconocida";

        document.getElementById(
            "ultimoDispositivo"
        ).textContent =
            evento.dispositivo;

        // Facilita registrar una tarjeta nueva.
        document.getElementById(
            "uid"
        ).value =
            evento.uid_rfid;

        cargarEventos();
    }
);

document.addEventListener(
    "DOMContentLoaded",
    async function()
    {
        await cargarCajas();

        cargarEventos();
    }
);
```

---

# 30. Agregar estilos

Al final de `public/css/estilos.css` agregar:

```css
.numero-rfid {
    font-size: 25px;
    font-weight: bold;
    overflow-wrap: anywhere;
}

code {
    font-family: Consolas, monospace;
}
```

---

# 31. Agregar el enlace al menú

En las páginas privadas agregar:

```html
<a href="/cajas">
    Cajas RFID
</a>
```

---

# Bloque 6. Registrar las cajas

## 32. Procedimiento para registrar una caja nueva

1. Iniciar Node.js.
2. Abrir:

```text
http://localhost:3000/cajas
```

3. Encender el ESP32 con el programa final de esta práctica.
4. Acercar un llavero no registrado.
5. La página mostrará el UID.
6. El UID se colocará automáticamente en el formulario.
7. Escribir un nombre:

```text
CAJA_01
```

8. Escribir su capacidad máxima, por ejemplo:

```text
5000 g
```

9. Presionar:

```text
Registrar caja
```

10. Acercar nuevamente el llavero.
11. La caja deberá ser identificada correctamente.

---

# Bloque 7. Programa final del ESP32

## 33. Funcionamiento

El programa integrará:

* WiFi.
* Solicitudes HTTP.
* Celda de carga con HX711.
* Lector RC522.
* Identificación de caja.
* Tara automática.
* Detección de una pieza.
* Estabilización del peso.
* Envío de mediciones.

La estación permanecerá detenida hasta que se identifique una caja válida.

---

# 34. Código completo

```cpp
#include <WiFi.h>
#include <HTTPClient.h>

#include <SPI.h>
#include <MFRC522.h>

#include <HX711.h>
#include <math.h>

// ==================================================
// CONFIGURACIÓN DE RED
// ==================================================

const char* NOMBRE_WIFI =
    "NOMBRE_DE_LA_RED";

const char* CONTRASENA_WIFI =
    "CONTRASENA_DE_LA_RED";

const char* URL_IDENTIFICAR_CAJA =
    "http://192.168.1.25:3000/api/rfid/identificar";

const char* URL_MEDICIONES_PESO =
    "http://192.168.1.25:3000/api/mediciones-peso";

const char* CLAVE_DISPOSITIVO =
    "clave-banda-industria-40";

// ==================================================
// IDENTIFICACIÓN DEL SISTEMA
// ==================================================

const int PRODUCTO_ID = 1;

const char* NOMBRE_DISPOSITIVO =
    "bascula_01";

// ==================================================
// PINES DEL HX711
// ==================================================

const int PIN_HX711_DOUT = 16;
const int PIN_HX711_SCK = 17;

const float FACTOR_CALIBRACION =
    -421.782013;

HX711 bascula;

// ==================================================
// PINES DEL RC522
// ==================================================

const int PIN_RFID_SS = 5;
const int PIN_RFID_RST = 22;

const int PIN_SPI_SCK = 18;
const int PIN_SPI_MISO = 19;
const int PIN_SPI_MOSI = 23;

MFRC522 lectorRFID(
    PIN_RFID_SS,
    PIN_RFID_RST
);

// ==================================================
// CONFIGURACIÓN DEL PESAJE
// ==================================================

const float UMBRAL_NUEVA_PIEZA_G =
    10.0;

const float TOLERANCIA_ESTABLE_G =
    2.0;

const int LECTURAS_ESTABLES_NECESARIAS =
    4;

const unsigned long
    TIEMPO_MAXIMO_ESTABILIDAD_MS =
        10000;

const float UMBRAL_RETIRO_G =
    30.0;

// ==================================================
// VARIABLES DE LA CAJA
// ==================================================

bool cajaActiva = false;
bool sistemaListo = false;

String uidCajaActiva = "";
String nombreCajaActiva = "";

float capacidadCajaG = 0.0;

// ==================================================
// VARIABLES DEL PESO
// ==================================================

float pesoAnterior = 0.0;

int contadorPiezas = 0;

unsigned long ultimaLecturaSerial = 0;

// ==================================================
// CONECTAR A WIFI
// ==================================================

void conectarWiFi()
{
    Serial.println();
    Serial.println(
        "Conectando a WiFi"
    );

    WiFi.begin(
        NOMBRE_WIFI,
        CONTRASENA_WIFI
    );

    while(
        WiFi.status() !=
        WL_CONNECTED
    )
    {
        delay(500);
        Serial.print(".");
    }

    Serial.println();
    Serial.println(
        "WiFi conectado"
    );

    Serial.print(
        "IP del ESP32: "
    );

    Serial.println(
        WiFi.localIP()
    );
}

// ==================================================
// CONVERTIR UID A TEXTO
// ==================================================

String convertirUidATexto()
{
    String uid = "";

    for(
        byte i = 0;
        i < lectorRFID.uid.size;
        i++
    )
    {
        if(
            lectorRFID.uid.uidByte[i] <
            0x10
        )
        {
            uid += "0";
        }

        uid += String(
            lectorRFID.uid.uidByte[i],
            HEX
        );
    }

    uid.toUpperCase();

    return uid;
}

// ==================================================
// LEER UNA TARJETA
// ==================================================

String leerUidRFID()
{
    if(
        !lectorRFID.PICC_IsNewCardPresent()
    )
    {
        return "";
    }

    if(
        !lectorRFID.PICC_ReadCardSerial()
    )
    {
        return "";
    }

    String uid =
        convertirUidATexto();

    lectorRFID.PICC_HaltA();

    lectorRFID.PCD_StopCrypto1();

    return uid;
}

// ==================================================
// IDENTIFICAR CAJA EN EL SERVIDOR
// ==================================================

bool identificarCaja(
    const String& uid,
    String& nombre,
    float& capacidad
)
{
    if(
        WiFi.status() !=
        WL_CONNECTED
    )
    {
        conectarWiFi();
    }

    HTTPClient http;

    http.begin(
        URL_IDENTIFICAR_CAJA
    );

    http.addHeader(
        "Content-Type",
        "application/json"
    );

    http.addHeader(
        "x-api-key",
        CLAVE_DISPOSITIVO
    );

    String datos = "{";

    datos +=
        "\"uid_rfid\":\"" +
        uid +
        "\",";

    datos +=
        "\"dispositivo\":\"" +
        String(NOMBRE_DISPOSITIVO) +
        "\"";

    datos += "}";

    int codigoHTTP =
        http.POST(datos);

    String respuesta =
        http.getString();

    Serial.print(
        "Código HTTP RFID: "
    );

    Serial.println(
        codigoHTTP
    );

    Serial.print(
        "Respuesta RFID: "
    );

    Serial.println(
        respuesta
    );

    http.end();

    if(codigoHTTP != 200)
    {
        return false;
    }

    if(
        !respuesta.startsWith(
            "OK|"
        )
    )
    {
        return false;
    }

    int separador1 =
        respuesta.indexOf('|');

    int separador2 =
        respuesta.indexOf(
            '|',
            separador1 + 1
        );

    int separador3 =
        respuesta.indexOf(
            '|',
            separador2 + 1
        );

    if(
        separador1 < 0 ||
        separador2 < 0 ||
        separador3 < 0
    )
    {
        return false;
    }

    nombre =
        respuesta.substring(
            separador2 + 1,
            separador3
        );

    capacidad =
        respuesta.substring(
            separador3 + 1
        ).toFloat();

    return true;
}

// ==================================================
// REALIZAR TARA
// ==================================================

void prepararCaja()
{
    Serial.println();
    Serial.println(
        "La caja fue autorizada."
    );

    Serial.println(
        "No toque la plataforma."
    );

    Serial.println(
        "Realizando tara..."
    );

    delay(2000);

    bascula.tare(20);

    pesoAnterior = 0.0;

    contadorPiezas = 0;

    sistemaListo = true;

    Serial.println(
        "Tara terminada."
    );

    Serial.println(
        "Estación preparada."
    );
}

// ==================================================
// PROCESAR UNA TARJETA
// ==================================================

void revisarRFID()
{
    String uid =
        leerUidRFID();

    if(uid == "")
    {
        return;
    }

    Serial.println();
    Serial.print(
        "UID leído: "
    );

    Serial.println(uid);

    if(cajaActiva)
    {
        if(uid == uidCajaActiva)
        {
            Serial.println(
                "Esta caja ya se encuentra activa."
            );
        }
        else
        {
            Serial.println(
                "No es posible cambiar de caja"
            );

            Serial.println(
                "mientras exista una caja activa."
            );

            Serial.println(
                "Retire primero la caja actual."
            );
        }

        delay(1000);
        return;
    }

    String nombre = "";
    float capacidad = 0.0;

    bool identificada =
        identificarCaja(
            uid,
            nombre,
            capacidad
        );

    if(!identificada)
    {
        Serial.println(
            "Caja no registrada o inactiva."
        );

        Serial.println(
            "La estación permanecerá detenida."
        );

        delay(1000);
        return;
    }

    uidCajaActiva = uid;
    nombreCajaActiva = nombre;
    capacidadCajaG = capacidad;

    cajaActiva = true;

    Serial.println();
    Serial.println(
        "CAJA IDENTIFICADA"
    );

    Serial.print(
        "Nombre: "
    );

    Serial.println(
        nombreCajaActiva
    );

    Serial.print(
        "UID: "
    );

    Serial.println(
        uidCajaActiva
    );

    if(capacidadCajaG > 0)
    {
        Serial.print(
            "Capacidad: "
        );

        Serial.print(
            capacidadCajaG,
            2
        );

        Serial.println(" g");
    }
    else
    {
        Serial.println(
            "Capacidad: sin límite configurado"
        );
    }

    prepararCaja();
}

// ==================================================
// LEER PESO
// ==================================================

float leerPeso()
{
    if(
        !bascula.wait_ready_timeout(
            1000
        )
    )
    {
        return NAN;
    }

    float peso =
        bascula.get_units(5);

    if(
        peso > -1.0 &&
        peso < 1.0
    )
    {
        peso = 0.0;
    }

    return peso;
}

// ==================================================
// ESPERAR ESTABILIDAD
// ==================================================

float esperarPesoEstable()
{
    unsigned long inicio =
        millis();

    float lecturaAnterior =
        leerPeso();

    if(isnan(lecturaAnterior))
    {
        return NAN;
    }

    int lecturasEstables = 0;

    while(
        millis() - inicio <
        TIEMPO_MAXIMO_ESTABILIDAD_MS
    )
    {
        revisarRFID();

        delay(250);

        float lecturaActual =
            leerPeso();

        if(isnan(lecturaActual))
        {
            continue;
        }

        float diferencia =
            fabs(
                lecturaActual -
                lecturaAnterior
            );

        Serial.print(
            "Estabilizando: "
        );

        Serial.print(
            lecturaActual,
            2
        );

        Serial.print(
            " g | variación: "
        );

        Serial.println(
            diferencia,
            2
        );

        if(
            diferencia <=
            TOLERANCIA_ESTABLE_G
        )
        {
            lecturasEstables++;
        }
        else
        {
            lecturasEstables = 0;
        }

        lecturaAnterior =
            lecturaActual;

        if(
            lecturasEstables >=
            LECTURAS_ESTABLES_NECESARIAS
        )
        {
            return lecturaActual;
        }
    }

    return NAN;
}

// ==================================================
// ENVIAR PESO AL SERVIDOR
// ==================================================

bool enviarMedicionPeso(
    float pesoAcumuladoAnterior,
    float pesoAcumuladoActual,
    int numeroPieza
)
{
    if(
        WiFi.status() !=
        WL_CONNECTED
    )
    {
        conectarWiFi();
    }

    HTTPClient http;

    http.begin(
        URL_MEDICIONES_PESO
    );

    http.addHeader(
        "Content-Type",
        "application/json"
    );

    http.addHeader(
        "x-api-key",
        CLAVE_DISPOSITIVO
    );

    String datos = "{";

    datos +=
        "\"producto_id\":" +
        String(PRODUCTO_ID) +
        ",";

    datos +=
        "\"uid_rfid\":\"" +
        uidCajaActiva +
        "\",";

    datos +=
        "\"numero_pieza\":" +
        String(numeroPieza) +
        ",";

    datos +=
        "\"peso_anterior_g\":" +
        String(
            pesoAcumuladoAnterior,
            2
        ) +
        ",";

    datos +=
        "\"peso_actual_g\":" +
        String(
            pesoAcumuladoActual,
            2
        ) +
        ",";

    datos +=
        "\"dispositivo\":\"" +
        String(NOMBRE_DISPOSITIVO) +
        "\"";

    datos += "}";

    Serial.println();
    Serial.println(
        "Enviando medición:"
    );

    Serial.println(datos);

    int codigoHTTP =
        http.POST(datos);

    Serial.print(
        "Código HTTP: "
    );

    Serial.println(
        codigoHTTP
    );

    if(codigoHTTP > 0)
    {
        String respuesta =
            http.getString();

        Serial.println(
            "Respuesta:"
        );

        Serial.println(
            respuesta
        );
    }

    bool correcto =
        codigoHTTP >= 200 &&
        codigoHTTP < 300;

    http.end();

    return correcto;
}

// ==================================================
// DESACTIVAR CAJA
// ==================================================

void desactivarCaja()
{
    Serial.println();
    Serial.println(
        "La caja fue retirada."
    );

    Serial.print(
        "Caja terminada: "
    );

    Serial.println(
        nombreCajaActiva
    );

    Serial.print(
        "Piezas registradas: "
    );

    Serial.println(
        contadorPiezas
    );

    cajaActiva = false;
    sistemaListo = false;

    uidCajaActiva = "";
    nombreCajaActiva = "";

    capacidadCajaG = 0.0;
    pesoAnterior = 0.0;

    contadorPiezas = 0;

    Serial.println();
    Serial.println(
        "Coloque una caja vacía"
    );

    Serial.println(
        "y acerque su llavero RFID."
    );
}

// ==================================================
// CONFIGURACIÓN
// ==================================================

void setup()
{
    Serial.begin(115200);

    // Iniciar la báscula.
    bascula.begin(
        PIN_HX711_DOUT,
        PIN_HX711_SCK
    );

    bascula.set_scale(
        FACTOR_CALIBRACION
    );

    // Iniciar SPI y RFID.
    SPI.begin(
        PIN_SPI_SCK,
        PIN_SPI_MISO,
        PIN_SPI_MOSI,
        PIN_RFID_SS
    );

    lectorRFID.PCD_Init();

    delay(100);

    conectarWiFi();

    Serial.println();
    Serial.println(
        "ESTACIÓN DE PESAJE CON RFID"
    );

    Serial.println(
        "Coloque una caja vacía"
    );

    Serial.println(
        "y acerque su llavero RFID."
    );
}

// ==================================================
// PROGRAMA PRINCIPAL
// ==================================================

void loop()
{
    revisarRFID();

    if(
        !cajaActiva ||
        !sistemaListo
    )
    {
        delay(100);
        return;
    }

    float pesoActual =
        leerPeso();

    if(isnan(pesoActual))
    {
        Serial.println(
            "No fue posible leer el HX711"
        );

        delay(500);
        return;
    }

    if(
        millis() -
        ultimaLecturaSerial >=
        1000
    )
    {
        ultimaLecturaSerial =
            millis();

        Serial.print(
            "Caja: "
        );

        Serial.print(
            nombreCajaActiva
        );

        Serial.print(
            " | Peso acumulado: "
        );

        Serial.print(
            pesoActual,
            2
        );

        Serial.println(" g");

        if(
            capacidadCajaG > 0 &&
            pesoActual >= capacidadCajaG
        )
        {
            Serial.println(
                "ALERTA: capacidad máxima alcanzada"
            );
        }
    }

    float cambio =
        pesoActual -
        pesoAnterior;

    // Detectar una nueva pieza.
    if(
        cambio >=
        UMBRAL_NUEVA_PIEZA_G
    )
    {
        Serial.println();
        Serial.println(
            "Aumento de peso detectado."
        );

        float pesoEstable =
            esperarPesoEstable();

        if(isnan(pesoEstable))
        {
            Serial.println(
                "El peso no logró estabilizarse."
            );

            delay(500);
            return;
        }

        float pesoPieza =
            pesoEstable -
            pesoAnterior;

        if(
            pesoPieza <
            UMBRAL_NUEVA_PIEZA_G
        )
        {
            Serial.println(
                "El cambio no corresponde a una pieza."
            );

            return;
        }

        int siguientePieza =
            contadorPiezas + 1;

        Serial.println();
        Serial.print(
            "Caja: "
        );

        Serial.println(
            nombreCajaActiva
        );

        Serial.print(
            "UID: "
        );

        Serial.println(
            uidCajaActiva
        );

        Serial.print(
            "Número de pieza: "
        );

        Serial.println(
            siguientePieza
        );

        Serial.print(
            "Peso de la pieza: "
        );

        Serial.print(
            pesoPieza,
            2
        );

        Serial.println(" g");

        Serial.print(
            "Peso acumulado: "
        );

        Serial.print(
            pesoEstable,
            2
        );

        Serial.println(" g");

        bool enviado =
            enviarMedicionPeso(
                pesoAnterior,
                pesoEstable,
                siguientePieza
            );

        if(enviado)
        {
            pesoAnterior =
                pesoEstable;

            contadorPiezas =
                siguientePieza;

            Serial.println(
                "Medición registrada."
            );
        }
        else
        {
            Serial.println(
                "El servidor no confirmó la medición."
            );
        }

        delay(500);
    }

    // Detectar retiro de la caja.
    if(
        cambio <=
        -UMBRAL_RETIRO_G
    )
    {
        desactivarCaja();
    }

    delay(100);
}
```

---

# 35. Variables que deberán modificarse

## Red WiFi

```cpp
const char* NOMBRE_WIFI =
    "NOMBRE_DE_LA_RED";
```

```cpp
const char* CONTRASENA_WIFI =
    "CONTRASENA_DE_LA_RED";
```

## Dirección del servidor

```cpp
const char* URL_IDENTIFICAR_CAJA =
    "http://DIRECCION_IP:3000/api/rfid/identificar";
```

```cpp
const char* URL_MEDICIONES_PESO =
    "http://DIRECCION_IP:3000/api/mediciones-peso";
```

## Factor de calibración

```cpp
const float FACTOR_CALIBRACION =
    -421.782013;
```

Debe utilizarse el factor obtenido para la celda de carga de cada equipo.

## Producto

```cpp
const int PRODUCTO_ID = 1;
```

---

# Bloque 8. Actualizar la página de peso

## 36. Agregar el UID a la tabla

En `mediciones-peso.html`, agregar una columna:

```html
<th>UID RFID</th>
```

Puede colocarse después de:

```html
<th>Caja</th>
```

---

# 37. Actualizar `crearFila()`

En `mediciones-peso.js`, agregar:

```javascript
<td>
    <code>
        ${medicion.uid_rfid || ""}
    </code>
</td>
```

Debe colocarse después de la celda que muestra el nombre de la caja.

---

# 38. Mostrar alerta de caja llena

Dentro de:

```javascript
actualizarUltimaMedicion(medicion)
```

se puede agregar:

```javascript
if(medicion.caja_llena)
{
    document.getElementById(
        "ultimoResultado"
    ).textContent =
        "Caja llena";
}
```

Esto reemplazará momentáneamente el resultado visual cuando el peso acumulado haya alcanzado la capacidad configurada.

---

# Bloque 9. Ejecutar el sistema

## 39. Iniciar MySQL

Verificar que MySQL Server esté ejecutándose.

---

## 40. Iniciar Node.js

```bash
node server.js
```

Resultado esperado:

```text
Conexión correcta con MySQL
Servidor disponible en http://localhost:3000
```

---

## 41. Abrir la página de cajas

```text
http://localhost:3000/cajas
```

---

## 42. Encender el ESP32

Abrir el monitor serial a:

```text
115200 baudios
```

El sistema deberá mostrar:

```text
ESTACIÓN DE PESAJE CON RFID

Coloque una caja vacía
y acerque su llavero RFID.
```

---

# Bloque 10. Pruebas obligatorias

## Prueba 1. Leer un UID no registrado

Acercar un llavero nuevo.

Resultado esperado:

```text
Caja no registrada o inactiva.
La estación permanecerá detenida.
```

La página deberá mostrar:

```text
UID no registrado
```

El evento deberá guardarse en:

```text
eventos_rfid
```

---

## Prueba 2. Registrar la caja

Desde la página:

```text
/cajas
```

registrar:

```text
Nombre: CAJA_01
UID: UID leído
Capacidad: 5000 g
```

---

## Prueba 3. Identificar la caja registrada

Acercar nuevamente el llavero.

Resultado esperado:

```text
CAJA IDENTIFICADA
Nombre: CAJA_01
Realizando tara...
Tara terminada.
```

---

## Prueba 4. Comprobar la tara

Después de identificar la caja vacía:

```text
Peso acumulado: 0.00 g
```

Se aceptan pequeñas variaciones alrededor de cero.

---

## Prueba 5. Registrar una pieza

Dejar caer una pieza.

Comprobar:

* Caja identificada.
* UID.
* Número de pieza.
* Peso individual.
* Peso acumulado.
* Registro en MySQL.

---

## Prueba 6. Acumular piezas

Registrar al menos cinco piezas.

| Caja    | UID | Pieza | Peso individual | Peso acumulado |
| ------- | --- | ----: | --------------: | -------------: |
| CAJA_01 |     |     1 |                 |                |
| CAJA_01 |     |     2 |                 |                |
| CAJA_01 |     |     3 |                 |                |
| CAJA_01 |     |     4 |                 |                |
| CAJA_01 |     |     5 |                 |                |

---

## Prueba 7. Intentar cambiar de caja

Con `CAJA_01` todavía sobre la báscula, acercar el llavero de `CAJA_02`.

Resultado esperado:

```text
No es posible cambiar de caja
mientras exista una caja activa.
```

---

## Prueba 8. Retirar la caja

Retirar la caja llena.

El sistema deberá detectar una disminución importante y mostrar:

```text
La caja fue retirada.
Coloque una caja vacía
y acerque su llavero RFID.
```

---

## Prueba 9. Colocar una caja diferente

1. Colocar `CAJA_02` vacía.
2. Acercar su llavero.
3. Comprobar que se realice una nueva tara.
4. Registrar al menos tres piezas.

Las nuevas mediciones deberán relacionarse con:

```text
CAJA_02
```

---

## Prueba 10. Caja inactiva

Desde MySQL ejecutar:

```sql
UPDATE cajas
SET estado = 'inactiva'
WHERE nombre = 'CAJA_02';
```

Acercar su llavero.

Resultado esperado:

```text
Caja no registrada o inactiva.
```

Volver a activarla:

```sql
UPDATE cajas
SET estado = 'activa'
WHERE nombre = 'CAJA_02';
```

---

## Prueba 11. Capacidad máxima

Registrar una caja con una capacidad baja para facilitar la prueba:

```text
Capacidad: 500 g
```

Agregar piezas hasta superar ese peso.

Resultado esperado:

```text
ALERTA: capacidad máxima alcanzada
```

---

## Prueba 12. Persistencia

Detener Node.js:

```text
Ctrl + C
```

Volver a iniciarlo:

```bash
node server.js
```

Las cajas, UID, eventos y mediciones deberán continuar disponibles.

---

# 43. Consultas SQL de comprobación

## Consultar cajas

```sql
SELECT
    id,
    nombre,
    uid_rfid,
    capacidad_max_g,
    estado,
    fecha_registro
FROM cajas;
```

---

## Consultar eventos RFID

```sql
SELECT
    eventos_rfid.id,
    eventos_rfid.uid_rfid,
    cajas.nombre AS caja,
    eventos_rfid.dispositivo,
    eventos_rfid.resultado,
    eventos_rfid.fecha

FROM eventos_rfid

LEFT JOIN cajas
    ON eventos_rfid.caja_id =
       cajas.id

ORDER BY
    eventos_rfid.fecha DESC;
```

---

## Consultar pesos por caja

```sql
SELECT
    cajas.nombre AS caja,
    cajas.uid_rfid,

    COUNT(
        mediciones_peso.id
    ) AS total_piezas,

    ROUND(
        SUM(
            mediciones_peso.peso_pieza_g
        ),
        2
    ) AS peso_total,

    ROUND(
        AVG(
            mediciones_peso.peso_pieza_g
        ),
        2
    ) AS peso_promedio

FROM cajas

LEFT JOIN mediciones_peso
    ON cajas.id =
       mediciones_peso.caja_id

GROUP BY
    cajas.id,
    cajas.nombre,
    cajas.uid_rfid;
```

---

## Consultar el contenido de una caja

```sql
SELECT
    cajas.nombre AS caja,
    mediciones_peso.numero_pieza,
    productos.nombre AS producto,
    mediciones_peso.peso_pieza_g,
    mediciones_peso.peso_actual_g,
    mediciones_peso.resultado,
    mediciones_peso.fecha

FROM mediciones_peso

INNER JOIN cajas
    ON mediciones_peso.caja_id =
       cajas.id

INNER JOIN productos
    ON mediciones_peso.producto_id =
       productos.id

WHERE cajas.nombre = 'CAJA_01'

ORDER BY
    mediciones_peso.numero_pieza;
```

---

## Contar lecturas RFID correctas e incorrectas

```sql
SELECT
    resultado,
    COUNT(*) AS cantidad

FROM eventos_rfid

GROUP BY resultado;
```

---

# 44. Flujo completo de una caja

```text
La caja se registra en MySQL
               │
               ▼
Se relaciona su nombre con un UID
               │
               ▼
La caja vacía se coloca en la báscula
               │
               ▼
El operador acerca el llavero
               │
               ▼
El ESP32 lee el UID
               │
               ▼
Node.js valida la caja
               │
               ▼
El ESP32 realiza la tara
               │
               ▼
Las piezas comienzan a caer
               │
               ▼
Cada peso se registra con:
    - caja_id
    - nombre de la caja
    - UID
    - número de pieza
    - producto
    - peso individual
    - peso acumulado
    - resultado
    - fecha
               │
               ▼
La caja se retira
               │
               ▼
La estación espera otra caja
```

---

# 45. Evidencias que deberán entregarse

El reporte deberá incluir:

1. Portada.
2. Objetivo.
3. Diagrama de conexión del RC522.
4. Diagrama completo de la estación.
5. Fotografía del montaje.
6. Tabla de UID obtenidos.
7. Captura del monitor serial leyendo un UID.
8. Captura de una caja no registrada.
9. Captura de una caja identificada.
10. Captura de la tabla `cajas`.
11. Captura de la tabla `eventos_rfid`.
12. Captura de las mediciones relacionadas con una caja.
13. Captura de la página de cajas.
14. Prueba con al menos dos cajas.
15. Prueba de caja inactiva.
16. Prueba de cambio de caja.
17. Explicación del flujo completo.
18. Conclusiones.

---

# 46. Investigar

Investigar y explicar con palabras propias:

* ¿Qué significa RFID?
* ¿Qué diferencia existe entre RFID y NFC?
* ¿Qué frecuencia utiliza el MFRC522?
* ¿Qué es el UID?
* ¿Por qué el UID se almacena como texto?
* ¿Qué es una representación hexadecimal?
* ¿Qué función realiza el lector RFID?
* ¿La tarjeta necesita batería?
* ¿Qué función tiene la antena del RC522?
* ¿Qué es el protocolo SPI?
* ¿Qué función tiene SCK?
* ¿Qué función tiene MOSI?
* ¿Qué función tiene MISO?
* ¿Qué función tiene SS o CS?
* ¿Por qué el pin SDA del módulo funciona como SS?
* ¿Por qué el RC522 debe alimentarse con 3.3 V?
* ¿Qué función realiza `PICC_IsNewCardPresent()`?
* ¿Qué función realiza `PICC_ReadCardSerial()`?
* ¿Qué función realiza `PICC_HaltA()`?
* ¿Por qué se normaliza el UID en el servidor?
* ¿Qué es una clave única en MySQL?
* ¿Por qué `uid_rfid` utiliza `UNIQUE`?
* ¿Qué es una llave foránea?
* ¿Qué relación existe entre `cajas` y `mediciones_peso`?
* ¿Por qué se conserva el nombre anterior de la caja?
* ¿Qué ocurre si una caja no está registrada?
* ¿Qué ocurre si se acerca otra tarjeta mientras existe una caja activa?
* ¿Por qué se realiza la tara después de identificar la caja?
* ¿Por qué un UID no debe considerarse autenticación segura?
* ¿Cómo ayuda el RFID a la trazabilidad de un proceso?

---

# 47. Consideraciones y limitaciones

El sistema puede presentar problemas cuando:

* El RC522 se alimenta con 5 V.
* Los cables SPI son demasiado largos.
* El llavero se encuentra lejos del lector.
* Existe metal directamente detrás de la antena.
* El lector está cerca del motor.
* La fuente de alimentación es inestable.
* El UID fue capturado incorrectamente.
* La misma tarjeta fue registrada dos veces.
* La caja se coloca después de realizar la tara.
* Se cambia de caja sin retirar la anterior.
* Se retira una pieza sin cambiar la caja.
* Se presentan pérdidas de comunicación con el servidor.

El sistema supone que:

* Cada llavero corresponde a una sola caja.
* La caja se encuentra vacía al ser identificada.
* Solo existe una caja activa en la estación.
* Las piezas caen de una en una.
* La caja permanece sobre la báscula durante su llenado.

---

# 48. Preparación para la práctica de color

La siguiente práctica utilizará el sensor:

```text
TCS34725
```

El sensor se colocará cerca del detector infrarrojo y del HC-SR04.

El flujo será:

```text
El infrarrojo detecta la pieza
              │
      ┌───────┴────────┐
      ▼                ▼
HC-SR04 mide       TCS34725 mide
la altura          el color
      │                │
      └───────┬────────┘
              ▼
       La pieza cae
              │
              ▼
      HX711 mide el peso
              │
              ▼
RFID identifica la caja receptora
```

Al finalizar la práctica de color, cada pieza podrá contener:

```text
Producto
Altura
Peso
Color
Resultado
Caja receptora
UID de la caja
Fecha
Dispositivo
```
