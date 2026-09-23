# Fuerza Bruta vs. Ataque de Diccionario en formularios de login

## 1. Contexto común: ¿qué tienen en cuenta?

Ambos son técnicas de **cracking de credenciales** aplicadas contra un formulario de autenticación: un script automatizado envía repetidas peticiones de login probando distintas combinaciones de usuario/contraseña hasta encontrar una que sea aceptada por el backend. La diferencia entre ambos no está en el objetivo (adivinar una credencial válida), sino en **cómo se genera el conjunto de valores que se prueban**.

## 2. Ataque de Fuerza Bruta (tradicional)

### Qué es
Consiste en probar **todas las combinaciones posibles** de caracteres dentro de un espacio definido (por ejemplo, todas las combinaciones de letras minúsculas, mayúsculas, números y símbolos, hasta cierta longitud), sin ningún criterio de probabilidad ni conocimiento previo sobre la contraseña. Es un enfoque puramente **exhaustivo y matemático**.

### Cómo funciona en la práctica
Si se define un alfabeto de caracteres (por ejemplo, 62 caracteres: a-z, A-Z, 0-9) y una longitud máxima de 8, el atacante generaría y probaría secuencialmente:
```
aaaaaaaa, aaaaaaab, aaaaaaac, ..., aaaaaaaz, aaaaaaba, ...
```
hasta cubrir absolutamente todo el espacio de combinaciones posible, o hasta encontrar la correcta.

### Característica clave
Garantiza matemáticamente encontrar la contraseña **si se dispone de tiempo y recursos ilimitados**, porque no deja ninguna combinación sin probar dentro del espacio definido. El problema es que ese espacio crece exponencialmente con la longitud y complejidad de la contraseña: una contraseña de 10 caracteres alfanuméricos con símbolos puede representar billones de combinaciones, haciendo que el ataque sea computacionalmente inviable en un tiempo razonable contra contraseñas robustas, incluso con hardware potente (GPUs, clusters).

## 3. Ataque de Diccionario

### Qué es
En lugar de probar combinaciones exhaustivas, este ataque prueba una **lista predefinida de candidatos** ("diccionario") compuesta por contraseñas reales que la gente efectivamente usa: palabras comunes, nombres propios, fechas, contraseñas filtradas en brechas de datos previas (como las que circulan en bases como "RockYou" o los dumps de *Have I Been Pwned*), y variaciones típicas de esas palabras (agregar números al final, reemplazar letras por símbolos similares — "leetspeak" — como `p4ssw0rd`, `Admin123!`, `Contraseña2024`).

### Cómo funciona en la práctica
El atacante carga un archivo de texto con miles o millones de contraseñas candidatas (por ejemplo, `123456`, `password`, `qwerty`, `admin123`, nombres de mascotas comunes, etc.) y las prueba una por una contra el formulario de login, generalmente combinándolas también con nombres de usuario probables (a veces obtenidos por OSINT previo).

### Característica clave
No garantiza cobertura del 100% del espacio de contraseñas posibles (si la contraseña real no está en el diccionario, el ataque nunca la va a encontrar), pero es **mucho más eficiente en la práctica**, porque se basa en la observación de que las personas eligen contraseñas predecibles, no combinaciones aleatorias. Con muchísimas menos peticiones que la fuerza bruta pura, la tasa de éxito real contra usuarios promedio suele ser considerablemente más alta.

## 4. La diferencia fundamental, resumida

| Aspecto | Fuerza Bruta | Diccionario |
|---|---|---|
| **Fuente de los candidatos** | Generación matemática exhaustiva de todas las combinaciones posibles | Lista curada de contraseñas reales/probables conocidas de antemano |
| **Cobertura** | Total, dentro del espacio definido (garantiza encontrarla eventualmente) | Parcial (solo encuentra lo que está en la lista) |
| **Eficiencia práctica** | Baja: el espacio crece exponencialmente con la complejidad | Alta: aprovecha patrones reales de comportamiento humano |
| **Cantidad de intentos típica** | Puede llegar a miles de millones/billones | Generalmente miles a pocos millones |
| **Variante intermedia** | — | **Ataque híbrido**: combina el diccionario con reglas de mutación (agregar números, mayúsculas, símbolos) para ampliar la cobertura sin llegar al volumen de la fuerza bruta pura |

En una frase: la fuerza bruta **no asume nada** sobre la contraseña y prueba todo; el diccionario **asume que la contraseña es "humana"** y prueba solo lo probable.

## 5. Mecanismos de defensa para el desarrollador

Cualquiera de los dos ataques depende de poder enviar **muchos intentos automatizados** contra el mismo endpoint de login. Por eso, la defensa más efectiva no distingue entre uno y otro: apunta a frenar la automatización en sí. Algunos mecanismos concretos:

### a) Rate limiting / Throttling
Limitar la cantidad de intentos de login permitidos desde una misma IP, cuenta o combinación de ambas en una ventana de tiempo (por ejemplo, máximo 5 intentos por minuto). Se puede implementar a nivel de aplicación o directamente en el reverse proxy / WAF.

### b) Bloqueo progresivo de cuenta (account lockout)
Tras N intentos fallidos consecutivos sobre un mismo usuario, bloquear temporalmente esa cuenta (o exigir un paso adicional de verificación) antes de permitir nuevos intentos. Hay que balancearlo con cuidado para no habilitar un ataque de **denegación de servicio dirigido** (bloquear intencionalmente la cuenta de otra persona a propósito).

### c) CAPTCHA / hCaptcha / reCAPTCHA
Introducir un desafío que distinga humanos de bots después de cierta cantidad de intentos fallidos, dificultando la automatización masiva del formulario.

### d) Autenticación multifactor (MFA/2FA)
Aunque el atacante adivine la contraseña correcta, sin el segundo factor (código temporal, notificación push, llave física) no puede completar el login. Es una de las defensas más robustas porque neutraliza el ataque incluso si tiene éxito en la parte de contraseña.

### e) Retraso incremental (exponential backoff)
Aumentar progresivamente el tiempo de espera exigido entre intentos fallidos (1s, 2s, 4s, 8s...), lo cual vuelve impráctico para el atacante escalar el volumen de intentos necesario, sin necesariamente bloquear al usuario legítimo.

### f) Política de contraseñas robustas + detección de contraseñas filtradas
Exigir contraseñas de longitud/complejidad razonable y, adicionalmente, verificar contra listas de contraseñas ya filtradas (como la API de *Have I Been Pwned Passwords*) al momento del registro, para reducir directamente la probabilidad de que un ataque de diccionario tenga éxito.

## 6. Conclusión

La fuerza bruta y el ataque de diccionario resuelven el mismo objetivo (adivinar credenciales) con estrategias opuestas: exhaustividad matemática ciega vs. eficiencia basada en el comportamiento humano real al elegir contraseñas. Sin embargo, ambos comparten el mismo talón de Aquiles desde el punto de vista defensivo: **dependen de poder automatizar muchísimos intentos contra el mismo endpoint sin restricción**. Por eso, medidas como rate limiting, bloqueo progresivo, CAPTCHA y, sobre todo, MFA, son efectivas contra los dos tipos de ataque simultáneamente, sin necesidad de distinguir cuál de las dos técnicas está usando el atacante en un momento dado.
