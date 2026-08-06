# Práctica final: Sistema integral de inspección de piezas

## Integración de altura, color, peso, RFID, dos ESP32, Node.js y MySQL

---

# 1. Introducción

En las prácticas anteriores se desarrollaron por separado los siguientes sistemas:

* Detección de piezas mediante un sensor infrarrojo.
* Medición de altura mediante el HC-SR04.
* Inspección de color mediante el TCS34725.
* Medición de peso mediante una celda de carga y el HX711.
* Identificación de cajas mediante el lector RFID RC522.
* Registro de usuarios, productos, inventario y movimientos.
* Almacenamiento de mediciones en MySQL.
* Monitoreo y estadísticas mediante páginas web.

En esta práctica se integrarán todos los sensores en un solo proceso.

Debido a que los sensores están ubicados en dos zonas físicas diferentes, se utilizarán dos ESP32:

```text
ESP32 de inspección de la banda
├── Sensor infrarrojo
├── HC-SR04
└── TCS34725
```

```text
ESP32 de recepción
├── Celda de carga con HX711
└── Lector RFID RC522
```

El primer ESP32 creará una inspección con la altura y el color de la pieza.

Cuando esa pieza caiga dentro de la caja, el segundo ESP32 medirá su peso y completará la inspección pendiente.

---

# 2. Objetivo general

Integrar en un solo sistema las mediciones de altura, color y peso de cada pieza, así como la identificación RFID de la caja donde se deposita, almacenando todos los datos en una inspección unificada dentro de MySQL.

---

# 3. Objetivos específicos

Al finalizar la práctica, el estudiante será capaz de:

* Integrar varios sensores en un proceso industrial.
* Dividir un sistema entre dos microcontroladores.
* Detectar y medir una pieza en diferentes estaciones.
* Crear una inspección pendiente desde el ESP32 de la banda.
* Completar la inspección desde el ESP32 de recepción.
* Relacionar las mediciones mediante una cola FIFO.
* Identificar la caja receptora mediante RFID.
* Calcular el resultado de altura.
* Calcular el resultado de color.
* Calcular el resultado de peso.
* Determinar un resultado general.
* Almacenar una inspección completa en MySQL.
* Mostrar el proceso en tiempo real mediante Socket.IO.
* Obtener estadísticas generales y por producto.
* Detectar errores de sincronización entre estaciones.

---

# 4. Arquitectura del sistema

```text
                     ESTACIÓN 1
                Inspección en la banda

                Sensor infrarrojo
                       │
                       ▼
          ┌────────────┴────────────┐
          ▼                         ▼
      HC-SR04                  TCS34725
    mide altura               mide color
          │                         │
          └────────────┬────────────┘
                       ▼
                  ESP32 BANDA
                       │
                       │ HTTP POST
                       ▼
                  Servidor Node.js
                       │
                       ▼
        Crea una inspección pendiente
                       │
                       ▼
                     MySQL


                     ESTACIÓN 2
                 Recepción de piezas

                 RFID identifica caja
                       │
                       ▼
             La pieza cae en la caja
                       │
                       ▼
             HX711 obtiene el peso
                       │
                       ▼
                ESP32 BÁSCULA
                       │
                       │ Consulta la inspección
                       │ pendiente más antigua
                       ▼
                  Servidor Node.js
                       │
                       ▼
           Completa la inspección en MySQL
```

---

# 5. Relación entre las estaciones

El ESP32 de la banda no conoce directamente el peso de la pieza.

El ESP32 de la báscula no mide directamente su altura ni su color.

El servidor relacionará ambos eventos por orden de llegada.

```text
Primera pieza medida en la banda
        │
        ▼
Primera inspección pendiente
        │
        ▼
Primera pieza que cae en la caja
        │
        ▼
Completa la primera inspección
```

Este sistema utiliza una cola FIFO:

```text
FIFO = First In, First Out

Primero en entrar
Primero en salir
```

Ejemplo:

```text
Inspección 101 → Pieza 1
Inspección 102 → Pieza 2
Inspección 103 → Pieza 3
```

La estación de peso siempre tomará la inspección pendiente más antigua.

---

# 6. Condiciones necesarias

Para que la relación FIFO sea correcta:

* Las piezas deberán pasar de una en una.
* Las piezas no deberán adelantarse entre sí.
* No deberá retirarse una pieza entre las dos estaciones.
* No deberán caer dos piezas al mismo tiempo.
* Deberá existir una separación suficiente entre piezas.
* Solo deberá existir una estación de recepción.
* La banda deberá conservar el orden de las piezas.

Si una pieza se retira después de medir altura y color, la siguiente pieza podría relacionarse con la inspección equivocada.

---

# 7. Resultado esperado

Cada inspección contendrá:

```text
ID de inspección
Producto
Altura
Resultado de altura
Valores RGB
Color detectado
Resultado de color
Peso individual
Resultado de peso
Caja receptora
UID de la caja
Número de pieza dentro de la caja
Resultado general
Fecha de inicio
Fecha de finalización
```

Ejemplo:

```text
Inspección: 125
Producto: Pieza roja
Altura: 10.20 cm
Resultado de altura: correcto
Color detectado: rojo
Resultado de color: correcto
Peso: 101.40 g
Resultado de peso: correcto
Caja: CAJA_03
UID: A4D38B21
Resultado general: aceptada
```

---

# 8. Material necesario

## Estación de inspección

* Un ESP32.
* Un sensor infrarrojo.
* Un sensor ultrasónico HC-SR04.
* Un sensor de color TCS34725.
* Divisor de voltaje para `ECHO`.
* Soporte para los sensores.
* Cubierta para controlar la iluminación.
* Cables Dupont.
* Protoboard.

## Estación de recepción

* Un ESP32.
* Una celda de carga.
* Un módulo HX711.
* Un lector RFID RC522.
* Una tarjeta o llavero RFID por caja.
* Caja receptora.
* Plataforma de pesaje.
* Cables Dupont.
* Protoboard.

## Sistema informático

* Computadora conectada a la misma red.
* Node.js.
* MySQL.
* Arduino IDE.
* Visual Studio Code.
* Navegador web.
* Proyecto de las prácticas anteriores.

---

# 9. Distribución de sensores

## ESP32 de la banda

| Dispositivo       | Señal |               ESP32 |
| ----------------- | ----- | ------------------: |
| Sensor infrarrojo | OUT   |             GPIO 19 |
| HC-SR04           | TRIG  |              GPIO 5 |
| HC-SR04           | ECHO  | GPIO 18 con divisor |
| TCS34725          | SDA   |             GPIO 21 |
| TCS34725          | SCL   |             GPIO 22 |
| TCS34725          | VCC   |               3.3 V |
| TCS34725          | GND   |                 GND |

## ESP32 de recepción

### HX711

| HX711 |   ESP32 |
| ----- | ------: |
| VCC   |   3.3 V |
| GND   |     GND |
| DOUT  | GPIO 16 |
| SCK   | GPIO 17 |

### RC522

| RC522  |        ESP32 |
| ------ | -----------: |
| 3.3 V  |        3.3 V |
| GND    |          GND |
| SDA/SS |       GPIO 5 |
| SCK    |      GPIO 18 |
| MOSI   |      GPIO 23 |
| MISO   |      GPIO 19 |
| RST    |      GPIO 22 |
| IRQ    | Sin conexión |

---

# 10. Flujo completo de una pieza

```text
1. La pieza llega a la zona de inspección.

2. El infrarrojo detecta la pieza.

3. El ESP32 de banda espera que quede centrada.

4. El HC-SR04 mide la distancia.

5. Se calcula la altura.

6. El TCS34725 mide el color.

7. El ESP32 envía altura y color.

8. Node.js crea una inspección:
   estado = esperando_peso.

9. La pieza continúa avanzando.

10. La pieza cae dentro de la caja.

11. El HX711 detecta un aumento de peso.

12. El ESP32 espera que la lectura se estabilice.

13. Consulta la inspección pendiente más antigua.

14. Calcula el peso por diferencia.

15. Envía el peso y el UID de la caja.

16. Node.js completa la inspección.

17. Se calcula el resultado general.

18. Socket.IO actualiza las páginas.
```

---

# Bloque 1. Verificar los productos

## 11. Entrar a MySQL

```bash
mysql -u root -p
```

Seleccionar la base:

```sql
USE industria40_web;
```

---

# 12. Verificar las especificaciones

Las prácticas anteriores agregaron las siguientes columnas a `productos`:

```text
altura_min_cm
altura_max_cm

peso_min_g
peso_max_g

color_esperado
color_ref_r_pct
color_ref_g_pct
color_ref_b_pct
tolerancia_color
```

Comprobar:

```sql
SELECT
    id,
    nombre,

    altura_min_cm,
    altura_max_cm,

    peso_min_g,
    peso_max_g,

    color_esperado,
    color_ref_r_pct,
    color_ref_g_pct,
    color_ref_b_pct,
    tolerancia_color

FROM productos;
```

---

# 13. Configurar un producto de prueba

Ejemplo:

```sql
UPDATE productos
SET
    altura_min_cm = 9.50,
    altura_max_cm = 10.50,

    peso_min_g = 95.00,
    peso_max_g = 105.00,

    color_esperado = 'rojo',
    color_ref_r_pct = 48.50,
    color_ref_g_pct = 31.20,
    color_ref_b_pct = 20.30,
    tolerancia_color = 10.00

WHERE id = 1;
```

Los valores deben corresponder a las calibraciones realizadas por cada equipo.

---

# Bloque 2. Crear la tabla de inspecciones

## 14. Crear la tabla principal

Ejecutar:

```sql
CREATE TABLE inspecciones (
    id INT AUTO_INCREMENT PRIMARY KEY,

    producto_id INT NOT NULL,

    caja_id INT NULL,

    uid_rfid VARCHAR(32) NULL,

    numero_pieza_caja INT NULL,

    dispositivo_banda VARCHAR(50) NOT NULL,

    dispositivo_bascula VARCHAR(50) NULL,

    distancia_banda_cm DECIMAL(7,2) NOT NULL,

    distancia_objeto_cm DECIMAL(7,2) NOT NULL,

    altura_cm DECIMAL(7,2) NOT NULL,

    resultado_altura VARCHAR(30) NOT NULL,

    rojo_raw INT UNSIGNED NOT NULL,

    verde_raw INT UNSIGNED NOT NULL,

    azul_raw INT UNSIGNED NOT NULL,

    claro_raw INT UNSIGNED NOT NULL,

    rojo_pct DECIMAL(6,2) NOT NULL,

    verde_pct DECIMAL(6,2) NOT NULL,

    azul_pct DECIMAL(6,2) NOT NULL,

    color_detectado VARCHAR(30) NOT NULL,

    diferencia_color DECIMAL(8,2) NULL,

    resultado_color VARCHAR(30) NOT NULL,

    peso_anterior_g DECIMAL(10,2) NULL,

    peso_actual_g DECIMAL(10,2) NULL,

    peso_pieza_g DECIMAL(8,2) NULL,

    resultado_peso VARCHAR(30) NOT NULL
        DEFAULT 'pendiente',

    resultado_general VARCHAR(40) NOT NULL
        DEFAULT 'pendiente',

    estado VARCHAR(30) NOT NULL
        DEFAULT 'esperando_peso',

    caja_llena BOOLEAN NOT NULL
        DEFAULT FALSE,

    fecha_inicio TIMESTAMP
        DEFAULT CURRENT_TIMESTAMP,

    fecha_fin TIMESTAMP NULL
        DEFAULT NULL,

    FOREIGN KEY (producto_id)
        REFERENCES productos(id),

    FOREIGN KEY (caja_id)
        REFERENCES cajas(id)
        ON DELETE SET NULL
);
```

---

# 15. Crear índices

```sql
CREATE INDEX idx_inspecciones_estado
ON inspecciones(estado);
```

```sql
CREATE INDEX idx_inspecciones_fecha
ON inspecciones(fecha_inicio);
```

```sql
CREATE INDEX idx_inspecciones_producto
ON inspecciones(producto_id);
```

Los índices ayudan al servidor a localizar con mayor rapidez:

* Inspecciones pendientes.
* Inspecciones recientes.
* Inspecciones por producto.

---

# 16. Verificar

```sql
DESCRIBE inspecciones;
```

Salir:

```sql
EXIT;
```

---

# Bloque 3. Funciones auxiliares del servidor

Las siguientes funciones se agregarán a `server.js`.

Si alguna ya existe por las prácticas anteriores, no deberá duplicarse.

## 17. Redondear valores

```javascript
function redondear2(valor)
{
    return Number(
        Number(valor).toFixed(2)
    );
}
```

---

# 18. Normalizar un UID

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

---

# 19. Clasificar un color básico

```javascript
function clasificarColorBasico(
    rojo,
    verde,
    azul
)
{
    const maximo =
        Math.max(
            rojo,
            verde,
            azul
        );

    const minimo =
        Math.min(
            rojo,
            verde,
            azul
        );

    if(maximo - minimo <= 6)
    {
        return "neutro";
    }

    if(
        rojo >= 40 &&
        verde >= 34 &&
        azul <= 25
    )
    {
        return "amarillo";
    }

    if(
        rojo >= verde + 8 &&
        rojo >= azul + 8
    )
    {
        return "rojo";
    }

    if(
        verde >= rojo + 7 &&
        verde >= azul + 7
    )
    {
        return "verde";
    }

    if(
        azul >= rojo + 7 &&
        azul >= verde + 7
    )
    {
        return "azul";
    }

    return "indeterminado";
}
```

---

# 20. Calcular la diferencia de color

```javascript
function calcularDiferenciaColor(
    rojo,
    verde,
    azul,
    rojoReferencia,
    verdeReferencia,
    azulReferencia
)
{
    return Math.sqrt(
        Math.pow(
            rojo -
            rojoReferencia,
            2
        ) +
        Math.pow(
            verde -
            verdeReferencia,
            2
        ) +
        Math.pow(
            azul -
            azulReferencia,
            2
        )
    );
}
```

---

# 21. Evaluar el resultado general

```javascript
function calcularResultadoGeneral(
    resultadoAltura,
    resultadoColor,
    resultadoPeso
)
{
    const sinEspecificacion =
        resultadoAltura ===
            "sin_especificacion" ||

        resultadoColor ===
            "sin_especificacion" ||

        resultadoPeso ===
            "sin_especificacion";

    if(sinEspecificacion)
    {
        return "sin_especificacion";
    }

    const fallaAltura =
        resultadoAltura !== "correcto";

    const fallaColor =
        resultadoColor !== "correcto";

    const fallaPeso =
        resultadoPeso !== "correcto";

    const totalFallas =
        Number(fallaAltura) +
        Number(fallaColor) +
        Number(fallaPeso);

    if(totalFallas === 0)
    {
        return "aceptada";
    }

    if(totalFallas > 1)
    {
        return "rechazada_varias_causas";
    }

    if(fallaAltura)
    {
        return "rechazada_altura";
    }

    if(fallaColor)
    {
        return "rechazada_color";
    }

    return "rechazada_peso";
}
```

---

# Bloque 4. Rutas de las páginas

## 22. Agregar las rutas

Agregar en `server.js`:

```javascript
app.get(
    "/inspecciones",
    requiereSesionPagina,
    (req, res) =>
    {
        enviarPagina(
            res,
            "inspecciones.html"
        );
    }
);
```

```javascript
app.get(
    "/estadisticas-inspecciones",
    requiereSesionPagina,
    (req, res) =>
    {
        enviarPagina(
            res,
            "estadisticas-inspecciones.html"
        );
    }
);
```

---

# Bloque 5. Crear una inspección desde la banda

## 23. API para iniciar la inspección

Agregar en `server.js`:

```javascript
// ==================================================
// API: CREAR INSPECCIÓN DE ALTURA Y COLOR
// ==================================================

app.post(
    "/api/inspecciones/iniciar",
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

            const rojoRaw =
                Number(
                    req.body.rojo_raw
                );

            const verdeRaw =
                Number(
                    req.body.verde_raw
                );

            const azulRaw =
                Number(
                    req.body.azul_raw
                );

            const claroRaw =
                Number(
                    req.body.claro_raw
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
                        "Dispositivo inválido"
                });
            }

            if(
                !Number.isFinite(distanciaBanda) ||
                !Number.isFinite(distanciaObjeto) ||
                distanciaBanda <= 0 ||
                distanciaObjeto <= 0 ||
                distanciaObjeto >= distanciaBanda
            )
            {
                return res.status(400).json({
                    correcto: false,
                    mensaje:
                        "Distancias inválidas"
                });
            }

            const canales = [
                rojoRaw,
                verdeRaw,
                azulRaw,
                claroRaw
            ];

            const canalesValidos =
                canales.every(
                    valor =>
                        Number.isInteger(valor) &&
                        valor >= 0 &&
                        valor <= 65535
                );

            if(!canalesValidos)
            {
                return res.status(400).json({
                    correcto: false,
                    mensaje:
                        "Canales de color inválidos"
                });
            }

            const sumaRgb =
                rojoRaw +
                verdeRaw +
                azulRaw;

            if(sumaRgb <= 0)
            {
                return res.status(400).json({
                    correcto: false,
                    mensaje:
                        "No existe información de color"
                });
            }

            const [productos] =
                await conexion.execute(
                    `SELECT
                        id,
                        nombre,

                        altura_min_cm,
                        altura_max_cm,

                        color_esperado,
                        color_ref_r_pct,
                        color_ref_g_pct,
                        color_ref_b_pct,
                        tolerancia_color

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

            // ======================================
            // ALTURA
            // ======================================

            const altura =
                redondear2(
                    distanciaBanda -
                    distanciaObjeto
                );

            let resultadoAltura =
                "sin_especificacion";

            if(
                producto.altura_min_cm !== null &&
                producto.altura_max_cm !== null
            )
            {
                if(
                    altura <
                    Number(
                        producto.altura_min_cm
                    )
                )
                {
                    resultadoAltura =
                        "demasiado_bajo";
                }
                else if(
                    altura >
                    Number(
                        producto.altura_max_cm
                    )
                )
                {
                    resultadoAltura =
                        "demasiado_alto";
                }
                else
                {
                    resultadoAltura =
                        "correcto";
                }
            }

            // ======================================
            // COLOR
            // ======================================

            const rojoPct =
                redondear2(
                    rojoRaw *
                    100 /
                    sumaRgb
                );

            const verdePct =
                redondear2(
                    verdeRaw *
                    100 /
                    sumaRgb
                );

            const azulPct =
                redondear2(
                    azulRaw *
                    100 /
                    sumaRgb
                );

            const colorDetectado =
                clasificarColorBasico(
                    rojoPct,
                    verdePct,
                    azulPct
                );

            let diferenciaColor = null;

            let resultadoColor =
                "sin_especificacion";

            const tieneColorReferencia =
                producto.color_ref_r_pct !== null &&
                producto.color_ref_g_pct !== null &&
                producto.color_ref_b_pct !== null &&
                producto.tolerancia_color !== null;

            if(tieneColorReferencia)
            {
                diferenciaColor =
                    redondear2(
                        calcularDiferenciaColor(
                            rojoPct,
                            verdePct,
                            azulPct,

                            Number(
                                producto.color_ref_r_pct
                            ),

                            Number(
                                producto.color_ref_g_pct
                            ),

                            Number(
                                producto.color_ref_b_pct
                            )
                        )
                    );

                if(
                    diferenciaColor <=
                    Number(
                        producto.tolerancia_color
                    )
                )
                {
                    resultadoColor =
                        "correcto";
                }
                else
                {
                    resultadoColor =
                        "color_incorrecto";
                }
            }

            // ======================================
            // GUARDAR INSPECCIÓN PENDIENTE
            // ======================================

            const [insercion] =
                await conexion.execute(
                    `INSERT INTO inspecciones
                    (
                        producto_id,
                        dispositivo_banda,

                        distancia_banda_cm,
                        distancia_objeto_cm,
                        altura_cm,
                        resultado_altura,

                        rojo_raw,
                        verde_raw,
                        azul_raw,
                        claro_raw,

                        rojo_pct,
                        verde_pct,
                        azul_pct,

                        color_detectado,
                        diferencia_color,
                        resultado_color,

                        resultado_peso,
                        resultado_general,
                        estado
                    )
                    VALUES
                    (
                        ?, ?,
                        ?, ?, ?, ?,
                        ?, ?, ?, ?,
                        ?, ?, ?,
                        ?, ?, ?,
                        'pendiente',
                        'pendiente',
                        'esperando_peso'
                    )`,
                    [
                        productoId,
                        dispositivo,

                        distanciaBanda,
                        distanciaObjeto,
                        altura,
                        resultadoAltura,

                        rojoRaw,
                        verdeRaw,
                        azulRaw,
                        claroRaw,

                        rojoPct,
                        verdePct,
                        azulPct,

                        colorDetectado,
                        diferenciaColor,
                        resultadoColor
                    ]
                );

            const inspeccion = {
                id:
                    insercion.insertId,

                producto_id:
                    productoId,

                producto:
                    producto.nombre,

                altura_cm:
                    altura,

                resultado_altura:
                    resultadoAltura,

                rojo_pct:
                    rojoPct,

                verde_pct:
                    verdePct,

                azul_pct:
                    azulPct,

                color_detectado:
                    colorDetectado,

                diferencia_color:
                    diferenciaColor,

                resultado_color:
                    resultadoColor,

                estado:
                    "esperando_peso",

                resultado_general:
                    "pendiente",

                fecha_inicio:
                    new Date().toISOString()
            };

            io.emit(
                "inspeccion_iniciada",
                inspeccion
            );

            res.json({
                correcto: true,

                mensaje:
                    "Inspección creada y esperando peso",

                inspeccion:
                    inspeccion
            });
        }
        catch(error)
        {
            console.log(error);

            res.status(500).json({
                correcto: false,
                mensaje:
                    "No fue posible iniciar la inspección"
            });
        }
    }
);
```

---

# Bloque 6. Consultar la inspección pendiente

## 24. API para el ESP32 de la báscula

Agregar:

```javascript
// ==================================================
// API: OBTENER INSPECCIÓN PENDIENTE
// ==================================================

app.get(
    "/api/inspecciones/pendiente",
    requiereClaveDispositivo,
    async (req, res) =>
    {
        try
        {
            const [inspecciones] =
                await conexion.execute(
                    `SELECT
                        inspecciones.id

                    FROM inspecciones

                    WHERE
                        inspecciones.estado =
                        'esperando_peso'

                    ORDER BY
                        inspecciones.id ASC

                    LIMIT 1`
                );

            if(inspecciones.length === 0)
            {
                return res.status(404)
                    .type("text/plain")
                    .send("SIN_PENDIENTES");
            }

            res.type("text/plain").send(
                `OK|${inspecciones[0].id}`
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

La respuesta será:

```text
OK|125
```

Donde `125` es el ID de la inspección que debe completarse.

---

# Bloque 7. Completar la inspección con peso y RFID

## 25. API para completar una inspección

Agregar:

```javascript
// ==================================================
// API: COMPLETAR INSPECCIÓN CON PESO Y RFID
// ==================================================

app.post(
    "/api/inspecciones/:id/completar-peso",
    requiereClaveDispositivo,
    async (req, res) =>
    {
        const inspeccionId =
            Number(
                req.params.id
            );

        const uid =
            normalizarUid(
                req.body.uid_rfid
            );

        const numeroPieza =
            Number(
                req.body.numero_pieza_caja
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
            !Number.isInteger(inspeccionId) ||
            inspeccionId <= 0
        )
        {
            return res.status(400).json({
                correcto: false,
                mensaje:
                    "Inspección inválida"
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

        try
        {
            await conexion.beginTransaction();

            const [inspecciones] =
                await conexion.execute(
                    `SELECT
                        inspecciones.*,

                        productos.nombre
                            AS producto,

                        productos.peso_min_g,
                        productos.peso_max_g

                    FROM inspecciones

                    INNER JOIN productos
                        ON inspecciones.producto_id =
                           productos.id

                    WHERE inspecciones.id = ?

                    FOR UPDATE`,
                    [inspeccionId]
                );

            if(inspecciones.length === 0)
            {
                await conexion.rollback();

                return res.status(404).json({
                    correcto: false,
                    mensaje:
                        "La inspección no existe"
                });
            }

            const inspeccion =
                inspecciones[0];

            // Permite que el ESP32 repita la solicitud
            // si la respuesta anterior no llegó.
            if(inspeccion.estado === "completa")
            {
                await conexion.commit();

                return res.json({
                    correcto: true,
                    ya_completa: true,
                    mensaje:
                        "La inspección ya estaba completa",
                    inspeccion_id:
                        inspeccionId
                });
            }

            if(
                inspeccion.estado !==
                "esperando_peso"
            )
            {
                await conexion.rollback();

                return res.status(409).json({
                    correcto: false,
                    mensaje:
                        "La inspección no puede completarse"
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
                await conexion.rollback();

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
                await conexion.rollback();

                return res.status(403).json({
                    correcto: false,
                    mensaje:
                        "La caja está inactiva"
                });
            }

            const pesoPieza =
                redondear2(
                    pesoActual -
                    pesoAnterior
                );

            if(pesoPieza <= 0)
            {
                await conexion.rollback();

                return res.status(400).json({
                    correcto: false,
                    mensaje:
                        "El peso debe ser mayor que cero"
                });
            }

            let resultadoPeso =
                "sin_especificacion";

            if(
                inspeccion.peso_min_g !== null &&
                inspeccion.peso_max_g !== null
            )
            {
                if(
                    pesoPieza <
                    Number(
                        inspeccion.peso_min_g
                    )
                )
                {
                    resultadoPeso =
                        "demasiado_ligero";
                }
                else if(
                    pesoPieza >
                    Number(
                        inspeccion.peso_max_g
                    )
                )
                {
                    resultadoPeso =
                        "demasiado_pesado";
                }
                else
                {
                    resultadoPeso =
                        "correcto";
                }
            }

            const resultadoGeneral =
                calcularResultadoGeneral(
                    inspeccion.resultado_altura,
                    inspeccion.resultado_color,
                    resultadoPeso
                );

            const capacidad =
                caja.capacidad_max_g === null
                ? 0
                : Number(
                    caja.capacidad_max_g
                );

            const cajaLlena =
                capacidad > 0 &&
                pesoActual >= capacidad;

            await conexion.execute(
                `UPDATE inspecciones

                SET
                    caja_id = ?,
                    uid_rfid = ?,
                    numero_pieza_caja = ?,

                    dispositivo_bascula = ?,

                    peso_anterior_g = ?,
                    peso_actual_g = ?,
                    peso_pieza_g = ?,

                    resultado_peso = ?,
                    resultado_general = ?,

                    estado = 'completa',
                    caja_llena = ?,
                    fecha_fin = CURRENT_TIMESTAMP

                WHERE id = ?`,
                [
                    caja.id,
                    uid,
                    numeroPieza,

                    dispositivo,

                    pesoAnterior,
                    pesoActual,
                    pesoPieza,

                    resultadoPeso,
                    resultadoGeneral,

                    cajaLlena,
                    inspeccionId
                ]
            );

            await conexion.commit();

            const inspeccionCompleta = {
                id:
                    inspeccionId,

                producto:
                    inspeccion.producto,

                caja:
                    caja.nombre,

                uid_rfid:
                    uid,

                numero_pieza_caja:
                    numeroPieza,

                altura_cm:
                    Number(
                        inspeccion.altura_cm
                    ),

                resultado_altura:
                    inspeccion.resultado_altura,

                color_detectado:
                    inspeccion.color_detectado,

                resultado_color:
                    inspeccion.resultado_color,

                peso_pieza_g:
                    pesoPieza,

                peso_actual_g:
                    pesoActual,

                resultado_peso:
                    resultadoPeso,

                resultado_general:
                    resultadoGeneral,

                caja_llena:
                    cajaLlena,

                estado:
                    "completa",

                fecha_fin:
                    new Date().toISOString()
            };

            io.emit(
                "inspeccion_completada",
                inspeccionCompleta
            );

            res.json({
                correcto: true,

                mensaje:
                    "Inspección completada",

                inspeccion:
                    inspeccionCompleta
            });
        }
        catch(error)
        {
            try
            {
                await conexion.rollback();
            }
            catch(errorRollback)
            {
                console.log(
                    errorRollback
                );
            }

            console.log(error);

            res.status(500).json({
                correcto: false,
                mensaje:
                    "No fue posible completar la inspección"
            });
        }
    }
);
```

---

# Bloque 8. Consultar inspecciones

## 26. API de historial

Agregar:

```javascript
// ==================================================
// API: CONSULTAR INSPECCIONES
// ==================================================

app.get(
    "/api/inspecciones",
    requiereSesionAPI,
    async (req, res) =>
    {
        try
        {
            const [inspecciones] =
                await conexion.execute(
                    `SELECT
                        inspecciones.id,

                        productos.nombre AS producto,

                        cajas.nombre AS caja,

                        inspecciones.uid_rfid,
                        inspecciones.numero_pieza_caja,

                        inspecciones.altura_cm,
                        inspecciones.resultado_altura,

                        inspecciones.rojo_pct,
                        inspecciones.verde_pct,
                        inspecciones.azul_pct,

                        inspecciones.color_detectado,
                        inspecciones.diferencia_color,
                        inspecciones.resultado_color,

                        inspecciones.peso_pieza_g,
                        inspecciones.peso_actual_g,
                        inspecciones.resultado_peso,

                        inspecciones.resultado_general,
                        inspecciones.estado,
                        inspecciones.caja_llena,

                        inspecciones.fecha_inicio,
                        inspecciones.fecha_fin

                    FROM inspecciones

                    INNER JOIN productos
                        ON inspecciones.producto_id =
                           productos.id

                    LEFT JOIN cajas
                        ON inspecciones.caja_id =
                           cajas.id

                    ORDER BY
                        inspecciones.id DESC

                    LIMIT 100`
                );

            res.json(inspecciones);
        }
        catch(error)
        {
            console.log(error);

            res.status(500).json({
                mensaje:
                    "No fue posible consultar las inspecciones"
            });
        }
    }
);
```

---

# Bloque 9. Estadísticas integrales

## 27. API de estadísticas

Agregar:

```javascript
// ==================================================
// API: ESTADÍSTICAS DE INSPECCIONES
// ==================================================

app.get(
    "/api/estadisticas-inspecciones",
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

                        SUM(
                            estado =
                            'esperando_peso'
                        ) AS pendientes,

                        SUM(
                            estado =
                            'completa'
                        ) AS completas,

                        SUM(
                            resultado_general =
                            'aceptada'
                        ) AS aceptadas,

                        SUM(
                            resultado_general LIKE
                            'rechazada%'
                        ) AS rechazadas,

                        SUM(
                            resultado_general =
                            'sin_especificacion'
                        ) AS sin_especificacion,

                        ROUND(
                            AVG(altura_cm),
                            2
                        ) AS altura_promedio,

                        ROUND(
                            AVG(peso_pieza_g),
                            2
                        ) AS peso_promedio,

                        ROUND(
                            AVG(diferencia_color),
                            2
                        ) AS diferencia_color_promedio,

                        SUM(
                            resultado_altura !=
                            'correcto' AND
                            resultado_altura !=
                            'sin_especificacion'
                        ) AS fallas_altura,

                        SUM(
                            resultado_color =
                            'color_incorrecto'
                        ) AS fallas_color,

                        SUM(
                            resultado_peso IN
                            (
                                'demasiado_ligero',
                                'demasiado_pesado'
                            )
                        ) AS fallas_peso

                    FROM inspecciones

                    ${filtro}`,
                    parametros
                );

            const filtroSerie =
                productoId
                ? `WHERE
                    inspecciones.producto_id = ?`
                : "";

            const [serie] =
                await conexion.execute(
                    `SELECT
                        inspecciones.id,
                        inspecciones.altura_cm,
                        inspecciones.peso_pieza_g,
                        inspecciones.diferencia_color,
                        inspecciones.resultado_general,
                        inspecciones.estado,
                        inspecciones.fecha_inicio,

                        productos.nombre AS producto

                    FROM inspecciones

                    INNER JOIN productos
                        ON inspecciones.producto_id =
                           productos.id

                    ${filtroSerie}

                    ORDER BY
                        inspecciones.id DESC

                    LIMIT 30`,
                    parametros
                );

            const [distribucion] =
                await conexion.execute(
                    `SELECT
                        resultado_general,
                        COUNT(*) AS cantidad

                    FROM inspecciones

                    ${filtro}

                    GROUP BY
                        resultado_general

                    ORDER BY
                        cantidad DESC`,
                    parametros
                );

            res.json({
                resumen:
                    resumen[0],

                serie:
                    serie,

                distribucion:
                    distribucion
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

# Bloque 10. Página de monitoreo integral

## 28. Crear `paginas/inspecciones.html`

```html
<!DOCTYPE html>
<html lang="es">

<head>
    <meta charset="UTF-8">

    <meta
        name="viewport"
        content="width=device-width, initial-scale=1.0"
    >

    <title>Inspecciones integrales</title>

    <link
        rel="stylesheet"
        href="/css/estilos.css"
    >
</head>

<body>

    <header>

        <div>
            <h1>Sistema integral de inspección</h1>

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
        <a href="/cajas">Cajas RFID</a>
        <a href="/inspecciones">Inspecciones</a>
        <a href="/estadisticas-inspecciones">
            Estadísticas integrales
        </a>
    </nav>

    <main>

        <section class="tarjetas">

            <article class="tarjeta">
                <h3>Última inspección</h3>

                <p
                    id="ultimaInspeccion"
                    class="numero"
                >
                    --
                </p>
            </article>

            <article class="tarjeta">
                <h3>Producto</h3>

                <p id="ultimoProducto">
                    Sin inspecciones
                </p>
            </article>

            <article class="tarjeta">
                <h3>Estado</h3>

                <p id="ultimoEstado">
                    Sin inspecciones
                </p>
            </article>

            <article class="tarjeta">
                <h3>Resultado general</h3>

                <p id="ultimoResultado">
                    Sin inspecciones
                </p>
            </article>

            <article class="tarjeta">
                <h3>Altura</h3>

                <p id="ultimaAltura">
                    -- cm
                </p>
            </article>

            <article class="tarjeta">
                <h3>Color</h3>

                <p id="ultimoColor">
                    --
                </p>
            </article>

            <article class="tarjeta">
                <h3>Peso</h3>

                <p id="ultimoPeso">
                    -- g
                </p>
            </article>

            <article class="tarjeta">
                <h3>Caja</h3>

                <p id="ultimaCaja">
                    Sin asignar
                </p>
            </article>

        </section>

        <section class="seccion">

            <h2>Historial de inspecciones</h2>

            <div class="tabla-contenedor">

                <table>

                    <thead>
                        <tr>
                            <th>ID</th>
                            <th>Producto</th>
                            <th>Altura</th>
                            <th>Resultado altura</th>
                            <th>Color</th>
                            <th>Resultado color</th>
                            <th>Peso</th>
                            <th>Resultado peso</th>
                            <th>Caja</th>
                            <th>Pieza en caja</th>
                            <th>Resultado general</th>
                            <th>Estado</th>
                            <th>Fecha</th>
                        </tr>
                    </thead>

                    <tbody
                        id="tablaInspecciones"
                    ></tbody>

                </table>

            </div>

        </section>

    </main>

    <script src="/socket.io/socket.io.js"></script>
    <script src="/js/comun.js"></script>
    <script src="/js/inspecciones.js"></script>

</body>
</html>
```

---

# 29. Crear `public/js/inspecciones.js`

```javascript
const socket = io();

const tabla =
    document.getElementById(
        "tablaInspecciones"
    );

function textoResultado(valor)
{
    const textos = {
        correcto:
            "Correcto",

        demasiado_bajo:
            "Demasiado bajo",

        demasiado_alto:
            "Demasiado alto",

        color_incorrecto:
            "Color incorrecto",

        demasiado_ligero:
            "Demasiado ligero",

        demasiado_pesado:
            "Demasiado pesado",

        sin_especificacion:
            "Sin especificación",

        pendiente:
            "Pendiente",

        aceptada:
            "Aceptada",

        rechazada_altura:
            "Rechazada por altura",

        rechazada_color:
            "Rechazada por color",

        rechazada_peso:
            "Rechazada por peso",

        rechazada_varias_causas:
            "Rechazada por varias causas"
    };

    return textos[valor] || valor;
}

function actualizarResumen(inspeccion)
{
    document.getElementById(
        "ultimaInspeccion"
    ).textContent =
        `#${inspeccion.id}`;

    document.getElementById(
        "ultimoProducto"
    ).textContent =
        inspeccion.producto ||
        "Sin producto";

    document.getElementById(
        "ultimoEstado"
    ).textContent =
        inspeccion.estado ===
        "esperando_peso"
        ? "Esperando peso"
        : "Completa";

    document.getElementById(
        "ultimoResultado"
    ).textContent =
        textoResultado(
            inspeccion.resultado_general
        );

    document.getElementById(
        "ultimaAltura"
    ).textContent =
        `${inspeccion.altura_cm} cm`;

    document.getElementById(
        "ultimoColor"
    ).textContent =
        inspeccion.color_detectado;

    document.getElementById(
        "ultimoPeso"
    ).textContent =
        inspeccion.peso_pieza_g === null ||
        inspeccion.peso_pieza_g === undefined
        ? "-- g"
        : `${inspeccion.peso_pieza_g} g`;

    document.getElementById(
        "ultimaCaja"
    ).textContent =
        inspeccion.caja ||
        "Sin asignar";
}

function crearFila(inspeccion)
{
    const fecha =
        inspeccion.fecha_fin ||
        inspeccion.fecha_inicio;

    return `
        <tr>
            <td>${inspeccion.id}</td>

            <td>
                ${inspeccion.producto}
            </td>

            <td>
                ${inspeccion.altura_cm} cm
            </td>

            <td>
                ${textoResultado(
                    inspeccion.resultado_altura
                )}
            </td>

            <td>
                ${inspeccion.color_detectado}
            </td>

            <td>
                ${textoResultado(
                    inspeccion.resultado_color
                )}
            </td>

            <td>
                ${
                    inspeccion.peso_pieza_g === null
                    ? "--"
                    : `${inspeccion.peso_pieza_g} g`
                }
            </td>

            <td>
                ${textoResultado(
                    inspeccion.resultado_peso
                )}
            </td>

            <td>
                ${inspeccion.caja || "Sin asignar"}
            </td>

            <td>
                ${
                    inspeccion.numero_pieza_caja ||
                    "--"
                }
            </td>

            <td>
                <strong>
                    ${textoResultado(
                        inspeccion.resultado_general
                    )}
                </strong>
            </td>

            <td>
                ${
                    inspeccion.estado ===
                    "esperando_peso"
                    ? "Esperando peso"
                    : "Completa"
                }
            </td>

            <td>
                ${new Date(
                    fecha
                ).toLocaleString()}
            </td>
        </tr>
    `;
}

async function cargarInspecciones()
{
    const respuesta = await fetch(
        "/api/inspecciones"
    );

    if(!respuesta.ok)
    {
        return;
    }

    const inspecciones =
        await respuesta.json();

    tabla.innerHTML = "";

    inspecciones.forEach(
        inspeccion =>
        {
            tabla.innerHTML +=
                crearFila(inspeccion);
        }
    );

    if(inspecciones.length > 0)
    {
        actualizarResumen(
            inspecciones[0]
        );
    }
}

socket.on(
    "inspeccion_iniciada",
    (inspeccion) =>
    {
        actualizarResumen(
            inspeccion
        );

        cargarInspecciones();
    }
);

socket.on(
    "inspeccion_completada",
    (inspeccion) =>
    {
        actualizarResumen(
            inspeccion
        );

        cargarInspecciones();
    }
);

document.addEventListener(
    "DOMContentLoaded",
    cargarInspecciones
);
```

---

# Bloque 11. Página de estadísticas

## 30. Crear `paginas/estadisticas-inspecciones.html`

```html
<!DOCTYPE html>
<html lang="es">

<head>
    <meta charset="UTF-8">

    <meta
        name="viewport"
        content="width=device-width, initial-scale=1.0"
    >

    <title>Estadísticas integrales</title>

    <link
        rel="stylesheet"
        href="/css/estilos.css"
    >
</head>

<body>

    <header>

        <div>
            <h1>Estadísticas integrales</h1>

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
        <a href="/cajas">Cajas RFID</a>
        <a href="/inspecciones">Inspecciones</a>
        <a href="/estadisticas-inspecciones">
            Estadísticas integrales
        </a>
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
                <h3>Total</h3>
                <p id="total" class="numero">0</p>
            </article>

            <article class="tarjeta">
                <h3>Pendientes</h3>
                <p id="pendientes" class="numero">0</p>
            </article>

            <article class="tarjeta">
                <h3>Completas</h3>
                <p id="completas" class="numero">0</p>
            </article>

            <article class="tarjeta">
                <h3>Aceptadas</h3>
                <p id="aceptadas" class="numero">0</p>
            </article>

            <article class="tarjeta">
                <h3>Rechazadas</h3>
                <p id="rechazadas" class="numero">0</p>
            </article>

            <article class="tarjeta">
                <h3>Cumplimiento</h3>
                <p id="cumplimiento" class="numero">0 %</p>
            </article>

            <article class="tarjeta">
                <h3>Altura promedio</h3>
                <p id="alturaPromedio" class="numero">0 cm</p>
            </article>

            <article class="tarjeta">
                <h3>Peso promedio</h3>
                <p id="pesoPromedio" class="numero">0 g</p>
            </article>

            <article class="tarjeta">
                <h3>Fallas de altura</h3>
                <p id="fallasAltura" class="numero">0</p>
            </article>

            <article class="tarjeta">
                <h3>Fallas de color</h3>
                <p id="fallasColor" class="numero">0</p>
            </article>

            <article class="tarjeta">
                <h3>Fallas de peso</h3>
                <p id="fallasPeso" class="numero">0</p>
            </article>

        </section>

        <section class="seccion">

            <h2>Altura y peso por inspección</h2>

            <div class="contenedor-grafica">
                <canvas id="graficaMediciones"></canvas>
            </div>

        </section>

        <section class="seccion">

            <h2>Resultados generales</h2>

            <div class="contenedor-grafica">
                <canvas id="graficaResultados"></canvas>
            </div>

        </section>

        <section class="seccion">

            <h2>Tipos de falla</h2>

            <div class="contenedor-grafica">
                <canvas id="graficaFallas"></canvas>
            </div>

        </section>

    </main>

    <script
        src="https://cdn.jsdelivr.net/npm/chart.js"
    ></script>

    <script src="/socket.io/socket.io.js"></script>
    <script src="/js/comun.js"></script>
    <script src="/js/estadisticas-inspecciones.js"></script>

</body>
</html>
```

---

# 31. Crear `public/js/estadisticas-inspecciones.js`

```javascript
const socket = io();

const selectorProducto =
    document.getElementById(
        "filtroProducto"
    );

let graficaMediciones = null;
let graficaResultados = null;
let graficaFallas = null;

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
        "/api/estadisticas-inspecciones";

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

    const pendientes =
        numero(resumen.pendientes);

    const completas =
        numero(resumen.completas);

    const aceptadas =
        numero(resumen.aceptadas);

    const rechazadas =
        numero(resumen.rechazadas);

    let cumplimiento = 0;

    if(completas > 0)
    {
        cumplimiento =
            (
                aceptadas /
                completas *
                100
            ).toFixed(1);
    }

    document.getElementById(
        "total"
    ).textContent =
        total;

    document.getElementById(
        "pendientes"
    ).textContent =
        pendientes;

    document.getElementById(
        "completas"
    ).textContent =
        completas;

    document.getElementById(
        "aceptadas"
    ).textContent =
        aceptadas;

    document.getElementById(
        "rechazadas"
    ).textContent =
        rechazadas;

    document.getElementById(
        "cumplimiento"
    ).textContent =
        `${cumplimiento} %`;

    document.getElementById(
        "alturaPromedio"
    ).textContent =
        `${numero(
            resumen.altura_promedio
        )} cm`;

    document.getElementById(
        "pesoPromedio"
    ).textContent =
        `${numero(
            resumen.peso_promedio
        )} g`;

    document.getElementById(
        "fallasAltura"
    ).textContent =
        numero(
            resumen.fallas_altura
        );

    document.getElementById(
        "fallasColor"
    ).textContent =
        numero(
            resumen.fallas_color
        );

    document.getElementById(
        "fallasPeso"
    ).textContent =
        numero(
            resumen.fallas_peso
        );

    const serie =
        [...datos.serie].reverse();

    if(graficaMediciones)
    {
        graficaMediciones.destroy();
    }

    graficaMediciones =
        new Chart(
            document.getElementById(
                "graficaMediciones"
            ),
            {
                type: "line",

                data: {
                    labels:
                        serie.map(
                            elemento =>
                                `#${elemento.id}`
                        ),

                    datasets: [
                        {
                            label:
                                "Altura en cm",

                            data:
                                serie.map(
                                    elemento =>
                                        Number(
                                            elemento.altura_cm
                                        )
                                )
                        },

                        {
                            label:
                                "Peso en g",

                            data:
                                serie.map(
                                    elemento =>
                                        elemento.peso_pieza_g ===
                                        null
                                        ? null
                                        : Number(
                                            elemento.peso_pieza_g
                                        )
                                )
                        }
                    ]
                },

                options: {
                    responsive: true,

                    spanGaps: false
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
                type: "bar",

                data: {
                    labels:
                        datos.distribucion.map(
                            elemento =>
                                elemento.resultado_general
                        ),

                    datasets: [
                        {
                            label:
                                "Inspecciones",

                            data:
                                datos.distribucion.map(
                                    elemento =>
                                        Number(
                                            elemento.cantidad
                                        )
                                )
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

    if(graficaFallas)
    {
        graficaFallas.destroy();
    }

    graficaFallas =
        new Chart(
            document.getElementById(
                "graficaFallas"
            ),
            {
                type: "doughnut",

                data: {
                    labels: [
                        "Altura",
                        "Color",
                        "Peso"
                    ],

                    datasets: [
                        {
                            data: [
                                numero(
                                    resumen.fallas_altura
                                ),

                                numero(
                                    resumen.fallas_color
                                ),

                                numero(
                                    resumen.fallas_peso
                                )
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
    "inspeccion_iniciada",
    cargarEstadisticas
);

socket.on(
    "inspeccion_completada",
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

# 32. Agregar enlaces al menú

En las páginas privadas agregar:

```html
<a href="/inspecciones">
    Inspecciones
</a>

<a href="/estadisticas-inspecciones">
    Estadísticas integrales
</a>
```

---

# Bloque 12. Código del ESP32 de la banda

## 33. Funciones del ESP32 de inspección

Este ESP32 realizará:

* Detección de la pieza.
* Medición de distancia.
* Cálculo local de altura.
* Medición de color.
* Promedio de varias muestras.
* Envío de una inspección al servidor.

---

# 34. Código completo del ESP32 de la banda

```cpp
#include <WiFi.h>
#include <HTTPClient.h>

#include <Wire.h>
#include <Adafruit_TCS34725.h>

// ==================================================
// RED
// ==================================================

const char* NOMBRE_WIFI =
    "NOMBRE_DE_LA_RED";

const char* CONTRASENA_WIFI =
    "CONTRASENA_DE_LA_RED";

const char* URL_SERVIDOR =
    "http://192.168.1.25:3000/api/inspecciones/iniciar";

const char* CLAVE_DISPOSITIVO =
    "clave-banda-industria-40";

// ==================================================
// IDENTIFICACIÓN
// ==================================================

const int PRODUCTO_ID = 1;

const char* NOMBRE_DISPOSITIVO =
    "banda_01";

// ==================================================
// PINES
// ==================================================

const int PIN_SENSOR_IR = 19;

const int PIN_TRIG = 5;
const int PIN_ECHO = 18;

const int PIN_SDA = 21;
const int PIN_SCL = 22;

const int ESTADO_OBJETO = LOW;

// ==================================================
// REFERENCIA DE ALTURA
// ==================================================

// Medir físicamente con la banda vacía.
const float DISTANCIA_BANDA_CM =
    30.0;

// ==================================================
// SENSOR DE COLOR
// ==================================================

Adafruit_TCS34725 sensorColor(
    TCS34725_INTEGRATIONTIME_50MS,
    TCS34725_GAIN_4X
);

// ==================================================
// MEDICIÓN
// ==================================================

const int TOTAL_MUESTRAS_DISTANCIA = 7;
const int TOTAL_MUESTRAS_COLOR = 8;

const unsigned long RETARDO_CENTRADO_MS =
    80;

const unsigned long TIEMPO_LIBRE_MS =
    150;

bool objetoProcesado = false;

unsigned long inicioZonaLibre = 0;

// ==================================================
// ESTRUCTURA DE COLOR
// ==================================================

struct LecturaColor
{
    uint32_t rojo;
    uint32_t verde;
    uint32_t azul;
    uint32_t claro;
};

// ==================================================
// WIFI
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

    return
        duracion *
        0.0343 /
        2.0;
}

// ==================================================
// ORDENAR DISTANCIAS
// ==================================================

void ordenarValores(
    float valores[],
    int cantidad
)
{
    for(
        int i = 0;
        i < cantidad - 1;
        i++
    )
    {
        for(
            int j = 0;
            j < cantidad - i - 1;
            j++
        )
        {
            if(
                valores[j] >
                valores[j + 1]
            )
            {
                float temporal =
                    valores[j];

                valores[j] =
                    valores[j + 1];

                valores[j + 1] =
                    temporal;
            }
        }
    }
}

// ==================================================
// DISTANCIA ESTABLE
// ==================================================

float obtenerDistanciaEstable()
{
    float muestras[
        TOTAL_MUESTRAS_DISTANCIA
    ];

    int validas = 0;
    int intentos = 0;

    while(
        validas <
        TOTAL_MUESTRAS_DISTANCIA &&

        intentos <
        TOTAL_MUESTRAS_DISTANCIA * 2
    )
    {
        float distancia =
            medirDistanciaCm();

        intentos++;

        if(
            distancia > 1.0 &&
            distancia <
            DISTANCIA_BANDA_CM
        )
        {
            muestras[validas] =
                distancia;

            validas++;
        }

        delay(30);
    }

    if(validas < 3)
    {
        return -1.0;
    }

    ordenarValores(
        muestras,
        validas
    );

    int centro =
        validas / 2;

    if(validas % 2 == 1)
    {
        return muestras[centro];
    }

    return
        (
            muestras[centro - 1] +
            muestras[centro]
        ) /
        2.0;
}

// ==================================================
// COLOR PROMEDIO
// ==================================================

bool obtenerColorPromedio(
    LecturaColor& resultado
)
{
    uint64_t sumaRojo = 0;
    uint64_t sumaVerde = 0;
    uint64_t sumaAzul = 0;
    uint64_t sumaClaro = 0;

    int validas = 0;

    for(
        int i = 0;
        i < TOTAL_MUESTRAS_COLOR;
        i++
    )
    {
        uint16_t rojo;
        uint16_t verde;
        uint16_t azul;
        uint16_t claro;

        sensorColor.getRawData(
            &rojo,
            &verde,
            &azul,
            &claro
        );

        if(
            rojo == 0 &&
            verde == 0 &&
            azul == 0
        )
        {
            delay(60);
            continue;
        }

        sumaRojo += rojo;
        sumaVerde += verde;
        sumaAzul += azul;
        sumaClaro += claro;

        validas++;

        delay(60);
    }

    if(validas < 3)
    {
        return false;
    }

    resultado.rojo =
        sumaRojo / validas;

    resultado.verde =
        sumaVerde / validas;

    resultado.azul =
        sumaAzul / validas;

    resultado.claro =
        sumaClaro / validas;

    return true;
}

// ==================================================
// ENVIAR INSPECCIÓN
// ==================================================

bool enviarInspeccion(
    float distanciaObjeto,
    const LecturaColor& color
)
{
    if(
        WiFi.status() !=
        WL_CONNECTED
    )
    {
        conectarWiFi();
    }

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
        ) +
        ",";

    datos +=
        "\"rojo_raw\":" +
        String(color.rojo) +
        ",";

    datos +=
        "\"verde_raw\":" +
        String(color.verde) +
        ",";

    datos +=
        "\"azul_raw\":" +
        String(color.azul) +
        ",";

    datos +=
        "\"claro_raw\":" +
        String(color.claro);

    datos += "}";

    for(int intento = 1; intento <= 3; intento++)
    {
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

        Serial.println();
        Serial.print(
            "Envío, intento "
        );

        Serial.println(intento);

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
            Serial.println(
                http.getString()
            );
        }

        bool correcto =
            codigoHTTP >= 200 &&
            codigoHTTP < 300;

        http.end();

        if(correcto)
        {
            return true;
        }

        delay(1000);
    }

    return false;
}

// ==================================================
// CONFIGURACIÓN
// ==================================================

void setup()
{
    Serial.begin(115200);

    pinMode(
        PIN_SENSOR_IR,
        INPUT
    );

    pinMode(
        PIN_TRIG,
        OUTPUT
    );

    pinMode(
        PIN_ECHO,
        INPUT
    );

    digitalWrite(
        PIN_TRIG,
        LOW
    );

    Wire.begin(
        PIN_SDA,
        PIN_SCL
    );

    if(!sensorColor.begin())
    {
        Serial.println(
            "No se encontró el TCS34725."
        );

        while(true)
        {
            delay(1000);
        }
    }

    conectarWiFi();

    Serial.println();
    Serial.println(
        "ESTACIÓN DE INSPECCIÓN PREPARADA"
    );

    Serial.print(
        "Distancia a la banda: "
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
        ) ==
        ESTADO_OBJETO;

    if(
        objetoPresente &&
        !objetoProcesado
    )
    {
        Serial.println();
        Serial.println(
            "Pieza detectada"
        );

        delay(
            RETARDO_CENTRADO_MS
        );

        float distanciaObjeto =
            obtenerDistanciaEstable();

        LecturaColor color;

        bool colorCorrecto =
            obtenerColorPromedio(
                color
            );

        if(
            distanciaObjeto > 0 &&
            colorCorrecto
        )
        {
            float altura =
                DISTANCIA_BANDA_CM -
                distanciaObjeto;

            Serial.print(
                "Altura estimada: "
            );

            Serial.print(
                altura,
                2
            );

            Serial.println(" cm");

            Serial.print("R: ");
            Serial.println(color.rojo);

            Serial.print("G: ");
            Serial.println(color.verde);

            Serial.print("B: ");
            Serial.println(color.azul);

            Serial.print("C: ");
            Serial.println(color.claro);

            bool enviada =
                enviarInspeccion(
                    distanciaObjeto,
                    color
                );

            if(enviada)
            {
                Serial.println(
                    "Inspección creada."
                );
            }
            else
            {
                Serial.println(
                    "No fue posible crear la inspección."
                );
            }
        }
        else
        {
            Serial.println(
                "No se obtuvieron mediciones válidas."
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

    delay(10);
}
```

---

# 35. Configuración del ESP32 de banda

Modificar:

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
    "http://DIRECCION_IP:3000/api/inspecciones/iniciar";
```

```cpp
const int PRODUCTO_ID = 1;
```

```cpp
const float DISTANCIA_BANDA_CM =
    30.0;
```

---

# Bloque 13. Código del ESP32 de recepción

## 36. Funciones del ESP32 de recepción

Este ESP32 realizará:

* Lectura RFID.
* Validación de la caja.
* Tara automática.
* Detección del aumento de peso.
* Espera de estabilidad.
* Consulta de la inspección pendiente.
* Envío del peso.
* Asociación con la caja.
* Detección de caja llena.
* Detección de retiro de la caja.

La ruta para identificar cajas creada en la práctica RFID deberá conservarse:

```text
POST /api/rfid/identificar
```

---

# 37. Código completo del ESP32 de recepción

```cpp
#include <WiFi.h>
#include <HTTPClient.h>

#include <SPI.h>
#include <MFRC522.h>

#include <HX711.h>
#include <math.h>

// ==================================================
// RED
// ==================================================

const char* NOMBRE_WIFI =
    "NOMBRE_DE_LA_RED";

const char* CONTRASENA_WIFI =
    "CONTRASENA_DE_LA_RED";

const char* URL_IDENTIFICAR_CAJA =
    "http://192.168.1.25:3000/api/rfid/identificar";

const char* URL_PENDIENTE =
    "http://192.168.1.25:3000/api/inspecciones/pendiente";

const char* URL_BASE_COMPLETAR =
    "http://192.168.1.25:3000/api/inspecciones";

const char* CLAVE_DISPOSITIVO =
    "clave-banda-industria-40";

// ==================================================
// IDENTIFICACIÓN
// ==================================================

const char* NOMBRE_DISPOSITIVO =
    "bascula_01";

// ==================================================
// HX711
// ==================================================

const int PIN_HX711_DOUT = 16;
const int PIN_HX711_SCK = 17;

const float FACTOR_CALIBRACION =
    -421.782013;

HX711 bascula;

// ==================================================
// RC522
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
// PARÁMETROS DE PESO
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
// ESTADO DE LA CAJA
// ==================================================

bool cajaActiva = false;
bool sistemaListo = false;

String uidCajaActiva = "";
String nombreCajaActiva = "";

float capacidadCajaG = 0.0;

// ==================================================
// ESTADO DE PESAJE
// ==================================================

float pesoAnterior = 0.0;

int contadorPiezas = 0;

// Se conserva hasta que el servidor confirme
// que la inspección fue completada.
int inspeccionPendienteId = 0;

unsigned long ultimaLecturaSerial = 0;

// ==================================================
// WIFI
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
// UID A TEXTO
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
// LEER RFID
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
// IDENTIFICAR CAJA
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
        "Respuesta RFID: "
    );

    Serial.println(
        respuesta
    );

    http.end();

    if(
        codigoHTTP != 200 ||
        !respuesta.startsWith("OK|")
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
// REVISAR RFID
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
                "La caja ya está activa."
            );
        }
        else
        {
            Serial.println(
                "Retire la caja actual antes de cambiarla."
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

        delay(1000);
        return;
    }

    uidCajaActiva = uid;
    nombreCajaActiva = nombre;
    capacidadCajaG = capacidad;

    cajaActiva = true;

    Serial.println();
    Serial.print(
        "Caja identificada: "
    );

    Serial.println(
        nombreCajaActiva
    );

    Serial.println(
        "Realizando tara..."
    );

    delay(2000);

    bascula.tare(20);

    pesoAnterior = 0.0;
    contadorPiezas = 0;
    inspeccionPendienteId = 0;

    sistemaListo = true;

    Serial.println(
        "Estación preparada."
    );
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

    int estables = 0;

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
            estables++;
        }
        else
        {
            estables = 0;
        }

        lecturaAnterior =
            lecturaActual;

        if(
            estables >=
            LECTURAS_ESTABLES_NECESARIAS
        )
        {
            return lecturaActual;
        }
    }

    return NAN;
}

// ==================================================
// OBTENER INSPECCIÓN PENDIENTE
// ==================================================

bool obtenerInspeccionPendiente(
    int& inspeccionId
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
        URL_PENDIENTE
    );

    http.addHeader(
        "x-api-key",
        CLAVE_DISPOSITIVO
    );

    int codigoHTTP =
        http.GET();

    String respuesta =
        http.getString();

    http.end();

    if(codigoHTTP != 200)
    {
        Serial.println(
            "No existe una inspección pendiente."
        );

        return false;
    }

    if(!respuesta.startsWith("OK|"))
    {
        return false;
    }

    inspeccionId =
        respuesta.substring(
            3
        ).toInt();

    return inspeccionId > 0;
}

// ==================================================
// COMPLETAR INSPECCIÓN
// ==================================================

bool completarInspeccion(
    int inspeccionId,
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

    String url =
        String(URL_BASE_COMPLETAR) +
        "/" +
        String(inspeccionId) +
        "/completar-peso";

    HTTPClient http;

    http.begin(url);

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
        uidCajaActiva +
        "\",";

    datos +=
        "\"numero_pieza_caja\":" +
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
    Serial.print(
        "Completando inspección #"
    );

    Serial.println(
        inspeccionId
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
        Serial.println(
            http.getString()
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
        "Caja retirada."
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
    inspeccionPendienteId = 0;

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

    bascula.begin(
        PIN_HX711_DOUT,
        PIN_HX711_SCK
    );

    bascula.set_scale(
        FACTOR_CALIBRACION
    );

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
        "ESTACIÓN DE RECEPCIÓN PREPARADA"
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
            "No fue posible leer el HX711."
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
            " | Peso: "
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
                "ALERTA: caja llena"
            );
        }
    }

    float cambio =
        pesoActual -
        pesoAnterior;

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
                "El peso no se estabilizó."
            );

            delay(1000);
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

        if(inspeccionPendienteId == 0)
        {
            bool encontrada =
                obtenerInspeccionPendiente(
                    inspeccionPendienteId
                );

            if(!encontrada)
            {
                Serial.println(
                    "La pieza ya cayó, pero todavía"
                );

                Serial.println(
                    "no existe una inspección pendiente."
                );

                Serial.println(
                    "Se volverá a intentar."
                );

                delay(2000);
                return;
            }
        }

        int siguientePieza =
            contadorPiezas + 1;

        Serial.print(
            "Inspección: #"
        );

        Serial.println(
            inspeccionPendienteId
        );

        Serial.print(
            "Peso de la pieza: "
        );

        Serial.print(
            pesoPieza,
            2
        );

        Serial.println(" g");

        bool completada =
            completarInspeccion(
                inspeccionPendienteId,
                pesoAnterior,
                pesoEstable,
                siguientePieza
            );

        if(completada)
        {
            pesoAnterior =
                pesoEstable;

            contadorPiezas =
                siguientePieza;

            inspeccionPendienteId = 0;

            Serial.println(
                "Inspección completada."
            );
        }
        else
        {
            Serial.println(
                "El servidor no confirmó la inspección."
            );

            Serial.println(
                "Se conservará el mismo ID para reintentar."
            );
        }

        delay(1000);
    }

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

# 38. Configuración del ESP32 de recepción

Modificar:

```cpp
const char* NOMBRE_WIFI =
    "NOMBRE_DE_LA_RED";
```

```cpp
const char* CONTRASENA_WIFI =
    "CONTRASENA_DE_LA_RED";
```

```cpp
const char* URL_IDENTIFICAR_CAJA =
    "http://DIRECCION_IP:3000/api/rfid/identificar";
```

```cpp
const char* URL_PENDIENTE =
    "http://DIRECCION_IP:3000/api/inspecciones/pendiente";
```

```cpp
const char* URL_BASE_COMPLETAR =
    "http://DIRECCION_IP:3000/api/inspecciones";
```

```cpp
const float FACTOR_CALIBRACION =
    -421.782013;
```

---

# Bloque 14. Ejecutar el sistema

## 39. Orden de encendido

1. Iniciar MySQL.
2. Iniciar Node.js.
3. Abrir la página de inspecciones.
4. Encender el ESP32 de la banda.
5. Encender el ESP32 de recepción.
6. Colocar una caja vacía en la báscula.
7. Acercar su llavero RFID.
8. Esperar que termine la tara.
9. Encender o iniciar la banda.
10. Enviar las piezas de una en una.

---

# 40. Iniciar Node.js

```bash
node server.js
```

Resultado esperado:

```text
Conexión correcta con MySQL
Servidor disponible en http://localhost:3000
```

---

# 41. Abrir el monitoreo

```text
http://localhost:3000/inspecciones
```

---

# 42. Abrir las estadísticas

```text
http://localhost:3000/estadisticas-inspecciones
```

---

# Bloque 15. Pruebas obligatorias

## Prueba 1. Crear una inspección pendiente

Detener temporalmente la pieza antes de que caiga en la caja.

Pasarla por la estación de altura y color.

Consultar:

```sql
SELECT
    id,
    altura_cm,
    color_detectado,
    resultado_altura,
    resultado_color,
    estado
FROM inspecciones
ORDER BY id DESC
LIMIT 1;
```

Resultado esperado:

```text
estado = esperando_peso
resultado_peso = pendiente
resultado_general = pendiente
```

---

## Prueba 2. Completar la inspección

Dejar que la pieza caiga en la caja.

Resultado esperado:

```text
estado = completa
peso_pieza_g = valor medido
resultado_peso = resultado calculado
resultado_general = resultado final
```

---

## Prueba 3. Pieza completamente correcta

Utilizar una pieza que cumpla:

* Altura.
* Color.
* Peso.

Resultado esperado:

```text
aceptada
```

---

## Prueba 4. Falla de altura

Utilizar una pieza con color y peso correctos, pero altura incorrecta.

Resultado esperado:

```text
rechazada_altura
```

---

## Prueba 5. Falla de color

Utilizar una pieza con altura y peso correctos, pero color incorrecto.

Resultado esperado:

```text
rechazada_color
```

---

## Prueba 6. Falla de peso

Utilizar una pieza con altura y color correctos, pero peso incorrecto.

Resultado esperado:

```text
rechazada_peso
```

---

## Prueba 7. Varias fallas

Utilizar una pieza que no cumpla al menos dos especificaciones.

Resultado esperado:

```text
rechazada_varias_causas
```

---

## Prueba 8. Dos inspecciones pendientes

Pasar dos piezas por la estación de altura y color sin dejarlas caer todavía.

Consultar:

```sql
SELECT
    id,
    estado
FROM inspecciones
WHERE estado = 'esperando_peso'
ORDER BY id;
```

Después dejar caer las piezas en el mismo orden.

Comprobar que:

```text
La primera caída completa el ID menor.
La segunda caída completa el siguiente ID.
```

---

## Prueba 9. Alterar el orden

Realizar una prueba controlada cambiando físicamente el orden de dos piezas después de la estación de inspección.

Observar que la asociación FIFO ya no será correcta.

Explicar por qué ocurrió.

---

## Prueba 10. Pieza sin inspección previa

Colocar manualmente una pieza dentro de la caja sin pasarla por la estación de altura y color.

Resultado esperado:

```text
No existe una inspección pendiente.
```

El ESP32 conservará el cambio de peso y volverá a intentar.

Para recuperar el sistema:

1. Retirar la pieza.
2. Realizar una nueva tara o cambiar la caja.
3. Pasar correctamente la pieza por la banda.

---

## Prueba 11. Caja no identificada

Intentar recibir piezas sin acercar un RFID.

La estación de peso deberá permanecer detenida.

---

## Prueba 12. Caja inactiva

Desactivar una caja:

```sql
UPDATE cajas
SET estado = 'inactiva'
WHERE nombre = 'CAJA_01';
```

Intentar identificarla.

Después volver a activarla:

```sql
UPDATE cajas
SET estado = 'activa'
WHERE nombre = 'CAJA_01';
```

---

## Prueba 13. Caja llena

Configurar una capacidad pequeña:

```sql
UPDATE cajas
SET capacidad_max_g = 500
WHERE nombre = 'CAJA_01';
```

Agregar piezas hasta alcanzar la capacidad.

Comprobar la alerta.

---

## Prueba 14. Actualización en tiempo real

Mantener abierta:

```text
/inspecciones
```

Comprobar dos cambios:

```text
1. Inspección iniciada:
   esperando peso.

2. Inspección completada:
   muestra peso, caja y resultado general.
```

---

## Prueba 15. Estadísticas

Registrar al menos:

* Diez piezas aceptadas.
* Tres piezas rechazadas por altura.
* Tres piezas rechazadas por color.
* Tres piezas rechazadas por peso.
* Dos piezas rechazadas por varias causas.

Comprobar:

* Total.
* Pendientes.
* Completas.
* Aceptadas.
* Rechazadas.
* Cumplimiento.
* Fallas por tipo.
* Promedio de altura.
* Promedio de peso.
* Gráficas.

---

# 43. Consultas SQL de comprobación

## Consultar inspecciones completas

```sql
SELECT
    inspecciones.id,

    productos.nombre AS producto,

    inspecciones.altura_cm,
    inspecciones.resultado_altura,

    inspecciones.color_detectado,
    inspecciones.resultado_color,

    inspecciones.peso_pieza_g,
    inspecciones.resultado_peso,

    cajas.nombre AS caja,

    inspecciones.numero_pieza_caja,

    inspecciones.resultado_general,

    inspecciones.fecha_inicio,
    inspecciones.fecha_fin

FROM inspecciones

INNER JOIN productos
    ON inspecciones.producto_id =
       productos.id

LEFT JOIN cajas
    ON inspecciones.caja_id =
       cajas.id

ORDER BY
    inspecciones.id DESC;
```

---

## Consultar inspecciones pendientes

```sql
SELECT
    id,
    producto_id,
    altura_cm,
    color_detectado,
    fecha_inicio

FROM inspecciones

WHERE estado = 'esperando_peso'

ORDER BY id;
```

---

## Contar resultados generales

```sql
SELECT
    resultado_general,
    COUNT(*) AS cantidad

FROM inspecciones

GROUP BY resultado_general;
```

---

## Contar fallas de altura

```sql
SELECT
    resultado_altura,
    COUNT(*) AS cantidad

FROM inspecciones

GROUP BY resultado_altura;
```

---

## Contar fallas de color

```sql
SELECT
    resultado_color,
    COUNT(*) AS cantidad

FROM inspecciones

GROUP BY resultado_color;
```

---

## Contar fallas de peso

```sql
SELECT
    resultado_peso,
    COUNT(*) AS cantidad

FROM inspecciones

GROUP BY resultado_peso;
```

---

## Estadísticas por producto

```sql
SELECT
    productos.nombre,

    COUNT(*) AS total,

    ROUND(
        AVG(inspecciones.altura_cm),
        2
    ) AS altura_promedio,

    ROUND(
        AVG(inspecciones.peso_pieza_g),
        2
    ) AS peso_promedio,

    SUM(
        inspecciones.resultado_general =
        'aceptada'
    ) AS aceptadas,

    SUM(
        inspecciones.resultado_general LIKE
        'rechazada%'
    ) AS rechazadas

FROM inspecciones

INNER JOIN productos
    ON inspecciones.producto_id =
       productos.id

GROUP BY
    productos.id,
    productos.nombre;
```

---

## Contenido de una caja

```sql
SELECT
    cajas.nombre AS caja,

    inspecciones.numero_pieza_caja,

    productos.nombre AS producto,

    inspecciones.altura_cm,

    inspecciones.color_detectado,

    inspecciones.peso_pieza_g,

    inspecciones.resultado_general,

    inspecciones.fecha_fin

FROM inspecciones

INNER JOIN cajas
    ON inspecciones.caja_id =
       cajas.id

INNER JOIN productos
    ON inspecciones.producto_id =
       productos.id

WHERE cajas.nombre = 'CAJA_01'

ORDER BY
    inspecciones.numero_pieza_caja;
```

---

# 44. Evidencias que deberán entregarse

El reporte deberá incluir:

1. Portada.
2. Objetivo general.
3. Objetivos específicos.
4. Diagrama general del sistema.
5. Diagrama de conexión del ESP32 de banda.
6. Diagrama de conexión del ESP32 de recepción.
7. Fotografías de ambas estaciones.
8. Captura de una inspección pendiente.
9. Captura de una inspección completa.
10. Captura del monitor serial del ESP32 de banda.
11. Captura del monitor serial del ESP32 de recepción.
12. Captura de la tabla `inspecciones`.
13. Captura de la página de monitoreo.
14. Captura de las estadísticas.
15. Prueba de pieza aceptada.
16. Prueba de rechazo por altura.
17. Prueba de rechazo por color.
18. Prueba de rechazo por peso.
19. Prueba de varias causas.
20. Prueba de dos inspecciones pendientes.
21. Análisis del funcionamiento FIFO.
22. Análisis de errores.
23. Propuestas de mejora.
24. Conclusiones.

---

# 45. Investigar

Investigar y explicar con palabras propias:

* ¿Qué es una arquitectura distribuida?
* ¿Por qué se utilizaron dos ESP32?
* ¿Qué ventajas tiene separar las estaciones?
* ¿Qué significa FIFO?
* ¿Qué es una cola de eventos?
* ¿Cómo relaciona el servidor las mediciones?
* ¿Qué ocurre si las piezas cambian de orden?
* ¿Qué ocurre si una pieza se retira entre estaciones?
* ¿Qué es una inspección pendiente?
* ¿Qué es una inspección completa?
* ¿Qué función realiza una transacción en MySQL?
* ¿Qué función cumple `FOR UPDATE`?
* ¿Por qué se bloquea temporalmente la inspección?
* ¿Qué problema podría producir que dos dispositivos completen el mismo registro?
* ¿Qué es una solicitud idempotente?
* ¿Por qué el servidor responde correctamente si la inspección ya estaba completa?
* ¿Qué es Socket.IO?
* ¿Qué diferencia existe entre HTTP y Socket.IO?
* ¿Qué función realiza una llave foránea?
* ¿Qué relación existe entre productos, cajas e inspecciones?
* ¿Qué es un índice en una base de datos?
* ¿Qué representa el resultado general?
* ¿Por qué una inspección puede quedar sin especificación?
* ¿Qué significa trazabilidad?
* ¿Cómo se identifica el contenido de una caja?
* ¿Cómo podrían sincronizarse las estaciones sin depender únicamente de FIFO?
* ¿Cómo podría utilizarse un código QR para identificar directamente una pieza?
* ¿Cómo podría utilizarse RFID en cada pieza?
* ¿Cómo podría utilizarse visión artificial?
* ¿Cómo podría detenerse automáticamente la banda si existen demasiadas inspecciones pendientes?
* ¿Cómo podría generarse una alarma cuando una caja esté llena?
* ¿Cómo podría registrarse automáticamente un movimiento de inventario?

---

# 46. Consideraciones y limitaciones

El sistema puede perder la relación correcta cuando:

* Dos piezas cambian de orden.
* Una pieza es retirada.
* Dos piezas caen simultáneamente.
* Una pieza no es detectada por el infrarrojo.
* La estación de color o altura falla.
* La báscula no detecta una pieza.
* El servidor deja de funcionar.
* Se pierde la conexión WiFi.
* Se reinicia un ESP32 durante el proceso.
* La caja se retira con una inspección pendiente.
* Se cambia el producto sin actualizar ambos sistemas.
* La banda avanza demasiado rápido.

La práctica supone que:

* Las piezas pasan de una en una.
* Las piezas conservan su orden.
* El producto configurado corresponde al producto real.
* La caja se identifica antes de comenzar.
* La caja está vacía cuando se realiza la tara.
* La estación de recepción tiene una sola báscula.
* Todos los dispositivos usan el mismo servidor.

---

# 47. Posibles mejoras

El sistema podría mejorarse posteriormente mediante:

## Identificador individual por pieza

Cada pieza podría llevar:

* RFID.
* Código QR.
* Código de barras.
* Identificador impreso.

Así las estaciones no dependerían exclusivamente del orden FIFO.

## Sensor adicional en la caída

Un sensor infrarrojo antes de la caja podría confirmar que la pieza salió de la banda.

## Control automático de la banda

El servidor podría detener la banda cuando:

* No exista caja.
* La caja esté llena.
* Existan demasiadas inspecciones pendientes.
* Se pierda la comunicación con la báscula.
* Ocurra un error de sensor.

## Rechazo automático

Un servo o cilindro podría separar:

* Piezas aceptadas.
* Piezas rechazadas.
* Piezas que requieren revisión.

## Inventario automático

Al completar una inspección aceptada, el servidor podría:

* Incrementar el inventario.
* Registrar un movimiento.
* Asociar la pieza con un lote.
* Generar una etiqueta.
* Actualizar la cantidad de la caja.

---

# 48. Resultado final del proyecto

Al concluir esta práctica, el sistema será capaz de:

```text
Detectar una pieza
        │
        ▼
Medir su altura
        │
        ▼
Medir su color
        │
        ▼
Crear una inspección
        │
        ▼
Medir su peso
        │
        ▼
Identificar la caja receptora
        │
        ▼
Evaluar todas las especificaciones
        │
        ▼
Guardar la inspección completa
        │
        ▼
Mostrar estadísticas y trazabilidad
```

La base de datos permitirá conocer:

```text
Qué producto fue inspeccionado
Cuándo fue inspeccionado
Cuál fue su altura
Cuál fue su color
Cuál fue su peso
Qué especificación incumplió
En qué caja fue depositado
Qué número de pieza ocupa dentro de la caja
Cuál fue el resultado final
```
