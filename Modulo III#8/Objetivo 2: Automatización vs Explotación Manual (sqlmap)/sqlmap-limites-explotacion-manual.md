# Dónde falla sqlmap y el pentester manual gana

sqlmap automatiza lo repetitivo de una inyección SQL: detecta el punto de inyección, prueba técnicas, arma los payloads y extrae datos. Funciona bien cuando la petición es simple y el punto de inyección está donde la herramienta espera encontrarlo. El problema aparece cuando la aplicación mete lógica entre el navegador y la consulta: tokens que cambian, parámetros transformados, flujos de varios pasos. Ahí sqlmap, que trata cada petición de forma bastante mecánica, se queda afuera. Un atacante con Burp Suite entiende el contexto y arma la petición a mano.

Van cuatro escenarios reales.

## 1. Tokens anti-CSRF y valores que cambian en cada petición

Muchas aplicaciones meten un token anti-CSRF en cada formulario, o un nonce que solo sirve una vez. El servidor rechaza cualquier POST cuyo token no coincida con el que acaba de emitir. El parámetro vulnerable (digamos `search`) viaja en el mismo cuerpo que ese token.

sqlmap, a menos que lo configures con cuidado, dispara cientos de peticiones reutilizando el mismo cuerpo. Desde la segunda, el token ya venció y el servidor devuelve 403 o redirige al login. sqlmap ve respuestas uniformes de error y concluye que el parámetro no es inyectable. Un falso negativo.

sqlmap tiene `--csrf-token` y `--csrf-url` para esto, pero dependen de que el token tenga un nombre y una ubicación estándar. Cuando el token está anidado en un JSON, derivado de un valor de sesión, o atado a un header particular, la opción no alcanza.

En Burp, el flujo manual es directo. Usás la pestaña Repeater: pedís una página nueva, extraés el token fresco de la respuesta, lo pegás en la siguiente petición junto con tu payload, y la mandás. Una petición, un token válido, una respuesta observable. Si querés iterar, la macro de Burp (Session handling rules) regenera el token antes de cada envío y automatiza eso mismo sin perder validez. El ser humano ve que el token es el problema; sqlmap solo ve 403.

## 2. Inyección de segundo orden (stored)

Esta es la que más seguido se le escapa a una herramienta automática, por cómo está construida.

En una inyección de segundo orden, el payload entra por un lugar y se ejecuta en otro, más tarde. El caso típico: te registrás con un nombre de usuario como `admin'--`. Al registrarte, la aplicación guarda ese string de forma segura, con la consulta parametrizada. No pasa nada. El `INSERT` no se rompe.

El problema viene después. Cuando entrás a "editar mi perfil", otra parte del código lee ese nombre de la base y lo concatena sin escapar en una consulta nueva, por ejemplo un `UPDATE ... WHERE username = 'admin'--'`. Ahí recién dispara.

sqlmap trabaja observando la respuesta a la misma petición donde inyectó. Mete el payload en el registro, mira la respuesta del registro, no ve nada raro, y pasa de largo. No tiene forma de saber que el efecto aparece tres clics más adelante, en un endpoint distinto, en otro momento. La herramienta carece del modelo mental de "esto se guarda acá y se usa allá".

El pentester manual sí lo tiene. El flujo es: inyectar en el registro, navegar hasta la pantalla de edición, y observar si ahí algo se rompe o cambia. Una vez confirmado el camino de entrada y el de ejecución, Burp sirve para repetir el ciclo variando el payload almacenado. sqlmap incluso puede ayudar en la fase de extracción una vez que sabés exactamente qué endpoint es el que ejecuta, pero el descubrimiento del flujo de dos tiempos es trabajo humano.

## 3. Parámetros transformados antes de llegar al servidor

Cada vez más aplicaciones no mandan el parámetro en texto plano. Lo envuelven: base64, un JSON serializado con firma, un blob cifrado del lado del cliente, o una estructura anidada propia.

Supongamos que el `id` viaja así:

```
POST /api/item
{"data":"eyJpZCI6NX0="}    // base64 de {"id":5}
```

El campo inyectable real es el `id` dentro de ese JSON codificado. sqlmap, apuntado al parámetro `data`, inyecta dentro del base64. El resultado es un base64 inválido, o un JSON malformado que el servidor descarta antes de que la consulta se arme. El payload nunca toca la base. sqlmap concluye que no hay inyección.

sqlmap ofrece tamper scripts para codificaciones conocidas, y con `--prefix`/`--suffix` podés ajustar la forma del payload. Pero cuando la transformación es a medida (una firma HMAC sobre el cuerpo, un esquema de cifrado propio, varias capas combinadas), no hay tamper que lo cubra, porque la herramienta no puede reconstruir la firma ni rearmar la estructura.

El trabajo manual acá es de ingeniería inversa. Entendés cómo se arma el `data`: lo decodificás, ves el JSON, modificás el `id` con tu payload, volvés a codificar a base64, y si hay firma, la recalculás. Después mandás la petición bien formada desde Repeater. La extensión Hackvertor de Burp automatiza las capas de codificación y firma dentro de la misma petición, así podés iterar payloads sin rehacer el encoding a mano cada vez. sqlmap no sabe que tiene que decodificar, modificar y recodificar; el humano sí.

## 4. WAF que detecta a sqlmap por su comportamiento, no por el payload

Un WAF moderno no mira solo el contenido de cada petición. Mira el patrón: volumen, cadencia, User-Agent, el orden y la forma de las pruebas. sqlmap, por defecto, es ruidoso. Dispara una ráfaga de peticiones con una firma reconocible, prueba técnicas en una secuencia conocida, y usa un User-Agent delator si no lo cambiás. El WAF lo identifica como herramienta automática y empieza a bloquear o a servir respuestas falsas, que envenenan la detección de sqlmap.

sqlmap tiene contramedidas: `--random-agent`, `--delay`, `--tamper` para ofuscar, `--safe-url`. Ayudan, pero son una carrera contra reglas que también evolucionan, y muchas veces no alcanzan contra un WAF bien afinado que correlaciona comportamiento.

El atacante manual tiene la ventaja del bajo volumen. En vez de cientos de pruebas, manda unas pocas, muy pensadas, desde Repeater, con tiempos humanos entre una y otra. Puede estudiar qué variantes pasan el filtro probando una por una, ajustar la codificación a lo que ese WAF específico deja pasar, y quedar por debajo del umbral de detección. Donde sqlmap grita, el humano susurra. Una vez que encontró la variante que el WAF no bloquea, recién ahí puede pasarle esa sintaxis a sqlmap con un tamper a medida para la fase de extracción.

## El patrón de fondo

Los cuatro casos comparten la misma causa. sqlmap es un motor estadístico sobre peticiones HTTP: asume que el punto de inyección está en un parámetro visible, que la respuesta a esa misma petición revela el resultado, y que puede repetir la petición tal cual muchas veces. Cuando la aplicación rompe cualquiera de esos tres supuestos (el parámetro está transformado, el efecto aparece en otro lado, o la petición no se puede repetir idéntica), la herramienta se queda ciega.

El pentester entiende la aplicación como un sistema con estado y con lógica, no como una lista de parámetros. Por eso la recomendación práctica es la del título: sqlmap para lo mecánico, una vez que ya mapeaste a mano el contexto. La detección y el entendimiento del flujo siguen siendo trabajo manual, y Burp es la herramienta para ese trabajo.
