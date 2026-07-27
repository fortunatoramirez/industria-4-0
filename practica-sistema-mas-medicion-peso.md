# Práctica: Medición automática del peso de piezas

## Celda de carga con HX711, ESP32, Node.js y MySQL

---

# 1. Introducción

En las prácticas anteriores se desarrolló un sistema de Industria 4.0 que permite:

* Registrar usuarios.
* Iniciar sesión.
* Administrar productos.
* Registrar movimientos de inventario.
* Detectar piezas sobre una banda.
* Medir la altura de las piezas con un HC-SR04.
* Guardar mediciones en MySQL.
* Consultar estadísticas desde páginas web.

En esta práctica se agregará una estación para medir el peso de las piezas que salen de la banda transportadora.

Las piezas caerán dentro de una caja colocada sobre una celda de carga. Como la caja irá acumulando piezas, el peso de cada nueva pieza se calculará mediante una resta:

```text
Peso de la pieza =
Peso acumulado actual − Peso acumulado anterior
```

Ejemplo:

```text
Peso acumulado anterior: 1,250 g
Peso acumulado actual:   1,372 g

Peso de la pieza:          122 g
```

La lectura no se realizará inmediatamente después de la caída, porque el impacto produce oscilaciones. El ESP32 esperará hasta que el peso sea estable y después enviará la información al servidor.

---

# 2. Objetivo general

Desarrollar una estación de pesaje capaz de detectar el aumento de peso producido por la caída de una pieza, calcular su peso individual, evaluar si cumple con las especificaciones y almacenar el resultado en MySQL.

---

# 3. Objetivos específicos

Al finalizar la práctica, el estudiante será capaz de:

* Conectar una celda de carga al módulo HX711.
* Conectar el HX711 al ESP32.
* Calibrar una celda de carga utilizando una masa conocida.
* Realizar la tara de una caja vacía.
* Obtener lecturas de peso en gramos.
* Detectar un incremento significativo de peso.
* Esperar hasta que la lectura sea estable.
* Calcular el peso individual mediante una resta.
* Enviar mediciones mediante HTTP.
* Guardar el peso individual y acumulado en MySQL.
* Clasificar piezas por rango de peso.
* Actualizar páginas mediante Socket.IO.
* Obtener estadísticas de las piezas pesadas.

---

# 4. Resultado esperado

El sistema tendrá el siguiente funcionamiento:

```text
La pieza sale de la banda
            │
            ▼
Cae dentro de la caja
            │
            ▼
La celda de carga detecta un aumento
            │
            ▼
El ESP32 espera que el peso se estabilice
            │
            ▼
Calcula la diferencia de peso
            │
            ▼
Peso de la pieza =
peso actual − peso anterior
            │
            ▼
Envía los datos a Node.js
            │
            ▼
Node.js compara con las especificaciones
            │
            ▼
Guarda la medición en MySQL
            │
            ▼
Actualiza monitoreo y estadísticas
```

---

# 5. Material necesario

## Hardware

* Una tarjeta ESP32.
* Una celda de carga.
* Un módulo HX711.
* Una plataforma o base para colocar la caja.
* Una caja para recibir las piezas.
* Protoboard.
* Cables Dupont.
* Tornillos, separadores o soportes para montar la celda.
* Una masa conocida para calibrar.
* Piezas de diferentes pesos.
* Banda transportadora.

## Software

* Arduino IDE.
* Node.js.
* MySQL Server.
* Visual Studio Code.
* Navegador web.
* Proyecto desarrollado en las prácticas anteriores.

---

# 6. Funcionamiento de la celda de carga

Una celda de carga convierte una deformación mecánica muy pequeña en una señal eléctrica.

Cuando se coloca peso sobre la plataforma:

```text
Peso aplicado
     │
     ▼
La celda se deforma ligeramente
     │
     ▼
Cambia su señal eléctrica
     │
     ▼
El HX711 amplifica y digitaliza la señal
     │
     ▼
El ESP32 obtiene el peso
```

La señal de la celda es demasiado pequeña para conectarse directamente al ESP32. Por ello se utiliza el módulo HX711.

---

# 7. Instalación mecánica

La instalación mecánica es tan importante como la conexión eléctrica.

Una celda de carga de barra normalmente tiene:

* Un extremo fijo.
* Un extremo móvil.
* Una flecha que indica la dirección de la carga.

Montaje general:

```text
                 Caja receptora
          ┌─────────────────────┐
          │                     │
          └──────────┬──────────┘
                     │
                Plataforma
          ┌──────────┴──────────┐
          │                     │
          └──────────┬──────────┘
                     │
              Extremo móvil
         ┌───────────────────────┐
         │    Celda de carga     │
         └───────────────────────┘
              Extremo fijo
                     │
               Base rígida
════════════════════════════════════
```

## Recomendaciones

* Sujetar firmemente el extremo fijo.
* Colocar la plataforma sobre el extremo móvil.
* Evitar que la plataforma toque la estructura.
* Aplicar el peso verticalmente.
* Evitar fuerzas laterales.
* No exceder la capacidad máxima de la celda.
* Evitar que la banda golpee directamente la celda.
* Procurar que las piezas caigan cerca del centro de la caja.
* Reducir las vibraciones de la estructura.

---

# 8. Conexión de la celda al HX711

Las celdas de carga pueden utilizar diferentes colores de cables. Por esta razón, se deben revisar las etiquetas o la documentación de la celda.

La conexión general es:

| Celda de carga      | HX711 |
| ------------------- | ----- |
| Excitación positiva | E+    |
| Excitación negativa | E−    |
| Señal positiva      | A+    |
| Señal negativa      | A−    |

> No se deben confiar únicamente en los colores de los cables, porque pueden cambiar entre fabricantes.

Si al colocar peso la lectura disminuye, se puede:

* Intercambiar `A+` y `A−`.
* Conservar la conexión y utilizar un factor de calibración negativo.

---

# 9. Conexión del HX711 al ESP32

| HX711     | ESP32   |
| --------- | ------- |
| VCC       | 3.3 V   |
| GND       | GND     |
| DT o DOUT | GPIO 16 |
| SCK o CLK | GPIO 17 |

El módulo se alimentará con 3.3 V para que la señal digital entregada al ESP32 sea compatible con sus entradas.

---

# Bloque 1. Instalar la biblioteca

## 10. Instalar la biblioteca HX711

Desde Arduino IDE:

1. Abrir el administrador de bibliotecas.
2. Buscar:

```text
HX711
```

3. Instalar la biblioteca:

```text
HX711 Arduino Library
```

4. Comprobar que pueda utilizarse:

```cpp
#include <HX711.h>
```

---

# Bloque 2. Calibrar la celda de carga

## 11. Preparar una masa conocida

Para calibrar se necesita un objeto cuyo peso sea conocido.

Ejemplos:

```text
100 g
200 g
500 g
1,000 g
```

Se recomienda utilizar una masa suficientemente grande respecto a la capacidad de la celda.

Ejemplo:

```text
Celda de 5 kg
Masa de calibración recomendada: 500 g o 1 kg
```

No es recomendable calibrar una celda de 5 kg usando solamente una masa de 5 g.

---

## 12. Programa de calibración

Crear un programa nuevo en Arduino IDE:

```cpp
#include <HX711.h>

const int PIN_DOUT = 16;
const int PIN_SCK = 17;

// Cambiar por el peso real del objeto
// utilizado para calibrar.
const float PESO_CONOCIDO_G = 500.0;

HX711 bascula;

void limpiarEntradaSerial()
{
    while(Serial.available())
    {
        Serial.read();
    }
}

void esperarEnter()
{
    while(!Serial.available())
    {
        delay(10);
    }

    limpiarEntradaSerial();
}

void setup()
{
    Serial.begin(115200);

    bascula.begin(
        PIN_DOUT,
        PIN_SCK
    );

    Serial.println();
    Serial.println(
        "CALIBRACIÓN DE LA CELDA DE CARGA"
    );

    Serial.println();
    Serial.println(
        "1. Coloque la caja o plataforma vacía."
    );

    Serial.println(
        "2. Presione Enter para realizar la tara."
    );

    esperarEnter();

    bascula.set_scale();

    bascula.tare(20);

    Serial.println();
    Serial.println(
        "Tara terminada."
    );

    Serial.println();
    Serial.print(
        "Coloque una masa conocida de "
    );

    Serial.print(
        PESO_CONOCIDO_G
    );

    Serial.println(" g.");

    Serial.println(
        "Presione Enter cuando la lectura esté estable."
    );

    esperarEnter();

    long lectura =
        bascula.get_value(30);

    float factorCalibracion =
        lectura /
        PESO_CONOCIDO_G;

    Serial.println();
    Serial.print(
        "Lectura sin convertir: "
    );

    Serial.println(
        lectura
    );

    Serial.print(
        "Factor de calibración: "
    );

    Serial.println(
        factorCalibracion,
        6
    );

    Serial.println();
    Serial.println(
        "Guarde este factor para el programa final."
    );
}

void loop()
{
}
```

---

## 13. Procedimiento de calibración

1. Colocar la caja vacía sobre la plataforma.
2. Iniciar el monitor serial a 115200 baudios.
3. Presionar Enter para realizar la tara.
4. Colocar la masa conocida dentro de la caja.
5. Esperar a que no existan movimientos.
6. Presionar Enter.
7. Copiar el factor mostrado.

Ejemplo:

```text
Factor de calibración: -421.782013
```

El valor puede ser positivo o negativo.

Cada celda y cada montaje tendrán un factor diferente.

---

## 14. Comprobar la calibración

Crear una prueba sencilla:

```cpp
#include <HX711.h>

const int PIN_DOUT = 16;
const int PIN_SCK = 17;

const float FACTOR_CALIBRACION =
    -421.782013;

HX711 bascula;

void setup()
{
    Serial.begin(115200);

    bascula.begin(
        PIN_DOUT,
        PIN_SCK
    );

    bascula.set_scale(
        FACTOR_CALIBRACION
    );

    Serial.println(
        "Coloque la caja vacía."
    );

    delay(3000);

    bascula.tare(20);

    Serial.println(
        "Tara terminada."
    );
}

void loop()
{
    if(bascula.wait_ready_timeout(1000))
    {
        float peso =
            bascula.get_units(10);

        Serial.print("Peso: ");
        Serial.print(peso, 2);
        Serial.println(" g");
    }
    else
    {
        Serial.println(
            "HX711 no disponible"
        );
    }

    delay(500);
}
```

Colocar diferentes objetos y comparar el resultado con una báscula de referencia.

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

## 16. Agregar límites de peso a los productos

Ejecutar una sola vez:

```sql
ALTER TABLE productos
ADD peso_min_g DECIMAL(8,2) NULL,
ADD peso_max_g DECIMAL(8,2) NULL;
```

Los nuevos campos representarán:

| Campo        | Función               |
| ------------ | --------------------- |
| `peso_min_g` | Peso mínimo permitido |
| `peso_max_g` | Peso máximo permitido |

---

## 17. Configurar un producto

Consultar los productos:

```sql
SELECT
    id,
    nombre,
    peso_min_g,
    peso_max_g
FROM productos;
```

Configurar un producto.

Ejemplo:

```sql
UPDATE productos
SET
    peso_min_g = 95.00,
    peso_max_g = 105.00
WHERE id = 1;
```

En este ejemplo, una pieza será correcta si pesa entre 95 y 105 gramos.

---

## 18. Crear la tabla de mediciones

Ejecutar:

```sql
CREATE TABLE mediciones_peso (
    id INT AUTO_INCREMENT PRIMARY KEY,

    producto_id INT NOT NULL,

    caja VARCHAR(50) NOT NULL,

    numero_pieza INT NOT NULL,

    peso_anterior_g DECIMAL(10,2) NOT NULL,

    peso_actual_g DECIMAL(10,2) NOT NULL,

    peso_pieza_g DECIMAL(8,2) NOT NULL,

    resultado VARCHAR(30) NOT NULL,

    dispositivo VARCHAR(50) NOT NULL,

    fecha TIMESTAMP
        DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY (producto_id)
        REFERENCES productos(id)
);
```

El campo `resultado` podrá contener:

```text
correcto
demasiado_ligero
demasiado_pesado
sin_especificacion
```

La caja se identificará temporalmente como:

```text
CAJA_01
```

En la práctica posterior, este nombre fijo será sustituido por el UID obtenido mediante el lector RFID RC522.

---

## 19. Verificar la tabla

```sql
DESCRIBE mediciones_peso;
```

Salir de MySQL:

```sql
EXIT;
```

---

# Bloque 4. Modificar el servidor Node.js

Esta práctica parte del servidor desarrollado anteriormente. Por lo tanto, se reutilizarán:

* Express.
* MySQL.
* Sesiones.
* Socket.IO.
* La conexión `conexion`.
* El middleware `requiereSesionPagina`.
* El middleware `requiereSesionAPI`.
* El middleware `requiereClaveDispositivo`.
* La variable `io`.

---

## 20. Crear las rutas de las páginas

Agregar en `server.js`:

```javascript
app.get(
    "/mediciones-peso",
    requiereSesionPagina,
    (req, res) =>
    {
        enviarPagina(
            res,
            "mediciones-peso.html"
        );
    }
);

app.get(
    "/estadisticas-peso",
    requiereSesionPagina,
    (req, res) =>
    {
        enviarPagina(
            res,
            "estadisticas-peso.html"
        );
    }
);
```

---

## 21. API para recibir una medición

Agregar en `server.js` antes de iniciar el servidor:

```javascript
// ==================================================
// API: RECIBIR MEDICIÓN DE PESO
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

            const caja =
                String(
                    req.body.caja || ""
                ).trim();

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
                caja === "" ||
                dispositivo === ""
            )
            {
                return res.status(400).json({
                    correcto: false,
                    mensaje:
                        "Falta identificar la caja o el dispositivo"
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

            const [insercion] =
                await conexion.execute(
                    `INSERT INTO mediciones_peso
                    (
                        producto_id,
                        caja,
                        numero_pieza,
                        peso_anterior_g,
                        peso_actual_g,
                        peso_pieza_g,
                        resultado,
                        dispositivo
                    )
                    VALUES (?, ?, ?, ?, ?, ?, ?, ?)`,
                    [
                        productoId,
                        caja,
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

                caja:
                    caja,

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
                    "Medición de peso registrada",

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

El servidor vuelve a calcular el peso individual mediante la resta. De esta manera, no depende únicamente del cálculo realizado por el ESP32.

---

## 22. API para consultar las mediciones

Agregar:

```javascript
// ==================================================
// API: CONSULTAR MEDICIONES DE PESO
// ==================================================

app.get(
    "/api/mediciones-peso",
    requiereSesionAPI,
    async (req, res) =>
    {
        try
        {
            const [mediciones] =
                await conexion.execute(
                    `SELECT
                        mediciones_peso.id,
                        mediciones_peso.caja,
                        mediciones_peso.numero_pieza,
                        mediciones_peso.peso_anterior_g,
                        mediciones_peso.peso_actual_g,
                        mediciones_peso.peso_pieza_g,
                        mediciones_peso.resultado,
                        mediciones_peso.dispositivo,
                        mediciones_peso.fecha,

                        productos.id AS producto_id,
                        productos.nombre AS producto,
                        productos.peso_min_g,
                        productos.peso_max_g

                    FROM mediciones_peso

                    INNER JOIN productos
                        ON mediciones_peso.producto_id =
                           productos.id

                    ORDER BY
                        mediciones_peso.fecha DESC

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
// API: ESTADÍSTICAS DE PESO
// ==================================================

app.get(
    "/api/estadisticas-peso",
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
                            AVG(peso_pieza_g),
                            2
                        ) AS promedio,

                        MIN(peso_pieza_g)
                            AS minimo,

                        MAX(peso_pieza_g)
                            AS maximo,

                        ROUND(
                            STDDEV_POP(
                                peso_pieza_g
                            ),
                            2
                        ) AS desviacion,

                        SUM(
                            resultado = 'correcto'
                        ) AS correctas,

                        SUM(
                            resultado IN
                            (
                                'demasiado_ligero',
                                'demasiado_pesado'
                            )
                        ) AS rechazadas,

                        SUM(
                            resultado =
                            'sin_especificacion'
                        ) AS sin_especificacion,

                        MAX(peso_actual_g)
                            AS peso_acumulado

                    FROM mediciones_peso

                    ${filtro}`,
                    parametros
                );

            const filtroSerie =
                productoId
                ? "WHERE mediciones_peso.producto_id = ?"
                : "";

            const [serie] =
                await conexion.execute(
                    `SELECT
                        mediciones_peso.id,
                        mediciones_peso.numero_pieza,
                        mediciones_peso.peso_pieza_g,
                        mediciones_peso.resultado,
                        mediciones_peso.fecha,
                        productos.nombre AS producto

                    FROM mediciones_peso

                    INNER JOIN productos
                        ON mediciones_peso.producto_id =
                           productos.id

                    ${filtroSerie}

                    ORDER BY
                        mediciones_peso.fecha DESC

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

# Bloque 5. Crear la página de monitoreo

## 24. Crear `paginas/mediciones-peso.html`

```html
<!DOCTYPE html>
<html lang="es">

<head>
    <meta charset="UTF-8">

    <meta
        name="viewport"
        content="width=device-width, initial-scale=1.0"
    >

    <title>Mediciones de peso</title>

    <link
        rel="stylesheet"
        href="/css/estilos.css"
    >
</head>

<body>

    <header>

        <div>
            <h1>Medición automática de peso</h1>

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
        <a href="/estadisticas-altura">Estadísticas de altura</a>
        <a href="/mediciones-peso">Pesos</a>
        <a href="/estadisticas-peso">Estadísticas de peso</a>
    </nav>

    <main>

        <section class="tarjetas">

            <article class="tarjeta">
                <h3>Última pieza</h3>

                <p
                    id="ultimoPeso"
                    class="numero"
                >
                    -- g
                </p>
            </article>

            <article class="tarjeta">
                <h3>Peso acumulado</h3>

                <p
                    id="pesoAcumulado"
                    class="numero"
                >
                    -- g
                </p>
            </article>

            <article class="tarjeta">
                <h3>Número de pieza</h3>

                <p
                    id="numeroPieza"
                    class="numero"
                >
                    0
                </p>
            </article>

            <article class="tarjeta">
                <h3>Resultado</h3>

                <p id="ultimoResultado">
                    Sin mediciones
                </p>
            </article>

            <article class="tarjeta">
                <h3>Caja</h3>

                <p id="ultimaCaja">
                    CAJA_01
                </p>
            </article>

            <article class="tarjeta">
                <h3>Dispositivo</h3>

                <p id="ultimoDispositivo">
                    Sin mediciones
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
                            <th>Caja</th>
                            <th>Pieza</th>
                            <th>Peso anterior</th>
                            <th>Peso actual</th>
                            <th>Peso de la pieza</th>
                            <th>Resultado</th>
                            <th>Fecha</th>
                        </tr>
                    </thead>

                    <tbody
                        id="tablaPesos"
                    ></tbody>

                </table>

            </div>

        </section>

    </main>

    <script src="/socket.io/socket.io.js"></script>
    <script src="/js/comun.js"></script>
    <script src="/js/mediciones-peso.js"></script>

</body>
</html>
```

---

## 25. Crear `public/js/mediciones-peso.js`

```javascript
const socket = io();

const tabla =
    document.getElementById(
        "tablaPesos"
    );

function textoResultado(resultado)
{
    if(resultado === "correcto")
    {
        return "Correcto";
    }

    if(resultado === "demasiado_ligero")
    {
        return "Demasiado ligero";
    }

    if(resultado === "demasiado_pesado")
    {
        return "Demasiado pesado";
    }

    return "Sin especificación";
}

function actualizarUltimaMedicion(
    medicion
)
{
    document.getElementById(
        "ultimoPeso"
    ).textContent =
        `${medicion.peso_pieza_g} g`;

    document.getElementById(
        "pesoAcumulado"
    ).textContent =
        `${medicion.peso_actual_g} g`;

    document.getElementById(
        "numeroPieza"
    ).textContent =
        medicion.numero_pieza;

    document.getElementById(
        "ultimoResultado"
    ).textContent =
        textoResultado(
            medicion.resultado
        );

    document.getElementById(
        "ultimaCaja"
    ).textContent =
        medicion.caja;

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
                ${medicion.caja}
            </td>

            <td>
                ${medicion.numero_pieza}
            </td>

            <td>
                ${medicion.peso_anterior_g} g
            </td>

            <td>
                ${medicion.peso_actual_g} g
            </td>

            <td>
                <strong>
                    ${medicion.peso_pieza_g} g
                </strong>
            </td>

            <td>
                ${textoResultado(
                    medicion.resultado
                )}
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
        "/api/mediciones-peso"
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
    "nueva_medicion_peso",
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

# Bloque 6. Crear la página de estadísticas

## 26. Crear `paginas/estadisticas-peso.html`

```html
<!DOCTYPE html>
<html lang="es">

<head>
    <meta charset="UTF-8">

    <meta
        name="viewport"
        content="width=device-width, initial-scale=1.0"
    >

    <title>Estadísticas de peso</title>

    <link
        rel="stylesheet"
        href="/css/estilos.css"
    >
</head>

<body>

    <header>

        <div>
            <h1>Estadísticas de peso</h1>

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
        <a href="/mediciones-altura">Alturas</a>
        <a href="/estadisticas-altura">Estadísticas de altura</a>
        <a href="/mediciones-peso">Pesos</a>
        <a href="/estadisticas-peso">Estadísticas de peso</a>
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
                <h3>Peso promedio</h3>
                <p id="promedio" class="numero">0 g</p>
            </article>

            <article class="tarjeta">
                <h3>Peso mínimo</h3>
                <p id="minimo" class="numero">0 g</p>
            </article>

            <article class="tarjeta">
                <h3>Peso máximo</h3>
                <p id="maximo" class="numero">0 g</p>
            </article>

            <article class="tarjeta">
                <h3>Desviación</h3>
                <p id="desviacion" class="numero">0 g</p>
            </article>

            <article class="tarjeta">
                <h3>Peso acumulado</h3>
                <p id="acumulado" class="numero">0 g</p>
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
                Peso de las últimas piezas
            </h2>

            <div class="contenedor-grafica">
                <canvas id="graficaPesos"></canvas>
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
    <script src="/js/estadisticas-peso.js"></script>

</body>
</html>
```

---

## 27. Crear `public/js/estadisticas-peso.js`

```javascript
const socket = io();

let graficaPesos = null;
let graficaResultados = null;

const selectorProducto =
    document.getElementById(
        "filtroProducto"
    );

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

async function cargarEstadisticas()
{
    const productoId =
        selectorProducto.value;

    let url =
        "/api/estadisticas-peso";

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
        `${numero(resumen.promedio)} g`;

    document.getElementById(
        "minimo"
    ).textContent =
        `${numero(resumen.minimo)} g`;

    document.getElementById(
        "maximo"
    ).textContent =
        `${numero(resumen.maximo)} g`;

    document.getElementById(
        "desviacion"
    ).textContent =
        `${numero(resumen.desviacion)} g`;

    document.getElementById(
        "acumulado"
    ).textContent =
        `${numero(
            resumen.peso_acumulado
        )} g`;

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
                `Pieza ${medicion.numero_pieza}`
        );

    const pesos =
        serie.map(
            medicion =>
                Number(
                    medicion.peso_pieza_g
                )
        );

    if(graficaPesos)
    {
        graficaPesos.destroy();
    }

    graficaPesos =
        new Chart(
            document.getElementById(
                "graficaPesos"
            ),
            {
                type: "line",

                data: {
                    labels:
                        etiquetas,

                    datasets: [
                        {
                            label:
                                "Peso de la pieza en gramos",

                            data:
                                pesos
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
    "nueva_medicion_peso",
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

# Bloque 7. Programar el ESP32

## 28. Funcionamiento del programa

El programa utilizará dos valores principales:

```text
pesoAnterior
pesoActual
```

Cuando una pieza caiga:

1. El peso aumentará.
2. El ESP32 detectará el cambio.
3. Esperará que la lectura sea estable.
4. Restará el peso anterior.
5. Enviará el resultado.
6. Guardará el nuevo peso como referencia.

---

## 29. Configuración que deberá modificarse

Antes de cargar el código, identificar:

* Nombre del WiFi.
* Contraseña del WiFi.
* Dirección IPv4 de la computadora.
* Factor de calibración.
* ID del producto.
* Nombre de la caja.

---

## 30. Código completo del ESP32

```cpp
#include <WiFi.h>
#include <HTTPClient.h>
#include <HX711.h>
#include <math.h>

// ==================================================
// CONFIGURACIÓN DE RED
// ==================================================

const char* NOMBRE_WIFI =
    "NOMBRE_DE_LA_RED";

const char* CONTRASENA_WIFI =
    "CONTRASENA_DE_LA_RED";

const char* URL_SERVIDOR =
    "http://192.168.1.25:3000/api/mediciones-peso";

const char* CLAVE_DISPOSITIVO =
    "clave-banda-industria-40";

// ==================================================
// IDENTIFICACIÓN
// ==================================================

const int PRODUCTO_ID = 1;

const char* NOMBRE_CAJA =
    "CAJA_01";

const char* NOMBRE_DISPOSITIVO =
    "bascula_01";

// ==================================================
// HX711
// ==================================================

const int PIN_DOUT = 16;
const int PIN_SCK = 17;

// Sustituir por el factor obtenido
// durante la calibración.
const float FACTOR_CALIBRACION =
    -421.782013;

HX711 bascula;

// ==================================================
// PARÁMETROS DE DETECCIÓN
// ==================================================

// Cambio mínimo para considerar que cayó
// una nueva pieza.
const float UMBRAL_NUEVA_PIEZA_G =
    10.0;

// Diferencia máxima permitida entre lecturas
// para considerar que el peso es estable.
const float TOLERANCIA_ESTABLE_G =
    2.0;

// Número de lecturas estables consecutivas.
const int LECTURAS_ESTABLES_NECESARIAS =
    4;

// Tiempo máximo para esperar estabilidad.
const unsigned long
    TIEMPO_MAXIMO_ESTABILIDAD_MS =
        10000;

// Cambio negativo utilizado para detectar
// que se retiró la caja o alguna pieza.
const float UMBRAL_RETIRO_G =
    30.0;

// ==================================================
// VARIABLES DEL SISTEMA
// ==================================================

float pesoAnterior = 0.0;

int contadorPiezas = 0;

bool sistemaListo = false;

unsigned long ultimaLecturaSerial = 0;

// ==================================================
// CONEXIÓN WIFI
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
// LEER PESO PROMEDIO
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

    // Eliminar pequeños valores cercanos a cero.
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
// ESPERAR QUE EL PESO SEA ESTABLE
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
            "Esperando estabilidad: "
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
// ENVIAR MEDICIÓN
// ==================================================

bool enviarMedicion(
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
        "\"caja\":\"" +
        String(NOMBRE_CAJA) +
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

    bool enviado =
        codigoHTTP >= 200 &&
        codigoHTTP < 300;

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

    http.end();

    return enviado;
}

// ==================================================
// REALIZAR TARA
// ==================================================

void realizarTara()
{
    Serial.println();
    Serial.println(
        "Coloque la caja vacía."
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
        "El sistema está listo."
    );
}

// ==================================================
// REVISAR COMANDOS SERIALES
// ==================================================

void revisarSerial()
{
    if(!Serial.available())
    {
        return;
    }

    char comando =
        Serial.read();

    if(
        comando == 'T' ||
        comando == 't'
    )
    {
        realizarTara();
    }
}

// ==================================================
// CONFIGURACIÓN
// ==================================================

void setup()
{
    Serial.begin(115200);

    bascula.begin(
        PIN_DOUT,
        PIN_SCK
    );

    bascula.set_scale(
        FACTOR_CALIBRACION
    );

    conectarWiFi();

    Serial.println();
    Serial.println(
        "ESTACIÓN DE PESAJE"
    );

    Serial.println(
        "Coloque la caja vacía."
    );

    Serial.println(
        "La tara comenzará en 5 segundos."
    );

    delay(5000);

    realizarTara();

    Serial.println();
    Serial.println(
        "Escriba T en el monitor serial"
    );

    Serial.println(
        "para realizar una nueva tara."
    );
}

// ==================================================
// PROGRAMA PRINCIPAL
// ==================================================

void loop()
{
    revisarSerial();

    if(!sistemaListo)
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

    // Mostrar el peso cada segundo.
    if(
        millis() -
        ultimaLecturaSerial >=
        1000
    )
    {
        ultimaLecturaSerial =
            millis();

        Serial.print(
            "Peso acumulado: "
        );

        Serial.print(
            pesoActual,
            2
        );

        Serial.println(" g");
    }

    float cambio =
        pesoActual -
        pesoAnterior;

    // Detectar que una pieza cayó.
    if(
        cambio >=
        UMBRAL_NUEVA_PIEZA_G
    )
    {
        Serial.println();
        Serial.println(
            "Se detectó un aumento de peso."
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
                "El cambio final no corresponde a una pieza."
            );

            return;
        }

        int siguientePieza =
            contadorPiezas + 1;

        Serial.println();
        Serial.print(
            "Peso anterior: "
        );

        Serial.print(
            pesoAnterior,
            2
        );

        Serial.println(" g");

        Serial.print(
            "Peso actual: "
        );

        Serial.print(
            pesoEstable,
            2
        );

        Serial.println(" g");

        Serial.print(
            "Peso de la pieza: "
        );

        Serial.print(
            pesoPieza,
            2
        );

        Serial.println(" g");

        bool enviado =
            enviarMedicion(
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
                "Medición registrada correctamente."
            );
        }
        else
        {
            Serial.println(
                "La medición no fue confirmada por el servidor."
            );
        }

        delay(500);
    }

    // Detectar que la caja o alguna pieza
    // fue retirada.
    if(
        cambio <=
        -UMBRAL_RETIRO_G
    )
    {
        sistemaListo = false;

        Serial.println();
        Serial.println(
            "Se detectó una reducción importante de peso."
        );

        Serial.println(
            "La caja o alguna pieza pudo ser retirada."
        );

        Serial.println(
            "Coloque una caja vacía y escriba T."
        );
    }

    delay(100);
}
```

---

# 31. Variables que deben modificarse

## Red WiFi

```cpp
const char* NOMBRE_WIFI =
    "NOMBRE_DE_LA_RED";
```

```cpp
const char* CONTRASENA_WIFI =
    "CONTRASENA_DE_LA_RED";
```

---

## Dirección del servidor

```cpp
const char* URL_SERVIDOR =
    "http://DIRECCION_IP:3000/api/mediciones-peso";
```

Ejemplo:

```cpp
const char* URL_SERVIDOR =
    "http://192.168.1.25:3000/api/mediciones-peso";
```

---

## Factor de calibración

```cpp
const float FACTOR_CALIBRACION =
    -421.782013;
```

Debe sustituirse por el valor obtenido durante la calibración.

---

## Producto

```cpp
const int PRODUCTO_ID = 1;
```

Debe corresponder a un producto registrado en MySQL.

---

## Identificación temporal de la caja

```cpp
const char* NOMBRE_CAJA =
    "CAJA_01";
```

En la siguiente práctica este valor será obtenido mediante RFID.

---

# Bloque 8. Ajustar la detección

## 32. Umbral de nueva pieza

```cpp
const float UMBRAL_NUEVA_PIEZA_G =
    10.0;
```

Este valor evita que pequeñas vibraciones sean interpretadas como una nueva pieza.

Ejemplo:

```text
Piezas de aproximadamente 100 g:
umbral sugerido de 10 g.

Piezas de aproximadamente 20 g:
el umbral deberá reducirse.
```

---

## 33. Tolerancia de estabilidad

```cpp
const float TOLERANCIA_ESTABLE_G =
    2.0;
```

Una medición se considera estable cuando varias lecturas consecutivas cambian menos que esta tolerancia.

Ejemplo:

```text
Lectura 1: 122.8 g
Lectura 2: 122.1 g
Lectura 3: 122.5 g
Lectura 4: 122.3 g
```

Si la tolerancia es de 2 g, estas lecturas pueden considerarse estables.

---

## 34. Detección de retiro

```cpp
const float UMBRAL_RETIRO_G =
    30.0;
```

Una reducción importante puede significar que:

* Se retiró la caja.
* Se retiraron piezas.
* La plataforma fue levantada.
* La celda sufrió un movimiento mecánico.

Cuando esto ocurra, el sistema se detendrá hasta realizar una nueva tara.

---

# Bloque 9. Ejecutar el sistema

## 35. Preparar la estación

1. Colocar la celda de carga.
2. Colocar la plataforma.
3. Colocar la caja vacía.
4. Verificar que nada toque la caja.
5. Comprobar que la caja no reciba apoyo de la banda.
6. Encender el ESP32.
7. Evitar tocar la estructura durante la tara.

---

## 36. Iniciar MySQL

Verificar que MySQL Server esté ejecutándose.

---

## 37. Iniciar Node.js

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

## 38. Abrir la página de monitoreo

```text
http://localhost:3000/mediciones-peso
```

---

## 39. Abrir la página de estadísticas

```text
http://localhost:3000/estadisticas-peso
```

---

## 40. Abrir el monitor serial

Configurar:

```text
115200 baudios
```

El resultado inicial será parecido a:

```text
WiFi conectado
ESTACIÓN DE PESAJE
Coloque la caja vacía
Realizando tara
Tara terminada
El sistema está listo
```

---

# Bloque 10. Pruebas obligatorias

## Prueba 1. Tara de la caja

Colocar la caja vacía y reiniciar el ESP32.

Resultado esperado:

```text
Peso acumulado: 0.00 g
```

Se aceptan pequeñas variaciones cercanas a cero.

---

## Prueba 2. Colocar una pieza manualmente

Colocar una pieza dentro de la caja sin dejarla caer.

Comprobar:

* Peso anterior.
* Peso actual.
* Peso individual.
* Envío al servidor.
* Registro en MySQL.

---

## Prueba 3. Dejar caer una pieza

Dejar que una pieza caiga desde la banda.

Observar las oscilaciones:

```text
120 g
143 g
127 g
131 g
129 g
129 g
```

El sistema deberá esperar hasta obtener lecturas estables.

---

## Prueba 4. Acumular piezas

Colocar al menos cinco piezas.

Ejemplo:

| Pieza | Peso anterior | Peso actual | Peso individual |
| ----- | ------------: | ----------: | --------------: |
| 1     |           0 g |       102 g |           102 g |
| 2     |         102 g |       201 g |            99 g |
| 3     |         201 g |       305 g |           104 g |
| 4     |         305 g |       405 g |           100 g |
| 5     |         405 g |       506 g |           101 g |

---

## Prueba 5. Pieza correcta

Configurar:

```text
Peso mínimo: 95 g
Peso máximo: 105 g
```

Colocar una pieza de aproximadamente 100 g.

Resultado esperado:

```text
correcto
```

---

## Prueba 6. Pieza demasiado ligera

Colocar una pieza cuyo peso sea inferior al mínimo.

Resultado esperado:

```text
demasiado_ligero
```

---

## Prueba 7. Pieza demasiado pesada

Colocar una pieza cuyo peso sea superior al máximo.

Resultado esperado:

```text
demasiado_pesado
```

---

## Prueba 8. Retirar la caja

Retirar la caja con piezas.

El sistema deberá mostrar:

```text
Se detectó una reducción importante de peso.
Coloque una caja vacía y escriba T.
```

---

## Prueba 9. Cambiar la caja

1. Retirar la caja llena.
2. Colocar una caja vacía.
3. Escribir:

```text
T
```

en el monitor serial.

El contador deberá regresar a cero.

---

## Prueba 10. Actualización en tiempo real

Mantener abierta la página:

```text
/mediciones-peso
```

Cada nueva medición deberá aparecer sin recargarla.

---

## Prueba 11. Estadísticas

Registrar al menos diez piezas.

Comprobar:

* Total de piezas.
* Peso promedio.
* Peso mínimo.
* Peso máximo.
* Desviación.
* Piezas correctas.
* Piezas rechazadas.
* Cumplimiento.
* Gráficas.

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

Las mediciones deberán continuar en la base de datos.

---

# 41. Consultas SQL de comprobación

## Consultar todas las mediciones

```sql
SELECT
    *
FROM mediciones_peso
ORDER BY fecha DESC;
```

---

## Consultar mediciones con producto

```sql
SELECT
    mediciones_peso.id,
    productos.nombre AS producto,
    mediciones_peso.caja,
    mediciones_peso.numero_pieza,
    mediciones_peso.peso_pieza_g,
    mediciones_peso.resultado,
    mediciones_peso.fecha

FROM mediciones_peso

INNER JOIN productos
    ON mediciones_peso.producto_id =
       productos.id

ORDER BY
    mediciones_peso.fecha DESC;
```

---

## Promedio, mínimo y máximo

```sql
SELECT
    COUNT(*) AS total,

    ROUND(
        AVG(peso_pieza_g),
        2
    ) AS promedio,

    MIN(peso_pieza_g)
        AS minimo,

    MAX(peso_pieza_g)
        AS maximo,

    ROUND(
        STDDEV_POP(peso_pieza_g),
        2
    ) AS desviacion

FROM mediciones_peso;
```

---

## Contar resultados

```sql
SELECT
    resultado,
    COUNT(*) AS cantidad

FROM mediciones_peso

GROUP BY resultado;
```

---

## Peso acumulado por caja

```sql
SELECT
    caja,
    MAX(peso_actual_g)
        AS peso_acumulado,

    COUNT(*)
        AS total_piezas

FROM mediciones_peso

GROUP BY caja;
```

---

## Estadísticas por producto

```sql
SELECT
    productos.nombre,

    COUNT(*)
        AS total,

    ROUND(
        AVG(
            mediciones_peso.peso_pieza_g
        ),
        2
    ) AS promedio,

    MIN(
        mediciones_peso.peso_pieza_g
    ) AS minimo,

    MAX(
        mediciones_peso.peso_pieza_g
    ) AS maximo

FROM mediciones_peso

INNER JOIN productos
    ON mediciones_peso.producto_id =
       productos.id

GROUP BY
    productos.id,
    productos.nombre;
```

---

# 42. Flujo completo

```text
La caja vacía se coloca sobre la báscula
                 │
                 ▼
Se realiza la tara
                 │
                 ▼
Peso acumulado inicial = 0 g
                 │
                 ▼
Una pieza cae dentro de la caja
                 │
                 ▼
La lectura comienza a oscilar
                 │
                 ▼
El ESP32 espera estabilidad
                 │
                 ▼
Obtiene el nuevo peso acumulado
                 │
                 ▼
Resta el peso acumulado anterior
                 │
                 ▼
Obtiene el peso individual
                 │
                 ▼
Envía la medición mediante HTTP
                 │
                 ▼
Node.js valida y recalcula
                 │
                 ▼
Compara con peso mínimo y máximo
                 │
                 ▼
Guarda en MySQL
                 │
                 ▼
Socket.IO actualiza las páginas
```

---

# 43. Evidencias que deberán entregarse

El reporte deberá incluir:

1. Portada.
2. Objetivo.
3. Diagrama de conexión.
4. Fotografía del montaje mecánico.
5. Capacidad de la celda utilizada.
6. Peso conocido utilizado para calibrar.
7. Factor de calibración obtenido.
8. Tabla de comparación con una báscula.
9. Captura del monitor serial.
10. Captura de `mediciones_peso`.
11. Captura de la página de monitoreo.
12. Captura de la página de estadísticas.
13. Prueba con al menos diez piezas.
14. Cálculo manual de tres diferencias de peso.
15. Explicación de la detección de estabilidad.
16. Análisis de errores.
17. Conclusiones.

---

# 44. Tabla de calibración

Completar:

| Objeto   | Peso de referencia | Peso medido | Error absoluto |
| -------- | -----------------: | ----------: | -------------: |
| Objeto 1 |                    |             |                |
| Objeto 2 |                    |             |                |
| Objeto 3 |                    |             |                |
| Objeto 4 |                    |             |                |
| Objeto 5 |                    |             |                |

Calcular:

```text
Error absoluto =
|Peso medido − Peso de referencia|
```

---

# 45. Investigar

Investigar y explicar con palabras propias:

* ¿Qué es una celda de carga?
* ¿Qué son los extensómetros?
* ¿Qué función realiza un puente de Wheatstone?
* ¿Por qué la señal de una celda de carga necesita amplificación?
* ¿Qué función realiza el HX711?
* ¿Qué diferencia existe entre calibrar y realizar una tara?
* ¿Qué es el factor de calibración?
* ¿Por qué puede ser negativo?
* ¿Por qué no se toma la primera lectura después del impacto?
* ¿Qué significa que una lectura sea estable?
* ¿Qué diferencia existe entre peso acumulado y peso individual?
* ¿Por qué se utiliza una resta?
* ¿Qué función tiene el umbral de nueva pieza?
* ¿Qué función tiene la tolerancia de estabilidad?
* ¿Qué ocurre si caen dos piezas al mismo tiempo?
* ¿Por qué debe evitarse que la caja toque la estructura?
* ¿Qué representa la desviación estándar?
* ¿Qué diferencia existe entre promedio, mínimo, máximo y desviación?
* ¿Qué función realiza HTTP POST?
* ¿Por qué el servidor vuelve a calcular el peso de la pieza?
* ¿Qué información se almacena en MySQL?
* ¿Cómo podría detectarse que la caja está llena?

---

# 46. Consideraciones y limitaciones

El sistema puede presentar errores cuando:

* La celda está mal instalada.
* La plataforma toca la estructura.
* La caja se apoya en otro elemento.
* Las piezas golpean con demasiada fuerza.
* La banda transmite vibraciones a la báscula.
* La masa de calibración no es confiable.
* Dos piezas caen casi al mismo tiempo.
* Una persona toca la caja.
* Se retiran piezas durante la medición.
* La celda se sobrecarga.
* La temperatura cambia considerablemente.
* La fuente de alimentación es inestable.

Para esta práctica se recomienda:

* Permitir que caiga una pieza a la vez.
* Usar piezas con pesos claramente mayores al ruido.
* Mantener fija la caja.
* Reducir la altura de caída.
* Separar mecánicamente la banda y la báscula.
* Realizar una nueva tara al cambiar la caja.
* No exceder la capacidad de la celda.

---

# 47. Preparación para la práctica de RFID

En esta práctica la caja se identifica manualmente mediante:

```cpp
const char* NOMBRE_CAJA =
    "CAJA_01";
```

En la siguiente práctica se utilizará el lector RFID RC522.

Cada caja tendrá una tarjeta o llavero asociado.

El flujo será:

```text
El operador acerca el llavero RFID
               │
               ▼
El RC522 obtiene el UID
               │
               ▼
Node.js busca la caja en MySQL
               │
               ▼
El sistema identifica la caja
               │
               ▼
Las nuevas mediciones quedan
asociadas automáticamente
```

De esta manera ya no será necesario escribir manualmente:

```text
CAJA_01
CAJA_02
CAJA_03
```

El UID del RFID permitirá identificar la caja receptora de manera automática.
