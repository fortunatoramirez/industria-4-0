# Práctica: Inspección y clasificación de piezas por color

## Integración del sensor TCS34725 con ESP32, sensor infrarrojo, Node.js, Socket.IO y MySQL

---

# 1. Introducción

En las prácticas anteriores se desarrolló un sistema de Industria 4.0 capaz de:

* Detectar piezas sobre una banda transportadora.
* Medir la altura de cada pieza.
* Medir el peso individual de las piezas.
* Identificar mediante RFID la caja receptora.
* Guardar las mediciones en MySQL.
* Mostrar resultados y estadísticas desde páginas web.

En esta práctica se agregará una estación para inspeccionar el color de las piezas.

El sensor TCS34725 se colocará cerca del sensor infrarrojo. Cuando el infrarrojo detecte una pieza, el ESP32 tomará varias lecturas de color, calculará un promedio y enviará al servidor los valores de los canales:

```text
Rojo
Verde
Azul
Claro
```

El TCS34725 proporciona canales filtrados rojo, verde y azul, además de un canal claro sin filtro de color. La biblioteca utilizada permite leer directamente estos cuatro valores mediante `getRawData()`.

El servidor:

* Normalizará los valores RGB.
* Comparará la medición con una referencia previamente calibrada.
* Determinará si el color es correcto.
* Guardará la información en MySQL.
* Actualizará las páginas mediante Socket.IO.
* Generará estadísticas del proceso.

---

# 2. Objetivo general

Desarrollar una estación de inspección automática que permita medir el color de las piezas que pasan por una banda transportadora, compararlo con una referencia y almacenar los resultados en una base de datos.

---

# 3. Objetivos específicos

Al finalizar la práctica, el estudiante será capaz de:

* Conectar un sensor TCS34725 al ESP32.
* Utilizar comunicación I²C.
* Leer los canales rojo, verde, azul y claro.
* Utilizar el sensor infrarrojo como disparador de la medición.
* Tomar varias muestras por pieza.
* Obtener el promedio de las lecturas.
* Normalizar los canales RGB.
* Calibrar el color esperado de un producto.
* Calcular la diferencia entre un color medido y uno de referencia.
* Clasificar una pieza como correcta o incorrecta.
* Enviar las lecturas mediante HTTP.
* Guardar mediciones de color en MySQL.
* Actualizar una página mediante Socket.IO.
* Generar estadísticas de color.

---

# 4. Funcionamiento general

```text
La pieza avanza sobre la banda
                │
                ▼
El sensor infrarrojo la detecta
                │
                ▼
El ESP32 espera que la pieza quede
debajo del TCS34725
                │
                ▼
El sensor toma varias muestras
                │
                ▼
Se obtiene el promedio RGB y claro
                │
                ▼
El ESP32 envía los valores a Node.js
                │
                ▼
Node.js normaliza los canales
                │
                ▼
Compara con el color de referencia
                │
        ┌───────┴─────────┐
        ▼                 ▼
 Color correcto     Color incorrecto
        │                 │
        └───────┬─────────┘
                ▼
       Guarda en MySQL
                │
                ▼
Actualiza monitoreo y estadísticas
```

---

# 5. Resultado esperado

Cuando pase una pieza, el monitor serial mostrará información parecida a:

```text
Objeto detectado

R: 2150
G: 1320
B: 890
C: 4580

R normalizado: 49.31 %
G normalizado: 30.28 %
B normalizado: 20.41 %

Enviando medición...
Código HTTP: 200
Resultado: correcto
```

En la página web deberá mostrarse:

```text
Producto: Pieza roja
Color esperado: rojo
Color detectado: rojo
Diferencia: 4.28
Resultado: correcto
```

---

# 6. Material necesario

## Hardware

* Una tarjeta ESP32.
* Un sensor de color TCS34725.
* Un sensor infrarrojo para detectar las piezas.
* Banda transportadora.
* Protoboard.
* Cables Dupont.
* Soporte para colocar el sensor.
* Cubierta o túnel para controlar la iluminación.
* Piezas de diferentes colores.
* Piezas del mismo color para calibración.

## Software

* Arduino IDE.
* Node.js.
* MySQL Server.
* Visual Studio Code.
* Navegador web.
* Proyecto desarrollado en las prácticas anteriores.

---

# 7. Funcionamiento del TCS34725

El TCS34725 mide la luz que llega al sensor y proporciona cuatro canales:

| Canal | Descripción                           |
| ----- | ------------------------------------- |
| `R`   | Componente roja                       |
| `G`   | Componente verde                      |
| `B`   | Componente azul                       |
| `C`   | Cantidad general de luz o canal claro |

La comunicación con el ESP32 se realiza mediante I²C. El circuito integrado trabaja como dispositivo esclavo I²C y su dirección predeterminada es `0x29`.

La biblioteca Adafruit permite:

* Inicializar el sensor con `begin()`.
* Cambiar el tiempo de integración.
* Cambiar la ganancia.
* Leer los valores crudos con `getRawData()`.
* Obtener valores RGB normalizados con `getRGB()`.

En esta práctica se utilizarán los valores crudos para que los alumnos realicen explícitamente el procesamiento.

---

# 8. Importancia de la iluminación

El sensor no genera el color. Mide la luz reflejada por el objeto.

Por lo tanto, una misma pieza puede generar valores diferentes si cambia:

* La iluminación del salón.
* La distancia al sensor.
* La inclinación del objeto.
* La presencia de sombras.
* El brillo de la superficie.
* La luz del Sol.
* La posición de la pieza.

Si el módulo incluye un LED blanco, puede utilizarse para iluminar la superficie. Este tipo de lectura se basa en iluminar el objeto y medir la luz que refleja.

Para obtener resultados consistentes se recomienda construir un pequeño túnel:

```text
Vista lateral

             TCS34725
          ┌────────────┐
          │ Sensor/LED │
          └──────┬─────┘
                 │
        ┌────────┴────────┐
        │ Cubierta oscura │
        │                 │
        │      Pieza      │
════════┴─────────────────┴════════
        Banda transportadora
```

La cubierta deberá:

* Bloquear parte de la luz exterior.
* Mantener fija la distancia.
* Evitar sombras producidas por personas.
* Permitir el paso libre de la pieza.
* Evitar que el sensor toque el objeto.

---

# 9. Colocación de los sensores

El sensor infrarrojo deberá detectar la llegada de la pieza.

El TCS34725 deberá colocarse de manera que mida una zona representativa de su superficie.

```text
Vista superior

Movimiento de la banda ─────────────►

       Sensor IR          TCS34725
           │                 │
           ▼                 ▼
═════════════════════════════════════
             Pieza
        ┌───────────┐
        │           │
        └───────────┘
═════════════════════════════════════
```

Si el infrarrojo se encuentra antes del sensor de color, se utilizará un pequeño tiempo de espera:

```text
Sensor IR detecta
       │
       ▼
Esperar el desplazamiento
       │
       ▼
Medir el color
```

El tiempo dependerá de:

* La velocidad de la banda.
* La separación entre sensores.
* La longitud de la pieza.
* La posición del área que se desea medir.

---

# 10. Conexión del TCS34725 al ESP32

## Tabla de conexión

| TCS34725  | ESP32              |
| --------- | ------------------ |
| VCC o VIN | 3.3 V              |
| GND       | GND                |
| SDA       | GPIO 21            |
| SCL       | GPIO 22            |
| LED       | Depende del módulo |
| INT       | Sin conexión       |

En el ESP32 genérico, los pines predeterminados para I²C son GPIO 21 para SDA y GPIO 22 para SCL. También pueden configurarse explícitamente desde `Wire`.

El circuito integrado TCS34725 trabaja con alimentación de bajo voltaje; conectar el módulo a 3.3 V es la opción segura para el ESP32. Algunos módulos comerciales incluyen regulador y pueden aceptar otros voltajes, pero esto debe verificarse específicamente en su tarjeta.

---

# 11. Conexión del sensor infrarrojo

Se conservará la conexión de la práctica de altura:

| Sensor infrarrojo | ESP32   |
| ----------------- | ------- |
| VCC               | 3.3 V   |
| GND               | GND     |
| OUT               | GPIO 19 |

La mayoría de los módulos infrarrojos entregan:

```text
LOW  → objeto detectado
HIGH → zona libre
```

Si el módulo utilizado funciona al contrario, deberá modificarse:

```cpp
const int ESTADO_OBJETO = LOW;
```

por:

```cpp
const int ESTADO_OBJETO = HIGH;
```

---

# 12. Conexión completa

| Dispositivo | Pin |   ESP32 |
| ----------- | --- | ------: |
| TCS34725    | SDA | GPIO 21 |
| TCS34725    | SCL | GPIO 22 |
| TCS34725    | VCC |   3.3 V |
| TCS34725    | GND |     GND |
| Sensor IR   | OUT | GPIO 19 |
| Sensor IR   | VCC |   3.3 V |
| Sensor IR   | GND |     GND |

El HC-SR04 puede permanecer instalado, pero no se utilizará en el programa de esta práctica. Esto permite estudiar el sensor de color por separado antes de realizar la integración completa.

---

# Bloque 1. Instalar la biblioteca

## 13. Instalar Adafruit TCS34725

Desde Arduino IDE:

1. Abrir el administrador de bibliotecas.
2. Buscar:

```text
Adafruit TCS34725
```

3. Instalar:

```text
Adafruit TCS34725
```

4. Instalar las dependencias solicitadas por Arduino IDE.

La guía oficial indica que la biblioteca puede instalarse desde el administrador de bibliotecas buscando `Adafruit TCS34725`.

El programa utilizará:

```cpp
#include <Wire.h>
#include <Adafruit_TCS34725.h>
```

---

# Bloque 2. Probar solamente el sensor

## 14. Código básico de prueba

Antes de utilizar la banda, el infrarrojo o el servidor, probar solamente el TCS34725.

```cpp
#include <Wire.h>
#include <Adafruit_TCS34725.h>

const int PIN_SDA = 21;
const int PIN_SCL = 22;

Adafruit_TCS34725 sensorColor(
    TCS34725_INTEGRATIONTIME_50MS,
    TCS34725_GAIN_4X
);

void setup()
{
    Serial.begin(115200);

    Wire.begin(
        PIN_SDA,
        PIN_SCL
    );

    Serial.println();
    Serial.println(
        "Iniciando TCS34725..."
    );

    if(!sensorColor.begin())
    {
        Serial.println(
            "No se encontró el TCS34725."
        );

        Serial.println(
            "Revise la alimentación y el cableado."
        );

        while(true)
        {
            delay(1000);
        }
    }

    Serial.println(
        "Sensor encontrado correctamente."
    );
}

void loop()
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

    uint32_t suma =
        rojo +
        verde +
        azul;

    float rojoPct = 0;
    float verdePct = 0;
    float azulPct = 0;

    if(suma > 0)
    {
        rojoPct =
            rojo * 100.0 / suma;

        verdePct =
            verde * 100.0 / suma;

        azulPct =
            azul * 100.0 / suma;
    }

    Serial.print("R: ");
    Serial.print(rojo);

    Serial.print(" | G: ");
    Serial.print(verde);

    Serial.print(" | B: ");
    Serial.print(azul);

    Serial.print(" | C: ");
    Serial.println(claro);

    Serial.print("R%: ");
    Serial.print(rojoPct, 2);

    Serial.print(" | G%: ");
    Serial.print(verdePct, 2);

    Serial.print(" | B%: ");
    Serial.println(azulPct, 2);

    Serial.println();

    delay(500);
}
```

La configuración utiliza un tiempo de integración de 50 ms y una ganancia de 4×. La biblioteca permite modificar ambos parámetros según la cantidad de luz disponible.

---

# 15. Realizar pruebas iniciales

Colocar frente al sensor objetos:

* Rojos.
* Verdes.
* Azules.
* Amarillos.
* Blancos.
* Negros.

Completar:

| Objeto   |  R |  G |  B |  C | R % | G % | B % |
| -------- | -: | -: | -: | -: | --: | --: | --: |
| Rojo     |    |    |    |    |     |     |     |
| Verde    |    |    |    |    |     |     |     |
| Azul     |    |    |    |    |     |     |     |
| Amarillo |    |    |    |    |     |     |     |
| Blanco   |    |    |    |    |     |     |     |
| Negro    |    |    |    |    |     |     |     |

No continuar hasta que los valores cambien de manera coherente al cambiar el objeto.

---

# 16. Interpretación de los porcentajes

Los porcentajes se calculan mediante:

```text
Suma = R + G + B
```

```text
R normalizado = R × 100 / Suma
G normalizado = G × 100 / Suma
B normalizado = B × 100 / Suma
```

Ejemplo:

```text
R = 2400
G = 1500
B = 900

Suma = 4800
```

```text
R = 50.00 %
G = 31.25 %
B = 18.75 %
```

La normalización reduce parcialmente el efecto de cambios generales de intensidad.

Sin embargo, no elimina completamente los efectos de:

* Distancia.
* Luz ambiental.
* Sombras.
* Brillo.
* Saturación del sensor.

---

# Bloque 3. Calibrar un color de referencia

## 17. Importancia de la calibración

No se recomienda definir que un objeto es rojo únicamente porque:

```text
R > G y R > B
```

Esa regla puede reconocer el canal dominante, pero no determina si el color corresponde exactamente al producto esperado.

En esta práctica cada producto tendrá una referencia formada por:

```text
R de referencia
G de referencia
B de referencia
Tolerancia
```

---

# 18. Código para obtener una referencia

Este programa toma diez muestras y calcula un promedio.

```cpp
#include <Wire.h>
#include <Adafruit_TCS34725.h>

const int PIN_SDA = 21;
const int PIN_SCL = 22;

const int TOTAL_MUESTRAS = 10;

Adafruit_TCS34725 sensorColor(
    TCS34725_INTEGRATIONTIME_50MS,
    TCS34725_GAIN_4X
);

void esperarEnter()
{
    while(!Serial.available())
    {
        delay(10);
    }

    while(Serial.available())
    {
        Serial.read();
    }
}

void setup()
{
    Serial.begin(115200);

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

    Serial.println();
    Serial.println(
        "CALIBRACIÓN DE COLOR"
    );

    Serial.println(
        "Coloque la muestra debajo del sensor."
    );

    Serial.println(
        "Presione Enter para comenzar."
    );

    esperarEnter();

    uint64_t sumaRojo = 0;
    uint64_t sumaVerde = 0;
    uint64_t sumaAzul = 0;
    uint64_t sumaClaro = 0;

    for(int i = 0; i < TOTAL_MUESTRAS; i++)
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

        sumaRojo += rojo;
        sumaVerde += verde;
        sumaAzul += azul;
        sumaClaro += claro;

        Serial.print(
            "Muestra "
        );

        Serial.print(i + 1);

        Serial.print(": ");

        Serial.print(rojo);
        Serial.print(", ");

        Serial.print(verde);
        Serial.print(", ");

        Serial.print(azul);
        Serial.print(", ");

        Serial.println(claro);

        delay(100);
    }

    float rojoPromedio =
        sumaRojo /
        (float)TOTAL_MUESTRAS;

    float verdePromedio =
        sumaVerde /
        (float)TOTAL_MUESTRAS;

    float azulPromedio =
        sumaAzul /
        (float)TOTAL_MUESTRAS;

    float claroPromedio =
        sumaClaro /
        (float)TOTAL_MUESTRAS;

    float sumaRGB =
        rojoPromedio +
        verdePromedio +
        azulPromedio;

    float rojoPct =
        rojoPromedio *
        100.0 /
        sumaRGB;

    float verdePct =
        verdePromedio *
        100.0 /
        sumaRGB;

    float azulPct =
        azulPromedio *
        100.0 /
        sumaRGB;

    Serial.println();
    Serial.println(
        "RESULTADO DE CALIBRACIÓN"
    );

    Serial.print(
        "R promedio: "
    );
    Serial.println(rojoPromedio, 2);

    Serial.print(
        "G promedio: "
    );
    Serial.println(verdePromedio, 2);

    Serial.print(
        "B promedio: "
    );
    Serial.println(azulPromedio, 2);

    Serial.print(
        "C promedio: "
    );
    Serial.println(claroPromedio, 2);

    Serial.println();

    Serial.print(
        "R normalizado: "
    );
    Serial.print(rojoPct, 2);
    Serial.println(" %");

    Serial.print(
        "G normalizado: "
    );
    Serial.print(verdePct, 2);
    Serial.println(" %");

    Serial.print(
        "B normalizado: "
    );
    Serial.print(azulPct, 2);
    Serial.println(" %");
}

void loop()
{
}
```

---

# 19. Obtener varias referencias

No se debe calibrar con una sola pieza.

Utilizar al menos cinco piezas correctas del mismo producto:

| Pieza    | R % | G % | B % |
| -------- | --: | --: | --: |
| 1        |     |     |     |
| 2        |     |     |     |
| 3        |     |     |     |
| 4        |     |     |     |
| 5        |     |     |     |
| Promedio |     |     |     |

El promedio final será la referencia del producto.

Ejemplo:

```text
Color esperado: rojo

R referencia: 48.50 %
G referencia: 31.20 %
B referencia: 20.30 %
```

---

# 20. Diferencia entre colores

El servidor calculará una distancia entre el color medido y el color de referencia:

```text
Diferencia =
√[
  (Rmedido − Rreferencia)² +
  (Gmedido − Greferencia)² +
  (Bmedido − Breferencia)²
]
```

Ejemplo:

```text
Referencia:
R = 48 %
G = 31 %
B = 21 %

Medición:
R = 45 %
G = 34 %
B = 21 %
```

```text
Diferencia =
√[(45−48)² + (34−31)² + (21−21)²]

Diferencia =
√[9 + 9 + 0]

Diferencia = 4.24
```

Si la tolerancia establecida es:

```text
10
```

la pieza se considerará correcta.

---

# Bloque 4. Preparar la base de datos

## 21. Entrar a MySQL

```bash
mysql -u root -p
```

Seleccionar la base:

```sql
USE industria40_web;
```

---

# 22. Agregar la referencia de color a los productos

Ejecutar una sola vez:

```sql
ALTER TABLE productos
ADD color_esperado VARCHAR(30) NULL,

ADD color_ref_r_pct DECIMAL(6,2) NULL,

ADD color_ref_g_pct DECIMAL(6,2) NULL,

ADD color_ref_b_pct DECIMAL(6,2) NULL,

ADD tolerancia_color DECIMAL(6,2) NULL
    DEFAULT 10.00;
```

Los campos representan:

| Campo              | Descripción                    |
| ------------------ | ------------------------------ |
| `color_esperado`   | Nombre del color               |
| `color_ref_r_pct`  | Porcentaje rojo de referencia  |
| `color_ref_g_pct`  | Porcentaje verde de referencia |
| `color_ref_b_pct`  | Porcentaje azul de referencia  |
| `tolerancia_color` | Diferencia máxima aceptada     |

---

# 23. Configurar un producto

Consultar:

```sql
SELECT
    id,
    nombre,
    color_esperado,
    color_ref_r_pct,
    color_ref_g_pct,
    color_ref_b_pct,
    tolerancia_color
FROM productos;
```

Ejemplo para el producto con ID 1:

```sql
UPDATE productos
SET
    color_esperado = 'rojo',
    color_ref_r_pct = 48.50,
    color_ref_g_pct = 31.20,
    color_ref_b_pct = 20.30,
    tolerancia_color = 10.00
WHERE id = 1;
```

Los valores deberán sustituirse por los obtenidos durante la calibración.

---

# 24. Crear la tabla de mediciones

```sql
CREATE TABLE mediciones_color (
    id INT AUTO_INCREMENT PRIMARY KEY,

    producto_id INT NOT NULL,

    dispositivo VARCHAR(50) NOT NULL,

    rojo_raw INT UNSIGNED NOT NULL,

    verde_raw INT UNSIGNED NOT NULL,

    azul_raw INT UNSIGNED NOT NULL,

    claro_raw INT UNSIGNED NOT NULL,

    rojo_pct DECIMAL(6,2) NOT NULL,

    verde_pct DECIMAL(6,2) NOT NULL,

    azul_pct DECIMAL(6,2) NOT NULL,

    color_detectado VARCHAR(30) NOT NULL,

    diferencia_color DECIMAL(8,2) NULL,

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
color_incorrecto
sin_especificacion
```

---

# 25. Verificar

```sql
DESCRIBE productos;
```

```sql
DESCRIBE mediciones_color;
```

Salir:

```sql
EXIT;
```

---

# Bloque 5. Modificar el servidor Node.js

La práctica reutiliza:

* Express.
* MySQL.
* Sesiones.
* Socket.IO.
* `requiereSesionPagina`.
* `requiereSesionAPI`.
* `requiereClaveDispositivo`.
* La conexión `conexion`.
* La variable `io`.

---

# 26. Crear funciones para procesar el color

Agregar en `server.js`, junto con las demás funciones auxiliares:

```javascript
function redondear2(valor)
{
    return Number(
        Number(valor).toFixed(2)
    );
}

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

    // Los tres canales son parecidos.
    if(maximo - minimo <= 6)
    {
        return "neutro";
    }

    // Mezcla con rojo y verde elevados.
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

La clasificación básica solo proporciona un nombre aproximado para mostrar en la página.

La evaluación de conformidad utilizará la referencia calibrada del producto.

---

# 27. Crear las rutas de las páginas

Agregar:

```javascript
app.get(
    "/mediciones-color",
    requiereSesionPagina,
    (req, res) =>
    {
        enviarPagina(
            res,
            "mediciones-color.html"
        );
    }
);

app.get(
    "/estadisticas-color",
    requiereSesionPagina,
    (req, res) =>
    {
        enviarPagina(
            res,
            "estadisticas-color.html"
        );
    }
);
```

---

# 28. API para recibir una medición

Agregar antes del inicio del servidor:

```javascript
// ==================================================
// API: RECIBIR MEDICIÓN DE COLOR
// ==================================================

app.post(
    "/api/mediciones-color",
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
                        "Los canales de color son inválidos"
                });
            }

            const suma =
                rojoRaw +
                verdeRaw +
                azulRaw;

            if(suma <= 0)
            {
                return res.status(400).json({
                    correcto: false,
                    mensaje:
                        "No existe suficiente información de color"
                });
            }

            const rojoPct =
                redondear2(
                    rojoRaw *
                    100 /
                    suma
                );

            const verdePct =
                redondear2(
                    verdeRaw *
                    100 /
                    suma
                );

            const azulPct =
                redondear2(
                    azulRaw *
                    100 /
                    suma
                );

            const colorDetectado =
                clasificarColorBasico(
                    rojoPct,
                    verdePct,
                    azulPct
                );

            const [productos] =
                await conexion.execute(
                    `SELECT
                        id,
                        nombre,
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

            let diferenciaColor = null;

            let resultado =
                "sin_especificacion";

            const tieneReferencia =
                producto.color_ref_r_pct !== null &&
                producto.color_ref_g_pct !== null &&
                producto.color_ref_b_pct !== null &&
                producto.tolerancia_color !== null;

            if(tieneReferencia)
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
                    resultado =
                        "correcto";
                }
                else
                {
                    resultado =
                        "color_incorrecto";
                }
            }

            const [insercion] =
                await conexion.execute(
                    `INSERT INTO mediciones_color
                    (
                        producto_id,
                        dispositivo,
                        rojo_raw,
                        verde_raw,
                        azul_raw,
                        claro_raw,
                        rojo_pct,
                        verde_pct,
                        azul_pct,
                        color_detectado,
                        diferencia_color,
                        resultado
                    )
                    VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)`,
                    [
                        productoId,
                        dispositivo,
                        rojoRaw,
                        verdeRaw,
                        azulRaw,
                        claroRaw,
                        rojoPct,
                        verdePct,
                        azulPct,
                        colorDetectado,
                        diferenciaColor,
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

                color_esperado:
                    producto.color_esperado,

                dispositivo:
                    dispositivo,

                rojo_raw:
                    rojoRaw,

                verde_raw:
                    verdeRaw,

                azul_raw:
                    azulRaw,

                claro_raw:
                    claroRaw,

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

                resultado:
                    resultado,

                fecha:
                    new Date().toISOString()
            };

            io.emit(
                "nueva_medicion_color",
                medicion
            );

            res.json({
                correcto: true,

                mensaje:
                    "Medición de color registrada",

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

El servidor calcula nuevamente los porcentajes. De esta manera no depende de porcentajes calculados por el ESP32.

---

# 29. API para consultar el historial

Agregar:

```javascript
// ==================================================
// API: CONSULTAR MEDICIONES DE COLOR
// ==================================================

app.get(
    "/api/mediciones-color",
    requiereSesionAPI,
    async (req, res) =>
    {
        try
        {
            const [mediciones] =
                await conexion.execute(
                    `SELECT
                        mediciones_color.id,
                        mediciones_color.dispositivo,
                        mediciones_color.rojo_raw,
                        mediciones_color.verde_raw,
                        mediciones_color.azul_raw,
                        mediciones_color.claro_raw,
                        mediciones_color.rojo_pct,
                        mediciones_color.verde_pct,
                        mediciones_color.azul_pct,
                        mediciones_color.color_detectado,
                        mediciones_color.diferencia_color,
                        mediciones_color.resultado,
                        mediciones_color.fecha,

                        productos.id AS producto_id,
                        productos.nombre AS producto,
                        productos.color_esperado,
                        productos.tolerancia_color

                    FROM mediciones_color

                    INNER JOIN productos
                        ON mediciones_color.producto_id =
                           productos.id

                    ORDER BY
                        mediciones_color.fecha DESC

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

# 30. API para consultar estadísticas

Agregar:

```javascript
// ==================================================
// API: ESTADÍSTICAS DE COLOR
// ==================================================

app.get(
    "/api/estadisticas-color",
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
                            AVG(rojo_pct),
                            2
                        ) AS rojo_promedio,

                        ROUND(
                            AVG(verde_pct),
                            2
                        ) AS verde_promedio,

                        ROUND(
                            AVG(azul_pct),
                            2
                        ) AS azul_promedio,

                        ROUND(
                            AVG(diferencia_color),
                            2
                        ) AS diferencia_promedio,

                        SUM(
                            resultado = 'correcto'
                        ) AS correctas,

                        SUM(
                            resultado =
                            'color_incorrecto'
                        ) AS incorrectas,

                        SUM(
                            resultado =
                            'sin_especificacion'
                        ) AS sin_especificacion

                    FROM mediciones_color

                    ${filtro}`,
                    parametros
                );

            const filtroSerie =
                productoId
                ? `WHERE
                    mediciones_color.producto_id = ?`
                : "";

            const [serie] =
                await conexion.execute(
                    `SELECT
                        mediciones_color.id,
                        mediciones_color.rojo_pct,
                        mediciones_color.verde_pct,
                        mediciones_color.azul_pct,
                        mediciones_color.color_detectado,
                        mediciones_color.diferencia_color,
                        mediciones_color.resultado,
                        mediciones_color.fecha,

                        productos.nombre AS producto

                    FROM mediciones_color

                    INNER JOIN productos
                        ON mediciones_color.producto_id =
                           productos.id

                    ${filtroSerie}

                    ORDER BY
                        mediciones_color.fecha DESC

                    LIMIT 30`,
                    parametros
                );

            const filtroDistribucion =
                productoId
                ? "WHERE producto_id = ?"
                : "";

            const [distribucion] =
                await conexion.execute(
                    `SELECT
                        color_detectado,
                        COUNT(*) AS cantidad

                    FROM mediciones_color

                    ${filtroDistribucion}

                    GROUP BY color_detectado

                    ORDER BY cantidad DESC`,
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

# Bloque 6. Crear la página de monitoreo

## 31. Crear `paginas/mediciones-color.html`

```html
<!DOCTYPE html>
<html lang="es">

<head>
    <meta charset="UTF-8">

    <meta
        name="viewport"
        content="width=device-width, initial-scale=1.0"
    >

    <title>Mediciones de color</title>

    <link
        rel="stylesheet"
        href="/css/estilos.css"
    >
</head>

<body>

    <header>

        <div>
            <h1>Inspección de color</h1>

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
        <a href="/cajas">Cajas RFID</a>
        <a href="/mediciones-color">Colores</a>
        <a href="/estadisticas-color">Estadísticas de color</a>
    </nav>

    <main>

        <section class="tarjetas">

            <article class="tarjeta">
                <h3>Color detectado</h3>

                <p
                    id="ultimoColor"
                    class="numero"
                >
                    --
                </p>
            </article>

            <article class="tarjeta">
                <h3>Color esperado</h3>

                <p id="colorEsperado">
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
                <h3>Diferencia</h3>

                <p
                    id="ultimaDiferencia"
                    class="numero"
                >
                    --
                </p>
            </article>

            <article class="tarjeta">
                <h3>Producto</h3>

                <p id="ultimoProducto">
                    Sin mediciones
                </p>
            </article>

            <article class="tarjeta">
                <h3>Vista aproximada</h3>

                <div
                    id="muestraColor"
                    class="muestra-color"
                ></div>
            </article>

        </section>

        <section class="seccion">

            <h2>
                Últimas mediciones
            </h2>

            <div class="tabla-contenedor">

                <table>

                    <thead>
                        <tr>
                            <th>ID</th>
                            <th>Producto</th>
                            <th>R %</th>
                            <th>G %</th>
                            <th>B %</th>
                            <th>Canal claro</th>
                            <th>Color</th>
                            <th>Diferencia</th>
                            <th>Resultado</th>
                            <th>Fecha</th>
                        </tr>
                    </thead>

                    <tbody
                        id="tablaColores"
                    ></tbody>

                </table>

            </div>

        </section>

    </main>

    <script src="/socket.io/socket.io.js"></script>
    <script src="/js/comun.js"></script>
    <script src="/js/mediciones-color.js"></script>

</body>
</html>
```

---

# 32. Crear `public/js/mediciones-color.js`

```javascript
const socket = io();

const tabla =
    document.getElementById(
        "tablaColores"
    );

function textoResultado(resultado)
{
    if(resultado === "correcto")
    {
        return "Correcto";
    }

    if(resultado === "color_incorrecto")
    {
        return "Color incorrecto";
    }

    return "Sin especificación";
}

function obtenerColorVisual(
    rojo,
    verde,
    azul
)
{
    const maximo =
        Math.max(
            rojo,
            verde,
            azul,
            1
        );

    const r =
        Math.round(
            rojo /
            maximo *
            255
        );

    const g =
        Math.round(
            verde /
            maximo *
            255
        );

    const b =
        Math.round(
            azul /
            maximo *
            255
        );

    return `rgb(${r}, ${g}, ${b})`;
}

function actualizarUltimaMedicion(
    medicion
)
{
    document.getElementById(
        "ultimoColor"
    ).textContent =
        medicion.color_detectado;

    document.getElementById(
        "colorEsperado"
    ).textContent =
        medicion.color_esperado ||
        "Sin especificación";

    document.getElementById(
        "ultimoResultado"
    ).textContent =
        textoResultado(
            medicion.resultado
        );

    document.getElementById(
        "ultimaDiferencia"
    ).textContent =
        medicion.diferencia_color === null
        ? "--"
        : medicion.diferencia_color;

    document.getElementById(
        "ultimoProducto"
    ).textContent =
        medicion.producto;

    document.getElementById(
        "muestraColor"
    ).style.background =
        obtenerColorVisual(
            Number(
                medicion.rojo_pct
            ),
            Number(
                medicion.verde_pct
            ),
            Number(
                medicion.azul_pct
            )
        );
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
                ${medicion.rojo_pct} %
            </td>

            <td>
                ${medicion.verde_pct} %
            </td>

            <td>
                ${medicion.azul_pct} %
            </td>

            <td>
                ${medicion.claro_raw}
            </td>

            <td>
                ${medicion.color_detectado}
            </td>

            <td>
                ${
                    medicion.diferencia_color === null
                    ? "--"
                    : medicion.diferencia_color
                }
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
        "/api/mediciones-color"
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
    "nueva_medicion_color",
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

# 33. Agregar estilos

Al final de `public/css/estilos.css` agregar:

```css
.muestra-color {
    width: 100%;
    height: 70px;
    border: 2px solid #94a3b8;
    border-radius: 8px;
    background: #e2e8f0;
}
```

La vista es solamente una representación aproximada de las proporciones RGB. No sustituye la medición del sensor.

---

# Bloque 7. Crear la página de estadísticas

## 34. Crear `paginas/estadisticas-color.html`

```html
<!DOCTYPE html>
<html lang="es">

<head>
    <meta charset="UTF-8">

    <meta
        name="viewport"
        content="width=device-width, initial-scale=1.0"
    >

    <title>Estadísticas de color</title>

    <link
        rel="stylesheet"
        href="/css/estilos.css"
    >
</head>

<body>

    <header>

        <div>
            <h1>Estadísticas de color</h1>

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
        <a href="/mediciones-peso">Pesos</a>
        <a href="/cajas">Cajas RFID</a>
        <a href="/mediciones-color">Colores</a>
        <a href="/estadisticas-color">Estadísticas de color</a>
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
                <h3>R promedio</h3>
                <p id="rojoPromedio" class="numero">0 %</p>
            </article>

            <article class="tarjeta">
                <h3>G promedio</h3>
                <p id="verdePromedio" class="numero">0 %</p>
            </article>

            <article class="tarjeta">
                <h3>B promedio</h3>
                <p id="azulPromedio" class="numero">0 %</p>
            </article>

            <article class="tarjeta">
                <h3>Diferencia promedio</h3>
                <p id="diferenciaPromedio" class="numero">0</p>
            </article>

            <article class="tarjeta">
                <h3>Piezas correctas</h3>
                <p id="correctas" class="numero">0</p>
            </article>

            <article class="tarjeta">
                <h3>Piezas incorrectas</h3>
                <p id="incorrectas" class="numero">0</p>
            </article>

            <article class="tarjeta">
                <h3>Cumplimiento</h3>
                <p id="cumplimiento" class="numero">0 %</p>
            </article>

        </section>

        <section class="seccion">

            <h2>
                Componentes RGB normalizados
            </h2>

            <div class="contenedor-grafica">
                <canvas id="graficaRgb"></canvas>
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

        <section class="seccion">

            <h2>
                Colores detectados
            </h2>

            <div class="contenedor-grafica">
                <canvas id="graficaDistribucion"></canvas>
            </div>

        </section>

    </main>

    <script
        src="https://cdn.jsdelivr.net/npm/chart.js"
    ></script>

    <script src="/socket.io/socket.io.js"></script>
    <script src="/js/comun.js"></script>
    <script src="/js/estadisticas-color.js"></script>

</body>
</html>
```

---

# 35. Crear `public/js/estadisticas-color.js`

```javascript
const socket = io();

const selectorProducto =
    document.getElementById(
        "filtroProducto"
    );

let graficaRgb = null;
let graficaResultados = null;
let graficaDistribucion = null;

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
        "/api/estadisticas-color";

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

    const incorrectas =
        numero(resumen.incorrectas);

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
        "rojoPromedio"
    ).textContent =
        `${numero(
            resumen.rojo_promedio
        )} %`;

    document.getElementById(
        "verdePromedio"
    ).textContent =
        `${numero(
            resumen.verde_promedio
        )} %`;

    document.getElementById(
        "azulPromedio"
    ).textContent =
        `${numero(
            resumen.azul_promedio
        )} %`;

    document.getElementById(
        "diferenciaPromedio"
    ).textContent =
        numero(
            resumen.diferencia_promedio
        );

    document.getElementById(
        "correctas"
    ).textContent =
        correctas;

    document.getElementById(
        "incorrectas"
    ).textContent =
        incorrectas;

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

    if(graficaRgb)
    {
        graficaRgb.destroy();
    }

    graficaRgb =
        new Chart(
            document.getElementById(
                "graficaRgb"
            ),
            {
                type: "line",

                data: {
                    labels:
                        etiquetas,

                    datasets: [
                        {
                            label:
                                "Rojo %",

                            data:
                                serie.map(
                                    medicion =>
                                        Number(
                                            medicion.rojo_pct
                                        )
                                )
                        },

                        {
                            label:
                                "Verde %",

                            data:
                                serie.map(
                                    medicion =>
                                        Number(
                                            medicion.verde_pct
                                        )
                                )
                        },

                        {
                            label:
                                "Azul %",

                            data:
                                serie.map(
                                    medicion =>
                                        Number(
                                            medicion.azul_pct
                                        )
                                )
                        }
                    ]
                },

                options: {
                    responsive: true,

                    scales: {
                        y: {
                            beginAtZero: true,
                            max: 100
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
                        "Incorrectas",
                        "Sin especificación"
                    ],

                    datasets: [
                        {
                            data: [
                                correctas,
                                incorrectas,
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

    if(graficaDistribucion)
    {
        graficaDistribucion.destroy();
    }

    graficaDistribucion =
        new Chart(
            document.getElementById(
                "graficaDistribucion"
            ),
            {
                type: "bar",

                data: {
                    labels:
                        datos.distribucion.map(
                            elemento =>
                                elemento.color_detectado
                        ),

                    datasets: [
                        {
                            label:
                                "Piezas",

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
}

selectorProducto.addEventListener(
    "change",
    cargarEstadisticas
);

socket.on(
    "nueva_medicion_color",
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

# 36. Agregar enlaces a los menús

En las páginas privadas agregar:

```html
<a href="/mediciones-color">
    Colores
</a>

<a href="/estadisticas-color">
    Estadísticas de color
</a>
```

---

# Bloque 8. Programar el ESP32

## 37. Funcionamiento del programa

El programa:

1. Esperará la detección del infrarrojo.
2. Esperará que la pieza llegue al TCS34725.
3. Obtendrá ocho muestras.
4. Calculará el promedio.
5. Mostrará los porcentajes.
6. Enviará los valores crudos al servidor.
7. Bloqueará la detección hasta que la pieza salga.

---

# 38. Código completo del ESP32

```cpp
#include <WiFi.h>
#include <HTTPClient.h>

#include <Wire.h>
#include <Adafruit_TCS34725.h>

// ==================================================
// CONFIGURACIÓN DE RED
// ==================================================

const char* NOMBRE_WIFI =
    "NOMBRE_DE_LA_RED";

const char* CONTRASENA_WIFI =
    "CONTRASENA_DE_LA_RED";

const char* URL_SERVIDOR =
    "http://192.168.1.25:3000/api/mediciones-color";

const char* CLAVE_DISPOSITIVO =
    "clave-banda-industria-40";

// ==================================================
// IDENTIFICACIÓN
// ==================================================

const int PRODUCTO_ID = 1;

const char* NOMBRE_DISPOSITIVO =
    "color_01";

// ==================================================
// PINES
// ==================================================

const int PIN_SENSOR_IR = 19;

const int PIN_SDA = 21;
const int PIN_SCL = 22;

// Cambiar a HIGH si el sensor infrarrojo
// utilizado funciona de manera inversa.
const int ESTADO_OBJETO = LOW;

// ==================================================
// SENSOR DE COLOR
// ==================================================

Adafruit_TCS34725 sensorColor(
    TCS34725_INTEGRATIONTIME_50MS,
    TCS34725_GAIN_4X
);

// ==================================================
// CONFIGURACIÓN DE MEDICIÓN
// ==================================================

const int TOTAL_MUESTRAS = 8;

// Ajustar de acuerdo con la distancia entre
// el infrarrojo y el sensor de color.
const unsigned long RETARDO_CENTRADO_MS =
    80;

// Tiempo durante el cual la zona debe estar
// libre antes de aceptar otra pieza.
const unsigned long TIEMPO_LIBRE_MS =
    150;

// ==================================================
// VARIABLES
// ==================================================

bool objetoProcesado = false;

unsigned long inicioZonaLibre = 0;

// ==================================================
// ESTRUCTURA PARA LA MEDICIÓN
// ==================================================

struct LecturaColor
{
    uint32_t rojo;
    uint32_t verde;
    uint32_t azul;
    uint32_t claro;
};

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
// OBTENER PROMEDIO
// ==================================================

bool obtenerColorPromedio(
    LecturaColor& resultado
)
{
    uint64_t sumaRojo = 0;
    uint64_t sumaVerde = 0;
    uint64_t sumaAzul = 0;
    uint64_t sumaClaro = 0;

    int muestrasValidas = 0;

    for(
        int i = 0;
        i < TOTAL_MUESTRAS;
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

        // Evitar muestras completamente vacías.
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

        muestrasValidas++;

        delay(60);
    }

    if(muestrasValidas < 3)
    {
        return false;
    }

    resultado.rojo =
        sumaRojo /
        muestrasValidas;

    resultado.verde =
        sumaVerde /
        muestrasValidas;

    resultado.azul =
        sumaAzul /
        muestrasValidas;

    resultado.claro =
        sumaClaro /
        muestrasValidas;

    return true;
}

// ==================================================
// MOSTRAR MEDICIÓN
// ==================================================

void mostrarMedicion(
    const LecturaColor& lectura
)
{
    uint32_t suma =
        lectura.rojo +
        lectura.verde +
        lectura.azul;

    float rojoPct = 0;
    float verdePct = 0;
    float azulPct = 0;

    if(suma > 0)
    {
        rojoPct =
            lectura.rojo *
            100.0 /
            suma;

        verdePct =
            lectura.verde *
            100.0 /
            suma;

        azulPct =
            lectura.azul *
            100.0 /
            suma;
    }

    Serial.println();

    Serial.print("R: ");
    Serial.println(
        lectura.rojo
    );

    Serial.print("G: ");
    Serial.println(
        lectura.verde
    );

    Serial.print("B: ");
    Serial.println(
        lectura.azul
    );

    Serial.print("C: ");
    Serial.println(
        lectura.claro
    );

    Serial.println();

    Serial.print(
        "R normalizado: "
    );

    Serial.print(
        rojoPct,
        2
    );

    Serial.println(" %");

    Serial.print(
        "G normalizado: "
    );

    Serial.print(
        verdePct,
        2
    );

    Serial.println(" %");

    Serial.print(
        "B normalizado: "
    );

    Serial.print(
        azulPct,
        2
    );

    Serial.println(" %");
}

// ==================================================
// ENVIAR MEDICIÓN
// ==================================================

bool enviarMedicion(
    const LecturaColor& lectura
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
        "\"dispositivo\":\"" +
        String(NOMBRE_DISPOSITIVO) +
        "\",";

    datos +=
        "\"rojo_raw\":" +
        String(lectura.rojo) +
        ",";

    datos +=
        "\"verde_raw\":" +
        String(lectura.verde) +
        ",";

    datos +=
        "\"azul_raw\":" +
        String(lectura.azul) +
        ",";

    datos +=
        "\"claro_raw\":" +
        String(lectura.claro);

    datos += "}";

    Serial.println();
    Serial.println(
        "Enviando al servidor:"
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
    else
    {
        Serial.println(
            "No fue posible conectar con el servidor."
        );
    }

    http.end();

    return enviado;
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

    Wire.begin(
        PIN_SDA,
        PIN_SCL
    );

    Serial.println();
    Serial.println(
        "Iniciando sensor TCS34725"
    );

    if(!sensorColor.begin())
    {
        Serial.println(
            "No se encontró el sensor."
        );

        Serial.println(
            "Revise VCC, GND, SDA y SCL."
        );

        while(true)
        {
            delay(1000);
        }
    }

    Serial.println(
        "Sensor de color preparado."
    );

    conectarWiFi();

    Serial.println();
    Serial.println(
        "ESTACIÓN DE INSPECCIÓN DE COLOR"
    );

    Serial.println(
        "Esperando una pieza..."
    );
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
            "Objeto detectado"
        );

        delay(
            RETARDO_CENTRADO_MS
        );

        LecturaColor lectura;

        bool lecturaCorrecta =
            obtenerColorPromedio(
                lectura
            );

        if(lecturaCorrecta)
        {
            mostrarMedicion(
                lectura
            );

            bool enviado =
                enviarMedicion(
                    lectura
                );

            if(enviado)
            {
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
        }
        else
        {
            Serial.println(
                "No se obtuvieron suficientes muestras."
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

# 39. Variables que deben modificarse

## WiFi

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
    "http://DIRECCION_IP:3000/api/mediciones-color";
```

Ejemplo:

```cpp
const char* URL_SERVIDOR =
    "http://192.168.1.25:3000/api/mediciones-color";
```

---

## Producto

```cpp
const int PRODUCTO_ID = 1;
```

Debe coincidir con un producto registrado en MySQL.

---

## Tiempo de centrado

```cpp
const unsigned long RETARDO_CENTRADO_MS =
    80;
```

Aumentar el valor si la medición ocurre antes de que la pieza quede debajo del sensor.

Reducirlo si la pieza avanza demasiado antes de realizar la lectura.

---

# Bloque 9. Ejecutar el sistema

## 40. Preparar la estación

1. Colocar el sensor de color.
2. Colocar el sensor infrarrojo.
3. Instalar la cubierta de iluminación.
4. Verificar que la pieza pase a una distancia constante.
5. Encender la iluminación utilizada durante la calibración.
6. No cambiar la posición del sensor después de calibrar.

---

## 41. Iniciar MySQL

Verificar que MySQL Server se encuentre ejecutándose.

---

## 42. Iniciar Node.js

```bash
node server.js
```

Resultado esperado:

```text
Conexión correcta con MySQL
Servidor disponible en http://localhost:3000
```

---

## 43. Abrir la página de monitoreo

```text
http://localhost:3000/mediciones-color
```

---

## 44. Abrir la página de estadísticas

```text
http://localhost:3000/estadisticas-color
```

---

## 45. Encender el ESP32

Abrir el monitor serial a:

```text
115200 baudios
```

Resultado esperado:

```text
Sensor de color preparado
WiFi conectado
ESTACIÓN DE INSPECCIÓN DE COLOR
Esperando una pieza
```

---

# Bloque 10. Pruebas obligatorias

## Prueba 1. Objeto correcto detenido

Colocar manualmente una pieza correcta debajo del sensor.

Comprobar:

* Valores R, G, B y C.
* Porcentajes normalizados.
* Color detectado.
* Diferencia respecto a la referencia.
* Resultado.

---

## Prueba 2. Pieza correcta sobre la banda

Colocar una pieza del color esperado sobre la banda.

Resultado esperado:

```text
correcto
```

---

## Prueba 3. Pieza de color incorrecto

Pasar una pieza de otro color.

Resultado esperado:

```text
color_incorrecto
```

---

## Prueba 4. Registro único

Mantener una pieza frente al sensor infrarrojo.

El sistema debe generar solamente un registro.

No deberá registrar continuamente la misma pieza.

---

## Prueba 5. Nueva pieza

Retirar completamente la primera pieza y pasar otra.

El sistema deberá aceptar una nueva medición después de que la zona permanezca libre.

---

## Prueba 6. Variación entre piezas correctas

Pasar al menos diez piezas correctas.

Completar:

| Pieza | R % | G % | B % | Diferencia | Resultado |
| ----: | --: | --: | --: | ---------: | --------- |
|     1 |     |     |     |            |           |
|     2 |     |     |     |            |           |
|     3 |     |     |     |            |           |
|     4 |     |     |     |            |           |
|     5 |     |     |     |            |           |
|     6 |     |     |     |            |           |
|     7 |     |     |     |            |           |
|     8 |     |     |     |            |           |
|     9 |     |     |     |            |           |
|    10 |     |     |     |            |           |

---

## Prueba 7. Cambiar la iluminación

Realizar una medición:

* Con la cubierta.
* Sin la cubierta.
* Con las luces del salón encendidas.
* Con una luz externa cercana.

Comparar los resultados.

No se deberá modificar la tolerancia para ocultar un montaje inestable. Primero deberá corregirse la iluminación.

---

## Prueba 8. Cambiar la distancia

Modificar ligeramente la distancia entre sensor y objeto.

Registrar cómo cambian:

* Canal claro.
* Valores crudos.
* Porcentajes normalizados.
* Diferencia de color.

---

## Prueba 9. Actualización en tiempo real

Mantener abierta:

```text
/mediciones-color
```

Cada nueva medición deberá aparecer sin recargar la página.

---

## Prueba 10. Estadísticas

Registrar como mínimo:

* Diez piezas correctas.
* Cinco piezas incorrectas.

Comprobar:

* Total de piezas.
* Promedio de R.
* Promedio de G.
* Promedio de B.
* Diferencia promedio.
* Correctas.
* Incorrectas.
* Cumplimiento.
* Gráficas.

---

## Prueba 11. Producto sin referencia

Eliminar temporalmente la referencia:

```sql
UPDATE productos
SET
    color_ref_r_pct = NULL,
    color_ref_g_pct = NULL,
    color_ref_b_pct = NULL
WHERE id = 1;
```

Pasar una pieza.

Resultado esperado:

```text
sin_especificacion
```

Restaurar posteriormente la referencia.

---

## Prueba 12. Persistencia

Detener el servidor:

```text
Ctrl + C
```

Volver a iniciarlo:

```bash
node server.js
```

Las mediciones deberán continuar disponibles.

---

# 46. Consultas SQL de comprobación

## Consultar mediciones

```sql
SELECT
    *
FROM mediciones_color
ORDER BY fecha DESC;
```

---

## Consultar mediciones con producto

```sql
SELECT
    mediciones_color.id,
    productos.nombre AS producto,
    productos.color_esperado,
    mediciones_color.rojo_pct,
    mediciones_color.verde_pct,
    mediciones_color.azul_pct,
    mediciones_color.color_detectado,
    mediciones_color.diferencia_color,
    mediciones_color.resultado,
    mediciones_color.fecha

FROM mediciones_color

INNER JOIN productos
    ON mediciones_color.producto_id =
       productos.id

ORDER BY
    mediciones_color.fecha DESC;
```

---

## Contar resultados

```sql
SELECT
    resultado,
    COUNT(*) AS cantidad

FROM mediciones_color

GROUP BY resultado;
```

---

## Obtener promedio RGB

```sql
SELECT
    ROUND(
        AVG(rojo_pct),
        2
    ) AS rojo_promedio,

    ROUND(
        AVG(verde_pct),
        2
    ) AS verde_promedio,

    ROUND(
        AVG(azul_pct),
        2
    ) AS azul_promedio,

    ROUND(
        AVG(diferencia_color),
        2
    ) AS diferencia_promedio

FROM mediciones_color;
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
            mediciones_color.rojo_pct
        ),
        2
    ) AS rojo_promedio,

    ROUND(
        AVG(
            mediciones_color.verde_pct
        ),
        2
    ) AS verde_promedio,

    ROUND(
        AVG(
            mediciones_color.azul_pct
        ),
        2
    ) AS azul_promedio,

    SUM(
        mediciones_color.resultado =
        'correcto'
    ) AS correctas,

    SUM(
        mediciones_color.resultado =
        'color_incorrecto'
    ) AS incorrectas

FROM mediciones_color

INNER JOIN productos
    ON mediciones_color.producto_id =
       productos.id

GROUP BY
    productos.id,
    productos.nombre;
```

---

## Distribución de colores detectados

```sql
SELECT
    color_detectado,
    COUNT(*) AS cantidad

FROM mediciones_color

GROUP BY color_detectado

ORDER BY cantidad DESC;
```

---

# 47. Flujo completo de una medición

```text
El producto se registra en MySQL
                │
                ▼
Se calibra su color de referencia
                │
                ▼
Se guardan R%, G%, B% y tolerancia
                │
                ▼
La pieza llega a la banda
                │
                ▼
El infrarrojo detecta la pieza
                │
                ▼
El ESP32 espera el centrado
                │
                ▼
El TCS34725 toma varias muestras
                │
                ▼
El ESP32 obtiene los promedios
                │
                ▼
Envía R, G, B y C mediante HTTP
                │
                ▼
Node.js normaliza los canales
                │
                ▼
Calcula la diferencia de color
                │
        ┌───────┴─────────┐
        ▼                 ▼
Diferencia menor     Diferencia mayor
que la tolerancia    que la tolerancia
        │                 │
        ▼                 ▼
    Correcto        Color incorrecto
        │                 │
        └───────┬─────────┘
                ▼
       Guarda en MySQL
                │
                ▼
Socket.IO actualiza las páginas
```

---

# 48. Evidencias que deberán entregarse

El reporte deberá incluir:

1. Portada.
2. Objetivo.
3. Diagrama de conexión.
4. Fotografía del montaje.
5. Fotografía de la cubierta de iluminación.
6. Captura del programa de prueba.
7. Tabla de colores iniciales.
8. Tabla de calibración.
9. Valores de referencia utilizados.
10. Tolerancia seleccionada.
11. Justificación de la tolerancia.
12. Captura del monitor serial.
13. Captura de una pieza correcta.
14. Captura de una pieza incorrecta.
15. Captura de `mediciones_color`.
16. Captura de la página de monitoreo.
17. Captura de la página de estadísticas.
18. Prueba con al menos quince piezas.
19. Análisis de iluminación.
20. Análisis de errores.
21. Conclusiones.

---

# 49. Investigar

Investigar y explicar con palabras propias:

* ¿Qué es el TCS34725?
* ¿Qué representa cada canal R, G, B y C?
* ¿Qué función tiene el filtro infrarrojo del sensor?
* ¿Qué es el protocolo I²C?
* ¿Qué función realiza SDA?
* ¿Qué función realiza SCL?
* ¿Cuál es la dirección I²C del TCS34725?
* ¿Qué es el tiempo de integración?
* ¿Qué es la ganancia?
* ¿Qué ocurre si la ganancia es demasiado alta?
* ¿Qué ocurre si el tiempo de integración es muy corto?
* ¿Qué función realiza `begin()`?
* ¿Qué función realiza `getRawData()`?
* ¿Por qué se toman varias muestras?
* ¿Por qué se utiliza el promedio?
* ¿Qué significa normalizar los canales?
* ¿Por qué R%, G% y B% suman aproximadamente 100 %?
* ¿Qué representa el canal claro?
* ¿Por qué debe controlarse la iluminación?
* ¿Por qué debe mantenerse fija la distancia?
* ¿Qué diferencia existe entre reconocer un color y comprobar una especificación?
* ¿Qué representa la diferencia de color utilizada en esta práctica?
* ¿Qué función tiene la tolerancia?
* ¿Por qué el servidor vuelve a calcular los porcentajes?
* ¿Por qué se almacenan también los valores crudos?
* ¿Por qué una superficie brillante puede producir errores?
* ¿Por qué el color negro y el blanco son más difíciles de distinguir usando únicamente porcentajes normalizados?
* ¿Cómo ayuda la inspección de color al control de calidad?
* ¿Cómo podría utilizarse el color para separar automáticamente productos?

---

# 50. Consideraciones y limitaciones

El sistema puede presentar errores cuando:

* Cambia la iluminación.
* El sensor se mueve.
* La pieza cambia de altura.
* La superficie está inclinada.
* La pieza es brillante.
* La pieza tiene varios colores.
* La zona medida contiene una etiqueta.
* El objeto pasa demasiado rápido.
* La banda vibra.
* La pieza no queda centrada.
* El sensor recibe luz solar directa.
* El canal claro se satura.
* La pieza es demasiado oscura.
* La cubierta permite demasiada luz exterior.

El sistema supone que:

* Las piezas tienen una zona de color relativamente uniforme.
* La distancia permanece constante.
* Se utiliza la misma iluminación durante calibración y operación.
* Las piezas pasan de una en una.
* El sensor infrarrojo detecta correctamente cada pieza.
* El producto seleccionado en el ESP32 corresponde a las piezas de la banda.

---

# 51. Ajuste de la tolerancia

Una tolerancia demasiado pequeña puede rechazar piezas correctas.

Ejemplo:

```text
Tolerancia: 2

Diferencias de piezas correctas:
3.2
4.1
2.8
```

En este caso se producirían rechazos falsos.

Una tolerancia demasiado grande puede aceptar piezas incorrectas.

Ejemplo:

```text
Tolerancia: 30
```

Para seleccionar la tolerancia:

1. Medir varias piezas correctas.
2. Registrar sus diferencias.
3. Identificar la diferencia máxima normal.
4. Agregar un pequeño margen.
5. Probar piezas incorrectas.
6. Verificar que continúen siendo rechazadas.

---

# 52. Preparación para la integración final

Hasta esta práctica existen estaciones separadas para:

```text
Altura
Peso
RFID
Color
```

En la siguiente etapa será necesario relacionar todas las mediciones con una misma pieza.

La integración final puede utilizar una tabla principal:

```text
inspecciones
- id
- producto_id
- caja_id
- numero_pieza
- altura_cm
- peso_g
- rojo_pct
- verde_pct
- azul_pct
- color_detectado
- resultado_altura
- resultado_peso
- resultado_color
- resultado_general
- fecha
```

El flujo completo será:

```text
El infrarrojo detecta la pieza
              │
      ┌───────┴────────┐
      ▼                ▼
HC-SR04 mide       TCS34725 mide
altura             color
      │                │
      └───────┬────────┘
              ▼
      Se crea una inspección
              │
              ▼
       La pieza cae
              │
              ▼
HX711 obtiene el peso por diferencia
              │
              ▼
RFID identifica la caja receptora
              │
              ▼
La inspección queda completa en MySQL
```

El resultado general podrá ser:

```text
aceptada
rechazada_por_altura
rechazada_por_peso
rechazada_por_color
rechazada_por_varias_causas
```
