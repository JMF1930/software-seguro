# Mensaje de Error Genérico vs. Stack Trace Expuesto en Producción

## 1. Introducción: la reacción ante el fallo como indicador de seguridad

Ninguna aplicación está exenta de errores — entradas inesperadas, fallos de red, excepciones no previstas por el desarrollador. Lo que diferencia a una aplicación segura de una insegura no es la ausencia total de errores (algo prácticamente imposible), sino **cómo reacciona** cuando ocurren. Esta es la idea central detrás de la categoría de OWASP que ya analizamos, *Security Misconfiguration*, y del concepto de *Information Disclosure* por manejo inadecuado de errores (CWE-209 / CWE-200): la forma en que el sistema responde ante un fallo puede, por sí sola, convertirse en una vulnerabilidad.

## 2. ¿Qué es un mensaje de error genérico?

Un **mensaje de error genérico** es una respuesta controlada y deliberadamente vaga que la aplicación muestra al usuario cuando ocurre una excepción, sin revelar ningún detalle técnico interno. Por ejemplo:

```
Ha ocurrido un error inesperado. Por favor, intente nuevamente más tarde.
Código de referencia: ERR-48291
```

### Características de un buen manejo de errores en producción
- **No revela ninguna información sobre la implementación interna**: ni lenguaje de programación, ni frameworks, ni estructura de archivos, ni consultas a bases de datos.
- **Es consistente**: el mismo tipo de error siempre produce el mismo mensaje genérico, sin variaciones que puedan filtrar información indirectamente (como vimos en el caso de los *timing attacks*, donde incluso un mensaje idéntico puede filtrar información por el tiempo de respuesta).
- **Incluye un código de referencia interno** (no el detalle del error), que el equipo de soporte o desarrollo puede usar para correlacionar el reclamo del usuario con el log completo almacenado de forma segura en el servidor.
- El **detalle completo del error se registra en logs internos** (no accesibles públicamente), para que el equipo de desarrollo pueda diagnosticar y corregir la causa raíz sin exponer esa información al usuario final ni a un posible atacante.

## 3. ¿Qué es un Stack Trace (traza de la pila)?

Un **Stack Trace** es un reporte técnico generado automáticamente por el entorno de ejecución (runtime) cuando ocurre una excepción no controlada. Describe la **secuencia completa de llamadas a funciones/métodos** que estaban activas en el momento exacto en que el error se produjo, desde el punto de entrada de la aplicación hasta la línea específica donde todo falló.

### Qué información típica contiene
- El **tipo de excepción** (ej. `PDOException`, `NullPointerException`, `TypeError`).
- El **mensaje de error exacto** generado internamente (que a veces incluye hasta fragmentos de la consulta SQL que falló).
- La **ruta absoluta de cada archivo** involucrado en la cadena de llamadas (ej. `/var/www/html/app/controllers/UserController.php`).
- El **número de línea exacto** donde ocurrió el fallo en cada archivo de la pila.
- A veces, **variables y valores** presentes en ese contexto de ejecución (dependiendo del nivel de verbosidad del framework).
- El **nombre y versión del framework/lenguaje/servidor** que generó el error (información que suele aparecer en el formato mismo del stack trace, ya que cada tecnología tiene un estilo de traza característico y reconocible).

### Ejemplo simplificado de cómo luce un stack trace expuesto
```
Fatal error: Uncaught PDOException: SQLSTATE[42S02]: Base table or view not found
Stack trace:
#0 /var/www/html/app/db.php(45): PDO->query('SELECT * FROM...')
#1 /var/www/html/app/models/User.php(112): Database->query(...)
#2 /var/www/html/app/controllers/UserController.php(28): User->find(45)
#3 /var/www/html/index.php(17): UserController->show(...)
thrown in /var/www/html/app/db.php on line 45

MySQL version: 5.7.32
```

Esto es, literalmente, el mismo tipo de hallazgo que analizamos en una consulta anterior sobre una aplicación que respondía con código fuente, rutas internas y versión de base de datos ante un parámetro malformado — ese caso era, precisamente, un Stack Trace expuesto en producción.

## 4. La diferencia fundamental entre ambos

| Aspecto | Mensaje de error genérico | Stack Trace expuesto |
|---|---|---|
| **Contenido** | Texto neutro, sin detalles técnicos | Rutas, código, líneas, versiones, tipo de excepción |
| **Diseño** | Intencional y controlado por el desarrollador | Generado automáticamente por el runtime, sin filtrar |
| **Audiencia pensada** | El usuario final | El propio equipo de desarrollo (en entornos de debug) |
| **Riesgo de seguridad** | Mínimo | Alto — divulgación de información interna |
| **Dónde debería aparecer** | En la respuesta al cliente, siempre | Únicamente en logs internos o en entornos de desarrollo controlados |

La diferencia no es solo de "cantidad de texto" — es una diferencia de **audiencia e intención**. El mensaje genérico está diseñado deliberadamente para no filtrar nada; el stack trace es información de diagnóstico interno que, por una configuración insegura (típicamente, modo debug habilitado en producción, como vimos antes), termina llegando a quien nunca debería verla: cualquier usuario externo, incluido un atacante.

## 5. Por qué un Stack Trace detallado facilita enormemente la fase de explotación

Para entender el impacto real, es útil pensarlo en términos de la metodología de un ataque (retomando PTES, que ya analizamos): normalmente, un atacante dedica una fase considerable de tiempo a **reconocimiento y enumeración** — descubrir qué tecnología usa el objetivo, qué estructura tiene, qué versiones corre, dónde están los puntos débiles. Un Stack Trace expuesto **le regala gran parte de ese trabajo**, saltando directamente a la fase de explotación con información de alta precisión.

### a) Revela el stack tecnológico exacto
Saber que la aplicación corre PHP con PDO y MySQL 5.7.32 permite al atacante **buscar directamente CVEs conocidos** para esa versión exacta (retomando el catálogo CVE que analizamos), en lugar de tener que adivinar o fingerprintear la tecnología a través de técnicas más indirectas y lentas.

### b) Expone la estructura interna de archivos y rutas
Conocer rutas como `/var/www/html/app/controllers/UserController.php` permite:
- Intentar ataques de **Path Traversal** o **Local File Inclusion (LFI)** apuntando directamente a archivos reales que se sabe que existen, en lugar de adivinar nombres.
- Entender la **arquitectura de la aplicación** (patrón MVC, nombres de controladores, modelos), lo cual ayuda a inferir qué otros endpoints o archivos podrían existir siguiendo la misma convención de nombres.

### c) Filtra fragmentos de lógica de negocio y consultas SQL
Si el stack trace muestra el fragmento de la consulta SQL que falló (`SELECT * FROM...`), el atacante obtiene **pistas directas sobre cómo está construida la query** — exactamente el tipo de información que facilita craftear un payload de SQL Injection preciso, en lugar de tener que probarlo a ciegas por fuerza bruta de sintaxis.

### d) Confirma qué tipo de validación (o falta de ella) existe
El propio hecho de que una excepción específica se haya disparado (por ejemplo, una excepción de tipo de dato al enviar texto donde se esperaba un número) le confirma al atacante que **ese parámetro llega directamente a una capa interna sin sanitizar correctamente**, orientando dónde seguir probando payloads más agresivos.

### e) Acorta drásticamente el ciclo de prueba y error del atacante
Sin esta información, un atacante debe iterar a ciegas: probar un payload, observar una respuesta genérica, inferir indirectamente si algo cambió. Con el stack trace, cada error funciona como una **ventana de depuración gratuita** hacia el funcionamiento interno del sistema — el atacante literalmente ve el mismo nivel de detalle que vería un desarrollador debuggeando su propio código, pero sin tener ningún acceso legítimo al servidor.

## 6. Conclusión

La diferencia entre un mensaje de error genérico y un Stack Trace expuesto no es meramente estética — es la diferencia entre **ocultar correctamente los detalles internos de implementación** y **entregárselos gratuitamente a cualquiera que sepa provocar un error**. Un Stack Trace detallado convierte cada fallo de la aplicación en una fuente involuntaria de inteligencia técnica: tecnología exacta, versión, estructura de archivos y hasta fragmentos de lógica de negocio, información que normalmente requeriría trabajo de reconocimiento activo. Por eso, aunque filtrar un stack trace no sea una vulnerabilidad "explotable" por sí misma en el sentido de otorgar acceso directo, actúa como un **multiplicador de eficiencia** para el atacante, acortando drásticamente el camino hacia la explotación exitosa de otras vulnerabilidades reales del sistema.
