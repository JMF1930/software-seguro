# Proxy vs. VPN: qué son y en qué se diferencian

## 1. ¿Qué es un proxy?

Un **proxy** es un servidor intermediario que se ubica entre el cliente (por ejemplo, el navegador de un usuario) y el servidor de destino. En lugar de que el cliente se conecte directamente al servidor, la petición pasa primero por el proxy, que la reenvía en su nombre, recibe la respuesta, y se la devuelve al cliente.

Desde el punto de vista del servidor de destino, la petición parece venir del proxy, no del cliente original. Esto es lo que permite, por ejemplo, ocultar la dirección IP real del cliente.

### Características principales de un proxy
- Actúa **a nivel de aplicación** (generalmente HTTP/HTTPS), aunque existen proxies para otros protocolos (SOCKS, por ejemplo, que es más genérico y opera a nivel de sesión/transporte).
- Normalmente se configura **por aplicación**: puedo configurar el proxy solo en el navegador, y el resto del tráfico del sistema operativo (otras apps, actualizaciones, etc.) sigue su camino normal sin pasar por él.
- No necesariamente cifra el tráfico. Un proxy HTTP simple solo reenvía las peticiones tal cual; si la conexión original ya usaba HTTPS, ese tramo sigue cifrado extremo a extremo entre cliente y servidor (el proxy solo ve el túnel TLS), pero el proxy en sí no agrega cifrado adicional.
- Casos de uso típicos: saltar restricciones geográficas para un sitio puntual, cachear contenido para acelerar accesos repetidos, filtrar contenido en una red corporativa, balancear carga (proxy inverso o *reverse proxy*), u ocultar la IP de origen para una tarea específica.

### Tipos comunes de proxy
- **Proxy directo (forward proxy)**: representa al cliente frente a servidores externos. Es el caso típico cuando hablamos de "usar un proxy" para navegar.
- **Proxy inverso (reverse proxy)**: se ubica del lado del servidor y representa al servidor frente a los clientes (por ejemplo, Nginx o Cloudflare distribuyendo tráfico hacia varios servidores backend). El cliente ni siquiera sabe que existe.

## 2. ¿Qué es una VPN?

Una **VPN (Virtual Private Network)** crea un **túnel cifrado a nivel de sistema operativo/red**, no solo para una aplicación puntual. Todo el tráfico que sale del dispositivo —sin importar la aplicación que lo genere (navegador, correo, apps móviles, juegos, etc.)— pasa por ese túnel cifrado hasta llegar a un servidor VPN, que recién ahí lo reenvía a su destino final en internet.

### Características principales de una VPN
- Opera a nivel de **red (capa 3)**, no a nivel de aplicación. Por eso captura todo el tráfico del dispositivo, no solo el de un navegador.
- **Cifra el tráfico entre el dispositivo y el servidor VPN**, independientemente de si la aplicación original usaba HTTPS o no. Esto es clave en redes no confiables, como el Wi-Fi público de un bar o aeropuerto.
- Requiere instalar un cliente o configurar el sistema operativo (no basta con configurar una sola app).
- Casos de uso típicos: acceder de forma segura a la red interna de una empresa desde afuera (VPN corporativa), proteger todo el tráfico en una red pública insegura, o enmascarar la ubicación/IP para todo el uso de internet del dispositivo.

## 3. Diferencias clave

| Aspecto | Proxy | VPN |
|---|---|---|
| **Nivel de operación** | Capa de aplicación (por app o protocolo específico) | Capa de red (todo el sistema) |
| **Alcance** | Generalmente una sola app o protocolo configurado | Todo el tráfico del dispositivo |
| **Cifrado** | No cifra por sí mismo (depende de si la app ya usaba HTTPS) | Cifra el túnel completo entre el dispositivo y el servidor VPN |
| **Configuración** | Se configura por aplicación (ej. en el navegador) | Se configura a nivel de sistema operativo |
| **Oculta la IP** | Sí, para el tráfico que pasa por él | Sí, para todo el tráfico del dispositivo |
| **Protege contra espionaje en la red local** | No necesariamente (si la app no usa TLS, el proxy no agrega protección) | Sí, porque todo el tráfico va cifrado desde el dispositivo |
| **Uso típico** | Acceder a un recurso puntual, cachear, filtrar, balancear carga | Proteger toda la conexión, acceso remoto seguro a una red privada |

## 4. Una analogía

- Un **proxy** es como pedirle a un mensajero de confianza que entregue *una* carta puntual en tu nombre: él la lleva, pero cómo viaje esa carta (si va en un sobre cerrado o abierto) depende de cómo la preparaste vos antes de dársela.
- Una **VPN** es como meter *todas* tus cartas, de todos los remitentes, dentro de un camión blindado que va directo hasta una oficina central, y recién ahí cada carta sigue su camino individual. No importa si el sobre original estaba bien cerrado o no: mientras viaja dentro del camión, está protegido.

## 5. Conclusión

Ambas tecnologías actúan como intermediarios que pueden ocultar la IP real del usuario, pero resuelven problemas distintos: el **proxy** es una solución puntual, generalmente por aplicación, y no necesariamente agrega cifrado propio. La **VPN** es una solución integral a nivel de sistema, que cifra todo el tráfico saliente del dispositivo hacia el servidor VPN, siendo más adecuada cuando se necesita seguridad completa (por ejemplo, en redes públicas no confiables) y no solo anonimato o acceso a un recurso específico.
