# Práctica: Diseño de interfaces web utilizando CSS

## Objetivo

Modificar la página web desarrollada en la práctica anterior para mejorar su apariencia visual mediante el uso de hojas de estilo CSS externas.

El estudiante aprenderá a separar el contenido de una página web de su presentación visual, utilizando archivos independientes para HTML y CSS, siguiendo las buenas prácticas utilizadas en el desarrollo de aplicaciones web industriales.

---

# 1. Introducción

En la práctica anterior se desarrolló una página web básica utilizando Node.js y Express.

Aunque la página cumplía su función, visualmente era muy simple.

En aplicaciones reales de Industria 4.0 es importante que la información se presente de manera clara y organizada.

Por ejemplo:

* Estado de sensores.
* Conteo de piezas.
* Estado de máquinas.
* Alarmas.
* Indicadores de producción.

Todo esto debe mostrarse de forma agradable y fácil de interpretar.

Para lograrlo se utiliza CSS.

---

# 2. ¿Qué es CSS?

CSS significa:

```text
Cascading Style Sheets
Hojas de Estilo en Cascada
```

CSS permite controlar:

* Colores.
* Tipografías.
* Bordes.
* Espaciados.
* Tamaños.
* Posiciones.
* Animaciones.
* Diseño visual general.

---

# 3. ¿Por qué separar HTML y CSS?

HTML define:

```text
Qué información existe
```

CSS define:

```text
Cómo se ve esa información
```

Ejemplo:

HTML:

```html
<h1>Industria 4.0</h1>
```

CSS:

```css
h1{
    color: blue;
    text-align: center;
}
```

Separar ambos archivos facilita:

* Mantenimiento.
* Organización.
* Reutilización.
* Trabajo en equipo.

---

# 4. Estructura del proyecto

La estructura ahora será:

```text
practica2_css
│
├── server.js
│
└── public
    ├── index.html
    └── styles.css
```

Observe que aparece un nuevo archivo:

```text
styles.css
```

---

# 5. Código del servidor

El servidor será exactamente el mismo que en la práctica anterior.

Archivo:

```text
server.js
```

```js
var express = require('express');
var app = express();

var puerto = 3000;

app.use(express.static('public'));

app.listen(puerto, function() {

    console.log("======================================");
    console.log(" Servidor web iniciado correctamente");
    console.log(" Carpeta pública: public");
    console.log(" Dirección: http://localhost:" + puerto);
    console.log("======================================");

});
```

---

# 6. Creación de la página HTML

Archivo:

```text
public/index.html
```

```html
<!DOCTYPE html>
<html lang="es">

<head>

    <meta charset="UTF-8">

    <title>Industria 4.0</title>

    <link rel="stylesheet" href="styles.css">

</head>

<body>

    <header>

        <h1>Industria 4.0</h1>

        <p>
            Panel de monitoreo industrial
        </p>

    </header>

    <main>

        <section class="tarjeta">

            <h2>Servidor Web</h2>

            <p>
                El servidor Node.js se encuentra funcionando correctamente.
            </p>

        </section>

        <section class="tarjeta">

            <h2>Estado del Sistema</h2>

            <ul>

                <li>Servidor: Activo</li>

                <li>Puerto: 3000</li>

                <li>Framework: Express</li>

            </ul>

        </section>

        <section class="tarjeta">

            <h2>Aplicaciones de Industria 4.0</h2>

            <ul>

                <li>Monitoreo de sensores</li>

                <li>Conteo de piezas</li>

                <li>Internet de las Cosas</li>

                <li>Mantenimiento predictivo</li>

            </ul>

        </section>

    </main>

</body>

</html>
```

---

# 7. Creación de la hoja de estilos

Archivo:

```text
public/styles.css
```

```css
body{

    margin:0;

    font-family:Arial, sans-serif;

    background:#eef2f5;

}

header{

    background:#1f3c88;

    color:white;

    padding:25px;

    text-align:center;

}

main{

    width:85%;

    max-width:1000px;

    margin:30px auto;

}

.tarjeta{

    background:white;

    padding:20px;

    margin-bottom:20px;

    border-radius:12px;

    box-shadow:0px 4px 12px rgba(0,0,0,0.15);

}

.tarjeta h2{

    color:#1f3c88;

}
```

---

# 8. Explicación del enlace CSS

La siguiente línea conecta la página HTML con la hoja de estilos:

```html
<link rel="stylesheet" href="styles.css">
```

Cuando el navegador carga la página:

```text
Solicita index.html
Solicita styles.css
Aplica los estilos
Muestra el resultado
```

---

# 9. Explicación de las clases

La siguiente línea:

```html
<section class="tarjeta">
```

asigna una clase llamada:

```text
tarjeta
```

La clase permite reutilizar el mismo diseño en múltiples elementos.

El estilo correspondiente es:

```css
.tarjeta{
    background:white;
    padding:20px;
}
```

Observe el punto:

```css
.tarjeta
```

El punto indica que se trata de una clase.

---

# 10. Resultado esperado

La página deberá mostrar:

```text
Encabezado azul
Tarjetas blancas
Sombras suaves
Bordes redondeados
Texto organizado
Aspecto profesional
```

Visualmente deberá parecerse más a un dashboard industrial que a una página básica.

---

# 11. Experimentos

Modificar el archivo CSS para probar:

## Experimento 1

Cambiar el color del encabezado:

```css
background:#00695c;
```

---

## Experimento 2

Cambiar el color del texto:

```css
color:orange;
```

---

## Experimento 3

Cambiar el radio de las esquinas:

```css
border-radius:25px;
```

---

## Experimento 4

Cambiar la sombra:

```css
box-shadow:0px 10px 20px rgba(0,0,0,0.3);
```

---

# 12. Actividad de comprobación

Agregar una nueva tarjeta llamada:

```text
Mi banda transportadora
```

La tarjeta deberá contener:

* Estado de la banda.
* Número de piezas.
* Nombre del operador.

Ejemplo:

```html
<section class="tarjeta">

    <h2>Mi Banda Transportadora</h2>

    <p>Estado: Activa</p>

    <p>Piezas procesadas: 0</p>

    <p>Operador: Juan Pérez</p>

</section>
```

---

# 13. Reto adicional

Investigar y aplicar:

```css
transition
hover
transform
```

para que las tarjetas cambien ligeramente cuando el mouse pase sobre ellas.

---

# 14. Conclusión

En esta práctica se aprendió a separar el contenido HTML de la presentación visual mediante hojas de estilo CSS.

El estudiante comprendió cómo los sistemas industriales modernos utilizan interfaces visuales organizadas para mostrar información de sensores y procesos.

La interfaz desarrollada servirá como base para la siguiente práctica, donde se integrará Socket.IO para recibir datos en tiempo real desde un ESP32 y actualizar el contenido de la página automáticamente.
