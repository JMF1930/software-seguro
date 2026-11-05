# Análisis de un JSON Web Token (JWT) interceptado

**Asignatura:** Ciberseguridad — Práctica de autenticación web
**Fecha:** 05/11/2026
**Entorno:** Aplicación de laboratorio propia (`http://127.0.0.1:8080`), sin tráfico de terceros.

---

## 1. Objetivo

Interceptar un JWT válido emitido por una aplicación web, decodificar su contenido e identificar:

1. El algoritmo de firma declarado en el **Header**.
2. Los datos sensibles y roles que viajan en el **Payload** en texto plano (solo codificados en Base64URL).

## 2. Entorno y consideraciones éticas

Para no interceptar tokens de servicios reales ni de otras personas, se levantó un servidor de pruebas en `localhost` que emite un JWT al hacer login (`POST /login`). Es el mismo flujo que siguen muchas aplicaciones reales: el servidor devuelve el token en el cuerpo JSON (`access_token`) y también en una cookie (`Set-Cookie: session=...`).

> En una aplicación real, el equivalente sería abrir **DevTools → Network** (o un proxy como **Burp Suite / OWASP ZAP**) y buscar la cabecera `Authorization: Bearer eyJ...`, una cookie de sesión o un valor en `localStorage`. Hay que hacerlo solo con cuentas propias o con autorización.

## 3. Intercepción del token

Se capturó la petición y la respuesta HTTP con `curl -v`:

```bash
curl -s -v -X POST http://127.0.0.1:8080/login -d 'user=juan&pass=lab123'
```

Respuesta relevante (recortada):

```http
HTTP/1.0 200 OK
Content-Type: application/json
Set-Cookie: session=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIi...; Path=/

{"access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."}
```

Token completo interceptado:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMDI0IiwibmFtZSI6Ikp1YW4gRXN0dWRpYW50ZSIsImVtYWlsIjoianVhbi5lc3R1ZGlhbnRlQGxhYi5sb2NhbCIsInJvbGUiOiJhZG1pbiIsInBlcm1pc3Npb25zIjpbInVzZXJzOnJlYWQiLCJ1c2Vyczp3cml0ZSIsImJpbGxpbmc6cmVhZCJdLCJkZXB0IjoiSVQiLCJpbnRlcm5hbF9pcCI6IjEwLjAuNC4xNyIsImlhdCI6MTc5MzkwODgwMCwiZXhwIjoxNzkzOTEyNDAwLCJpc3MiOiJsYWItYXV0aC5sb2NhbCJ9.reOhwe7_h4CPijHvmPzpS4TfpCl_hLy6Jz467Nb_c9I
```

## 4. Estructura técnica del JWT

Un JWT tiene tres partes separadas por puntos: `HEADER.PAYLOAD.SIGNATURE`

| Parte | Valor (Base64URL) | Contenido |
|---|---|---|
| Header | `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9` | Metadatos: algoritmo y tipo |
| Payload | `eyJzdWIiOiIxMDI0Ii...` | *Claims* (datos del usuario) |
| Signature | `reOhwe7_h4CPijHvmPzpS4TfpCl_hLy6Jz467Nb_c9I` | HMAC-SHA256 de `header.payload` |

Las dos primeras partes **no están cifradas**, solo codificadas en Base64URL. Cualquiera que tenga el token puede leerlas.

## 5. Decodificación desde la terminal

Se decodificó sin herramientas externas (no hace falta pegar el token en ninguna web):

```bash
dec(){ local s=$1; s=$(echo -n "$s" | tr '_-' '/+')
       while [ $(( ${#s} % 4 )) -ne 0 ]; do s="$s="; done
       echo -n "$s" | base64 -d; }

dec "$(echo $TOKEN | cut -d. -f1)" | python3 -m json.tool   # Header
dec "$(echo $TOKEN | cut -d. -f2)" | python3 -m json.tool   # Payload
```

> Si se usa **jwt.io**, conviene hacerlo solo con tokens de prueba o ya expirados: pegar un token vivo en un sitio de terceros equivale a entregar la sesión.

### 5.1 Header decodificado

```json
{
    "alg": "HS256",
    "typ": "JWT"
}
```

### 5.2 Payload decodificado

```json
{
    "sub": "1024",
    "name": "Juan Estudiante",
    "email": "juan.estudiante@lab.local",
    "role": "admin",
    "permissions": ["users:read", "users:write", "billing:read"],
    "dept": "IT",
    "internal_ip": "10.0.4.17",
    "iat": 1793908800,
    "exp": 1793912400,
    "iss": "lab-auth.local"
}
```

Conversión de las marcas de tiempo:

```bash
date -u -d @1793908800   # iat → Thu Nov  5 20:00:00 UTC 2026
date -u -d @1793912400   # exp → Thu Nov  5 21:00:00 UTC 2026  (vida útil: 1 hora)
```

## 6. Respuestas a las preguntas

### ¿Qué algoritmo de firma se usa en el Header?

**HS256**, es decir, **HMAC con SHA-256**. Es un algoritmo **simétrico**: la misma clave secreta sirve para firmar y para verificar el token.

Implicaciones de seguridad:

- Si la clave es débil o predecible, se puede atacar por **fuerza bruta offline** (por ejemplo `hashcat -m 16500` o `jwt_tool`), porque el atacante tiene el mensaje y la firma. Con la clave podría **falsificar tokens**, por ejemplo cambiando `"role"` a cualquier valor.
- Todos los servicios que verifican el token necesitan conocer el secreto, así que cada uno de ellos es un punto de fuga.
- El servidor tiene que **fijar el algoritmo esperado** y rechazar `alg: none` y la confusión de algoritmos (RS256 → HS256).
- Para arquitecturas con varios servicios suele ser preferible un algoritmo **asimétrico** (RS256 / ES256): se firma con clave privada y se verifica con la pública.

### ¿Qué datos sensibles o roles viajan en el Payload en texto plano?

| Claim | Valor | Tipo de dato | Riesgo |
|---|---|---|---|
| `sub` | `1024` | ID interno de usuario | Permite enumerar IDs y facilita ataques IDOR |
| `name` | `Juan Estudiante` | Dato personal (PII) | Exposición de identidad |
| `email` | `juan.estudiante@lab.local` | Dato personal (PII) | Phishing dirigido, correlación de cuentas |
| `role` | `admin` | **Rol / autorización** | Revela que la cuenta es privilegiada y la hace objetivo prioritario |
| `permissions` | `users:read`, `users:write`, `billing:read` | **Permisos granulares** | Mapea la superficie de ataque y los endpoints existentes |
| `dept` | `IT` | Dato organizativo | Ingeniería social |
| `internal_ip` | `10.0.4.17` | Infraestructura interna | Reconocimiento de la red interna |
| `iss` | `lab-auth.local` | Emisor | Revela el nombre del servicio de autenticación |
| `iat` / `exp` | 1 hora de vida | Temporales | Ventana durante la cual un token robado sirve |

**Conclusión clave:** la firma garantiza **integridad** (no se puede modificar sin la clave) pero **no confidencialidad**. Quien intercepte el token (XSS, logs, proxies, historial, extensiones del navegador) puede leer todo lo anterior.

## 7. Recomendaciones

1. **Minimizar el payload:** dejar solo `sub`, `exp`, `iat`, `iss`, `aud` y, si hace falta, un rol genérico. Los datos personales se consultan en el servidor.
2. **No incluir información de infraestructura** (IPs internas, hostnames, rutas).
3. **Si los datos deben viajar, cifrarlos** con **JWE** (JSON Web Encryption) en lugar de JWS.
4. **Clave HS256 robusta:** como mínimo 256 bits aleatorios, guardada en un gestor de secretos y rotada; o migrar a RS256/ES256.
5. **Validación estricta en el servidor:** lista blanca de algoritmos, verificación de `exp`, `iss` y `aud`, rechazo de `alg: none`.
6. **Autorización del lado del servidor:** no confiar ciegamente en `role` o `permissions` sin comprobarlos contra la base de datos en operaciones críticas.
7. **Transporte y almacenamiento:** solo HTTPS; cookies `HttpOnly; Secure; SameSite=Strict` en lugar de `localStorage`. En la app de laboratorio la cookie **no** tenía estas banderas, lo cual es otro hallazgo.
8. **Vida corta + refresh tokens** y un mecanismo de revocación (lista de bloqueo por `jti`).

## 8. Herramientas utilizadas

- `curl -v`: intercepción de la respuesta HTTP
- `base64`, `tr`, `cut`: decodificación Base64URL
- `python3 -m json.tool`: formateo del JSON
- `date`: conversión de timestamps Unix
- Alternativas: Burp Suite, OWASP ZAP, DevTools del navegador, `jwt_tool`, jwt.io (solo con tokens de prueba)

## Referencias

- RFC 7519 — JSON Web Token (JWT)
- RFC 7515 — JSON Web Signature (JWS)
- RFC 7516 — JSON Web Encryption (JWE)
- RFC 8725 — JWT Best Current Practices
- OWASP Cheat Sheet: *JSON Web Token for Java* / *Session Management*
