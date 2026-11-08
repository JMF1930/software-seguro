# Por qué el DOM-Based XSS es invisible para un WAF

## De dónde viene la diferencia

El XSS almacenado y el reflejado pasan por el servidor. En el almacenado, el payload se guarda en la base y vuelve en el HTML que el servidor arma. En el reflejado, el payload viaja en la petición y el servidor lo devuelve dentro de la respuesta. En los dos casos, el código malicioso aparece en el tráfico HTTP que el WAF o el IPS pueden inspeccionar, en la petición, en la respuesta, o en ambas.

El DOM-Based XSS no toca el servidor para ejecutarse. La vulnerabilidad vive entera en el JavaScript del lado del cliente: una entrada controlada por el atacante (la source) llega a una función que la interpreta como código o como HTML (la sink), y toda esa cadena ocurre dentro del navegador, después de que la página cargó. El servidor manda siempre la misma página inocente; el daño lo produce el script corriendo en el navegador de la víctima.

Ese es el primer motivo por el que un WAF no lo ve: el WAF inspecciona HTTP, y acá la explotación no está en el HTTP, está en el DOM.

## El fragmento `#` nunca llega al servidor

Acá está la pieza central, y es una regla del protocolo, no un truco.

En una URL, todo lo que va después del `#` es el fragmento (fragment identifier). El navegador lo trata como una referencia interna al documento, pensada originalmente para saltar a un ancla de la página. Y por diseño, el navegador no incluye el fragmento en la petición HTTP. Cuando pedís:

```
https://sitio.com/pagina#<img src=x onerror=alert(document.cookie)>
```

el navegador abre una conexión y manda únicamente:

```
GET /pagina HTTP/1.1
Host: sitio.com
```

El `#` y todo lo que le sigue se quedan en el navegador. No viajan por la red. El servidor nunca recibe ese payload, y por lo tanto el WAF que está delante del servidor tampoco. No hay nada que inspeccionar, porque el dato malicioso no cruzó el cable. Podés mandar el payload más obvio del mundo, con `<script>` y todo, y el WAF ni se entera.

## Qué pasa del lado del JavaScript

El fragmento, aunque no se mande al servidor, sí queda accesible para el JavaScript de la página a través de `location.hash`. El problema aparece cuando el código toma ese valor y lo mete en una sink peligrosa sin sanitizarlo.

Código vulnerable típico:

```javascript
// Lee el fragmento de la URL y lo escribe en la página
var seccion = location.hash.substring(1);   // saca el '#'
document.getElementById("contenido").innerHTML = seccion;
```

La intención del desarrollador era inocente: usar el `#` para mostrar una sección, algo como `pagina#bienvenida`. Pero `innerHTML` interpreta lo que le pasás como HTML. Si la víctima abre:

```
https://sitio.com/pagina#<img src=x onerror=alert(document.cookie)>
```

el JavaScript lee el fragmento, lo asigna a `innerHTML`, el navegador construye un elemento `<img>` con un `src` roto, se dispara `onerror`, y el código del atacante corre con la sesión de la víctima. Todo esto adentro del navegador, sin una sola petición que contenga el payload.

Las sources más comunes son las partes de la URL que el JavaScript puede leer: `location.hash`, `location.search`, `location.href`, y también cosas como `document.referrer` o mensajes de `postMessage`. Las sinks son las funciones que convierten un string en código o en HTML: `innerHTML`, `outerHTML`, `document.write()`, `eval()`, `setTimeout()` con un string, y la asignación a `location` con una URL `javascript:`. El patrón vulnerable es siempre una source que llega a una sink sin pasar por un saneamiento.

## Por qué esto deja ciego al WAF, en resumen

Se juntan dos cosas. Primero, la explotación ocurre en el cliente, así que ni la petición ni la respuesta del servidor contienen el ataque; no hay tráfico malicioso que filtrar. Segundo, cuando la source es el fragmento `#`, el payload directamente no sale del navegador, porque el protocolo no lo manda. Un IPS o un WAF que viven en la frontera de la red no tienen manera de observar algo que nunca pasó por ahí.

Vale aclarar que no todo DOM-XSS usa el fragmento. Si la source es `location.search` (el query string, que sí se envía), el WAF podría llegar a ver el payload en la petición. Pero sigue sin verlo ejecutarse: la respuesta del servidor es idéntica a la normal, así que un WAF que inspecciona salida no detecta nada raro, y el payload en el query puede ir ofuscado para esquivar las firmas. El caso del `#` es el más limpio porque elimina al servidor de la ecuación por completo.

## Cómo se defiende

Como la vulnerabilidad vive en el cliente, la defensa también tiene que estar en el cliente. Un WAF no es la herramienta.

Lo primero es elegir sinks seguras. Si solo querés mostrar texto, usá `textContent` o `innerText` en lugar de `innerHTML`: esas propiedades tratan el valor como texto plano y nunca lo interpretan como HTML, así que `<img onerror=...>` aparece escrito literalmente en la pantalla. Evitá por completo `eval()`, `document.write()` y `setTimeout`/`setInterval` con strings.

Cuando de verdad necesitás insertar HTML a partir de una entrada, sanitizalo con DOMPurify antes de pasarlo a la sink. Está hecho para esto y deja solo etiquetas y atributos de una lista blanca.

Como capas adicionales, Trusted Types (una API del navegador) obliga a que todo lo que llega a una sink peligrosa pase antes por una función de saneamiento, y hace que el patrón vulnerable falle de entrada. Y una Content Security Policy que prohíba el JavaScript inline limita el daño aunque algo se escape, porque un `onerror` inyectado no llega a ejecutarse.

## En una línea

El XSS almacenado y el reflejado dejan el payload en el tráfico HTTP, donde un WAF puede verlo; el DOM-Based XSS se ejecuta dentro del navegador, y cuando la entrada es el fragmento `#` el payload ni siquiera se envía al servidor. No hay nada en la red que inspeccionar, así que la defensa tiene que vivir en el código del cliente: sinks seguras, DOMPurify y Trusted Types.
