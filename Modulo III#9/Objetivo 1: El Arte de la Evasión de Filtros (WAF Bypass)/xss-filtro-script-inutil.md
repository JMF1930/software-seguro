# Por qué filtrar la etiqueta `<script>` no detiene un XSS

## El error conceptual

El desarrollador asume que un XSS necesita la etiqueta `<script>`. No la necesita. JavaScript se ejecuta en el navegador por muchos caminos, y los manejadores de eventos de HTML son el más directo: cualquier etiqueta puede llevar un atributo como `onerror`, `onload` u `onclick`, y cuando el evento se dispara, el navegador corre el código que pusiste ahí. La etiqueta `<script>` es apenas una de las entradas posibles.

Así que filtrar `<script>` y `</script>` deja abiertas casi todas las demás. El filtro apunta a una palabra concreta mientras el ataque vive en una familia entera de construcciones.

Hay además dos problemas técnicos que hacen al filtro todavía más débil.

El primero es la misma debilidad de toda lista negra: solo bloquea lo que alguien pensó en prohibir. El atacante solo necesita una variante que no esté contemplada, y en XSS hay cientos.

El segundo es más concreto, y hunde a los filtros ingenuos que hacen una sola pasada de reemplazo. Si el filtro elimina el texto `<script>` buscándolo una vez, podés anidarlo:

```html
<scr<script>ipt>alert(1)</scr</script>ipt>
```

El filtro encuentra el `<script>` del medio, lo borra, y los pedazos que quedaban a los costados se pegan y reconstruyen un `<script>` limpio:

```html
<script>alert(1)</script>
```

El filtro que iba a sacar la etiqueta terminó fabricándola. Esto pasa siempre que el reemplazo no se repite hasta que no queda nada que reemplazar.

## Dos payloads que ejecutan sin `<script>`

Estos son clásicos del cheat sheet de XSS y no usan la etiqueta en ningún momento.

### 1. Imagen con `onerror`

```html
<img src=x onerror="alert(document.cookie)">
```

La etiqueta `<img>` pide cargar una imagen desde `src=x`. Esa fuente no existe, así que la carga falla, y el navegador dispara el evento `onerror`. En ese atributo está tu código, que se ejecuta cuando el error ocurre. No hace falta `<script>`: el canal es el manejador de evento. El `document.cookie` del ejemplo es lo que convierte esto en robo de sesión cuando no hay flag `HttpOnly`.

### 2. SVG con `onload`

```html
<svg onload="alert(1)">
```

Un elemento `<svg>` admite el evento `onload`, que el navegador dispara apenas termina de procesar el elemento. No depende de ningún error ni de interacción del usuario: se ejecuta solo, al renderizar. Es uno de los vectores más limpios justamente porque no necesita que la víctima haga clic en nada.

### Más variantes, para ver el tamaño del problema

```html
<body onload="alert(1)">
<iframe src="javascript:alert(1)">
<a href="javascript:alert(1)">clic</a>
<input autofocus onfocus="alert(1)">
<details open ontoggle="alert(1)">
```

Entre manejadores de evento y la pseudo-URL `javascript:`, la cantidad de formas de ejecutar sin `<script>` es enorme. Ninguna lista negra las cubre todas.

## Cómo se arregla de verdad

El XSS, igual que la inyección SQL, viene de mezclar datos con código. Acá el dato del usuario termina interpretándose como HTML o JavaScript por el navegador. La defensa correcta no es buscar y borrar strings peligrosos, sino tratar la entrada siempre como dato inerte. Hay tres piezas.

Codificación de salida según el contexto (output encoding). La defensa de fondo. Cuando mostrás el comentario del usuario dentro de HTML, convertís los caracteres especiales a sus entidades: `<` pasa a `&lt;`, `>` a `&gt;`, `"` a `&quot;`. Así el navegador muestra `<img src=x onerror=...>` como texto literal en la pantalla y nunca lo interpreta como una etiqueta. El detalle importante es que la codificación depende de dónde insertás el dato: no se codifica igual dentro del cuerpo HTML, dentro de un atributo, dentro de un bloque `<script>` o dentro de una URL. Cada contexto tiene sus reglas.

Frameworks con escapado automático. React, Angular y las plantillas modernas (Jinja con autoescape, Razor, Thymeleaf) escapan la salida por defecto. Si no desactivás esa protección a mano (con cosas como `dangerouslySetInnerHTML` o `|safe`), el XSS reflejado y almacenado se corta solo. Apoyarse en el framework es más confiable que escribir tu propio filtro.

Sanitización con una librería probada, para cuando de verdad necesitás permitir HTML. Si el campo tiene que aceptar formato (un editor de texto rico, por ejemplo), no escribas tu propio limpiador: usá DOMPurify. Está hecho para esto, parsea el HTML como lo haría el navegador y deja solo las etiquetas y atributos de una lista blanca, descartando todo manejador de evento y toda URL peligrosa.

Como capa extra, una Content Security Policy (CSP) bien configurada limita el daño aunque algo se escape: podés prohibir el JavaScript inline, de modo que un `onerror` inyectado no corra aunque llegue al HTML. La CSP es defensa en profundidad, no el arreglo principal.

## En una línea

Filtrar `<script>` bloquea una sola puerta de un edificio lleno de ventanas: los manejadores de evento (`onerror`, `onload`) y la pseudo-URL `javascript:` ejecutan código sin esa etiqueta, y un filtro de una pasada hasta puede reconstruirla al borrar una versión anidada. La solución es codificar la salida según su contexto y tratar la entrada del usuario como dato, nunca como código.
