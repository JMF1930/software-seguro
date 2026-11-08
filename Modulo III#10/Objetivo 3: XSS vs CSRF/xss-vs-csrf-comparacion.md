# XSS contra CSRF: dos ataques con mecánicas opuestas

Los dos involucran el navegador de la víctima y los dos terminan afectando su cuenta, así que es fácil confundirlos. Pero funcionan al revés uno del otro. La diferencia se entiende mejor preguntando de quién abusa cada uno: a quién engaña para lograr su efecto.

## La diferencia de fondo: qué confianza explota cada uno

XSS abusa de la confianza que la víctima tiene en el sitio. El atacante logra que su código se ejecute dentro de la página legítima, así que el navegador de la víctima corre ese código creyendo que es parte del sitio en el que confía. El ataque va dirigido al cliente: inyecta y ejecuta JavaScript en el navegador de quien visita.

CSRF abusa de la confianza que el sitio tiene en el navegador de la víctima. El servidor confía en cualquier petición que llegue con la cookie de sesión válida, y el atacante aprovecha eso: fuerza al navegador de la víctima a mandar una petición que el servidor acepta como legítima. El ataque va dirigido al servidor: le hace ejecutar una acción autenticada que la víctima no pidió.

Esa es la oposición central. XSS mete código del atacante en el navegador de la víctima. CSRF hace que el navegador de la víctima mande una petición del atacante al servidor. Uno inyecta ejecución; el otro falsifica una solicitud.

## Comparación lado a lado

| Aspecto | XSS | CSRF |
|---|---|---|
| Confianza que explota | La de la víctima en el sitio | La del sitio en el navegador de la víctima |
| Qué logra el atacante | Ejecutar su JavaScript en el navegador de la víctima | Que el navegador de la víctima envíe una petición autenticada |
| Objetivo del ataque | El cliente (el navegador) | El servidor (la aplicación) |
| ¿Necesita que la víctima tenga sesión abierta? | No necesariamente | Sí, la sesión activa es la condición |
| ¿El atacante ve la respuesta? | Sí, su código lee el DOM y las respuestas | No, es un ataque "a ciegas"; dispara la acción pero no lee el resultado |
| Alcance de lo que puede hacer | Casi todo lo que el usuario puede hacer en la página, más robar datos | Solo acciones que cambian estado por una petición predecible |
| Causa raíz | Falta de codificación de salida: el dato se interpreta como código | El servidor autoriza solo con la cookie, que el navegador manda sola |
| Defensa principal | Output encoding según contexto + CSP | Tokens anti-CSRF + cookies SameSite |

## Por qué XSS es más poderoso

El alcance es la diferencia práctica más grande. En un XSS, el atacante tiene ejecución de código dentro de la página, así que controla el contexto completo de la víctima: lee el DOM, lee las respuestas del servidor, puede robar la cookie si no tiene `HttpOnly`, montar un formulario falso, registrar lo que se teclea, y disparar cualquier acción que el usuario podría hacer. Y como lee las respuestas, puede encadenar pasos: leer un token, usarlo, verificar que funcionó.

En un CSRF, el atacante trabaja a ciegas. Puede provocar una petición, pero no ve lo que el servidor responde, porque la política de mismo origen le impide a su página leer la respuesta de otro dominio. Por eso un CSRF sirve para acciones de un solo tiro y de forma predecible (cambiar el email, transferir un monto fijo, borrar algo), pero no para operaciones que requieran leer un valor del servidor y reaccionar. Es más acotado.

Hay una consecuencia importante de esto: un XSS rompe las defensas contra CSRF. Si el atacante tiene ejecución de JavaScript en la página (XSS), puede leer el token anti-CSRF del DOM e incluirlo en su petición, así que el token deja de protegerte. Por eso se dice que no alcanza con defenderse de CSRF si tenés XSS sin resolver; el XSS es el problema más grave de los dos.

## Dónde está la vulnerabilidad en cada uno

Conviene ubicar la causa, porque de ahí sale la defensa.

El XSS vive en cómo el sitio muestra los datos. Si la aplicación toma una entrada y la inserta en la página sin codificarla, el navegador la interpreta como código. Es un fallo de salida. Se arregla con codificación de salida consciente del contexto, reforzada con una CSP que impida ejecutar scripts no autorizados.

El CSRF vive en cómo el sitio autoriza las acciones. Si el servidor acepta una acción basándose solo en la cookie de sesión, que el navegador adjunta automáticamente, cualquier página puede disparar esa acción. Es un fallo de autorización de la petición. Se arregla exigiendo algo que una página externa no pueda conocer: un token anti-CSRF único por sesión, más cookies con `SameSite=Lax` o `Strict` para que la cookie no viaje en peticiones entre sitios.

Fijate que las defensas no se solapan. Codificar la salida no hace nada contra CSRF, porque el CSRF no inyecta nada. Un token anti-CSRF no hace nada contra XSS, porque el XSS puede leer el token. Son problemas distintos con soluciones distintas, y una aplicación seria necesita cubrir los dos.

## En una línea

XSS abusa de la confianza de la víctima en el sitio: ejecuta el código del atacante en el navegador de la víctima, es un fallo de salida, y el atacante ve lo que hace. CSRF abusa de la confianza del sitio en el navegador de la víctima: fuerza una petición autenticada que la víctima no pidió, es un fallo de autorización, y el atacante actúa a ciegas. Uno inyecta ejecución en el cliente, el otro falsifica una solicitud al servidor.
