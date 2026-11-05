# La vulnerabilidad del algoritmo `none` en JWT

**Asignatura:** Ciberseguridad — Autenticación web
**Fecha:** 05/11/2026
**Enfoque:** Defensivo. El objetivo es entender *por qué* ocurre el fallo para poder prevenirlo y detectarlo.

---

## 1. Resumen

El estándar JWT (RFC 7519) define, a través de JWS (RFC 7515), un valor de algoritmo especial llamado `none`. Significa "JWT no asegurado": un token **sin firma**. Fue pensado para casos donde la integridad ya está garantizada por otro medio (por ejemplo, un canal TLS cerrado punto a punto).

El problema no está en que el valor exista, sino en **backends que confían en el campo `alg` del propio token para decidir cómo verificarlo**. Si el servidor acepta `alg: none`, deja de comprobar la firma, y entonces cualquiera puede escribir el payload que quiera. Esto fue un fallo masivo en librerías alrededor de 2015 (divulgación de Tim McLean) y sigue apareciendo en implementaciones mal configuradas.

## 2. Repaso: qué garantiza la firma

Un JWT tiene tres partes: `HEADER.PAYLOAD.SIGNATURE`.

- El **header** y el **payload** solo están codificados en Base64URL (legibles por cualquiera, como vimos en la práctica anterior).
- La **firma** es lo único que da seguridad: es un valor criptográfico calculado sobre `header.payload` con una clave que **solo conoce el servidor**.

Con HS256, la firma es:

```
HMAC-SHA256( base64url(header) + "." + base64url(payload), clave_secreta )
```

La idea de seguridad: un atacante puede **leer** el payload, pero si lo **modifica** (por ejemplo, cambia `"role":"user"` por `"role":"admin"`), no puede recalcular una firma válida porque no tiene la clave. El servidor, al verificar, obtiene un HMAC distinto y rechaza el token.

**La seguridad del esquema depende enteramente de que el servidor verifique la firma.**

## 3. El fallo: el token decide cómo se valida a sí mismo

El error de diseño de las librerías vulnerables es tratar el campo `alg` del header como una **instrucción de confianza**. El flujo vulnerable es, en esencia:

```python
# PSEUDOCÓDIGO VULNERABLE — no usar
header = decode(token.header)
if header["alg"] == "none":
    return header, payload          # ← acepta sin verificar NADA
elif header["alg"] == "HS256":
    verify_hmac(token, secret)
...
```

El atacante controla el token entero, **incluido el header**. Por tanto controla el valor de `alg`. Si pone `alg: none`, empuja al servidor por la rama que **no verifica firma**. Resultado: el servidor acepta un token cuyo payload fue escrito íntegramente por el atacante.

Es un caso del error clásico: **dejar que la entrada no confiable determine su propia política de validación.** El dato que debería ser validado está diciéndole al validador qué hacer.

## 4. Por qué no hace falta conocer la clave secreta

Aquí está el núcleo de la pregunta. Con `alg: none`:

1. **La firma deja de usarse.** El token "no asegurado" se define con una **tercera parte vacía**: termina en un punto y nada después (`header.payload.`). No hay ningún valor criptográfico que comprobar.
2. **La clave secreta nunca entra en juego.** El servidor, al ver `none`, ni siquiera llega al código que usaría la clave. La clave del servidor es irrelevante para este ataque: no se adivina, no se roba, no se necesita. Simplemente **no se consulta**.
3. **El payload es texto bajo control del atacante.** Como el payload solo está en Base64URL, el atacante lo construye a mano con los claims que quiera (`sub`, `role`, `email`, etc.).

Dicho de otra forma: la clave secreta protege la *firma*. Si el servidor acepta tokens *sin firma*, la clave ya no protege nada, porque no hay firma que validar. Por eso el atacante puede falsificar **cualquier identidad** sin conocer el secreto.

## 5. Cómo se construiría un token malicioso (concepto)

No hace falta ninguna herramienta ofensiva ni la clave del servidor; solo codificación Base64URL, la misma que usamos para *decodificar* en la práctica anterior:

1. Header: `{"alg":"none","typ":"JWT"}` → Base64URL.
2. Payload: los claims deseados, p. ej. `{"sub":"1","role":"admin"}` → Base64URL.
3. Firma: **cadena vacía**.
4. Token = `base64url(header) + "." + base64url(payload) + "."` (nótese el punto final seguido de nada).

Contra un backend vulnerable, ese token se acepta como si lo hubiera emitido el servidor. Contra un backend correcto (sección 7), se rechaza de inmediato.

> Variantes relacionadas que explotan la misma raíz (confiar en `alg`):
> - **Mayúsculas/variantes**: `None`, `NONE`, `nOnE` — para saltarse filtros que solo comparan la cadena exacta `"none"`.
> - **Confusión de algoritmos RS256 → HS256**: el atacante cambia `alg` de RS256 a HS256 y firma con la **clave pública** (que es, precisamente, pública) usándola como si fuera el secreto HMAC. Es el mismo error: el servidor dejó que el token eligiera el algoritmo.

## 6. Demostración de laboratorio (verificador vulnerable vs. correcto)

Para ver el efecto sin atacar nada externo, comparo en local un verificador vulnerable y uno correcto frente a un token `alg: none` forjado. El código y su salida están en el archivo `demo_none.txt` que acompaña este informe.

Idea de la demo:

- **Verificador vulnerable**: si `alg == "none"`, devuelve el payload sin comprobar firma → **acepta** el token forjado con `role: admin`.
- **Verificador correcto**: exige explícitamente `HS256` (lista blanca) y verifica el HMAC → **rechaza** el mismo token.

Ambos reciben exactamente el mismo token forjado; la única diferencia es la política de validación del servidor.

Salida real de la demo (`demo_none.py`, ejecutada en el laboratorio):

```
Token forjado: eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJzdWIiOiIxIiwicm9sZSI6ImFkbWluIiwiZW1haWwiOiJhdGFjYW50ZUBldmlsLnRlc3QifQ.

Verificador VULNERABLE:
  ACEPTADO -> {'sub': '1', 'role': 'admin', 'email': 'atacante@evil.test'}
Verificador CORRECTO:
  RECHAZADO -> algoritmo no permitido: 'none'
```

El token se forjó **sin usar el secreto del servidor**: el header declara `alg: none`, el payload (`role: admin`) es texto bajo control del atacante y la firma es una cadena vacía (el token termina en punto). El verificador vulnerable lo acepta porque sigue la rama que no comprueba firma; el correcto lo rechaza en la lista blanca de algoritmos antes incluso de mirar la firma.

## 7. Mitigaciones (lo importante)

1. **Lista blanca de algoritmos en el servidor.** No leer `alg` del token para decidir; fijar el algoritmo esperado en el código de verificación:
   ```python
   # PyJWT
   jwt.decode(token, key, algorithms=["HS256"])   # solo HS256; 'none' se rechaza
   ```
   Nunca pasar algo como `algorithms=["HS256","none"]` ni derivar la lista del propio token.
2. **Rechazar `none` siempre** en producción, incluidas variantes de mayúsculas.
3. **Separar claramente simétrico y asimétrico** para evitar la confusión RS256→HS256: el verificador de RS256 debe usar la clave pública **solo** con RS256, nunca aceptar HS256 con esa clave.
4. **Usar librerías mantenidas y actualizadas** (el fallo masivo de 2015 ya está corregido en versiones modernas, pero reaparece con configuraciones laxas).
5. **Verificar también los claims**: `exp`, `iss`, `aud`, y no confiar en `role`/`permissions` para operaciones críticas sin contrastar contra el servidor.
6. **Pruebas de regresión de seguridad**: incluir un test que envíe un token `alg: none` y confirme que el backend responde 401.

## 8. Cómo detectarlo

- **Revisión de código**: buscar verificaciones donde `algorithms` dependa del header o incluya `none`.
- **Pruebas dinámicas**: enviar un token con `alg: none` y payload alterado contra un endpoint autenticado propio o autorizado; si responde 200, es vulnerable.
- **Herramientas**: `jwt_tool`, Burp Suite (extensión JWT), OWASP ZAP — siempre sobre sistemas propios o con permiso explícito.

## 9. Conclusión

La seguridad de un JWT firmado descansa por completo en que **el servidor verifique la firma con un algoritmo que él mismo decide**. El algoritmo `none` desactiva esa verificación, y como la clave secreta solo intervenía al comprobar la firma, deja de ser necesaria: el atacante no la rompe ni la roba, simplemente elige un camino en el que nunca se usa. La lección general es un principio de diseño seguro: **nunca dejar que un dato no confiable dicte cómo se valida a sí mismo.**

## Referencias

- RFC 7519 — JSON Web Token (JWT)
- RFC 7515 — JSON Web Signature (JWS), apéndice sobre "Unsecured JWS" (`alg: none`)
- RFC 8725 — JSON Web Token Best Current Practices (sección sobre confusión y `none`)
- Tim McLean, "Critical vulnerabilities in JSON Web Token libraries" (2015)
- OWASP — JSON Web Token Cheat Sheet
