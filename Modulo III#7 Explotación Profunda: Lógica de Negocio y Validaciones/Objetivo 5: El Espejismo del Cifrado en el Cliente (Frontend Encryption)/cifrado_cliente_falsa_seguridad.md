# Cifrado en el cliente: por qué no protege contra la manipulación de parámetros

**Asignatura:** Ciberseguridad — Arquitectura de aplicaciones web
**Fecha:** 05/11/2026
**Enfoque:** defensivo. Analizar una decisión de arquitectura y explicar por qué falla.

---

## 1. El escenario

Una aplicación financiera cifra con JavaScript, en el navegador del usuario, todo el cuerpo de las peticiones POST y los parámetros sensibles antes de enviarlos. Los arquitectos sostienen que así es imposible interceptar o manipular los valores con un proxy como Burp Suite, porque "todo viaja encriptado".

La afirmación es una **falsa sensación de seguridad**. El razonamiento confunde dos cosas distintas: proteger los datos *en tránsito frente a un tercero* y protegerlos *frente al propio usuario del navegador*. El atacante en este modelo no es un tercero en la red: es la persona que controla el navegador donde ese cifrado se ejecuta.

## 2. El fallo fundamental de diseño

El problema de raíz es el mismo que atraviesa todas las fallas de lógica y de manipulación de parámetros: **el cliente no es un entorno de confianza, y aquí se le está entregando el control del cifrado al cliente.**

Cuando el cifrado ocurre en el navegador con JavaScript, todo lo necesario para cifrar y descifrar está, por necesidad, en el navegador:

- El **código** que cifra (el propio JavaScript, visible y modificable).
- La **clave** o el material para derivarla, porque el código tiene que poder cifrar sin preguntarle a nadie.
- Los **datos en claro**, que existen en memoria *antes* de cifrarse y *después* de descifrarse.

El usuario, y por lo tanto un atacante que use su propia sesión, tiene acceso total a los tres. Puede leer el código, extraer la clave, ver los datos antes de que se cifren y cambiar lo que quiera en ese punto. El cifrado no lo detiene porque el atacante no está peleando contra el cifrado: opera en el mismo lado donde el cifrado se realiza, con las mismas llaves que el navegador legítimo.

Dicho de otro modo: cifrar en el cliente solo cambia *en qué punto* el atacante intercepta. En vez de tocar el dato en la red, lo toca en el navegador, un paso antes de que se cifre. El resultado para el servidor es idéntico, un payload cifrado correctamente que contiene el valor que el atacante eligió.

Y hay un segundo error conceptual: esto ni siquiera aporta confidencialidad en la red, porque las aplicaciones ya viajan sobre **HTTPS (TLS)**, que cifra el canal de forma correcta entre el navegador y el servidor. El cifrado en JavaScript se superpone a TLS sin agregar nada frente a un espía de red, y da una falsa tranquilidad frente al único atacante que importa aquí, el que controla el cliente.

## 3. Por qué Burp Suite sigue siendo útil

El argumento de los arquitectos imagina a Burp interceptando el tráfico *ya cifrado* entre el navegador y el servidor, y concluye que ahí no hay nada que manipular. Es verdad que en ese punto el payload está cifrado. El error es pensar que ese es el único punto donde se puede atacar.

El atacante no tiene por qué interceptar después del cifrado. Puede intervenir **antes**, dentro del navegador, o **reutilizar la propia maquinaria de cifrado de la aplicación** para producir payloads válidos con los valores que quiera. Burp sigue siendo útil, solo que combinado con el acceso que el atacante tiene al cliente.

## 4. Pasos tácticos que realizaría un atacante

Todo esto se hace sobre la **propia cuenta y el propio navegador** del atacante, que es donde el cifrado está a su disposición. No hace falta romper nada criptográfico.

**Paso 1: leer el código de cifrado.** Abrir las DevTools del navegador y examinar el JavaScript de la aplicación. La función que cifra está ahí, descargada y en ejecución. Aunque esté ofuscada o minimizada, se puede leer, embellecer y seguir.

**Paso 2: encontrar el punto donde el dato está en claro.** Poner un breakpoint en la función de cifrado, o justo antes de que se la llame. Cuando la aplicación procesa una operación real (por ejemplo una transferencia), la ejecución se detiene ahí y el atacante ve los valores en texto plano: el monto, la cuenta destino, lo que sea. Así lee los datos "protegidos" sin descifrar nada, simplemente mirándolos antes de que se cifren.

**Paso 3: manipular el valor en ese punto.** Con la ejecución detenida en el breakpoint, modificar la variable en claro desde la consola (cambiar el monto, la cuenta, el precio) y dejar que la función de cifrado continúe. El JavaScript cifra el valor alterado con la clave legítima y arma un payload perfectamente válido. El servidor lo recibe, lo descifra y obtiene el valor manipulado como si fuera auténtico.

**Alternativa al paso 3: llamar directamente a la función de cifrado.** En vez de usar breakpoints, el atacante invoca desde la consola la propia función de la aplicación, por ejemplo `cifrar({ monto: 0.01, destino: "atacante" })`, y obtiene un payload cifrado listo para enviar. Después lo manda con Burp Repeater, con `fetch` desde la consola o con un script. Está usando el cifrado de la aplicación como un oráculo que firma lo que él quiera.

**Paso 4 (opcional): automatizar.** Una vez que entiende el esquema, puede reproducir el cifrado fuera del navegador (en Python, o con un script de Turbo Intruder que llame a la lógica de cifrado) y lanzar muchas peticiones manipuladas, por ejemplo para combinar esto con un ataque de concurrencia.

En todos los casos el patrón es el mismo: el atacante no ataca el canal cifrado, ataca el eslabón anterior, donde los datos y la clave están a su alcance.

## 5. Dónde debe estar la defensa

La conclusión es la de siempre: **la seguridad de una operación la tiene que imponer el servidor, porque es el único entorno que el atacante no controla.**

- **Validar y decidir en el backend.** El servidor no debe confiar en ningún valor sensible que venga del cliente, esté cifrado o no. En una transferencia, el monto se valida contra las reglas y el saldo del lado del servidor; en una compra, el precio se toma del catálogo del servidor, nunca del payload (como vimos en manipulación de parámetros).
- **Usar TLS para el transporte, y nada más para eso.** HTTPS ya resuelve la confidencialidad e integridad en la red. El cifrado adicional en JavaScript no agrega seguridad frente al cliente y suele dar una falsa confianza.
- **Autorización por operación en el servidor.** Cada operación crítica se autoriza y verifica en el backend, con controles de estado y de propiedad, independientemente de lo que afirme el cliente.
- **Firmar del lado del servidor, no del cliente.** Si se necesita integridad de ciertos datos entre pasos, el servidor los firma con una clave que el cliente nunca ve (por ejemplo dentro de un token), de modo que el cliente no pueda generarlos ni alterarlos.

El cifrado en el cliente puede tener usos legítimos (por ejemplo, cifrado extremo a extremo donde el servidor *no debe* poder leer los datos), pero no sirve como control de seguridad contra la manipulación de parámetros, porque no cambia quién controla el entorno donde se cifra.

## 6. Conclusión

La afirmación de los arquitectos falla porque sitúa la defensa en el único lugar que el atacante controla por completo: su propio navegador. Cifrar en el cliente solo mueve el punto de intercepción un paso hacia atrás, a donde los datos y la clave todavía están en claro. El atacante lee y manipula ahí, deja que la aplicación cifre el valor falso con su clave legítima y se lo entrega al servidor. La protección no puede vivir en el cliente; tiene que vivir en el servidor, que valida y decide sobre datos de su propia fuente de confianza.

## Referencias

- OWASP — Client-Side Security / Testing for Weak Encryption (WSTG)
- OWASP Top 10 — A04:2021 Insecure Design
- OWASP — Parameter Tampering
- CWE-602: Client-Side Enforcement of Server-Side Security
- CWE-603: Use of Client-Side Authentication
