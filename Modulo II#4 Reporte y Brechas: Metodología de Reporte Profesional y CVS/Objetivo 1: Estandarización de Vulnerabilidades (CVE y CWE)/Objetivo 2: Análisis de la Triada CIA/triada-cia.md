# La Tríada CIA: Confidencialidad, Integridad y Disponibilidad

## 1. Introducción

Todo el campo de la seguridad de la información se organiza, en última instancia, alrededor de proteger tres propiedades fundamentales de los datos y los sistemas: **Confidencialidad, Integridad y Disponibilidad** (en inglés, *CIA Triad*: Confidentiality, Integrity, Availability). Cualquier control de seguridad que se implemente —cifrado, backups, firewalls, autenticación— existe, en el fondo, para sostener una o más de estas tres propiedades. Y, simétricamente, prácticamente cualquier vulnerabilidad o ataque puede analizarse identificando cuál de los tres pilares compromete principalmente.

## 2. Confidencialidad

### Qué significa
La confidencialidad garantiza que la información **solo sea accesible para quienes están autorizados a verla**, y que permanezca oculta o protegida frente a cualquier otra persona o sistema. No se trata solo de "que nadie la vea", sino de que el acceso esté restringido exactamente al conjunto de personas/sistemas que deberían tenerlo, ni más ni menos.

### Cómo se sostiene técnicamente
Se logra mediante mecanismos como el cifrado (en tránsito y en reposo), el control de acceso basado en roles y permisos, la autenticación robusta, y la clasificación de la información según su sensibilidad. Como vimos al analizar el TLS Handshake, el cifrado simétrico que protege el tráfico HTTPS es, precisamente, un mecanismo diseñado para sostener la confidencialidad de los datos que viajan entre cliente y servidor.

### Ejemplo de vulnerabilidad que compromete principalmente la Confidencialidad: IDOR (Insecure Direct Object Reference)

Ya lo analizamos antes en el contexto de la URL `https://app.softwareseguro.com.ar/dashboard/user?id=45`. Si el backend no valida que el usuario autenticado tenga permiso real sobre el recurso solicitado, un atacante puede simplemente cambiar el valor del parámetro (`id=46`, `id=47`...) y acceder a datos de **otros usuarios** que nunca debería poder ver: información personal, facturas, mensajes privados, historiales médicos, etc.

Este ataque compromete directamente la confidencialidad porque el sistema sigue funcionando con normalidad (la disponibilidad no se ve afectada) y los datos no se modifican (la integridad tampoco se ve afectada) — el único problema es que información que debía estar restringida queda expuesta a quien no tenía autorización para acceder a ella.

## 3. Integridad

### Qué significa
La integridad garantiza que la información sea **exacta, completa y no haya sido alterada de forma no autorizada** — ni por un atacante malicioso, ni por un error accidental, ni durante su transmisión o almacenamiento. Cuando alguien lee un dato, la integridad asegura que ese dato es fiel al original, tal como fue creado o autorizado por su dueño legítimo.

### Cómo se sostiene técnicamente
Se logra mediante funciones hash (para detectar si un archivo o mensaje fue modificado), firmas digitales, controles de versión, sumas de verificación (checksums), bitácoras de auditoría (logs) y validación estricta de las entradas de datos. En el TLS Handshake, el mensaje `Finished` que vimos —que resume mediante un hash todo lo negociado— es precisamente un mecanismo de integridad: confirma que ningún mensaje del handshake fue alterado en tránsito.

### Ejemplo de vulnerabilidad que compromete principalmente la Integridad: SQL Injection (con fines de modificación de datos)

Cuando un atacante explota una **SQL Injection** (CWE-89, como vimos en el informe de CVE/CWE) no solo para *leer* datos que no debería (lo cual afectaría confidencialidad), sino para **modificar o borrar registros** directamente en la base de datos —por ejemplo, inyectando una sentencia `UPDATE` o `DELETE`, o alterando el precio de un producto en un e-commerce, o cambiando la contraseña o el rol de un usuario (escalando sus propios privilegios a administrador)— el ataque compromete directamente la **integridad**: los datos dejan de reflejar la realidad que deberían reflejar, sin que el sistema ni los usuarios legítimos lo hayan autorizado.

Ejemplo concreto: `'; UPDATE productos SET precio = 1 WHERE id = 10; --` inyectado en un campo de búsqueda mal sanitizado, que termina alterando el precio real de un producto en la base de datos de la tienda.

## 4. Disponibilidad

### Qué significa
La disponibilidad garantiza que la información y los sistemas estén **accesibles y operativos cuando los usuarios autorizados los necesiten**. Un sistema perfectamente confidencial e íntegro, pero que nunca responde o está permanentemente caído, no cumple su función — la seguridad de la información también incluye asegurar que el servicio siga funcionando.

### Cómo se sostiene técnicamente
Se logra mediante redundancia de infraestructura (servidores replicados, balanceo de carga), planes de recuperación ante desastres (backups, sitios de contingencia), protección contra sobrecarga (rate limiting, como vimos al analizar las defensas contra fuerza bruta), y mitigación específica de ataques de denegación de servicio.

### Ejemplo de vulnerabilidad/ataque que compromete principalmente la Disponibilidad: DDoS (Distributed Denial of Service)

Un ataque de **denegación de servicio distribuido** consiste en saturar un servidor o servicio con un volumen de tráfico o solicitudes tan grande (a menudo generado por una red de dispositivos comprometidos, una *botnet*) que el sistema legítimo ya no puede procesar las peticiones de los usuarios reales, o directamente colapsa. También aplica el caso más simple de **DoS** (no distribuido) mediante, por ejemplo, explotar un endpoint costoso computacionalmente que agota los recursos del servidor con pocas peticiones bien diseñadas.

Este ataque no necesariamente compromete la confidencialidad de los datos (el atacante no necesariamente accede a información) ni su integridad (los datos siguen siendo correctos) — el daño es que el sistema deja de estar disponible para quienes legítimamente lo necesitan usar, lo cual puede tener un impacto económico y reputacional directo (un sitio de e-commerce caído durante horas, por ejemplo).

## 5. Tabla resumen

| Pilar | Qué protege | Mecanismo típico | Ejemplo de ataque/vulnerabilidad | Qué falla concretamente |
|---|---|---|---|---|
| **Confidencialidad** | Que solo accedan quienes están autorizados | Cifrado, control de acceso, autenticación | **IDOR** | Un usuario ve datos de otro usuario que no debería poder ver |
| **Integridad** | Que los datos sean exactos y no alterados sin autorización | Hashing, firmas digitales, validación de entradas | **SQL Injection** (con modificación de datos) | Los datos de la base quedan alterados sin autorización legítima |
| **Disponibilidad** | Que el sistema esté accesible cuando se lo necesita | Redundancia, rate limiting, planes de contingencia | **DDoS** | El sistema deja de responder a usuarios legítimos |

## 6. Una aclaración importante: los ataques rara vez son "puros"

Vale la pena remarcar, como cierre, que esta clasificación es útil para el análisis y el aprendizaje, pero en la práctica **muchos ataques comprometen más de un pilar a la vez**, o pueden encadenarse. Por ejemplo, una SQL Injection podría empezar comprometiendo confidencialidad (extraer datos con `SELECT`) y, con un payload distinto sobre el mismo endpoint vulnerable, terminar comprometiendo también integridad (`UPDATE`/`DELETE`). Del mismo modo, un atacante que compromete la integridad de un sistema de autenticación (por ejemplo, alterando la lógica de verificación de contraseñas) puede terminar afectando también la confidencialidad de todas las cuentas. Por eso, en un análisis de seguridad riguroso, conviene identificar el **impacto principal y directo** de cada hallazgo (como pide la consigna), pero sin perder de vista que el impacto real en un incidente concreto suele ser una combinación de los tres pilares.

## 7. Conclusión

La tríada CIA es el marco conceptual fundamental de la seguridad de la información porque cualquier objetivo de protección puede reducirse a sostener confidencialidad, integridad o disponibilidad — y cualquier vulnerabilidad, a su vez, puede entenderse como una falla en sostener alguno de esos tres pilares. Comprender esta clasificación permite, frente a cualquier hallazgo de seguridad nuevo, hacerse la pregunta clave: *¿qué es exactamente lo que este fallo pone en riesgo: que alguien vea algo que no debería, que algo se modifique sin autorización, o que el sistema deje de estar disponible?* Esa pregunta orienta tanto el análisis de impacto como la elección de la contramedida más adecuada.
