# Objetivo 1 — Man-in-the-Middle Local (Configuración y Certificado CA)

## 1. Enunciado

Para analizar el tráfico web como lo haría un atacante, es necesario ubicarse en el medio de la comunicación entre el navegador y el servidor. Para ello se instala **Burp Suite Community** (proxy de intercepción), se configura el navegador para enrutar todo su tráfico a través del proxy local (`127.0.0.1:8080`) y se instala el certificado de la CA de la herramienta en el navegador, de modo que este confíe en los certificados TLS emitidos "al vuelo" por Burp para cada sitio interceptado.

El objetivo puntual es interceptar una petición real de uno de los laboratorios de *Software Seguro* y detenerla en tránsito, antes de que llegue al servidor.

**Formato de entrega:** enlace al presente archivo `.md`, documentando los *endpoints* encontrados junto con sus métodos HTTP y una explicación de la función de cada uno.

## 2. Herramienta y configuración utilizada

| Componente | Detalle |
|---|---|
| Proxy de intercepción | Burp Suite Community Edition v2026.8 |
| Listener del proxy | `127.0.0.1:8080` (configuración por defecto) |
| Certificado CA | Certificado de Burp instalado en el almacén de confianza del navegador, para permitir la re-firma TLS de los sitios interceptados sin advertencias de certificado inválido |
| Configuración del navegador | Tráfico HTTP/HTTPS enrutado a través del proxy local |
| Objetivo analizado | `softwareseguro.com.ar` / `app.softwareseguro.com.ar` (laboratorios de Software Seguro) |
| Usuario de prueba | Juan Manuel Fernandez (sesión autenticada, cookie de sesión + token Bearer) |

### 2.1. Burp Suite instalado

Verificación de la instalación correcta de Burp Suite Community, con un *live passive crawl* configurado sobre todo el tráfico que pase por el proxy (alcance: mismo dominio y URLs dentro del *scope* de la suite).

![Burp Suite Community instalado — tarea de crawl pasivo en vivo](./assets_mitm/01-burp-instalado.png)

### 2.2. Burp en funcionamiento — Site map

Con el proxy activo y el navegador enrutando tráfico a través de él, Burp construye el **Site map** (pestaña *Target*) a partir de todas las peticiones observadas. Se identifican cuatro hosts distintos involucrados en la carga del sitio:

- `softwareseguro.com.ar` — aplicación principal (analizada en detalle en la sección 4).
- `app.softwareseguro.com.ar` — subdominio de aplicación autenticada, donde se listan y ejecutan los laboratorios (ver secciones 3 y 4).
- `static.cloudflareinsights.com` — recurso de terceros de Cloudflare (telemetría / RUM).
- `cdn.tailwindcss.com` — CDN del framework CSS Tailwind, utilizado para los estilos del sitio.

![Burp en funcionamiento — Site map con las peticiones capturadas hacia softwareseguro.com.ar](./assets_mitm/02-burp-sitemap.png)

## 3. Intercepción de una petición "en vuelo"

### 3.1. Acción disparadora en el navegador

Desde la sesión autenticada en `app.softwareseguro.com.ar/index`, el listado de **Laboratorios** permite iniciar cada práctica mediante el botón **"Iniciar"**. Al presionarlo sobre el laboratorio *"Uso del inspector"* (categoría *Introducción*), el botón cambia a estado **"INICIANDO.."**, disparando en segundo plano la petición que se intercepta a continuación.

![Panel de laboratorios de Software Seguro — botón "Iniciar" disparando la carga del laboratorio](./assets_mitm/04-software-seguro-lab-list.png)

### 3.2. Petición retenida en el proxy

Utilizando la pestaña **Proxy → Intercept**, con el interceptor activado (*Intercept is on*), se capturó la petición **`POST /start-challenge`** hacia `https://app.softwareseguro.com.ar/` **antes de que fuera reenviada al servidor** (IP de destino `104.21.59.209:443`). En este punto, Burp retiene la petición en el proxy local, dando al operador la posibilidad de inspeccionarla, modificarla, reenviarla (**Forward**) o descartarla (**Drop**) — evidenciando de forma concreta la posición de *hombre en el medio* entre el navegador y el servidor.

En la petición interceptada se observa, además:

- **`Authorization: Bearer <JWT>`** — token de sesión enviado en cada llamada autenticada a la API de la aplicación.
- **`Cookie: _ga=...; _ga_MR6ZT8PJ4Q=...`** — cookies de analítica de Google Analytics, ajenas a la lógica de autenticación propia de la app.
- Una segunda petición en cola, **`POST /cdn-cgi/rum?`**, correspondiente al beacon de telemetría de Cloudflare (ver sección 4).

![Petición POST /start-challenge interceptada en Burp Proxy, retenida antes de llegar al servidor](./assets_mitm/03-burp-intercept-start-challenge.png)

## 4. Endpoints identificados

### 4.1. Host `softwareseguro.com.ar` (sitio público)

| Método | Endpoint | Código | Tipo / MIME | Descripción |
|---|---|---|---|---|
| `GET` | `/` | 200 | HTML (*"Software Seguro"*) | Página principal (home) de la aplicación. |
| `GET` | `/js/avisos.js` | 200 | script | Script encargado de mostrar avisos o notificaciones dentro de la interfaz. |
| `GET` | `/js/iniciar-validar.js` | 200 | script | Lógica de validación en cliente del formulario de inicio de sesión (login), previa al envío al backend. |
| `GET` | `/js/menu.js` | 200 | script | Script de comportamiento del menú de navegación del sitio. |
| `GET` | `/src/login/` | 200 | HTML (*"Login - Software Seguro"*) | Página del formulario de inicio de sesión de la aplicación. |
| `POST` | `/cdn-cgi/rum?` (×3, con parámetros) | 204 | — | Beacons de **Cloudflare Real User Monitoring (RUM)**: envían métricas de rendimiento/experiencia de carga de la página hacia Cloudflare. Respuesta `204 No Content` típica de telemetría *fire-and-forget*. |
| `GET` | `/cdn-cgi/rum` (sin query string) | — | — | Carga del script/loader del beacon de Cloudflare RUM referenciado más arriba (antes de que este dispare los `POST` con los datos de telemetría). |

### 4.2. Host `app.softwareseguro.com.ar` (aplicación autenticada)

| Método | Endpoint | Descripción |
|---|---|---|
| `GET` | `/index` | Panel autenticado con el listado de laboratorios disponibles (ataque, defensa, categoría, acciones "Enunciado" / "Iniciar" / "Resolver"). |
| `POST` | `/start-challenge` | Endpoint que inicializa/arranca la instancia de un laboratorio concreto para el usuario autenticado (disparado al presionar "Iniciar"). Requiere `Authorization: Bearer <JWT>` y cookies de sesión válidas. |
| `POST` | `/cdn-cgi/rum?` | Mismo beacon de telemetría de Cloudflare descripto en 4.1, replicado también en este subdominio. |

> Los endpoints `/cdn-cgi/rum*` no pertenecen a la lógica de negocio de *Software Seguro*: son inyectados automáticamente por la capa de Cloudflare que protege/acelera el sitio, y se incluyen aquí porque aparecieron dentro del alcance interceptado.

## 5. Conclusiones

- La configuración de Burp Suite como proxy local (`127.0.0.1:8080`) junto con la instalación de su certificado CA en el navegador permite interceptar y descifrar tráfico HTTPS sin que el navegador reporte errores de certificado, habilitando el análisis completo de la comunicación cliente-servidor.
- El *Site map* construido de forma pasiva permite reconstruir la superficie de endpoints de la aplicación (`/`, `/src/login/`, `/index`, scripts estáticos) sin necesidad de interactuar activamente con cada recurso.
- La capacidad de retener una petición en la pestaña **Intercept** —en este caso, `POST /start-challenge`— antes de reenviarla (o descartarla) confirma de forma práctica la posición de *Man-in-the-Middle*: el proxy tiene control total sobre qué tráfico llega finalmente al servidor, incluyendo la posibilidad de leer o alterar el token `Authorization: Bearer` antes de que la petición continúe su curso.
- Se identifica tráfico de terceros (Cloudflare RUM, Google Analytics, Tailwind CDN) mezclado con el tráfico propio de la aplicación, lo cual es relevante al momento de acotar el alcance (*scope*) de un análisis de seguridad para evitar ruido ajeno al objetivo evaluado.
