# Por qué la lógica de negocio es el punto ciego de los escáneres automáticos

**Asignatura:** Ciberseguridad — Metodología de auditoría
**Fecha:** 05/11/2026

## La diferencia de fondo

Un escáner como Nessus, Acunetix o el escáner activo de Burp busca patrones que conoce de antemano. Sabe cómo se ve un XSS, una inyección SQL o una cabecera mal configurada, así que inyecta cargas conocidas, observa la respuesta y la compara con una firma. Es muy bueno en eso porque son fallos técnicos con una forma reconocible, iguales en casi cualquier aplicación.

Una falla de lógica de negocio no tiene esa forma. Es una regla del negocio que se puede violar usando la aplicación tal como fue construida, sin ninguna carga maliciosa. Comprar con una cantidad negativa y recibir saldo, canjear un cupón de un solo uso varias veces, saltar el paso de pago en un proceso de compra, transferir dinero que no se tiene. Para el servidor, cada una de esas peticiones es válida: tipos correctos, sesión correcta, sintaxis correcta. Lo que está mal es que el resultado contradice una regla que solo existe en la cabeza de quien diseñó el sistema.

## Por qué el escáner no puede verlo

El escáner no sabe qué se supone que hace la aplicación. No tiene idea de que un cupón debía usarse una sola vez, de que una cuenta no puede quedar en negativo, ni de cuál es el orden correcto de los pasos de una compra. Esa intención no está escrita en ningún lado que una herramienta pueda leer; vive en los requisitos, en la cabeza del negocio y en el contexto. Sin conocer la regla, el escáner no puede detectar que se rompió, porque desde el punto de vista técnico no se rompió nada.

A esto se suma que las fallas lógicas suelen necesitar varios pasos encadenados y en un orden concreto, muchas veces aprovechando cómo interactúan funciones distintas. Un escáner prueba entradas sobre puntos aislados; no entiende ese flujo de varios pasos ni sabe inventar la secuencia que lo rompe.

## Por qué la auditoría manual es indispensable

El auditor humano aporta justo lo que le falta a la herramienta: entiende para qué sirve la aplicación y qué significaría abusarla. Puede leer la lógica, preguntarse "¿qué pasa si hago los pasos al revés?" o "¿qué pasa si mando una cantidad negativa?", y probar esos abusos que para el servidor parecen uso normal. Razona sobre la intención del sistema, no sobre patrones de ataque.

Por eso los escáneres y la auditoría manual no compiten, se reparten el trabajo. La automatización cubre rápido y a gran escala los fallos técnicos conocidos, y deja al auditor tiempo para lo que solo él puede hacer: entender las reglas del negocio y buscar las formas de quebrarlas. En la etapa de lógica de negocio, la herramienta pasa de largo y la persona es la que encuentra el problema.

## Referencias

- OWASP — Testing for Business Logic Vulnerabilities (WSTG)
- OWASP Top 10 — A04:2021 Insecure Design
- PortSwigger Web Security Academy — Business logic vulnerabilities
