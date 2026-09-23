# Objetivo 2 — Manipulación Manual de Peticiones (Uso del Repeater)

## 1. Enunciado

El módulo **Repeater** (equivalente a *Requester* en OWASP ZAP) es la mesa de trabajo principal de un pentester: permite tomar una petición HTTP ya capturada, editarla manualmente cuantas veces sea necesario y reenviarla al servidor, sin pasar de nuevo por el navegador. El ejercicio consiste en tomar la petición interceptada en el **Objetivo 1**, enviarla al Repeater, modificar manualmente un parámetro (cookie, `User-Agent` o parámetro de URL) y reenviarla al servidor para observar su comportamiento.

**Formato de entrega:** enlace al presente archivo `.md`, con las pruebas y capturas de pantalla realizadas sobre el módulo Repeater para un laboratorio de *Software Seguro*.

## 2. Contexto

Se reutiliza la petición obtenida en el Objetivo 1 (*Man-in-the-Middle Local*): `POST /start-challenge` hacia `app.softwareseguro.com.ar`, generada al presionar **"Iniciar"** sobre el laboratorio *"Uso del inspector"*. Esta petición incluye:

- **Header `Authorization: Bearer <JWT>`** — token de sesión de la aplicación.
- **Cookies** `_ga`, `_ga_MR6ZT8PJ4Q` (Google Analytics) y `auth` (token de sesión propio, visible también en el header `Authorization`).
- **Body JSON** `{"challengeId": 1}` — identificador del laboratorio a inicializar.

## 3. Paso a paso en el Repeater

### 3.1. Petición interceptada (punto de partida)

Petición `POST /start-challenge` retenida en **Proxy → Intercept**, tal como se documentó en el Objetivo 1, antes de reenviarla al servidor.

![Petición POST /start-challenge interceptada en Burp Proxy](./assets_repeater/01-peticion-interceptada.png)

### 3.2. Envío al Repeater

Desde la vista de intercepción, la petición se envía al módulo **Repeater** (clic derecho → *Send to Repeater*, o `Ctrl+R`). En esta pestaña, la petición queda editable en el panel izquierdo y lista para reenviarse manualmente mediante el botón **Send**, mostrando la respuesta del servidor en el panel derecho.

![Petición enviada al módulo Repeater](./assets_repeater/02-enviada-a-repeater.png)

### 3.3. Modificación manual de la cookie

Se edita manualmente, dentro del propio Repeater, el valor de la cookie **`_ga`** (cookie de analítica de Google, no relacionada con la autenticación de la aplicación), reemplazándolo por una cadena arbitraria (`GA1.1714690krjahsjadhjakshdj`) para simular una manipulación de parámetros por parte de un atacante, sin tocar el header `Authorization` ni la cookie `auth`.

![Valor de la cookie _ga modificado manualmente en el Repeater](./assets_repeater/03-cookie-modificada.png)

### 3.4. Reenvío al servidor y respuesta

Al presionar **Send**, la petición modificada se envía al servidor. La respuesta es:

```http
HTTP/2 200 OK
Content-Type: application/json; charset=utf-8
Server: cloudflare

{
  "ok": true,
  "container_id": "0c780fc0ea3cbe3f3456f254e1b0ece4dde9dc7cf67843748721a7720d53ee01",
  "domain": "ch1-8c23adf7-9e59-464a-8bbe-d393fc704494-uso-inspector.softwareseguro.com.ar"
}
```

El servidor responde `200 OK` y **provisiona igualmente el contenedor del laboratorio** (`container_id` y un subdominio efímero `ch1-<uuid>-uso-inspector.softwareseguro.com.ar`), a pesar de haber recibido una cookie `_ga` inválida/manipulada.

![Respuesta 200 OK del servidor tras reenviar la petición con la cookie modificada](./assets_repeater/04-enviada-al-servidor.png)

## 4. Análisis del resultado

| Elemento modificado | ¿Afecta la autorización de la petición? | Motivo |
|---|---|---|
| Cookie `_ga` / `_ga_MR6ZT8PJ4Q` | **No** | Son cookies de telemetría de Google Analytics, ajenas al mecanismo de autenticación de la aplicación. El backend las ignora al validar la petición. |
| Header `Authorization: Bearer <JWT>` | **Sí** (no se modificó en esta prueba) | Es el mecanismo real de autorización: el backend valida contra este token, no contra las cookies de analítica. |
| Body `{"challengeId": 1}` | Sí, determina qué laboratorio se inicializa | Parámetro de negocio: cambiar su valor permitiría, en teoría, solicitar la inicialización de otro `challengeId` sin pasar por la UI. |

**Hallazgo relevante:** la aplicación no depende de la cookie `_ga` para autorizar la operación `/start-challenge`; el control de acceso recae exclusivamente sobre el token `Authorization: Bearer`. Esto confirma que, desde la óptica de un atacante, manipular cookies de analítica no tiene impacto de seguridad, mientras que el verdadero vector de interés para pruebas adicionales (fuera del alcance de este objetivo) sería el propio JWT o el parámetro `challengeId` del body.

## 5. Conclusiones

- El Repeater permite aislar una petición capturada del flujo normal del navegador y reenviarla de forma controlada tantas veces como sea necesario, siendo la herramienta central para probar hipótesis de manipulación de parámetros.
- Modificar manualmente un valor "inofensivo" (cookie de analítica) y observar que el servidor responde igualmente `200 OK` es en sí mismo un resultado válido de la prueba: permite descartar esa cookie como mecanismo de control de acceso y enfocar el análisis en los parámetros que realmente importan (`Authorization`, `challengeId`).
- Este flujo de trabajo (interceptar → enviar a Repeater → modificar → reenviar → comparar respuesta) es la base de la mayoría de las pruebas manuales de seguridad web (IDOR, bypass de autorización, fuzzing de parámetros).
