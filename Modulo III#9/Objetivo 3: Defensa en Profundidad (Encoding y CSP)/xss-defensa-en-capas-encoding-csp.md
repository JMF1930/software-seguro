# Defensa en capas contra XSS: output encoding y CSP

Un arquitecto de seguridad no confía en una sola barrera. Contra XSS, las dos capas que se complementan son la codificación de salida en el código y una Content Security Policy en las cabeceras HTTP. La primera evita que la inyección ocurra; la segunda limita el daño si la primera falla en algún punto. Conviene tener las dos, porque hacen cosas distintas en momentos distintos.

## Capa 1: output encoding, la defensa primaria

El XSS viene de que un dato del usuario termina interpretándose como código por el navegador. La codificación de salida corta eso en el momento exacto en que el dato se inserta en la página: convierte los caracteres que tienen significado especial en su representación inofensiva, de modo que el navegador los muestre como texto y nunca los ejecute.

En contexto HTML, los caracteres peligrosos se transforman en entidades:

```
<   →   &lt;
>   →   &gt;
"   →   &quot;
'   →   &#x27;
&   →   &amp;
```

Si un atacante manda `<img src=x onerror=alert(1)>` y lo codificás antes de insertarlo, en el HTML queda:

```
&lt;img src=x onerror=alert(1)&gt;
```

El navegador muestra ese texto literal en la pantalla. No construye ninguna etiqueta `<img>`, así que el `onerror` nunca existe como código. El payload quedó neutralizado sin necesidad de adivinar si era peligroso: lo tratás siempre como dato.

### La codificación depende del contexto

Acá está el punto que más se equivoca la gente. No alcanza con "codificar". Hay que codificar según dónde cae el dato, porque cada contexto del documento tiene sus propias reglas de escape. El mismo valor que es seguro en un lugar puede ejecutar en otro.

Los contextos principales son estos:

Dentro del cuerpo HTML (`<div>DATO</div>`), se usa la codificación de entidades de arriba.

Dentro de un atributo (`<input value="DATO">`), hay que codificar las comillas, porque si el dato cierra la comilla del atributo puede agregar un `onfocus` o un `onmouseover` propio. Por eso conviene además poner siempre los atributos entre comillas.

Dentro de un bloque de JavaScript (`<script>var x = "DATO";</script>`), la codificación HTML no sirve; necesitás escape de JavaScript (por ejemplo `\x3C` en vez de `<`), porque el parser que corre ahí es el de JS, no el de HTML. Meter datos del usuario adentro de un `<script>` es peligroso de por sí, y lo mejor es evitarlo.

Dentro de una URL (`<a href="/ver?x=DATO">`), se usa URL encoding, y hay que validar el esquema para bloquear `javascript:`.

Dentro de CSS (`<style>` o atributos `style`), hay reglas de escape de CSS aparte.

Si aplicás la codificación de un contexto en otro, dejás un hueco. Por eso la regla práctica del arquitecto es: codificación consciente del contexto (context-aware output encoding), no un escape genérico aplicado a ciegas.

### Apoyarse en el framework

Escribir estas reglas a mano es propenso a error. Los frameworks modernos ya traen escape automático según contexto: React, Angular, Vue, y las plantillas del lado del servidor como Jinja2 con autoescape, Razor o Thymeleaf escapan la salida por defecto. El trabajo del arquitecto es asegurarse de que nadie desactive esa protección sin pensarlo (con cosas como `dangerouslySetInnerHTML` en React o `|safe` en Jinja), y que cuando de verdad haga falta insertar HTML del usuario, se sanitice antes con una librería probada como DOMPurify.

Con esto bien hecho, el XSS reflejado y el almacenado quedan cerrados en origen. Pero "bien hecho en todos los puntos de la aplicación" es una condición fuerte. Una sola plantilla donde alguien se olvidó de escapar, o un contexto mal codificado, reabre el agujero. De ahí la segunda capa.

## Capa 2: CSP, la red de contención

La Content Security Policy es una cabecera HTTP que el servidor manda con cada respuesta, y que le dice al navegador qué fuentes de contenido puede ejecutar y cuáles tiene que ignorar. No evita que un payload se inyecte; evita que se ejecute si llegó a inyectarse. Es defensa en profundidad: la asumís como si la codificación fallara en algún lado.

Una cabecera se ve así:

```
Content-Security-Policy: default-src 'self'; script-src 'self'; object-src 'none'; base-uri 'none'
```

Lo que logra contra XSS depende de una decisión central: bloquear el JavaScript inline. La mayoría de los payloads de XSS dependen de script inline, sea un `<script>alert(1)</script>` o un manejador de evento como `onerror=`. Si tu política no incluye `'unsafe-inline'` en `script-src`, el navegador se niega a ejecutar cualquiera de esos dos. Un `<img onerror=...>` que se haya colado igual queda inerte, porque el navegador no corre el handler inline.

El problema es que casi todas las aplicaciones tienen algún script inline legítimo, y bloquearlo todo las rompe. La forma correcta de permitir solo los tuyos es con nonces o hashes, no con `'unsafe-inline'`:

```
Content-Security-Policy: script-src 'nonce-r4nd0m2024' 'strict-dynamic'; object-src 'none'; base-uri 'none'
```

Con esto, el servidor genera un nonce nuevo y aleatorio en cada carga de página y lo pone en sus propios scripts:

```html
<script nonce="r4nd0m2024"> ... tu código ... </script>
```

El navegador ejecuta solo los scripts que llevan el nonce correcto de esa carga. Un script inyectado por un atacante no conoce el nonce (cambia en cada respuesta), así que el navegador lo rechaza. El `'strict-dynamic'` deja que un script de confianza cargue a otros sin tener que listar cada dominio a mano, lo que hace la política mantenible.

Dos directivas más que conviene fijar: `object-src 'none'` corta plugins viejos como Flash que son vector de ejecución, y `base-uri 'none'` evita que un atacante inyecte una etiqueta `<base>` para secuestrar las URLs relativas.

### Ponerla en marcha sin romper el sitio

Una CSP estricta mal medida rompe funcionalidad, así que el arquitecto la despliega primero en modo observación:

```
Content-Security-Policy-Report-Only: script-src 'nonce-...'; report-uri /csp-reports
```

En este modo el navegador no bloquea nada, solo reporta a la URL indicada qué habría bloqueado. Con esos reportes ajustás la política hasta que no genere falsos positivos, y recién ahí la pasás a modo de bloqueo real quitando el `-Report-Only`. Los reportes, además, quedan como un canal de alerta: si empiezan a llegar violaciones de `script-src`, puede ser que alguien esté probando inyectar algo.

## Por qué las dos juntas y no una sola

Las capas cubren fallas distintas. La codificación de salida evita la inyección, pero depende de que esté aplicada correctamente en cada punto de salida de la aplicación, y basta un olvido para abrir un hueco. La CSP no evita la inyección, pero hace que un script inyectado no se ejecute, cubriendo justo ese olvido. Ninguna es perfecta sola: una codificación impecable hace innecesaria a la CSP, pero no podés garantizar que sea impecable en todos lados; una CSP fuerte contiene mucho, pero tiene bypasses conocidos y no debería ser tu única barrera.

Juntas, un atacante necesita que fallen las dos a la vez: encontrar un punto sin codificar y además una forma de ejecutar que la CSP permita. Esa es la idea de defensa en profundidad, y es lo que convierte al XSS de un riesgo común en uno que requiere encadenar varios errores del lado defensor.

## En una línea

La codificación de salida consciente del contexto cierra el XSS en el código, tratando cada dato del usuario como texto inerte según dónde se inserta. La CSP, en las cabeceras, actúa de red: si algo se escapó, impide que el script inyectado corra, sobre todo bloqueando el JavaScript inline y permitiendo solo scripts con nonce. La primera previene, la segunda contiene, y el valor está en tener las dos.
