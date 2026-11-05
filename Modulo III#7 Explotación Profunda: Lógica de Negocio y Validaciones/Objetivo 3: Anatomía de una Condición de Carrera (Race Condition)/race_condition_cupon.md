# Canjear varias veces un cupón de un solo uso con una condición de carrera

**Asignatura:** Ciberseguridad — Concurrencia y lógica de negocio
**Fecha:** 05/11/2026
**Enfoque:** defensivo. El objetivo es entender el fallo para prevenirlo.

## El problema en una frase

Un cupón marcado como "de un solo uso" no es realmente de un solo uso si el servidor comprueba su validez y recién después la marca como gastada, dejando un hueco de tiempo entre las dos cosas. Si durante ese hueco llegan varias peticiones a la vez, todas leen el cupón todavía disponible y todas lo canjean.

## Qué es una condición de carrera TOCTOU

TOCTOU viene de "Time Of Check to Time Of Use": el momento en que se comprueba algo y el momento en que se usa. El servidor hace dos pasos separados:

1. Comprobar: "¿este cupón sigue disponible?" (lee la base de datos).
2. Actuar: aplicar el descuento y marcar el cupón como usado (escribe la base de datos).

Entre esos dos pasos pasa algo de tiempo. La consulta a la base de datos, la validación, el cálculo del descuento. Si el código supone que el mundo no cambia entre el paso 1 y el paso 2, se equivoca: en un servidor web, decenas de peticiones corren en paralelo. Cuando muchas ejecutan el paso 1 antes de que cualquiera llegue al paso 2, todas ven el cupón disponible y siguen adelante. La escritura que lo marca como usado llega tarde para todas menos la primera, y para entonces el daño ya está hecho.

El patrón de código vulnerable se ve así:

```python
# VULNERABLE: comprobar y actuar son pasos separados, sin atomicidad
cupon = db.get(codigo)
if not cupon.usado:          # TOC: aqui estaba disponible
    aplicar_descuento(cuenta, cupon.valor)
    cupon.usado = True       # TOU: se marca, pero otros hilos ya pasaron el if
    db.save(cupon)
```

## Cómo lo explota un atacante con Turbo Intruder

Turbo Intruder es una extensión de Burp Suite pensada para enviar muchas peticiones muy rápido y con control fino sobre el momento exacto en que salen. Un atacante la usa así:

Primero captura una petición de canje legítima, por ejemplo `POST /api/canjear` con el código del cupón en el cuerpo. Después la carga en Turbo Intruder y la repite, digamos, 50 veces. La parte importante no es el número sino la sincronización: el objetivo es que las 50 lleguen al servidor dentro de la misma ventana de milisegundos, antes de que la primera termine de escribir "usado" en la base de datos.

Para lograr esa sincronización existe la técnica del "single-packet attack", que Turbo Intruder soporte sobre HTTP/2. En vez de dejar que cada petición compita por su cuenta, el atacante retiene la parte final de cada una y luego libera todas juntas, de modo que el servidor las recibe prácticamente en el mismo instante. Así maximiza la cantidad de peticiones que caen dentro del hueco entre comprobar y actuar.

Si el servidor es vulnerable, varias de esas peticiones pasan el `if not cupon.usado` cuando el cupón todavía figura disponible, y cada una aplica el descuento. Un cupón que debía valer una vez termina acreditándose muchas.

## Demostración en el laboratorio

Para ver el efecto sin tocar ningún sistema externo, en `demo_race.py` simulo 50 canjes concurrentes del mismo cupón con hilos. Una versión comprueba y actúa sin protección; la otra envuelve ambos pasos en un lock para hacerlos indivisibles.

Salida real:

```
Cupon VULNERABLE  -> canjes aceptados: 17
Cupon SEGURO (lock) -> canjes aceptados: 1
```

El cupón vulnerable aceptó 17 canjes de los 50 intentos. El número exacto cambia en cada ejecución, y esa variabilidad es justamente la firma de una condición de carrera: el resultado depende del azar de la planificación de los hilos. El cupón protegido aceptó uno solo, que es el comportamiento correcto.

## Cómo defenderse

La idea de fondo es que comprobar y actuar dejen de ser dos pasos separables y pasen a ser una sola operación que el sistema no puede partir por la mitad. Hay varias formas de conseguirlo, y en la práctica conviene combinarlas.

**Hacer atómica la operación en la base de datos.** En lugar de leer, decidir en el código y después escribir, dejar que la base de datos decida y escriba en un mismo paso. Un `UPDATE` condicional sirve bien:

```sql
UPDATE cupones
SET usado = TRUE, usado_por = :cuenta
WHERE codigo = :codigo AND usado = FALSE;
```

La base de datos garantiza que esa fila se actualiza de a una petición por vez. La primera que llega cambia `usado` a `TRUE` y afecta una fila; todas las siguientes encuentran `usado = FALSE` ya falso y afectan cero filas. El código solo aplica el descuento si el `UPDATE` reporta una fila modificada. Si reporta cero, el cupón ya estaba gastado.

**Bloqueo a nivel de fila.** Dentro de una transacción, leer el cupón con `SELECT ... FOR UPDATE`. Eso bloquea esa fila hasta que la transacción termina, así que las demás peticiones esperan su turno en vez de leer en paralelo un estado desactualizado.

**Restricción de unicidad.** Registrar cada canje en una tabla con una clave única que combine cupón y, según la regla de negocio, la cuenta. Un segundo canje que viole esa unicidad es rechazado por la propia base de datos. Esta red de seguridad funciona aunque falle la lógica de la aplicación.

**Control de concurrencia explícito.** Para recursos muy disputados, un bloqueo distribuido (por ejemplo con Redis) o una cola que serialice los canjes del mismo cupón evita que dos procesos lo toquen a la vez. Es más pesado y se reserva para cuando las opciones anteriores no alcanzan.

**Límite de frecuencia.** Un rate limit sobre el endpoint de canje no corrige la causa, pero reduce cuántas peticiones simultáneas puede lanzar un atacante y, con eso, su probabilidad de acertar dentro de la ventana. Es una capa más, no un reemplazo de la atomicidad.

## Conclusión

El fallo no está en validar el cupón, sino en suponer que nada cambia entre validarlo y marcarlo como usado. En un servidor que atiende muchas peticiones a la vez, esa suposición se rompe, y herramientas como Turbo Intruder existen precisamente para romperla a propósito. La defensa correcta es que la comprobación y la escritura ocurran como una sola operación indivisible, apoyada en lo que la base de datos ya sabe hacer: `UPDATE` condicional, bloqueo de fila o una restricción de unicidad.

## Referencias

- OWASP — Testing for Race Conditions (WSTG)
- PortSwigger Web Security Academy — Race conditions, y la investigación del single-packet attack
- CWE-367: Time-of-check Time-of-use (TOCTOU) Race Condition
- CWE-362: Concurrent Execution using Shared Resource with Improper Synchronization
