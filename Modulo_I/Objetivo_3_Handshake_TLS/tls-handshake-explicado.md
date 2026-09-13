# El TLS Handshake: qué es y cómo funciona

## 1. Contexto: ¿por qué existe este proceso?

Cuando un navegador quiere hablar con un servidor de forma segura (HTTPS), no puede simplemente abrir la conexión TCP y empezar a mandar el pedido HTTP en texto plano. Antes de que viaje un solo byte de la petición HTTP, ambas partes necesitan:

1. Verificar que están hablando con quien dicen ser (autenticación).
2. Ponerse de acuerdo en cómo van a cifrar los datos que intercambien.
3. Generar una clave secreta compartida que nadie más pueda conocer, aunque haya estado espiando toda la conversación.

A ese proceso previo se le llama **TLS Handshake** (apretón de manos TLS). TLS (Transport Layer Security) es el protocolo que se ubica entre la capa de transporte (TCP) y la capa de aplicación (HTTP), y es el sucesor de SSL. Todo esto ocurre *antes* de la primera línea `GET / HTTP/1.1`.

## 2. Las dos preguntas que el handshake tiene que resolver

El handshake resuelve dos problemas al mismo tiempo:

- **Confianza**: ¿el servidor con el que hablo es realmente el dominio que quiero visitar, o es un impostor haciendo un ataque "man-in-the-middle"?
- **Secreto compartido**: ¿cómo generamos una clave simétrica que solo el navegador y el servidor conocen, si toda la negociación viaja por una red pública donde cualquiera puede escuchar?

Para resolver esto se combinan certificados digitales (autenticación) y dos tipos de criptografía (cifrado asimétrico y simétrico), que explico más abajo.

## 3. Paso a paso del handshake (versión TLS 1.3, la actual)

Voy a describirlo en su forma moderna (TLS 1.3), que es más rápida que TLS 1.2, aunque la lógica de fondo es la misma.

### Paso 1 — ClientHello
El navegador abre la conexión TCP (three-way handshake) y, ya sobre esa conexión, envía un mensaje `ClientHello` que incluye:
- Las versiones de TLS que soporta.
- La lista de "cipher suites" (combinaciones de algoritmos de cifrado) que puede usar.
- Un número aleatorio generado por el cliente (*client random*).
- En TLS 1.3, además, el cliente ya se "adelanta" y envía sus parámetros de intercambio de claves (por ejemplo, su clave pública temporal de Diffie-Hellman), apostando a qué algoritmo va a elegir el servidor. Esto ahorra una ronda completa de ida y vuelta comparado con TLS 1.2.

### Paso 2 — ServerHello + certificado
El servidor responde con:
- La versión de TLS y el cipher suite elegidos.
- Su propio número aleatorio (*server random*).
- Su parte del intercambio de claves (su clave pública temporal).
- **Su certificado digital**, que contiene su clave pública y está firmado por una Autoridad Certificadora (CA).
- Una firma digital que prueba que el servidor efectivamente posee la clave privada correspondiente a ese certificado.

Todo esto (salvo el ClientHello/ServerHello iniciales) ya viaja cifrado en TLS 1.3, porque apenas ambas partes intercambian sus valores de Diffie-Hellman ya pueden derivar claves de sesión provisorias.

### Paso 3 — Verificación del certificado
El navegador toma el certificado del servidor y valida:
- Que esté firmado por una CA en la que el sistema operativo o el navegador confían (la cadena de confianza sube hasta un certificado raíz preinstalado).
- Que no esté vencido ni revocado.
- Que el nombre de dominio del certificado coincida con el dominio al que se está conectando.

Si algo de esto falla, el navegador corta la conexión y muestra la típica advertencia de "conexión no segura".

### Paso 4 — Generación de la clave de sesión
Usando el algoritmo de intercambio de claves (normalmente **ECDHE**, Diffie-Hellman sobre curvas elípticas con claves efímeras), ambas partes —cliente y servidor— combinan su propia clave privada temporal con la clave pública temporal que recibieron del otro lado. El resultado matemático de esa combinación es idéntico en ambos extremos, aunque nunca viajó por la red como tal. Ese valor se usa para derivar las **claves simétricas de sesión**.

Que las claves sean "efímeras" (se generan nuevas en cada conexión y se descartan después) es lo que da **forward secrecy**: si en el futuro alguien roba la clave privada del servidor, no podrá descifrar conversaciones pasadas, porque esas claves de sesión ya no existen en ningún lado.

### Paso 5 — Finished
Ambas partes envían un mensaje `Finished`, cifrado con las claves recién derivadas, que resume (mediante un hash) todo lo negociado hasta ese punto. Esto sirve para confirmar que nadie manipuló ningún mensaje anterior del handshake (protección de integridad).

A partir de aquí, el canal queda establecido con cifrado simétrico, y recién ahí el navegador manda la primera petición HTTP, ya viajando protegida dentro de ese túnel.

## 4. El rol de los certificados digitales

El certificado digital es, en esencia, una tarjeta de identidad firmada digitalmente. Contiene:
- El nombre del dominio (u organización) al que pertenece.
- Su clave pública.
- La firma de la Autoridad Certificadora que garantiza esos datos.
- Fechas de validez.

Su función en el handshake es puramente de **autenticación**, no de cifrado de los datos de la aplicación. El certificado le permite al navegador responder a la pregunta: "¿esta clave pública realmente pertenece al servidor con el que quiero hablar, o me la está entregando un atacante que se puso en el medio?". Sin esta verificación, el intercambio de claves (Diffie-Hellman) seguiría funcionando matemáticamente, pero no protegería contra un ataque man-in-the-middle: un atacante podría hacerse pasar por el servidor real, negociar una clave con el cliente, y otra con el servidor legítimo, quedando en el medio sin que nadie lo note. El certificado, respaldado por una cadena de confianza hacia una CA raíz, es lo que rompe esa posibilidad.

## 5. Por qué se usan dos tipos de cifrado en una misma conexión

Este es el punto clave del diseño de TLS: **combina lo mejor de dos mundos criptográficos** porque ninguno de los dos, solo, es una buena solución.

### Cifrado asimétrico (clave pública / clave privada)
- Permite que dos partes que nunca se vieron antes, sobre un canal inseguro, puedan autenticarse y acordar un secreto sin haber compartido nada previamente.
- El problema: es **computacionalmente costoso**. Los algoritmos como RSA o Diffie-Hellman sobre curvas elípticas requieren operaciones matemáticas mucho más pesadas que el cifrado simétrico. Si se usara para cifrar todo el tráfico de una página web (imágenes, HTML, video, etc.), el rendimiento sería inaceptable, especialmente en servidores con miles de conexiones simultáneas.

### Cifrado simétrico (misma clave para cifrar y descifrar)
- Algoritmos como AES son órdenes de magnitud más rápidos y eficientes en CPU.
- El problema: ambas partes necesitan la **misma clave secreta**, y no hay forma segura de "mandársela" por un canal abierto sin que alguien la intercepte. Si simplemente se enviara la clave simétrica en texto plano al inicio, cualquiera que esté espiando la red la obtendría y podría descifrar todo.

### La solución combinada
TLS usa el cifrado asimétrico únicamente para el **problema difícil**: autenticar al servidor (vía certificado) y acordar de forma segura un secreto compartido (vía Diffie-Hellman), sin que ese secreto viaje nunca expuesto por la red. Una vez resuelto ese problema puntual —que ocurre una sola vez al principio de la conexión—, ambas partes ya tienen una clave simétrica idéntica derivada de forma independiente. A partir de ahí, todo el tráfico real de la aplicación (la petición HTTP, la respuesta, cookies, etc.) se cifra con esa clave simétrica, que es rápida y escalable para mover grandes volúmenes de datos durante toda la sesión.

En otras palabras: **lo asimétrico resuelve "cómo confiar y cómo ponernos de acuerdo sin canal seguro previo"; lo simétrico resuelve "cómo cifrar mucho tráfico de forma eficiente una vez que ya nos pusimos de acuerdo"**. Usar solo uno de los dos dejaría, o bien una autenticación imposible de lograr de forma práctica, o bien un sistema demasiado lento para uso real en la web.

## 6. Resumen visual del flujo

```
Cliente                                              Servidor
   |------------------- ClientHello ------------------->|
   |         (versiones TLS, cipher suites,              |
   |          random del cliente, clave DH efímera)      |
   |                                                      |
   |<------------------ ServerHello ---------------------|
   |   (cipher elegido, random del servidor,             |
   |    clave DH efímera, Certificado, firma)            |
   |                                                      |
   [Verificación del certificado contra la CA raíz]
   [Ambos derivan la misma clave simétrica vía DH]
   |                                                      |
   |------------------- Finished ------------------------>|
   |<------------------ Finished -------------------------|
   |                                                      |
   |========= Canal cifrado con clave simétrica =========|
   |                                                      |
   |---------------- GET / HTTP/1.1 (cifrado) ----------->|
```

## 7. Conclusión

El TLS Handshake es el mecanismo que permite que la web sea usable de forma segura a gran escala: resuelve el problema de la confianza (mediante certificados digitales emitidos por CAs) y el problema de la eficiencia (combinando cifrado asimétrico para el intercambio inicial de claves con cifrado simétrico para el tráfico real). Recién cuando todo este proceso termina —típicamente en una o dos idas y vueltas de red con TLS 1.3— el navegador envía la primera petición HTTP, ya protegida dentro del canal cifrado que se acaba de negociar.
