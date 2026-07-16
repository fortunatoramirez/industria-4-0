# Práctica: Medición automática de altura en una banda transportadora

## Integración de ESP32, sensor infrarrojo, HC-SR04, Node.js, Socket.IO y MySQL

---

# 1. Introducción

En la práctica anterior se desarrolló un sistema web de Industria 4.0 con:

* Registro de usuarios.
* Inicio y cierre de sesión.
* Inventario de productos.
* Registro de entradas y salidas.
* Historial de movimientos.
* Almacenamiento permanente en MySQL.

En esta práctica se agregará el primer sistema físico de inspección automática.

Un sensor infrarrojo detectará el momento en que un objeto se encuentra en la zona de medición. En ese instante, un sensor ultrasónico HC-SR04 colocado sobre la banda medirá la distancia entre el sensor y la parte superior del objeto.

La altura se obtendrá mediante:

```text
Altura del objeto =
Distancia entre el sensor y la banda
−
Distancia entre el sensor y el objeto
```

La medición será enviada por el ESP32 al servidor Node.js, almacenada en MySQL y presentada en páginas de monitoreo, historial y estadísticas.

> El HC-SR04 permite medir distancias sin contacto utilizando un pulso de activación y el tiempo de retorno del eco. Su documentación especifica un pulso de disparo de aproximadamente 10 µs y un intervalo nominal de medición de 2 a 400 cm.

---

# 2. Objetivo general

Desarrollar un sistema de inspección automática capaz de detectar objetos sobre una banda transportadora, medir su altura, determinar si cumple con las especificaciones establecidas y almacenar los resultados en una base de datos.

---

# 3. Objetivos específicos

Al finalizar la práctica, el estudiante será capaz de:

* Conectar un sensor infrarrojo y un sensor ultrasónico al ESP32.
* Detectar una sola vez cada objeto que pase por la banda.
* Calibrar la distancia existente entre el sensor y la banda.
* Realizar varias mediciones ultrasónicas por objeto.
* Calcular la mediana de las mediciones obtenidas.
* Calcular la altura estimada de cada objeto.
* Enviar información desde el ESP32 mediante HTTP.
* Recibir y validar información en Node.js.
* Guardar mediciones automáticamente en MySQL.
* Actualizar una página web mediante Socket.IO.
* Obtener estadísticas utilizando consultas SQL.
* Identificar objetos correctos, demasiado bajos o demasiado altos.

---

# 4. Resultado esperado

El sistema tendrá la siguiente estructura:

```text
Objeto sobre la banda
          │
          ▼
Sensor infrarrojo detecta el objeto
          │
          ▼
HC-SR04 mide la distancia
          │
          ▼
ESP32 obtiene varias muestras
          │
          ▼
ESP32 envía los datos mediante HTTP
          │
          ▼
Servidor Node.js
          │
          ├── Calcula la altura
          ├── Compara con los límites
          ├── Guarda en MySQL
          └── Emite una actualización
                     │
                     ▼
        Página de monitoreo y estadísticas
```

---

# 5. Material necesario

## Hardware

* Una tarjeta ESP32.
* Un sensor ultrasónico HC-SR04.
* Un sensor infrarrojo para detección de objetos.
* Una banda transportadora.
* Protoboard.
* Cables Dupont.
* Dos resistencias para el divisor de voltaje:

  * Una resistencia de 1 kΩ.
  * Una resistencia de 2 kΩ.
* Regla o cinta métrica.
* Objetos de diferentes alturas y superficie superior plana.
* Computadora conectada a la misma red WiFi que el ESP32.

## Software

* Arduino IDE.
* Node.js.
* MySQL Server.
* Visual Studio Code.
* Navegador web.
* Proyecto desarrollado en la práctica anterior.

---

# 6. Funcionamiento de la medición

El HC-SR04 se colocará sobre la banda transportadora, apuntando verticalmente hacia abajo.

Primero se medirá la distancia entre el sensor y la banda vacía.

Ejemplo:

```text
Distancia sensor–banda: 30.0 cm
```

Cuando pase un objeto, el sensor medirá la distancia hasta la parte superior.

Ejemplo:

```text
Distancia sensor–objeto: 18.2 cm
```

Por lo tanto:

```text
Altura = 30.0 cm − 18.2 cm

Altura = 11.8 cm
```

La altura obtenida debe considerarse una **estimación calibrada**, no una medición metrológica exacta. La superficie, inclinación, velocidad de la banda y posición del objeto pueden afectar el resultado.

---

# 7. Colocación de los sensores

## Sensor ultrasónico

El HC-SR04 deberá colocarse:

* Sobre la banda.
* Apuntando verticalmente hacia abajo.
* Perpendicular a la superficie de la banda.
* Firmemente sujeto.
* Sin vibraciones.
* A una altura suficiente para medir los objetos más altos.

## Sensor infrarrojo

El sensor infrarrojo deberá colocarse en la misma zona de medición.

Su función será indicar que un objeto se encuentra debajo del sensor ultrasónico.

```text
Vista lateral

              HC-SR04
           ┌───────────┐
           │  ○     ○  │
           └─────┬─────┘
                 │
                 │ Distancia medida
                 ▼
            ┌─────────┐
            │ Objeto  │ ◄── Sensor infrarrojo
════════════┴─────────┴════════════
          Banda transportadora
```

El sensor infrarrojo puede colocarse ligeramente antes del HC-SR04. En ese caso será necesario ajustar un pequeño tiempo de espera para que el objeto llegue al centro de la zona de medición.

---

# 8. Conexión del HC-SR04

## Pines utilizados

| HC-SR04       | ESP32                    |
| ------------- | ------------------------ |
| VCC           | 5 V                      |
| GND           | GND                      |
| TRIG          | GPIO 5                   |
| ECHO          | GPIO 18 mediante divisor |
| Sensor IR OUT | GPIO 19                  |

## Protección de la entrada ECHO

El HC-SR04 normalmente trabaja con alimentación de 5 V. Por ello, la salida `ECHO` no debe conectarse directamente a una entrada de 3.3 V del ESP32.

Se utilizará un divisor resistivo:

```text
ECHO del HC-SR04
       │
      1 kΩ
       │
       ├──────── GPIO 18 del ESP32
       │
      2 kΩ
       │
      GND
```

El voltaje aproximado será:

```text
Vsalida = 5 V × 2 kΩ / (1 kΩ + 2 kΩ)

Vsalida ≈ 3.33 V
```

El uso de un divisor para adaptar la señal de 5 V del HC-SR04 a microcontroladores de 3.3 V es una medida de protección recomendada.

> Si el sensor infrarrojo utilizado entrega 5 V en su salida, también deberá utilizarse adaptación de nivel. Cuando sea posible, alimentar el módulo infrarrojo con 3.3 V y comprobar su funcionamiento.

---

# Bloque 1. Preparar la base de datos

## 9. Seleccionar la base de datos

Entrar a MySQL:

```bash
mysql -u root -p
```

Seleccionar la base de datos:

```sql
USE industria40_web;
```

---

## 10. Agregar especificaciones de altura a los productos

Ejecutar una sola vez:

```sql
ALTER TABLE productos
ADD altura_min_cm DECIMAL(6,2) NULL,
ADD altura_max_cm DECIMAL(6,2) NULL;
```

Los campos tendrán la siguiente función:

| Campo           | Función                 |
| --------------- | ----------------------- |
| `altura_min_cm` | Altura mínima permitida |
| `altura_max_cm` | Altura máxima permitida |

> Si se ejecuta nuevamente el comando, MySQL indicará que las columnas ya existen. No deben crearse por segunda vez.

---

## 11. Crear la tabla de mediciones

Ejecutar:

```sql
CREATE TABLE mediciones_altura (
    id INT AUTO_INCREMENT PRIMARY KEY,

    producto_id INT NOT NULL,

    dispositivo VARCHAR(50) NOT NULL,

    distancia_banda_cm DECIMAL(6,2) NOT NULL,

    distancia_objeto_cm DECIMAL(6,2) NOT NULL,

    altura_cm DECIMAL(6,2) NOT NULL,

    resultado VARCHAR(30) NOT NULL,

    fecha TIMESTAMP
        DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY (producto_id)
        REFERENCES productos(id)
);
```

El campo `resultado` podrá contener:

```text
correcto
demasiado_bajo
demasiado_alto
sin_especificacion
```

---

## 12. Configurar las especificaciones de un producto

Consultar los productos disponibles:

```sql
SELECT
    id,
    nombre,
    cantidad
FROM productos;
```

Seleccionar un producto y establecer sus límites.

Ejemplo para el producto con ID 1:

```sql
UPDATE productos
SET
    altura_min_cm = 9.50,
    altura_max_cm = 10.50
WHERE id = 1;
```

Comprobar:

```sql
SELECT
    id,
    nombre,
    altura_min_cm,
    altura_max_cm
FROM productos;
```

---

## 13. Verificar la tabla

Ejecutar:

```sql
DESCRIBE mediciones_altura;
```

Salir:

```sql
EXIT;
```

---

# Bloque 2. Preparar el proyecto de Node.js

## 14. Abrir el proyecto anterior

Abrir en Visual Studio Code la carpeta:

```text
industria40_web
```

Antes de continuar, verificar que todavía funcionen:

* Registro de usuarios.
* Inicio de sesión.
* Inventario.
* Movimientos.
* Historial.
* Conexión con MySQL.

---

## 15. Instalar Socket.IO

Ejecutar desde la terminal del proyecto:

```bash
npm install socket.io
```

Socket.IO se utilizará para avisar a las páginas web que se ha registrado una nueva medición, sin recargar manualmente el navegador. La documentación oficial permite conectar Socket.IO a un servidor HTTP de Node.js y cargar el cliente desde `/socket.io/socket.io.js`.

---

# Bloque 3. Modificar `server.js`

## 16. Importar las nuevas bibliotecas

Al inicio de `server.js`, después de las bibliotecas existentes, agregar:

```javascript
const http = require("http");

const {
    Server
} = require("socket.io");
```

---

## 17. Crear el servidor HTTP y Socket.IO

Localizar:

```javascript
const app = express();
```

Debajo agregar:

```javascript
const servidorHTTP =
    http.createServer(app);

const io =
    new Server(servidorHTTP);
```

---

## 18. Crear una clave para el ESP32

Debajo de la declaración del puerto agregar:

```javascript
const CLAVE_DISPOSITIVO =
    "clave-banda-industria-40";
```

Esta clave deberá ser igual en:

* El servidor Node.js.
* El programa del ESP32.

No se utilizará la sesión de usuario para el ESP32 porque el dispositivo no inicia sesión desde una página web.

---

## 19. Crear el middleware del dispositivo

Agregar después de los middleware de sesión:

```javascript
function requiereClaveDispositivo(
    req,
    res,
    next
)
{
    const clave =
        req.get("x-api-key");

    if(clave !== CLAVE_DISPOSITIVO)
    {
        return res.status(401).json({
            correcto: false,
            mensaje:
                "Dispositivo no autorizado"
        });
    }

    next();
}
```

---

## 20. Crear las rutas para las páginas

Agregar dentro de las rutas de páginas:

```javascript
app.get(
    "/mediciones-altura",
    requiereSesionPagina,
    (req, res) =>
    {
        enviarPagina(
            res,
            "mediciones-altura.html"
        );
    }
);

app.get(
    "/estadisticas-altura",
    requiereSesionPagina,
    (req, res) =>
    {
        enviarPagina(
            res,
            "estadisticas-altura.html"
        );
    }
);
```

---

## 21. API para recibir una medición del ESP32

Agregar antes de la sección donde se inicia el servidor:

```javascript
// ==================================================
// API: RECIBIR MEDICIÓN DE ALTURA DEL ESP32
// ==================================================

app.post(
    "/api/mediciones-altura",
    requiereClaveDispositivo,
    async (req, res) =>
    {
        try
        {
            const productoId =
                Number(
                    req.body.producto_id
                );

            const dispositivo =
                String(
                    req.body.dispositivo || ""
                ).trim();

            const distanciaBanda =
                Number(
                    req.body.distancia_banda_cm
                );

            const distanciaObjeto =
                Number(
                    req.body.distancia_objeto_cm
                );

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

            if(dispositivo === "")
            {
                return res.status(400).json({
                    correcto: false,
                    mensaje:
                        "Falta identificar el dispositivo"
                });
            }

            if(
                !Number.isFinite(distanciaBanda) ||
                !Number.isFinite(distanciaObjeto) ||
                distanciaBanda <= 0 ||
                distanciaObjeto <= 0
            )
            {
                return res.status(400).json({
                    correcto: false,
                    mensaje:
                        "Las distancias son inválidas"
                });
            }

            if(
                distanciaObjeto >= distanciaBanda
            )
            {
                return res.status(400).json({
                    correcto: false,
                    mensaje:
                        "La distancia al objeto debe ser menor que la distancia a la banda"
                });
            }

            const alturaCm =
                Number(
                    (
                        distanciaBanda -
                        distanciaObjeto
                    ).toFixed(2)
                );

            const [productos] =
                await conexion.execute(
                    `SELECT
                        id,
                        nombre,
                        altura_min_cm,
                        altura_max_cm
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
                producto.altura_min_cm !== null &&
                producto.altura_max_cm !== null
            )
            {
                const alturaMinima =
                    Number(
                        producto.altura_min_cm
                    );

                const alturaMaxima =
                    Number(
                        producto.altura_max_cm
                    );

                if(alturaCm < alturaMinima)
                {
                    resultado =
                        "demasiado_bajo";
                }
                else if(
                    alturaCm > alturaMaxima
                )
                {
                    resultado =
                        "demasiado_alto";
                }
                else
                {
                    resultado =
                        "correcto";
                }
            }

            const [insercion] =
                await conexion.execute(
                    `INSERT INTO mediciones_altura
                    (
                        producto_id,
                        dispositivo,
                        distancia_banda_cm,
                        distancia_objeto_cm,
                        altura_cm,
                        resultado
                    )
                    VALUES (?, ?, ?, ?, ?, ?)`,
                    [
                        productoId,
                        dispositivo,
                        distanciaBanda,
                        distanciaObjeto,
                        alturaCm,
                        resultado
                    ]
                );

            const medicion = {
                id:
                    insercion.insertId,

                producto_id:
                    productoId,

                producto:
                    producto.nombre,

                dispositivo:
                    dispositivo,

                distancia_banda_cm:
                    distanciaBanda,

                distancia_objeto_cm:
                    distanciaObjeto,

                altura_cm:
                    alturaCm,

                resultado:
                    resultado,

                fecha:
                    new Date().toISOString()
            };

            io.emit(
                "nueva_medicion_altura",
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

El servidor vuelve a calcular la altura utilizando las dos distancias recibidas. De esta manera, la base de datos no depende únicamente del cálculo realizado por el ESP32.

---

## 22. API para consultar el historial

Agregar:

```javascript
// ==================================================
// API: CONSULTAR MEDICIONES DE ALTURA
// ==================================================

app.get(
    "/api/mediciones-altura",
    requiereSesionAPI,
    async (req, res) =>
    {
        try
        {
            const [mediciones] =
                await conexion.execute(
                    `SELECT
                        mediciones_altura.id,
                        mediciones_altura.dispositivo,
                        mediciones_altura.distancia_banda_cm,
                        mediciones_altura.distancia_objeto_cm,
                        mediciones_altura.altura_cm,
                        mediciones_altura.resultado,
                        mediciones_altura.fecha,

                        productos.id AS producto_id,
                        productos.nombre AS producto,
                        productos.altura_min_cm,
                        productos.altura_max_cm

                    FROM mediciones_altura

                    INNER JOIN productos
                        ON mediciones_altura.producto_id =
                           productos.id

                    ORDER BY
                        mediciones_altura.fecha DESC

                    LIMIT 100`
                );

            res.json(mediciones);
        }
        catch(error)
        {
            console.log(error);

            res.status(500).json({
                mensaje:
                    "No fue posible consultar las mediciones"
            });
        }
    }
);
```

---

## 23. API para consultar estadísticas

Agregar:

```javascript
// ==================================================
// API: ESTADÍSTICAS DE ALTURA
// ==================================================

app.get(
    "/api/estadisticas-altura",
    requiereSesionAPI,
    async (req, res) =>
    {
        try
        {
            let productoId = null;

            if(req.query.producto_id)
            {
                productoId =
                    Number(
                        req.query.producto_id
                    );

                if(
                    !Number.isInteger(productoId) ||
                    productoId <= 0
                )
                {
                    return res.status(400).json({
                        mensaje:
                            "Producto inválido"
                    });
                }
            }

            const filtro =
                productoId
                ? "WHERE producto_id = ?"
                : "";

            const parametros =
                productoId
                ? [productoId]
                : [];

            const [resumen] =
                await conexion.execute(
                    `SELECT
                        COUNT(*) AS total,

                        ROUND(
                            AVG(altura_cm),
                            2
                        ) AS promedio,

                        MIN(altura_cm)
                            AS minima,

                        MAX(altura_cm)
                            AS maxima,

                        SUM(
                            resultado = 'correcto'
                        ) AS correctas,

                        SUM(
                            resultado IN
                            (
                                'demasiado_bajo',
                                'demasiado_alto'
                            )
                        ) AS rechazadas,

                        SUM(
                            resultado =
                            'sin_especificacion'
                        ) AS sin_especificacion

                    FROM mediciones_altura

                    ${filtro}`,
                    parametros
                );

            const filtroSerie =
                productoId
                ? "WHERE mediciones_altura.producto_id = ?"
                : "";

            const [serie] =
                await conexion.execute(
                    `SELECT
                        mediciones_altura.id,
                        mediciones_altura.altura_cm,
                        mediciones_altura.resultado,
                        mediciones_altura.fecha,
                        productos.nombre AS producto

                    FROM mediciones_altura

                    INNER JOIN productos
                        ON mediciones_altura.producto_id =
                           productos.id

                    ${filtroSerie}

                    ORDER BY
                        mediciones_altura.fecha DESC

                    LIMIT 30`,
                    parametros
                );

            res.json({
                resumen:
                    resumen[0],

                serie:
                    serie
            });
        }
        catch(error)
        {
            console.log(error);

            res.status(500).json({
                mensaje:
                    "No fue posible obtener las estadísticas"
            });
        }
    }
);
```

---

## 24. Mostrar conexiones de las páginas

Agregar de manera opcional:

```javascript
io.on(
    "connection",
    (socket) =>
    {
        console.log(
            "Página conectada mediante Socket.IO"
        );
    }
);
```

---

## 25. Modificar el inicio del servidor

Al final de `server.js`, localizar:

```javascript
app.listen(PUERTO, () =>
{
    console.log(
        `Servidor disponible en http://localhost:${PUERTO}`
    );
});
```

Sustituirlo por:

```javascript
servidorHTTP.listen(
    PUERTO,
    () =>
    {
        console.log(
            `Servidor disponible en http://localhost:${PUERTO}`
        );
    }
);
```

---

# Bloque 4. Crear la página de mediciones

## 26. Crear `paginas/mediciones-altura.html`

```html
<!DOCTYPE html>
<html lang="es">

<head>
    <meta charset="UTF-8">

    <meta
        name="viewport"
        content="width=device-width, initial-scale=1.0"
    >

    <title>Mediciones de altura</title>

    <link
        rel="stylesheet"
        href="/css/estilos.css"
    >
</head>

<body>

    <header>

        <div>
            <h1>Medición automática de altura</h1>

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
        <a href="/movimiento">Movimiento</a>
        <a href="/historial">Historial</a>
        <a href="/mediciones-altura">Alturas</a>
        <a href="/estadisticas-altura">Estadísticas</a>
    </nav>

    <main>

        <section class="tarjetas">

            <article class="tarjeta">
                <h3>Última altura</h3>

                <p
                    id="ultimaAltura"
                    class="numero"
                >
                    -- cm
                </p>
            </article>

            <article class="tarjeta">
                <h3>Producto</h3>

                <p id="ultimoProducto">
                    Sin mediciones
                </p>
            </article>

            <article class="tarjeta">
                <h3>Resultado</h3>

                <p id="ultimoResultado">
                    Sin mediciones
                </p>
            </article>

            <article class="tarjeta">
                <h3>Dispositivo</h3>

                <p id="ultimoDispositivo">
                    Sin conexión
                </p>
            </article>

        </section>

        <section class="seccion">

            <h2>
                Historial de mediciones
            </h2>

            <div class="tabla-contenedor">

                <table>

                    <thead>
                        <tr>
                            <th>ID</th>
                            <th>Producto</th>
                            <th>Distancia a banda</th>
                            <th>Distancia a objeto</th>
                            <th>Altura</th>
                            <th>Resultado</th>
                            <th>Dispositivo</th>
                            <th>Fecha</th>
                        </tr>
                    </thead>

                    <tbody
                        id="tablaMediciones"
                    ></tbody>

                </table>

            </div>

        </section>

    </main>

    <script src="/socket.io/socket.io.js"></script>
    <script src="/js/comun.js"></script>
    <script src="/js/mediciones-altura.js"></script>

</body>
</html>
```

---

## 27. Crear `public/js/mediciones-altura.js`

```javascript
const socket = io();

const tabla =
    document.getElementById(
        "tablaMediciones"
    );

function textoResultado(resultado)
{
    if(resultado === "correcto")
    {
        return "Correcto";
    }

    if(resultado === "demasiado_bajo")
    {
        return "Demasiado bajo";
    }

    if(resultado === "demasiado_alto")
    {
        return "Demasiado alto";
    }

    return "Sin especificación";
}

function actualizarUltimaMedicion(
    medicion
)
{
    document.getElementById(
        "ultimaAltura"
    ).textContent =
        `${medicion.altura_cm} cm`;

    document.getElementById(
        "ultimoProducto"
    ).textContent =
        medicion.producto;

    document.getElementById(
        "ultimoResultado"
    ).textContent =
        textoResultado(
            medicion.resultado
        );

    document.getElementById(
        "ultimoDispositivo"
    ).textContent =
        medicion.dispositivo;
}

function crearFila(medicion)
{
    return `
        <tr>
            <td>${medicion.id}</td>

            <td>
                ${medicion.producto}
            </td>

            <td>
                ${medicion.distancia_banda_cm} cm
            </td>

            <td>
                ${medicion.distancia_objeto_cm} cm
            </td>

            <td>
                <strong>
                    ${medicion.altura_cm} cm
                </strong>
            </td>

            <td>
                ${textoResultado(
                    medicion.resultado
                )}
            </td>

            <td>
                ${medicion.dispositivo}
            </td>

            <td>
                ${new Date(
                    medicion.fecha
                ).toLocaleString()}
            </td>
        </tr>
    `;
}

async function cargarMediciones()
{
    const respuesta = await fetch(
        "/api/mediciones-altura"
    );

    if(!respuesta.ok)
    {
        return;
    }

    const mediciones =
        await respuesta.json();

    tabla.innerHTML = "";

    mediciones.forEach(
        medicion =>
        {
            tabla.innerHTML +=
                crearFila(medicion);
        }
    );

    if(mediciones.length > 0)
    {
        actualizarUltimaMedicion(
            mediciones[0]
        );
    }
}

socket.on(
    "nueva_medicion_altura",
    (medicion) =>
    {
        actualizarUltimaMedicion(
            medicion
        );

        tabla.insertAdjacentHTML(
            "afterbegin",
            crearFila(medicion)
        );
    }
);

document.addEventListener(
    "DOMContentLoaded",
    cargarMediciones
);
```

---

# Bloque 5. Crear la página de estadísticas

## 28. Crear `paginas/estadisticas-altura.html`

```html
<!DOCTYPE html>
<html lang="es">

<head>
    <meta charset="UTF-8">

    <meta
        name="viewport"
        content="width=device-width, initial-scale=1.0"
    >

    <title>Estadísticas de altura</title>

    <link
        rel="stylesheet"
        href="/css/estilos.css"
    >
</head>

<body>

    <header>

        <div>
            <h1>Estadísticas de altura</h1>

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
        <a href="/movimiento">Movimiento</a>
        <a href="/historial">Historial</a>
        <a href="/mediciones-altura">Alturas</a>
        <a href="/estadisticas-altura">Estadísticas</a>
    </nav>

    <main>

        <section class="seccion">

            <label for="filtroProducto">
                Producto
            </label>

            <select id="filtroProducto">
                <option value="">
                    Todos los productos
                </option>
            </select>

        </section>

        <section class="tarjetas">

            <article class="tarjeta">
                <h3>Total de piezas</h3>
                <p id="total" class="numero">0</p>
            </article>

            <article class="tarjeta">
                <h3>Altura promedio</h3>
                <p id="promedio" class="numero">0</p>
            </article>

            <article class="tarjeta">
                <h3>Altura mínima</h3>
                <p id="minima" class="numero">0</p>
            </article>

            <article class="tarjeta">
                <h3>Altura máxima</h3>
                <p id="maxima" class="numero">0</p>
            </article>

            <article class="tarjeta">
                <h3>Piezas correctas</h3>
                <p id="correctas" class="numero">0</p>
            </article>

            <article class="tarjeta">
                <h3>Piezas rechazadas</h3>
                <p id="rechazadas" class="numero">0</p>
            </article>

            <article class="tarjeta">
                <h3>Cumplimiento</h3>
                <p id="cumplimiento" class="numero">0 %</p>
            </article>

        </section>

        <section class="seccion">

            <h2>
                Altura de las últimas piezas
            </h2>

            <div class="contenedor-grafica">
                <canvas id="graficaAlturas"></canvas>
            </div>

        </section>

        <section class="seccion">

            <h2>
                Resultados de inspección
            </h2>

            <div class="contenedor-grafica-pequena">
                <canvas id="graficaResultados"></canvas>
            </div>

        </section>

    </main>

    <script
        src="https://cdn.jsdelivr.net/npm/chart.js"
    ></script>

    <script src="/socket.io/socket.io.js"></script>
    <script src="/js/comun.js"></script>
    <script src="/js/estadisticas-altura.js"></script>

</body>
</html>
```

Chart.js puede incorporarse mediante una etiqueta `script` y utilizar elementos `canvas` para representar los datos.

---

## 29. Crear `public/js/estadisticas-altura.js`

```javascript
const socket = io();

let graficaAlturas = null;
let graficaResultados = null;

const selectorProducto =
    document.getElementById(
        "filtroProducto"
    );

async function cargarProductos()
{
    const respuesta = await fetch(
        "/api/productos"
    );

    if(!respuesta.ok)
    {
        return;
    }

    const productos =
        await respuesta.json();

    productos.forEach(
        producto =>
        {
            selectorProducto.innerHTML += `
                <option value="${producto.id}">
                    ${producto.nombre}
                </option>
            `;
        }
    );
}

function numero(valor)
{
    if(
        valor === null ||
        valor === undefined
    )
    {
        return 0;
    }

    return Number(valor);
}

async function cargarEstadisticas()
{
    const productoId =
        selectorProducto.value;

    let url =
        "/api/estadisticas-altura";

    if(productoId !== "")
    {
        url +=
            `?producto_id=${productoId}`;
    }

    const respuesta =
        await fetch(url);

    if(!respuesta.ok)
    {
        return;
    }

    const datos =
        await respuesta.json();

    const resumen =
        datos.resumen;

    const total =
        numero(resumen.total);

    const correctas =
        numero(resumen.correctas);

    const rechazadas =
        numero(resumen.rechazadas);

    const sinEspecificacion =
        numero(
            resumen.sin_especificacion
        );

    let cumplimiento = 0;

    if(total > 0)
    {
        cumplimiento =
            (
                correctas /
                total *
                100
            ).toFixed(1);
    }

    document.getElementById(
        "total"
    ).textContent =
        total;

    document.getElementById(
        "promedio"
    ).textContent =
        `${numero(resumen.promedio)} cm`;

    document.getElementById(
        "minima"
    ).textContent =
        `${numero(resumen.minima)} cm`;

    document.getElementById(
        "maxima"
    ).textContent =
        `${numero(resumen.maxima)} cm`;

    document.getElementById(
        "correctas"
    ).textContent =
        correctas;

    document.getElementById(
        "rechazadas"
    ).textContent =
        rechazadas;

    document.getElementById(
        "cumplimiento"
    ).textContent =
        `${cumplimiento} %`;

    const serie =
        [...datos.serie].reverse();

    const etiquetas =
        serie.map(
            medicion =>
                `#${medicion.id}`
        );

    const alturas =
        serie.map(
            medicion =>
                Number(
                    medicion.altura_cm
                )
        );

    if(graficaAlturas)
    {
        graficaAlturas.destroy();
    }

    graficaAlturas =
        new Chart(
            document.getElementById(
                "graficaAlturas"
            ),
            {
                type: "line",

                data: {
                    labels:
                        etiquetas,

                    datasets: [
                        {
                            label:
                                "Altura en centímetros",

                            data:
                                alturas
                        }
                    ]
                },

                options: {
                    responsive: true,

                    scales: {
                        y: {
                            beginAtZero: true
                        }
                    }
                }
            }
        );

    if(graficaResultados)
    {
        graficaResultados.destroy();
    }

    graficaResultados =
        new Chart(
            document.getElementById(
                "graficaResultados"
            ),
            {
                type: "doughnut",

                data: {
                    labels: [
                        "Correctas",
                        "Rechazadas",
                        "Sin especificación"
                    ],

                    datasets: [
                        {
                            data: [
                                correctas,
                                rechazadas,
                                sinEspecificacion
                            ]
                        }
                    ]
                },

                options: {
                    responsive: true
                }
            }
        );
}

selectorProducto.addEventListener(
    "change",
    cargarEstadisticas
);

socket.on(
    "nueva_medicion_altura",
    cargarEstadisticas
);

document.addEventListener(
    "DOMContentLoaded",
    async function()
    {
        await cargarProductos();

        cargarEstadisticas();
    }
);
```

---

## 30. Agregar estilos para las gráficas

Al final de `public/css/estilos.css` agregar:

```css
.contenedor-grafica {
    position: relative;
    width: 100%;
    min-height: 350px;
}

.contenedor-grafica-pequena {
    position: relative;
    width: 100%;
    max-width: 500px;
    margin: auto;
}
```

---

## 31. Agregar los enlaces al menú

En las demás páginas privadas, agregar dentro de `<nav>`:

```html
<a href="/mediciones-altura">
    Alturas
</a>

<a href="/estadisticas-altura">
    Estadísticas
</a>
```

---

# Bloque 6. Programar el ESP32

## 32. Información que deberá modificarse

Antes de cargar el programa, identificar:

* Nombre de la red WiFi.
* Contraseña de la red.
* Dirección IPv4 de la computadora.
* ID del producto que se medirá.
* Distancia entre el HC-SR04 y la banda vacía.

Para conocer la dirección IPv4 de la computadora, abrir una terminal de Windows y ejecutar:

```bash
ipconfig
```

Buscar un valor parecido a:

```text
Dirección IPv4: 192.168.1.25
```

No se debe utilizar:

```text
localhost
```

ni:

```text
127.0.0.1
```

desde el ESP32, porque esas direcciones representarían al propio ESP32 y no a la computadora.

---

## 33. Código completo del ESP32

```cpp
#include <WiFi.h>
#include <HTTPClient.h>

// ==================================================
// CONFIGURACIÓN DE RED
// ==================================================

const char* NOMBRE_WIFI =
    "NOMBRE_DE_LA_RED";

const char* CONTRASENA_WIFI =
    "CONTRASENA_DE_LA_RED";

// Cambiar la dirección IP por la dirección
// IPv4 de la computadora que ejecuta Node.js.
const char* URL_SERVIDOR =
    "http://192.168.1.25:3000/api/mediciones-altura";

// Debe ser exactamente igual a la clave
// configurada en server.js.
const char* CLAVE_DISPOSITIVO =
    "clave-banda-industria-40";

// ==================================================
// CONFIGURACIÓN DEL PRODUCTO Y LA BANDA
// ==================================================

// Cambiar por el ID mostrado en la tabla productos.
const int PRODUCTO_ID = 1;

const char* NOMBRE_DISPOSITIVO =
    "banda_01";

// Medir físicamente esta distancia.
// Corresponde al sensor y la banda vacía.
const float DISTANCIA_BANDA_CM =
    30.0;

// ==================================================
// PINES
// ==================================================

const int PIN_TRIG = 5;
const int PIN_ECHO = 18;
const int PIN_SENSOR_IR = 19;

// La mayoría de los módulos infrarrojos
// entregan LOW cuando detectan un objeto.
// Cambiar a HIGH si el módulo trabaja al revés.
const int ESTADO_OBJETO = LOW;

// ==================================================
// CONFIGURACIÓN DE MEDICIÓN
// ==================================================

const int TOTAL_MUESTRAS = 7;

// Tiempo para permitir que el objeto llegue
// al centro del sensor ultrasónico.
const unsigned long RETARDO_CENTRADO_MS =
    80;

// Tiempo durante el cual el sensor debe quedar
// libre antes de aceptar un nuevo objeto.
const unsigned long TIEMPO_LIBRE_MS =
    150;

bool objetoProcesado = false;

unsigned long inicioZonaLibre = 0;

// ==================================================
// CONECTAR A WIFI
// ==================================================

void conectarWiFi()
{
    Serial.println();
    Serial.println("Conectando a WiFi");

    WiFi.begin(
        NOMBRE_WIFI,
        CONTRASENA_WIFI
    );

    while(WiFi.status() != WL_CONNECTED)
    {
        delay(500);

        Serial.print(".");
    }

    Serial.println();
    Serial.println("WiFi conectado");

    Serial.print("IP del ESP32: ");
    Serial.println(
        WiFi.localIP()
    );
}

// ==================================================
// MEDIR DISTANCIA
// ==================================================

float medirDistanciaCm()
{
    digitalWrite(
        PIN_TRIG,
        LOW
    );

    delayMicroseconds(2);

    digitalWrite(
        PIN_TRIG,
        HIGH
    );

    delayMicroseconds(10);

    digitalWrite(
        PIN_TRIG,
        LOW
    );

    unsigned long duracion =
        pulseIn(
            PIN_ECHO,
            HIGH,
            30000
        );

    if(duracion == 0)
    {
        return -1.0;
    }

    float distancia =
        duracion *
        0.0343 /
        2.0;

    return distancia;
}

// ==================================================
// ORDENAR ARREGLO
// ==================================================

void ordenarMuestras(
    float datos[],
    int cantidad
)
{
    for(int i = 0; i < cantidad - 1; i++)
    {
        for(
            int j = 0;
            j < cantidad - i - 1;
            j++
        )
        {
            if(datos[j] > datos[j + 1])
            {
                float temporal =
                    datos[j];

                datos[j] =
                    datos[j + 1];

                datos[j + 1] =
                    temporal;
            }
        }
    }
}

// ==================================================
// OBTENER MEDIANA
// ==================================================

float obtenerDistanciaEstable()
{
    float muestras[TOTAL_MUESTRAS];

    int muestrasValidas = 0;

    int intentos = 0;

    while(
        muestrasValidas < TOTAL_MUESTRAS &&
        intentos < TOTAL_MUESTRAS * 2
    )
    {
        float distancia =
            medirDistanciaCm();

        intentos++;

        if(
            distancia > 1.0 &&
            distancia < DISTANCIA_BANDA_CM
        )
        {
            muestras[muestrasValidas] =
                distancia;

            muestrasValidas++;
        }

        delay(30);
    }

    if(muestrasValidas < 3)
    {
        return -1.0;
    }

    ordenarMuestras(
        muestras,
        muestrasValidas
    );

    int posicionCentral =
        muestrasValidas / 2;

    if(muestrasValidas % 2 == 1)
    {
        return
            muestras[posicionCentral];
    }

    return
        (
            muestras[posicionCentral - 1] +
            muestras[posicionCentral]
        ) /
        2.0;
}

// ==================================================
// ENVIAR MEDICIÓN AL SERVIDOR
// ==================================================

void enviarMedicion(
    float distanciaObjeto
)
{
    if(WiFi.status() != WL_CONNECTED)
    {
        conectarWiFi();
    }

    float altura =
        DISTANCIA_BANDA_CM -
        distanciaObjeto;

    HTTPClient http;

    http.begin(
        URL_SERVIDOR
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
        "\"dispositivo\":\"" +
        String(NOMBRE_DISPOSITIVO) +
        "\",";

    datos +=
        "\"distancia_banda_cm\":" +
        String(
            DISTANCIA_BANDA_CM,
            2
        ) +
        ",";

    datos +=
        "\"distancia_objeto_cm\":" +
        String(
            distanciaObjeto,
            2
        );

    datos += "}";

    Serial.println();
    Serial.println("Enviando al servidor:");

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
            "Respuesta del servidor:"
        );

        Serial.println(
            respuesta
        );
    }
    else
    {
        Serial.println(
            "No fue posible comunicarse con el servidor"
        );
    }

    http.end();

    Serial.print(
        "Altura calculada localmente: "
    );

    Serial.print(
        altura,
        2
    );

    Serial.println(" cm");
}

// ==================================================
// CONFIGURACIÓN
// ==================================================

void setup()
{
    Serial.begin(115200);

    pinMode(
        PIN_TRIG,
        OUTPUT
    );

    pinMode(
        PIN_ECHO,
        INPUT
    );

    pinMode(
        PIN_SENSOR_IR,
        INPUT
    );

    digitalWrite(
        PIN_TRIG,
        LOW
    );

    conectarWiFi();

    Serial.println();
    Serial.println(
        "Sistema preparado"
    );

    Serial.print(
        "Distancia de referencia: "
    );

    Serial.print(
        DISTANCIA_BANDA_CM,
        2
    );

    Serial.println(" cm");
}

// ==================================================
// PROGRAMA PRINCIPAL
// ==================================================

void loop()
{
    bool objetoPresente =
        digitalRead(
            PIN_SENSOR_IR
        ) == ESTADO_OBJETO;

    if(
        objetoPresente &&
        !objetoProcesado
    )
    {
        Serial.println();
        Serial.println(
            "Objeto detectado"
        );

        delay(
            RETARDO_CENTRADO_MS
        );

        float distanciaObjeto =
            obtenerDistanciaEstable();

        if(distanciaObjeto > 0)
        {
            float altura =
                DISTANCIA_BANDA_CM -
                distanciaObjeto;

            Serial.print(
                "Distancia al objeto: "
            );

            Serial.print(
                distanciaObjeto,
                2
            );

            Serial.println(" cm");

            Serial.print(
                "Altura estimada: "
            );

            Serial.print(
                altura,
                2
            );

            Serial.println(" cm");

            enviarMedicion(
                distanciaObjeto
            );
        }
        else
        {
            Serial.println(
                "No se obtuvieron suficientes muestras válidas"
            );
        }

        objetoProcesado = true;

        inicioZonaLibre = 0;
    }

    if(!objetoPresente)
    {
        if(inicioZonaLibre == 0)
        {
            inicioZonaLibre =
                millis();
        }

        if(
            millis() -
            inicioZonaLibre >=
            TIEMPO_LIBRE_MS
        )
        {
            objetoProcesado =
                false;
        }
    }
    else
    {
        inicioZonaLibre = 0;
    }
}
```

El ESP32 dispone de soporte para realizar solicitudes HTTP mediante sus bibliotecas de red.

---

# 34. Variables que deben modificarse

Antes de cargar el programa, cambiar:

```cpp
const char* NOMBRE_WIFI =
    "NOMBRE_DE_LA_RED";
```

```cpp
const char* CONTRASENA_WIFI =
    "CONTRASENA_DE_LA_RED";
```

```cpp
const char* URL_SERVIDOR =
    "http://DIRECCION_IP:3000/api/mediciones-altura";
```

```cpp
const int PRODUCTO_ID = 1;
```

```cpp
const float DISTANCIA_BANDA_CM =
    30.0;
```

La distancia de referencia debe medirse físicamente.

No se debe copiar el valor de 30 cm sin comprobarlo.

---

# Bloque 7. Calibrar el sistema

## 35. Medir la distancia hasta la banda

Con la banda vacía:

1. Apagar o detener la banda.
2. Colocar el HC-SR04 en su posición definitiva.
3. Medir con una regla la distancia entre el sensor y la banda.
4. Registrar el valor.
5. Colocarlo en `DISTANCIA_BANDA_CM`.
6. Ejecutar varias mediciones sin objetos.
7. Verificar que el valor sea estable.

Ejemplo:

```text
Medición con regla: 30.0 cm
Medición ultrasónica: 29.8 cm
```

---

## 36. Comprobar con objetos conocidos

Utilizar al menos tres objetos de altura conocida.

| Objeto   | Altura con regla | Altura medida | Error absoluto |
| -------- | ---------------: | ------------: | -------------: |
| Objeto 1 |           5.0 cm |               |                |
| Objeto 2 |          10.0 cm |               |                |
| Objeto 3 |          15.0 cm |               |                |

Calcular:

```text
Error absoluto =
|Altura medida − Altura con regla|
```

Si los errores son elevados, revisar:

* Posición vertical del HC-SR04.
* Vibraciones.
* Distancia de referencia.
* Posición del sensor infrarrojo.
* Tiempo de centrado.
* Superficie superior del objeto.
* Velocidad de la banda.

---

## 37. Ajustar el tiempo de centrado

Si el infrarrojo detecta el objeto antes de que quede debajo del HC-SR04, modificar:

```cpp
const unsigned long RETARDO_CENTRADO_MS =
    80;
```

Ejemplos:

```text
El objeto se mide demasiado pronto:
aumentar el retardo.

El objeto avanza demasiado antes de medirse:
reducir el retardo.
```

El valor dependerá de:

* Distancia entre los sensores.
* Velocidad de la banda.
* Longitud del objeto.

---

# Bloque 8. Ejecutar el sistema completo

## 38. Iniciar MySQL

Comprobar que MySQL Server se encuentre ejecutándose.

---

## 39. Iniciar Node.js

Desde la carpeta del proyecto:

```bash
node server.js
```

Resultado esperado:

```text
Conexión correcta con MySQL
Servidor disponible en http://localhost:3000
```

---

## 40. Abrir la página de mediciones

Abrir:

```text
http://localhost:3000/mediciones-altura
```

---

## 41. Abrir la página de estadísticas

Abrir:

```text
http://localhost:3000/estadisticas-altura
```

---

## 42. Encender el ESP32

Abrir el monitor serial a:

```text
115200 baudios
```

El resultado esperado será parecido a:

```text
WiFi conectado
IP del ESP32: 192.168.1.40
Sistema preparado
Distancia de referencia: 30.00 cm
```

---

# Bloque 9. Pruebas obligatorias

## Prueba 1. Detección de un objeto

Pasar un objeto por la banda.

El monitor serial debe mostrar:

```text
Objeto detectado
Distancia al objeto: 18.20 cm
Altura estimada: 11.80 cm
```

---

## Prueba 2. Registro único

Mantener un objeto frente al sensor infrarrojo.

El sistema deberá registrar solamente una medición.

No debe crear registros continuamente mientras el mismo objeto permanezca en la zona.

---

## Prueba 3. Nuevo objeto

Retirar completamente el primer objeto y pasar otro.

Después de que el sensor infrarrojo permanezca libre, el sistema deberá aceptar una nueva medición.

---

## Prueba 4. Producto correcto

Configurar:

```text
Altura mínima: 9.5 cm
Altura máxima: 10.5 cm
```

Pasar un objeto de aproximadamente 10 cm.

Resultado esperado:

```text
correcto
```

---

## Prueba 5. Producto demasiado bajo

Pasar un objeto cuya altura sea menor que el límite mínimo.

Resultado esperado:

```text
demasiado_bajo
```

---

## Prueba 6. Producto demasiado alto

Pasar un objeto cuya altura sea mayor que el límite máximo.

Resultado esperado:

```text
demasiado_alto
```

---

## Prueba 7. Actualización en tiempo real

Mantener abierta la página:

```text
/mediciones-altura
```

Al pasar un objeto, la nueva medición deberá aparecer sin recargar la página.

---

## Prueba 8. Estadísticas

Registrar al menos diez objetos de diferentes alturas.

Comprobar que la página muestre:

* Total de piezas.
* Altura promedio.
* Altura mínima.
* Altura máxima.
* Piezas correctas.
* Piezas rechazadas.
* Porcentaje de cumplimiento.
* Gráfica de las últimas alturas.
* Gráfica de resultados.

---

## Prueba 9. Persistencia

Detener el servidor con:

```text
Ctrl + C
```

Volver a iniciarlo:

```bash
node server.js
```

Las mediciones anteriores deben continuar disponibles porque fueron almacenadas en MySQL.

---

# 43. Consultas SQL de comprobación

## Consultar todas las mediciones

```sql
SELECT
    *
FROM mediciones_altura
ORDER BY fecha DESC;
```

---

## Consultar mediciones con el nombre del producto

```sql
SELECT
    mediciones_altura.id,
    productos.nombre AS producto,
    mediciones_altura.altura_cm,
    mediciones_altura.resultado,
    mediciones_altura.dispositivo,
    mediciones_altura.fecha

FROM mediciones_altura

INNER JOIN productos
    ON mediciones_altura.producto_id =
       productos.id

ORDER BY
    mediciones_altura.fecha DESC;
```

---

## Consultar promedio, mínimo y máximo

```sql
SELECT
    COUNT(*) AS total,

    ROUND(
        AVG(altura_cm),
        2
    ) AS altura_promedio,

    MIN(altura_cm)
        AS altura_minima,

    MAX(altura_cm)
        AS altura_maxima

FROM mediciones_altura;
```

---

## Contar resultados

```sql
SELECT
    resultado,
    COUNT(*) AS cantidad

FROM mediciones_altura

GROUP BY resultado;
```

---

## Obtener estadísticas por producto

```sql
SELECT
    productos.nombre,

    COUNT(*) AS total,

    ROUND(
        AVG(
            mediciones_altura.altura_cm
        ),
        2
    ) AS promedio,

    MIN(
        mediciones_altura.altura_cm
    ) AS minima,

    MAX(
        mediciones_altura.altura_cm
    ) AS maxima

FROM mediciones_altura

INNER JOIN productos
    ON mediciones_altura.producto_id =
       productos.id

GROUP BY
    productos.id,
    productos.nombre;
```

---

# 44. Flujo completo del sistema

```text
El objeto llega a la zona de medición
                 │
                 ▼
El sensor infrarrojo cambia de estado
                 │
                 ▼
El ESP32 espera el tiempo de centrado
                 │
                 ▼
El HC-SR04 obtiene siete muestras
                 │
                 ▼
Se eliminan lecturas inválidas
                 │
                 ▼
Se ordenan las muestras
                 │
                 ▼
Se obtiene la mediana
                 │
                 ▼
El ESP32 envía las distancias por HTTP
                 │
                 ▼
Node.js calcula la altura
                 │
                 ▼
Node.js consulta los límites del producto
                 │
                 ▼
Clasifica la pieza
                 │
                 ▼
Guarda el resultado en MySQL
                 │
                 ▼
Socket.IO actualiza las páginas
```

---

# 45. Evidencias que deberán entregarse

El reporte de la práctica deberá incluir:

1. Portada.
2. Objetivo de la práctica.
3. Diagrama de conexión.
4. Fotografía del montaje.
5. Distancia de referencia utilizada.
6. Tabla de calibración con tres objetos.
7. Captura del monitor serial.
8. Captura de la tabla `mediciones_altura`.
9. Captura de la página de mediciones.
10. Captura de la página de estadísticas.
11. Explicación del funcionamiento del sensor infrarrojo.
12. Explicación del funcionamiento del HC-SR04.
13. Explicación del cálculo de la altura.
14. Explicación de la mediana.
15. Análisis de los errores encontrados.
16. Conclusiones.

---

# 46. Investigar

Investigar y explicar con palabras propias:

* ¿Cómo funciona el sensor HC-SR04?
* ¿Cuál es la función del pin `TRIG`?
* ¿Cuál es la función del pin `ECHO`?
* ¿Por qué se divide entre dos el tiempo del eco?
* ¿Por qué no debe conectarse directamente `ECHO` al ESP32?
* ¿Cómo funciona un divisor de voltaje?
* ¿Qué diferencia existe entre detectar un objeto y medir su altura?
* ¿Por qué se utiliza el sensor infrarrojo?
* ¿Qué es la mediana?
* ¿Qué diferencia existe entre promedio y mediana?
* ¿Por qué se realizan varias mediciones?
* ¿Qué es una solicitud HTTP POST?
* ¿Qué información contiene un objeto JSON?
* ¿Qué función tiene la clave `x-api-key`?
* ¿Qué es Socket.IO?
* ¿Qué función realizan `COUNT`, `AVG`, `MIN` y `MAX` en MySQL?
* ¿Qué factores pueden producir errores en la medición ultrasónica?
* ¿Por qué el servidor vuelve a calcular la altura?
* ¿Qué diferencia existe entre una medición estimada y una medición exacta?

---

# 47. Consideraciones y limitaciones

El HC-SR04 puede producir mediciones incorrectas cuando:

* La superficie del objeto está inclinada.
* La superficie absorbe el sonido.
* El objeto es demasiado pequeño.
* El objeto tiene forma irregular.
* La banda se mueve demasiado rápido.
* El sensor vibra.
* Dos objetos pasan demasiado juntos.
* El objeto no se encuentra debajo del sensor.
* Existen obstáculos cercanos que reflejan el sonido.

Para esta práctica se recomienda utilizar:

* Cajas.
* Bloques.
* Envases con parte superior plana.
* Objetos suficientemente anchos.
* Velocidad baja o media en la banda.

---

# 48. Material para las prácticas posteriores

Cada equipo deberá conseguir los siguientes componentes:

## Celda de carga con módulo HX711

Se utilizará para:

* Medir el peso de cada objeto.
* Comparar el peso con especificaciones.
* Detectar productos incompletos.
* Guardar el peso en MySQL.

## Lector RFID RC522

Deberá incluir al menos una tarjeta o llavero RFID.

Se utilizará para:

* Identificar productos.
* Identificar lotes.
* Dar trazabilidad a cada pieza.
* Relacionar las mediciones con un identificador único.

## Sensor de color TCS34725

Se utilizará para:

* Medir el color de los objetos.
* Clasificar piezas por color.
* Detectar colores incorrectos.
* Guardar valores de color en MySQL.

## Componentes adicionales recomendados

* Protoboard.
* Cables Dupont macho-macho.
* Cables Dupont macho-hembra.
* Elementos para sujetar los sensores.
* Fuente de alimentación adecuada.
* Objetos de diferentes pesos y colores.

Estos sensores se incorporarán progresivamente al sistema:

```text
Práctica actual
    Altura con HC-SR04
            │
            ▼
Práctica posterior
    Peso con celda de carga y HX711
            │
            ▼
Práctica posterior
    Identificación con RFID RC522
            │
            ▼
Práctica posterior
    Clasificación por color con TCS34725
```

Al concluir las siguientes prácticas, cada objeto podrá tener almacenados:

```text
Identificación
Producto
Lote
Altura
Peso
Color
Resultado de inspección
Fecha
Dispositivo
Historial del proceso
```
