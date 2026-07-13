# Práctica: Sistema web básico de Industria 4.0 con Base de Datos

## Registro de usuarios, inicio de sesión, inventario e historial

---

# 1. Introducción

En el examen se desarrolló un sistema para contar objetos grandes y pequeños que circulaban sobre una banda transportadora.

Antes de volver a utilizar el ESP32 y los sensores infrarrojos, se desarrollará la parte correspondiente al servidor, la página web y la base de datos.

En esta práctica, los movimientos de objetos se registrarán manualmente desde una página web.

El sistema tendrá la siguiente estructura:

```text
Página HTML
     │
     │ fetch()
     ▼
Servidor Node.js
     │
     │ mysql2
     ▼
Base de datos MySQL
```

La práctica se realizará paso a paso para comprender cada bloque por separado.

---

# 2. Objetivo

Desarrollar un sistema web básico que permita:

* Registrar usuarios.
* Iniciar sesión.
* Cerrar sesión.
* Consultar los usuarios registrados.
* Consultar el historial de accesos.
* Registrar productos grandes y pequeños.
* Consultar el inventario.
* Registrar entradas y salidas.
* Consultar el historial de movimientos.
* Mantener permanentemente la información en MySQL.

En esta práctica todavía no se utilizarán sensores ni ESP32.

---

# 3. Páginas del sistema

El proyecto tendrá las siguientes páginas:

| Página            | Función                              |
| ----------------- | ------------------------------------ |
| `login.html`      | Iniciar sesión                       |
| `registro.html`   | Registrar usuarios                   |
| `panel.html`      | Mostrar un resumen                   |
| `usuarios.html`   | Mostrar usuarios registrados         |
| `accesos.html`    | Mostrar intentos de inicio de sesión |
| `inventario.html` | Registrar y consultar productos      |
| `movimiento.html` | Registrar entradas y salidas         |
| `historial.html`  | Consultar movimientos realizados     |

Cada página HTML tendrá su propio archivo JavaScript.

Por ejemplo:

```text
login.html
login.js
```

El HTML contiene los controles visuales.

El JavaScript:

* Lee los controles.
* Envía datos al servidor.
* Recibe la respuesta.
* Actualiza la página.

---

# 4. Software necesario

* Node.js.
* MySQL Server.
* Visual Studio Code.
* Navegador web.
* Terminal de Windows.
* MySQL Workbench, opcional.

---

# Bloque 1. Crear la base de datos

## 5. Entrar a MySQL

Abrir una terminal y ejecutar:

```bash
mysql -u root -p
```

Escribir la contraseña de MySQL.

---

## 6. Crear la base de datos

Ejecutar:

```sql
CREATE DATABASE industria40_web;

USE industria40_web;
```

---

## 7. Crear la tabla de usuarios

```sql
CREATE TABLE usuarios (
    id INT AUTO_INCREMENT PRIMARY KEY,

    nombre VARCHAR(100) NOT NULL,

    correo VARCHAR(150) NOT NULL UNIQUE,

    password_hash VARCHAR(255) NOT NULL,

    fecha_registro TIMESTAMP
        DEFAULT CURRENT_TIMESTAMP
);
```

Esta tabla guardará:

* El identificador del usuario.
* Su nombre.
* Su correo.
* La contraseña protegida.
* La fecha de registro.

---

## 8. Crear la tabla del historial de accesos

```sql
CREATE TABLE accesos (
    id INT AUTO_INCREMENT PRIMARY KEY,

    usuario_id INT NULL,

    correo_intentado VARCHAR(150) NOT NULL,

    resultado VARCHAR(20) NOT NULL,

    direccion_ip VARCHAR(50),

    fecha TIMESTAMP
        DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY (usuario_id)
        REFERENCES usuarios(id)
        ON DELETE SET NULL
);
```

La tabla guardará tanto los accesos correctos como los incorrectos.

Ejemplos:

```text
correcto
incorrecto
```

---

## 9. Crear la tabla de productos

```sql
CREATE TABLE productos (
    id INT AUTO_INCREMENT PRIMARY KEY,

    nombre VARCHAR(100) NOT NULL,

    tamano VARCHAR(20) NOT NULL,

    cantidad INT NOT NULL DEFAULT 0,

    fecha_registro TIMESTAMP
        DEFAULT CURRENT_TIMESTAMP
);
```

La columna `tamano` guardará:

```text
grande
pequeno
```

La columna `cantidad` representa las existencias actuales.

---

## 10. Crear la tabla de movimientos

```sql
CREATE TABLE movimientos (
    id INT AUTO_INCREMENT PRIMARY KEY,

    producto_id INT NOT NULL,

    usuario_id INT NULL,

    tipo VARCHAR(20) NOT NULL,

    cantidad INT NOT NULL,

    existencia_resultante INT NOT NULL,

    fecha TIMESTAMP
        DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY (producto_id)
        REFERENCES productos(id),

    FOREIGN KEY (usuario_id)
        REFERENCES usuarios(id)
        ON DELETE SET NULL
);
```

La columna `tipo` podrá contener:

```text
entrada
salida
```

La columna `existencia_resultante` guardará la cantidad que quedó después del movimiento.

---

## 11. Verificar las tablas

Ejecutar:

```sql
SHOW TABLES;
```

El resultado debe mostrar:

```text
accesos
movimientos
productos
usuarios
```

Para observar la estructura de una tabla:

```sql
DESCRIBE usuarios;
```

También se puede revisar:

```sql
DESCRIBE productos;
DESCRIBE movimientos;
DESCRIBE accesos;
```

Salir de MySQL:

```sql
EXIT;
```

---

# Bloque 2. Crear el proyecto de Node.js

## 12. Crear la carpeta

Crear una carpeta llamada:

```text
industria40_web
```

Abrirla desde Visual Studio Code.

---

## 13. Inicializar el proyecto

Abrir la terminal de Visual Studio Code y ejecutar:

```bash
npm init -y
```

---

## 14. Instalar las bibliotecas

Ejecutar:

```bash
npm install express mysql2 bcryptjs express-session
```

Las bibliotecas tienen las siguientes funciones:

| Biblioteca        | Función                     |
| ----------------- | --------------------------- |
| `express`         | Crear el servidor           |
| `mysql2`          | Conectar Node.js con MySQL  |
| `bcryptjs`        | Proteger las contraseñas    |
| `express-session` | Mantener iniciada la sesión |

---

# 15. Crear la estructura de carpetas

Crear la siguiente estructura:

```text
industria40_web/
│
├── server.js
├── package.json
│
├── paginas/
│   ├── login.html
│   ├── registro.html
│   ├── panel.html
│   ├── usuarios.html
│   ├── accesos.html
│   ├── inventario.html
│   ├── movimiento.html
│   └── historial.html
│
└── public/
    ├── css/
    │   └── estilos.css
    │
    └── js/
        ├── login.js
        ├── registro.js
        ├── comun.js
        ├── panel.js
        ├── usuarios.js
        ├── accesos.js
        ├── inventario.js
        ├── movimiento.js
        └── historial.js
```

---

# Bloque 3. Probar solamente la conexión con MySQL

## 16. Primera versión de `server.js`

Antes de crear páginas y rutas, se comprobará únicamente la conexión.

Crear:

```text
server.js
```

Agregar:

```javascript
const mysql = require("mysql2/promise");

async function conectarBaseDatos()
{
    try
    {
        const conexion = await mysql.createConnection({
            host: "localhost",
            user: "root",
            password: "CONTRASENA_MYSQL",
            database: "industria40_web"
        });

        console.log("Conexión correcta con MySQL");

        const [resultado] = await conexion.execute(
            "SHOW TABLES"
        );

        console.table(resultado);

        await conexion.end();
    }
    catch(error)
    {
        console.log("Error al conectar con MySQL");
        console.log(error.message);
    }
}

conectarBaseDatos();
```

Cambiar:

```text
CONTRASENA_MYSQL
```

por la contraseña real.

Ejecutar:

```bash
node server.js
```

El resultado esperado es:

```text
Conexión correcta con MySQL
```

Además, deberán aparecer las cuatro tablas.

No continuar hasta que esta prueba funcione.

---

# Bloque 4. Crear el servidor completo

Una vez comprobada la conexión, sustituir todo el contenido de `server.js` por el siguiente programa.

## 17. Código completo de `server.js`

```javascript
// ==================================================
// BIBLIOTECAS
// ==================================================

const express = require("express");
const mysql = require("mysql2/promise");
const bcrypt = require("bcryptjs");
const session = require("express-session");
const path = require("path");

// ==================================================
// CREAR EL SERVIDOR
// ==================================================

const app = express();

const PUERTO = 3000;

// Esta variable guardará la conexión con MySQL.
let conexion;

// ==================================================
// CONFIGURACIÓN DEL SERVIDOR
// ==================================================

// Permite recibir datos JSON enviados con fetch().
app.use(express.json());

// Permite recibir formularios tradicionales.
app.use(
    express.urlencoded({
        extended: true
    })
);

// Publicar los archivos CSS y JavaScript.
app.use(
    "/css",
    express.static(
        path.join(
            __dirname,
            "public",
            "css"
        )
    )
);

app.use(
    "/js",
    express.static(
        path.join(
            __dirname,
            "public",
            "js"
        )
    )
);

// Configuración de la sesión.
app.use(
    session({
        secret: "clave-industria-40",

        resave: false,

        saveUninitialized: false,

        cookie: {
            maxAge: 1000 * 60 * 60
        }
    })
);

// ==================================================
// CONEXIÓN CON MYSQL
// ==================================================

async function conectarBaseDatos()
{
    try
    {
        conexion = await mysql.createConnection({
            host: "localhost",
            user: "root",
            password: "CONTRASENA_MYSQL",
            database: "industria40_web"
        });

        console.log(
            "Conexión correcta con MySQL"
        );
    }
    catch(error)
    {
        console.log(
            "No fue posible conectar con MySQL"
        );

        console.log(error.message);

        process.exit(1);
    }
}

// ==================================================
// FUNCIONES PARA ENVIAR PÁGINAS
// ==================================================

function enviarPagina(res, nombreArchivo)
{
    res.sendFile(
        path.join(
            __dirname,
            "paginas",
            nombreArchivo
        )
    );
}

// Verifica que exista una sesión antes de mostrar
// una página privada.
function requiereSesionPagina(req, res, next)
{
    if(!req.session.usuario)
    {
        return res.redirect("/login");
    }

    next();
}

// Verifica la sesión antes de ejecutar una API.
function requiereSesionAPI(req, res, next)
{
    if(!req.session.usuario)
    {
        return res.status(401).json({
            correcto: false,
            mensaje: "Debe iniciar sesión"
        });
    }

    next();
}

// ==================================================
// RUTAS DE LAS PÁGINAS
// ==================================================

app.get("/", (req, res) =>
{
    if(req.session.usuario)
    {
        return res.redirect("/panel");
    }

    res.redirect("/login");
});

app.get("/login", (req, res) =>
{
    enviarPagina(res, "login.html");
});

app.get("/registro", (req, res) =>
{
    enviarPagina(res, "registro.html");
});

app.get(
    "/panel",
    requiereSesionPagina,
    (req, res) =>
    {
        enviarPagina(res, "panel.html");
    }
);

app.get(
    "/usuarios",
    requiereSesionPagina,
    (req, res) =>
    {
        enviarPagina(res, "usuarios.html");
    }
);

app.get(
    "/accesos",
    requiereSesionPagina,
    (req, res) =>
    {
        enviarPagina(res, "accesos.html");
    }
);

app.get(
    "/inventario",
    requiereSesionPagina,
    (req, res) =>
    {
        enviarPagina(res, "inventario.html");
    }
);

app.get(
    "/movimiento",
    requiereSesionPagina,
    (req, res) =>
    {
        enviarPagina(res, "movimiento.html");
    }
);

app.get(
    "/historial",
    requiereSesionPagina,
    (req, res) =>
    {
        enviarPagina(res, "historial.html");
    }
);

// ==================================================
// API: REGISTRAR USUARIO
// ==================================================

app.post("/api/usuarios", async (req, res) =>
{
    try
    {
        const nombre =
            String(req.body.nombre || "").trim();

        const correo =
            String(req.body.correo || "")
                .trim()
                .toLowerCase();

        const contrasena =
            String(req.body.contrasena || "");

        if(
            nombre === "" ||
            correo === "" ||
            contrasena === ""
        )
        {
            return res.status(400).json({
                correcto: false,
                mensaje:
                    "Todos los campos son obligatorios"
            });
        }

        if(contrasena.length < 6)
        {
            return res.status(400).json({
                correcto: false,
                mensaje:
                    "La contraseña debe tener al menos 6 caracteres"
            });
        }

        // Convertir la contraseña en un hash.
        const passwordHash = await bcrypt.hash(
            contrasena,
            10
        );

        await conexion.execute(
            `INSERT INTO usuarios
            (
                nombre,
                correo,
                password_hash
            )
            VALUES (?, ?, ?)`,
            [
                nombre,
                correo,
                passwordHash
            ]
        );

        res.json({
            correcto: true,
            mensaje:
                "Usuario registrado correctamente"
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
                    "El correo ya está registrado"
            });
        }

        res.status(500).json({
            correcto: false,
            mensaje:
                "No fue posible registrar el usuario"
        });
    }
});

// ==================================================
// API: INICIAR SESIÓN
// ==================================================

app.post("/api/login", async (req, res) =>
{
    try
    {
        const correo =
            String(req.body.correo || "")
                .trim()
                .toLowerCase();

        const contrasena =
            String(req.body.contrasena || "");

        if(correo === "" || contrasena === "")
        {
            return res.status(400).json({
                correcto: false,
                mensaje:
                    "Escriba el correo y la contraseña"
            });
        }

        const [usuarios] = await conexion.execute(
            `SELECT
                id,
                nombre,
                correo,
                password_hash
            FROM usuarios
            WHERE correo = ?`,
            [correo]
        );

        if(usuarios.length === 0)
        {
            await conexion.execute(
                `INSERT INTO accesos
                (
                    usuario_id,
                    correo_intentado,
                    resultado,
                    direccion_ip
                )
                VALUES (?, ?, ?, ?)`,
                [
                    null,
                    correo,
                    "incorrecto",
                    req.ip
                ]
            );

            return res.status(401).json({
                correcto: false,
                mensaje:
                    "Correo o contraseña incorrectos"
            });
        }

        const usuario = usuarios[0];

        const contrasenaCorrecta =
            await bcrypt.compare(
                contrasena,
                usuario.password_hash
            );

        if(!contrasenaCorrecta)
        {
            await conexion.execute(
                `INSERT INTO accesos
                (
                    usuario_id,
                    correo_intentado,
                    resultado,
                    direccion_ip
                )
                VALUES (?, ?, ?, ?)`,
                [
                    usuario.id,
                    correo,
                    "incorrecto",
                    req.ip
                ]
            );

            return res.status(401).json({
                correcto: false,
                mensaje:
                    "Correo o contraseña incorrectos"
            });
        }

        // Guardar los datos del usuario en la sesión.
        req.session.usuario = {
            id: usuario.id,
            nombre: usuario.nombre,
            correo: usuario.correo
        };

        await conexion.execute(
            `INSERT INTO accesos
            (
                usuario_id,
                correo_intentado,
                resultado,
                direccion_ip
            )
            VALUES (?, ?, ?, ?)`,
            [
                usuario.id,
                correo,
                "correcto",
                req.ip
            ]
        );

        res.json({
            correcto: true,
            mensaje:
                "Inicio de sesión correcto"
        });
    }
    catch(error)
    {
        console.log(error);

        res.status(500).json({
            correcto: false,
            mensaje:
                "Ocurrió un error en el servidor"
        });
    }
});

// ==================================================
// API: CONSULTAR LA SESIÓN
// ==================================================

app.get("/api/sesion", (req, res) =>
{
    if(!req.session.usuario)
    {
        return res.status(401).json({
            autenticado: false
        });
    }

    res.json({
        autenticado: true,
        usuario: req.session.usuario
    });
});

// ==================================================
// API: CERRAR SESIÓN
// ==================================================

app.post("/api/logout", (req, res) =>
{
    req.session.destroy((error) =>
    {
        if(error)
        {
            return res.status(500).json({
                correcto: false,
                mensaje:
                    "No fue posible cerrar la sesión"
            });
        }

        res.json({
            correcto: true
        });
    });
});

// ==================================================
// API: CONSULTAR USUARIOS
// ==================================================

app.get(
    "/api/usuarios",
    requiereSesionAPI,
    async (req, res) =>
    {
        try
        {
            const [usuarios] =
                await conexion.execute(
                    `SELECT
                        id,
                        nombre,
                        correo,
                        fecha_registro
                    FROM usuarios
                    ORDER BY fecha_registro DESC`
                );

            res.json(usuarios);
        }
        catch(error)
        {
            console.log(error);

            res.status(500).json({
                mensaje:
                    "No fue posible consultar los usuarios"
            });
        }
    }
);

// ==================================================
// API: CONSULTAR HISTORIAL DE ACCESOS
// ==================================================

app.get(
    "/api/accesos",
    requiereSesionAPI,
    async (req, res) =>
    {
        try
        {
            const [accesos] =
                await conexion.execute(
                    `SELECT
                        accesos.id,
                        accesos.correo_intentado,
                        accesos.resultado,
                        accesos.direccion_ip,
                        accesos.fecha,
                        usuarios.nombre

                    FROM accesos

                    LEFT JOIN usuarios
                        ON accesos.usuario_id =
                           usuarios.id

                    ORDER BY accesos.fecha DESC`
                );

            res.json(accesos);
        }
        catch(error)
        {
            console.log(error);

            res.status(500).json({
                mensaje:
                    "No fue posible consultar los accesos"
            });
        }
    }
);

// ==================================================
// API: REGISTRAR PRODUCTO
// ==================================================

app.post(
    "/api/productos",
    requiereSesionAPI,
    async (req, res) =>
    {
        try
        {
            const nombre =
                String(req.body.nombre || "").trim();

            const tamano =
                String(req.body.tamano || "").trim();

            const cantidad =
                Number(req.body.cantidad);

            if(
                nombre === "" ||
                !["grande", "pequeno"].includes(
                    tamano
                )
            )
            {
                return res.status(400).json({
                    correcto: false,
                    mensaje:
                        "Revise los datos del producto"
                });
            }

            if(
                !Number.isInteger(cantidad) ||
                cantidad < 0
            )
            {
                return res.status(400).json({
                    correcto: false,
                    mensaje:
                        "La cantidad debe ser un entero mayor o igual que cero"
                });
            }

            await conexion.execute(
                `INSERT INTO productos
                (
                    nombre,
                    tamano,
                    cantidad
                )
                VALUES (?, ?, ?)`,
                [
                    nombre,
                    tamano,
                    cantidad
                ]
            );

            res.json({
                correcto: true,
                mensaje:
                    "Producto registrado correctamente"
            });
        }
        catch(error)
        {
            console.log(error);

            res.status(500).json({
                correcto: false,
                mensaje:
                    "No fue posible registrar el producto"
            });
        }
    }
);

// ==================================================
// API: CONSULTAR PRODUCTOS
// ==================================================

app.get(
    "/api/productos",
    requiereSesionAPI,
    async (req, res) =>
    {
        try
        {
            const [productos] =
                await conexion.execute(
                    `SELECT
                        id,
                        nombre,
                        tamano,
                        cantidad,
                        fecha_registro
                    FROM productos
                    ORDER BY nombre`
                );

            res.json(productos);
        }
        catch(error)
        {
            console.log(error);

            res.status(500).json({
                mensaje:
                    "No fue posible consultar los productos"
            });
        }
    }
);

// ==================================================
// API: REGISTRAR MOVIMIENTO
// ==================================================

app.post(
    "/api/movimientos",
    requiereSesionAPI,
    async (req, res) =>
    {
        const productoId =
            Number(req.body.producto_id);

        const tipo =
            String(req.body.tipo || "").trim();

        const cantidad =
            Number(req.body.cantidad);

        if(
            !Number.isInteger(productoId) ||
            productoId <= 0
        )
        {
            return res.status(400).json({
                correcto: false,
                mensaje:
                    "Seleccione un producto"
            });
        }

        if(
            tipo !== "entrada" &&
            tipo !== "salida"
        )
        {
            return res.status(400).json({
                correcto: false,
                mensaje:
                    "Seleccione entrada o salida"
            });
        }

        if(
            !Number.isInteger(cantidad) ||
            cantidad <= 0
        )
        {
            return res.status(400).json({
                correcto: false,
                mensaje:
                    "La cantidad debe ser mayor que cero"
            });
        }

        try
        {
            // Iniciar una transacción.
            await conexion.beginTransaction();

            // Consultar y bloquear temporalmente
            // el producto seleccionado.
            const [productos] =
                await conexion.execute(
                    `SELECT
                        id,
                        nombre,
                        cantidad
                    FROM productos
                    WHERE id = ?
                    FOR UPDATE`,
                    [productoId]
                );

            if(productos.length === 0)
            {
                await conexion.rollback();

                return res.status(404).json({
                    correcto: false,
                    mensaje:
                        "El producto no existe"
                });
            }

            const producto = productos[0];

            let nuevaCantidad = producto.cantidad;

            if(tipo === "entrada")
            {
                nuevaCantidad =
                    producto.cantidad + cantidad;
            }
            else
            {
                nuevaCantidad =
                    producto.cantidad - cantidad;
            }

            if(nuevaCantidad < 0)
            {
                await conexion.rollback();

                return res.status(400).json({
                    correcto: false,
                    mensaje:
                        "No hay suficientes existencias"
                });
            }

            // Actualizar el inventario.
            await conexion.execute(
                `UPDATE productos
                SET cantidad = ?
                WHERE id = ?`,
                [
                    nuevaCantidad,
                    productoId
                ]
            );

            // Guardar el movimiento en el historial.
            await conexion.execute(
                `INSERT INTO movimientos
                (
                    producto_id,
                    usuario_id,
                    tipo,
                    cantidad,
                    existencia_resultante
                )
                VALUES (?, ?, ?, ?, ?)`,
                [
                    productoId,
                    req.session.usuario.id,
                    tipo,
                    cantidad,
                    nuevaCantidad
                ]
            );

            // Confirmar ambas operaciones.
            await conexion.commit();

            res.json({
                correcto: true,
                mensaje:
                    "Movimiento registrado correctamente"
            });
        }
        catch(error)
        {
            await conexion.rollback();

            console.log(error);

            res.status(500).json({
                correcto: false,
                mensaje:
                    "No fue posible registrar el movimiento"
            });
        }
    }
);

// ==================================================
// API: CONSULTAR HISTORIAL DE MOVIMIENTOS
// ==================================================

app.get(
    "/api/movimientos",
    requiereSesionAPI,
    async (req, res) =>
    {
        try
        {
            const [movimientos] =
                await conexion.execute(
                    `SELECT
                        movimientos.id,
                        movimientos.tipo,
                        movimientos.cantidad,
                        movimientos.existencia_resultante,
                        movimientos.fecha,

                        productos.nombre AS producto,
                        productos.tamano,

                        usuarios.nombre AS usuario

                    FROM movimientos

                    INNER JOIN productos
                        ON movimientos.producto_id =
                           productos.id

                    LEFT JOIN usuarios
                        ON movimientos.usuario_id =
                           usuarios.id

                    ORDER BY movimientos.fecha DESC`
                );

            res.json(movimientos);
        }
        catch(error)
        {
            console.log(error);

            res.status(500).json({
                mensaje:
                    "No fue posible consultar los movimientos"
            });
        }
    }
);

// ==================================================
// API: RESUMEN DEL PANEL
// ==================================================

app.get(
    "/api/resumen",
    requiereSesionAPI,
    async (req, res) =>
    {
        try
        {
            const [resultadoUsuarios] =
                await conexion.execute(
                    `SELECT COUNT(*) AS total
                    FROM usuarios`
                );

            const [resultadoProductos] =
                await conexion.execute(
                    `SELECT COUNT(*) AS total
                    FROM productos`
                );

            const [resultadoInventario] =
                await conexion.execute(
                    `SELECT
                        COALESCE(
                            SUM(cantidad),
                            0
                        ) AS total
                    FROM productos`
                );

            const [resultadoMovimientos] =
                await conexion.execute(
                    `SELECT COUNT(*) AS total
                    FROM movimientos`
                );

            res.json({
                usuarios:
                    resultadoUsuarios[0].total,

                productos:
                    resultadoProductos[0].total,

                inventario:
                    resultadoInventario[0].total,

                movimientos:
                    resultadoMovimientos[0].total
            });
        }
        catch(error)
        {
            console.log(error);

            res.status(500).json({
                mensaje:
                    "No fue posible cargar el resumen"
            });
        }
    }
);

// ==================================================
// INICIAR SERVIDOR
// ==================================================

async function iniciarServidor()
{
    await conectarBaseDatos();

    app.listen(PUERTO, () =>
    {
        console.log(
            `Servidor disponible en http://localhost:${PUERTO}`
        );
    });
}

iniciarServidor();
```

Cambiar nuevamente:

```text
CONTRASENA_MYSQL
```

por la contraseña real.

---

# Bloque 5. Crear los estilos

## 18. Archivo `public/css/estilos.css`

```css
* {
    box-sizing: border-box;
}

body {
    margin: 0;
    font-family: Arial, Helvetica, sans-serif;
    background: #f1f5f9;
    color: #1e293b;
}

header {
    background: #1e293b;
    color: white;
    padding: 20px 5%;
    display: flex;
    justify-content: space-between;
    align-items: center;
}

header h1 {
    margin: 0;
}

nav {
    background: white;
    padding: 14px 5%;
    border-bottom: 1px solid #cbd5e1;
}

nav a {
    margin-right: 20px;
    color: #0369a1;
    font-weight: bold;
    text-decoration: none;
}

main {
    width: 90%;
    max-width: 1100px;
    margin: 30px auto;
}

.contenedor-acceso {
    max-width: 420px;
    margin: 70px auto;
    background: white;
    padding: 30px;
    border-radius: 8px;
}

form {
    display: flex;
    flex-direction: column;
    gap: 10px;
}

input,
select {
    padding: 10px;
    border: 1px solid #94a3b8;
    border-radius: 5px;
    font-size: 16px;
}

button {
    padding: 10px 18px;
    border: none;
    border-radius: 5px;
    background: #0369a1;
    color: white;
    cursor: pointer;
    font-size: 15px;
}

button:hover {
    opacity: 0.9;
}

.boton-gris {
    background: #64748b;
}

.mensaje {
    display: none;
    padding: 12px;
    margin: 15px 0;
    border-radius: 5px;
}

.mensaje-correcto {
    display: block;
    background: #dcfce7;
    color: #166534;
}

.mensaje-error {
    display: block;
    background: #fee2e2;
    color: #991b1b;
}

.tarjetas {
    display: grid;
    grid-template-columns:
        repeat(auto-fit, minmax(200px, 1fr));
    gap: 20px;
}

.tarjeta {
    background: white;
    padding: 22px;
    border-radius: 8px;
}

.numero {
    font-size: 36px;
    font-weight: bold;
}

.seccion {
    margin-top: 25px;
    background: white;
    padding: 22px;
    border-radius: 8px;
}

.tabla-contenedor {
    overflow-x: auto;
    background: white;
    padding: 15px;
    border-radius: 8px;
}

table {
    width: 100%;
    border-collapse: collapse;
}

th,
td {
    padding: 11px;
    border-bottom: 1px solid #cbd5e1;
    text-align: left;
}

th {
    background: #e2e8f0;
}

.formulario-horizontal {
    display: grid;
    grid-template-columns:
        repeat(auto-fit, minmax(180px, 1fr));
    gap: 12px;
    align-items: end;
    margin-bottom: 25px;
}

.formulario-horizontal div {
    display: flex;
    flex-direction: column;
    gap: 6px;
}
```

---

# Bloque 6. Registrar usuarios

## 19. Página `paginas/registro.html`

```html
<!DOCTYPE html>
<html lang="es">

<head>
    <meta charset="UTF-8">

    <meta
        name="viewport"
        content="width=device-width, initial-scale=1.0"
    >

    <title>Registrar usuario</title>

    <link
        rel="stylesheet"
        href="/css/estilos.css"
    >
</head>

<body>

    <main class="contenedor-acceso">

        <h1>Registrar usuario</h1>

        <div
            id="mensaje"
            class="mensaje"
        ></div>

        <form id="formularioRegistro">

            <label for="nombre">
                Nombre
            </label>

            <input
                type="text"
                id="nombre"
                required
            >

            <label for="correo">
                Correo
            </label>

            <input
                type="email"
                id="correo"
                required
            >

            <label for="contrasena">
                Contraseña
            </label>

            <input
                type="password"
                id="contrasena"
                minlength="6"
                required
            >

            <button type="submit">
                Registrar usuario
            </button>

        </form>

        <p>
            <a href="/login">
                Regresar al inicio de sesión
            </a>
        </p>

    </main>

    <script src="/js/registro.js"></script>

</body>
</html>
```

---

## 20. JavaScript `public/js/registro.js`

```javascript
const formulario =
    document.getElementById(
        "formularioRegistro"
    );

const mensaje =
    document.getElementById("mensaje");

formulario.addEventListener(
    "submit",
    async function(evento)
    {
        evento.preventDefault();

        const datos = {
            nombre:
                document.getElementById(
                    "nombre"
                ).value,

            correo:
                document.getElementById(
                    "correo"
                ).value,

            contrasena:
                document.getElementById(
                    "contrasena"
                ).value
        };

        try
        {
            const respuesta = await fetch(
                "/api/usuarios",
                {
                    method: "POST",

                    headers: {
                        "Content-Type":
                            "application/json"
                    },

                    body: JSON.stringify(datos)
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

                setTimeout(() =>
                {
                    window.location.href =
                        "/login";
                }, 1500);
            }
            else
            {
                mensaje.className =
                    "mensaje mensaje-error";
            }
        }
        catch(error)
        {
            mensaje.textContent =
                "No se pudo conectar con el servidor";

            mensaje.className =
                "mensaje mensaje-error";
        }
    }
);
```

---

## 21. Probar el registro

Iniciar el servidor:

```bash
node server.js
```

Abrir:

```text
http://localhost:3000/registro
```

Registrar un usuario.

Después entrar a MySQL y comprobar:

```sql
USE industria40_web;

SELECT
    id,
    nombre,
    correo,
    password_hash,
    fecha_registro
FROM usuarios;
```

La contraseña no debe aparecer directamente.

Debe aparecer una cadena parecida a:

```text
$2b$10$...
```

---

# Bloque 7. Iniciar sesión

## 22. Página `paginas/login.html`

```html
<!DOCTYPE html>
<html lang="es">

<head>
    <meta charset="UTF-8">

    <meta
        name="viewport"
        content="width=device-width, initial-scale=1.0"
    >

    <title>Iniciar sesión</title>

    <link
        rel="stylesheet"
        href="/css/estilos.css"
    >
</head>

<body>

    <main class="contenedor-acceso">

        <h1>Iniciar sesión</h1>

        <p>
            Sistema de Industria 4.0
        </p>

        <div
            id="mensaje"
            class="mensaje"
        ></div>

        <form id="formularioLogin">

            <label for="correo">
                Correo
            </label>

            <input
                type="email"
                id="correo"
                required
            >

            <label for="contrasena">
                Contraseña
            </label>

            <input
                type="password"
                id="contrasena"
                required
            >

            <button type="submit">
                Iniciar sesión
            </button>

        </form>

        <p>
            ¿No tiene una cuenta?

            <a href="/registro">
                Registrar usuario
            </a>
        </p>

    </main>

    <script src="/js/login.js"></script>

</body>
</html>
```

---

## 23. JavaScript `public/js/login.js`

```javascript
const formulario =
    document.getElementById(
        "formularioLogin"
    );

const mensaje =
    document.getElementById("mensaje");

formulario.addEventListener(
    "submit",
    async function(evento)
    {
        evento.preventDefault();

        const datos = {
            correo:
                document.getElementById(
                    "correo"
                ).value,

            contrasena:
                document.getElementById(
                    "contrasena"
                ).value
        };

        try
        {
            const respuesta = await fetch(
                "/api/login",
                {
                    method: "POST",

                    headers: {
                        "Content-Type":
                            "application/json"
                    },

                    body: JSON.stringify(datos)
                }
            );

            const resultado =
                await respuesta.json();

            if(respuesta.ok)
            {
                window.location.href =
                    "/panel";
            }
            else
            {
                mensaje.textContent =
                    resultado.mensaje;

                mensaje.className =
                    "mensaje mensaje-error";
            }
        }
        catch(error)
        {
            mensaje.textContent =
                "No se pudo conectar con el servidor";

            mensaje.className =
                "mensaje mensaje-error";
        }
    }
);
```

---

# Bloque 8. Código común para las páginas privadas

## 24. Archivo `public/js/comun.js`

Este archivo será utilizado por todas las páginas que requieren una sesión.

```javascript
async function comprobarSesion()
{
    const respuesta = await fetch(
        "/api/sesion"
    );

    if(!respuesta.ok)
    {
        window.location.href = "/login";
        return null;
    }

    const datos = await respuesta.json();

    const espacioNombre =
        document.getElementById(
            "nombreUsuario"
        );

    if(espacioNombre)
    {
        espacioNombre.textContent =
            datos.usuario.nombre;
    }

    return datos.usuario;
}

async function cerrarSesion()
{
    await fetch(
        "/api/logout",
        {
            method: "POST"
        }
    );

    window.location.href = "/login";
}

document.addEventListener(
    "DOMContentLoaded",
    function()
    {
        comprobarSesion();

        const boton =
            document.getElementById(
                "botonCerrarSesion"
            );

        if(boton)
        {
            boton.addEventListener(
                "click",
                cerrarSesion
            );
        }
    }
);
```

---

# Bloque 9. Crear el panel principal

## 25. Página `paginas/panel.html`

```html
<!DOCTYPE html>
<html lang="es">

<head>
    <meta charset="UTF-8">

    <meta
        name="viewport"
        content="width=device-width, initial-scale=1.0"
    >

    <title>Panel principal</title>

    <link
        rel="stylesheet"
        href="/css/estilos.css"
    >
</head>

<body>

    <header>

        <div>
            <h1>Panel principal</h1>

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
        <a href="/usuarios">Usuarios</a>
        <a href="/accesos">Accesos</a>
        <a href="/inventario">Inventario</a>
        <a href="/movimiento">Movimiento</a>
        <a href="/historial">Historial</a>
    </nav>

    <main>

        <h2>Resumen del sistema</h2>

        <div class="tarjetas">

            <article class="tarjeta">
                <h3>Usuarios</h3>

                <p
                    id="totalUsuarios"
                    class="numero"
                >
                    0
                </p>
            </article>

            <article class="tarjeta">
                <h3>Productos</h3>

                <p
                    id="totalProductos"
                    class="numero"
                >
                    0
                </p>
            </article>

            <article class="tarjeta">
                <h3>Existencias</h3>

                <p
                    id="totalInventario"
                    class="numero"
                >
                    0
                </p>
            </article>

            <article class="tarjeta">
                <h3>Movimientos</h3>

                <p
                    id="totalMovimientos"
                    class="numero"
                >
                    0
                </p>
            </article>

        </div>

    </main>

    <script src="/js/comun.js"></script>
    <script src="/js/panel.js"></script>

</body>
</html>
```

---

## 26. JavaScript `public/js/panel.js`

```javascript
async function cargarResumen()
{
    const respuesta = await fetch(
        "/api/resumen"
    );

    if(!respuesta.ok)
    {
        return;
    }

    const datos = await respuesta.json();

    document.getElementById(
        "totalUsuarios"
    ).textContent = datos.usuarios;

    document.getElementById(
        "totalProductos"
    ).textContent = datos.productos;

    document.getElementById(
        "totalInventario"
    ).textContent = datos.inventario;

    document.getElementById(
        "totalMovimientos"
    ).textContent = datos.movimientos;
}

document.addEventListener(
    "DOMContentLoaded",
    cargarResumen
);
```

---

# Bloque 10. Consultar usuarios registrados

## 27. Página `paginas/usuarios.html`

```html
<!DOCTYPE html>
<html lang="es">

<head>
    <meta charset="UTF-8">

    <meta
        name="viewport"
        content="width=device-width, initial-scale=1.0"
    >

    <title>Usuarios</title>

    <link
        rel="stylesheet"
        href="/css/estilos.css"
    >
</head>

<body>

    <header>

        <div>
            <h1>Usuarios registrados</h1>

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
        <a href="/usuarios">Usuarios</a>
        <a href="/accesos">Accesos</a>
        <a href="/inventario">Inventario</a>
        <a href="/movimiento">Movimiento</a>
        <a href="/historial">Historial</a>
    </nav>

    <main>

        <div class="tabla-contenedor">

            <table>

                <thead>
                    <tr>
                        <th>ID</th>
                        <th>Nombre</th>
                        <th>Correo</th>
                        <th>Fecha de registro</th>
                    </tr>
                </thead>

                <tbody id="tablaUsuarios"></tbody>

            </table>

        </div>

    </main>

    <script src="/js/comun.js"></script>
    <script src="/js/usuarios.js"></script>

</body>
</html>
```

---

## 28. JavaScript `public/js/usuarios.js`

```javascript
async function cargarUsuarios()
{
    const respuesta = await fetch(
        "/api/usuarios"
    );

    if(!respuesta.ok)
    {
        return;
    }

    const usuarios = await respuesta.json();

    const tabla =
        document.getElementById(
            "tablaUsuarios"
        );

    tabla.innerHTML = "";

    usuarios.forEach(usuario =>
    {
        tabla.innerHTML += `
            <tr>
                <td>${usuario.id}</td>
                <td>${usuario.nombre}</td>
                <td>${usuario.correo}</td>
                <td>
                    ${new Date(
                        usuario.fecha_registro
                    ).toLocaleString()}
                </td>
            </tr>
        `;
    });
}

document.addEventListener(
    "DOMContentLoaded",
    cargarUsuarios
);
```

---

# Bloque 11. Consultar historial de accesos

## 29. Página `paginas/accesos.html`

```html
<!DOCTYPE html>
<html lang="es">

<head>
    <meta charset="UTF-8">

    <meta
        name="viewport"
        content="width=device-width, initial-scale=1.0"
    >

    <title>Historial de accesos</title>

    <link
        rel="stylesheet"
        href="/css/estilos.css"
    >
</head>

<body>

    <header>

        <div>
            <h1>Historial de accesos</h1>

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
        <a href="/usuarios">Usuarios</a>
        <a href="/accesos">Accesos</a>
        <a href="/inventario">Inventario</a>
        <a href="/movimiento">Movimiento</a>
        <a href="/historial">Historial</a>
    </nav>

    <main>

        <div class="tabla-contenedor">

            <table>

                <thead>
                    <tr>
                        <th>ID</th>
                        <th>Usuario</th>
                        <th>Correo intentado</th>
                        <th>Resultado</th>
                        <th>Dirección IP</th>
                        <th>Fecha</th>
                    </tr>
                </thead>

                <tbody id="tablaAccesos"></tbody>

            </table>

        </div>

    </main>

    <script src="/js/comun.js"></script>
    <script src="/js/accesos.js"></script>

</body>
</html>
```

---

## 30. JavaScript `public/js/accesos.js`

```javascript
async function cargarAccesos()
{
    const respuesta = await fetch(
        "/api/accesos"
    );

    if(!respuesta.ok)
    {
        return;
    }

    const accesos = await respuesta.json();

    const tabla =
        document.getElementById(
            "tablaAccesos"
        );

    tabla.innerHTML = "";

    accesos.forEach(acceso =>
    {
        tabla.innerHTML += `
            <tr>
                <td>${acceso.id}</td>

                <td>
                    ${acceso.nombre || "Desconocido"}
                </td>

                <td>
                    ${acceso.correo_intentado}
                </td>

                <td>${acceso.resultado}</td>

                <td>
                    ${acceso.direccion_ip || ""}
                </td>

                <td>
                    ${new Date(
                        acceso.fecha
                    ).toLocaleString()}
                </td>
            </tr>
        `;
    });
}

document.addEventListener(
    "DOMContentLoaded",
    cargarAccesos
);
```

---

# Bloque 12. Registrar productos y consultar inventario

## 31. Página `paginas/inventario.html`

```html
<!DOCTYPE html>
<html lang="es">

<head>
    <meta charset="UTF-8">

    <meta
        name="viewport"
        content="width=device-width, initial-scale=1.0"
    >

    <title>Inventario</title>

    <link
        rel="stylesheet"
        href="/css/estilos.css"
    >
</head>

<body>

    <header>

        <div>
            <h1>Inventario</h1>

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
        <a href="/usuarios">Usuarios</a>
        <a href="/accesos">Accesos</a>
        <a href="/inventario">Inventario</a>
        <a href="/movimiento">Movimiento</a>
        <a href="/historial">Historial</a>
    </nav>

    <main>

        <section class="seccion">

            <h2>Registrar producto</h2>

            <div
                id="mensaje"
                class="mensaje"
            ></div>

            <form
                id="formularioProducto"
                class="formulario-horizontal"
            >

                <div>
                    <label for="nombre">
                        Nombre
                    </label>

                    <input
                        type="text"
                        id="nombre"
                        required
                    >
                </div>

                <div>
                    <label for="tamano">
                        Tamaño
                    </label>

                    <select
                        id="tamano"
                        required
                    >
                        <option value="">
                            Seleccione
                        </option>

                        <option value="pequeno">
                            Pequeño
                        </option>

                        <option value="grande">
                            Grande
                        </option>
                    </select>
                </div>

                <div>
                    <label for="cantidad">
                        Cantidad inicial
                    </label>

                    <input
                        type="number"
                        id="cantidad"
                        value="0"
                        min="0"
                        step="1"
                        required
                    >
                </div>

                <button type="submit">
                    Registrar
                </button>

            </form>

        </section>

        <section>

            <h2>Productos registrados</h2>

            <div class="tabla-contenedor">

                <table>

                    <thead>
                        <tr>
                            <th>ID</th>
                            <th>Producto</th>
                            <th>Tamaño</th>
                            <th>Existencias</th>
                            <th>Fecha</th>
                        </tr>
                    </thead>

                    <tbody id="tablaProductos"></tbody>

                </table>

            </div>

        </section>

    </main>

    <script src="/js/comun.js"></script>
    <script src="/js/inventario.js"></script>

</body>
</html>
```

---

## 32. JavaScript `public/js/inventario.js`

```javascript
const formularioProducto =
    document.getElementById(
        "formularioProducto"
    );

const mensaje =
    document.getElementById("mensaje");

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

    const tabla =
        document.getElementById(
            "tablaProductos"
        );

    tabla.innerHTML = "";

    productos.forEach(producto =>
    {
        tabla.innerHTML += `
            <tr>
                <td>${producto.id}</td>
                <td>${producto.nombre}</td>
                <td>${producto.tamano}</td>
                <td>${producto.cantidad}</td>
                <td>
                    ${new Date(
                        producto.fecha_registro
                    ).toLocaleString()}
                </td>
            </tr>
        `;
    });
}

formularioProducto.addEventListener(
    "submit",
    async function(evento)
    {
        evento.preventDefault();

        const datos = {
            nombre:
                document.getElementById(
                    "nombre"
                ).value,

            tamano:
                document.getElementById(
                    "tamano"
                ).value,

            cantidad:
                Number(
                    document.getElementById(
                        "cantidad"
                    ).value
                )
        };

        const respuesta = await fetch(
            "/api/productos",
            {
                method: "POST",

                headers: {
                    "Content-Type":
                        "application/json"
                },

                body: JSON.stringify(datos)
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

            formularioProducto.reset();

            document.getElementById(
                "cantidad"
            ).value = 0;

            cargarProductos();
        }
        else
        {
            mensaje.className =
                "mensaje mensaje-error";
        }
    }
);

document.addEventListener(
    "DOMContentLoaded",
    cargarProductos
);
```

---

# Bloque 13. Registrar entradas y salidas

## 33. Página `paginas/movimiento.html`

```html
<!DOCTYPE html>
<html lang="es">

<head>
    <meta charset="UTF-8">

    <meta
        name="viewport"
        content="width=device-width, initial-scale=1.0"
    >

    <title>Registrar movimiento</title>

    <link
        rel="stylesheet"
        href="/css/estilos.css"
    >
</head>

<body>

    <header>

        <div>
            <h1>Registrar movimiento</h1>

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
        <a href="/usuarios">Usuarios</a>
        <a href="/accesos">Accesos</a>
        <a href="/inventario">Inventario</a>
        <a href="/movimiento">Movimiento</a>
        <a href="/historial">Historial</a>
    </nav>

    <main>

        <section class="seccion">

            <h2>
                Entrada o salida de producto
            </h2>

            <div
                id="mensaje"
                class="mensaje"
            ></div>

            <form id="formularioMovimiento">

                <label for="producto">
                    Producto
                </label>

                <select
                    id="producto"
                    required
                ></select>

                <label for="tipo">
                    Tipo de movimiento
                </label>

                <select
                    id="tipo"
                    required
                >
                    <option value="entrada">
                        Entrada
                    </option>

                    <option value="salida">
                        Salida
                    </option>
                </select>

                <label for="cantidad">
                    Cantidad
                </label>

                <input
                    type="number"
                    id="cantidad"
                    min="1"
                    step="1"
                    value="1"
                    required
                >

                <button type="submit">
                    Registrar movimiento
                </button>

            </form>

        </section>

    </main>

    <script src="/js/comun.js"></script>
    <script src="/js/movimiento.js"></script>

</body>
</html>
```

---

## 34. JavaScript `public/js/movimiento.js`

```javascript
const formulario =
    document.getElementById(
        "formularioMovimiento"
    );

const mensaje =
    document.getElementById("mensaje");

async function cargarProductos()
{
    const respuesta = await fetch(
        "/api/productos"
    );

    const productos =
        await respuesta.json();

    const selector =
        document.getElementById(
            "producto"
        );

    selector.innerHTML = "";

    productos.forEach(producto =>
    {
        selector.innerHTML += `
            <option value="${producto.id}">
                ${producto.nombre}
                -
                Existencia: ${producto.cantidad}
            </option>
        `;
    });
}

formulario.addEventListener(
    "submit",
    async function(evento)
    {
        evento.preventDefault();

        const datos = {
            producto_id:
                Number(
                    document.getElementById(
                        "producto"
                    ).value
                ),

            tipo:
                document.getElementById(
                    "tipo"
                ).value,

            cantidad:
                Number(
                    document.getElementById(
                        "cantidad"
                    ).value
                )
        };

        const respuesta = await fetch(
            "/api/movimientos",
            {
                method: "POST",

                headers: {
                    "Content-Type":
                        "application/json"
                },

                body: JSON.stringify(datos)
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

            document.getElementById(
                "cantidad"
            ).value = 1;

            cargarProductos();
        }
        else
        {
            mensaje.className =
                "mensaje mensaje-error";
        }
    }
);

document.addEventListener(
    "DOMContentLoaded",
    cargarProductos
);
```

---

# Bloque 14. Consultar el historial de movimientos

## 35. Página `paginas/historial.html`

```html
<!DOCTYPE html>
<html lang="es">

<head>
    <meta charset="UTF-8">

    <meta
        name="viewport"
        content="width=device-width, initial-scale=1.0"
    >

    <title>Historial</title>

    <link
        rel="stylesheet"
        href="/css/estilos.css"
    >
</head>

<body>

    <header>

        <div>
            <h1>Historial de movimientos</h1>

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
        <a href="/usuarios">Usuarios</a>
        <a href="/accesos">Accesos</a>
        <a href="/inventario">Inventario</a>
        <a href="/movimiento">Movimiento</a>
        <a href="/historial">Historial</a>
    </nav>

    <main>

        <div class="tabla-contenedor">

            <table>

                <thead>
                    <tr>
                        <th>ID</th>
                        <th>Producto</th>
                        <th>Tamaño</th>
                        <th>Movimiento</th>
                        <th>Cantidad</th>
                        <th>Existencia final</th>
                        <th>Usuario</th>
                        <th>Fecha</th>
                    </tr>
                </thead>

                <tbody id="tablaMovimientos"></tbody>

            </table>

        </div>

    </main>

    <script src="/js/comun.js"></script>
    <script src="/js/historial.js"></script>

</body>
</html>
```

---

## 36. JavaScript `public/js/historial.js`

```javascript
async function cargarMovimientos()
{
    const respuesta = await fetch(
        "/api/movimientos"
    );

    if(!respuesta.ok)
    {
        return;
    }

    const movimientos =
        await respuesta.json();

    const tabla =
        document.getElementById(
            "tablaMovimientos"
        );

    tabla.innerHTML = "";

    movimientos.forEach(movimiento =>
    {
        tabla.innerHTML += `
            <tr>
                <td>${movimiento.id}</td>

                <td>
                    ${movimiento.producto}
                </td>

                <td>
                    ${movimiento.tamano}
                </td>

                <td>
                    ${movimiento.tipo}
                </td>

                <td>
                    ${movimiento.cantidad}
                </td>

                <td>
                    ${movimiento.existencia_resultante}
                </td>

                <td>
                    ${movimiento.usuario ||
                      "Usuario eliminado"}
                </td>

                <td>
                    ${new Date(
                        movimiento.fecha
                    ).toLocaleString()}
                </td>
            </tr>
        `;
    });
}

document.addEventListener(
    "DOMContentLoaded",
    cargarMovimientos
);
```

---

# Bloque 15. Ejecutar el sistema completo

## 37. Iniciar MySQL

Verificar que MySQL Server esté ejecutándose.

---

## 38. Iniciar Node.js

Desde la carpeta del proyecto ejecutar:

```bash
node server.js
```

Resultado esperado:

```text
Conexión correcta con MySQL
Servidor disponible en http://localhost:3000
```

---

## 39. Abrir el sistema

Abrir en el navegador:

```text
http://localhost:3000
```

---

# Bloque 16. Pruebas que deben realizarse

## Prueba 1. Registrar usuarios

Registrar al menos tres usuarios.

Comprobar desde MySQL:

```sql
SELECT * FROM usuarios;
```

---

## Prueba 2. Inicio de sesión incorrecto

Intentar iniciar sesión con:

* Un correo inexistente.
* Una contraseña incorrecta.

Consultar:

```sql
SELECT * FROM accesos;
```

Los intentos deben aparecer como:

```text
incorrecto
```

---

## Prueba 3. Inicio de sesión correcto

Iniciar sesión con un usuario registrado.

Consultar:

```sql
SELECT * FROM accesos;
```

Debe aparecer un registro con:

```text
correcto
```

---

## Prueba 4. Registrar productos

Registrar:

```text
Caja pequeña
Tamaño: pequeño
Cantidad inicial: 10
```

Registrar:

```text
Caja grande
Tamaño: grande
Cantidad inicial: 5
```

Consultar:

```sql
SELECT * FROM productos;
```

---

## Prueba 5. Registrar una entrada

Registrar:

```text
Producto: Caja pequeña
Movimiento: entrada
Cantidad: 4
```

La nueva existencia deberá ser:

```text
14
```

---

## Prueba 6. Registrar una salida

Registrar:

```text
Producto: Caja pequeña
Movimiento: salida
Cantidad: 3
```

La nueva existencia deberá ser:

```text
11
```

---

## Prueba 7. Intentar una salida inválida

Intentar retirar una cantidad mayor que la disponible.

Ejemplo:

```text
Existencia actual: 5
Salida solicitada: 20
```

El servidor deberá responder:

```text
No hay suficientes existencias
```

La cantidad almacenada no deberá modificarse.

---

## Prueba 8. Persistencia

Detener el servidor:

```text
Ctrl + C
```

Volver a iniciarlo:

```bash
node server.js
```

Los usuarios, productos y movimientos deben continuar apareciendo porque están almacenados en MySQL.

---

# 40. Consultas SQL de comprobación

## Consultar usuarios

```sql
SELECT
    id,
    nombre,
    correo,
    fecha_registro
FROM usuarios;
```

## Consultar accesos

```sql
SELECT
    id,
    correo_intentado,
    resultado,
    direccion_ip,
    fecha
FROM accesos
ORDER BY fecha DESC;
```

## Consultar inventario

```sql
SELECT
    id,
    nombre,
    tamano,
    cantidad
FROM productos;
```

## Consultar historial completo

```sql
SELECT
    movimientos.id,
    productos.nombre AS producto,
    movimientos.tipo,
    movimientos.cantidad,
    movimientos.existencia_resultante,
    usuarios.nombre AS usuario,
    movimientos.fecha

FROM movimientos

INNER JOIN productos
    ON movimientos.producto_id =
       productos.id

LEFT JOIN usuarios
    ON movimientos.usuario_id =
       usuarios.id

ORDER BY movimientos.fecha DESC;
```

---

# 41. Flujo de registro de un usuario

```text
El usuario llena registro.html
             │
             ▼
registro.js lee los controles
             │
             ▼
fetch POST /api/usuarios
             │
             ▼
server.js recibe los datos
             │
             ▼
bcrypt protege la contraseña
             │
             ▼
INSERT INTO usuarios
             │
             ▼
MySQL guarda permanentemente al usuario
```

---

# 42. Flujo de inicio de sesión

```text
El usuario llena login.html
             │
             ▼
login.js envía correo y contraseña
             │
             ▼
server.js busca el correo en MySQL
             │
             ▼
bcrypt compara la contraseña
             │
             ├── Incorrecta → registra acceso incorrecto
             │
             └── Correcta → crea la sesión
```

---

# 43. Flujo de un movimiento

```text
El usuario selecciona un producto
             │
             ▼
Selecciona entrada o salida
             │
             ▼
movimiento.js envía los datos
             │
             ▼
server.js consulta la existencia
             │
             ▼
Calcula la nueva cantidad
             │
             ▼
Actualiza productos
             │
             ▼
Registra el movimiento
```

---

