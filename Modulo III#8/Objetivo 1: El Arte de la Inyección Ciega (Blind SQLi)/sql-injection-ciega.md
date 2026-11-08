# Inyección SQL a ciegas: Boolean-Based vs Time-Based

## El problema de fondo

Cuando una aplicación es vulnerable a inyección SQL pero no muestra los resultados de la consulta ni los mensajes de error de la base, hablamos de inyección "a ciegas" (blind SQL injection). El atacante sabe que puede inyectar, porque la aplicación reacciona distinto según lo que mete, pero no ve el dato directamente en pantalla. Lo tiene que inferir.

Las dos técnicas clásicas para esto son la basada en booleanos y la basada en tiempo. Las dos resuelven el mismo problema, extraer información sin verla, pero usan canales distintos para hacerlo.

## Boolean-Based Blind

La idea es convertir cada pregunta sobre la base en una condición verdadero/falso, y leer la respuesta en el comportamiento de la aplicación.

Supongamos un parámetro vulnerable:

```
https://ejemplo.com/producto?id=5
```

El backend arma algo como:

```sql
SELECT * FROM productos WHERE id = 5
```

El atacante prueba dos payloads que deberían dar resultados opuestos:

```sql
id=5 AND 1=1   -- condición verdadera
id=5 AND 1=2   -- condición falsa
```

Con `1=1`, la consulta devuelve el producto 5 normalmente: aparece la ficha, el precio, la imagen. Con `1=2`, la condición del WHERE nunca se cumple, la consulta no devuelve filas, y la página sale distinta: ficha vacía, "producto no encontrado", o simplemente falta un bloque que antes estaba.

Esa diferencia visible es el canal. El atacante no necesita el dato en texto; le alcanza con poder distinguir "verdadero" de "falso" mirando la respuesta. A partir de ahí reemplaza `1=1` por preguntas reales sobre la base.

## Time-Based Blind

¿Qué pasa cuando la página se ve idéntica en los dos casos? Cuando `1=1` y `1=2` devuelven exactamente la misma pantalla, el canal booleano se cae, porque no hay nada que observar.

Ahí entra la técnica basada en tiempo. En lugar de leer la respuesta en el contenido, el atacante la lee en cuánto tarda el servidor en responder. La pieza central es una función que le dice a la base "si esta condición es verdadera, dormí X segundos antes de contestar".

En MySQL:

```sql
id=5 AND IF(1=1, SLEEP(5), 0)
```

En PostgreSQL se usa `pg_sleep()`, en SQL Server `WAITFOR DELAY '0:0:5'`. El concepto es el mismo.

Si la condición es verdadera, la respuesta tarda cinco segundos de más. Si es falsa, vuelve al instante. El atacante no mira la página: mira el cronómetro. "Tardó cinco segundos" significa verdadero. "Volvió enseguida" significa falso.

## La diferencia en una línea

Las dos técnicas hacen preguntas de sí/no a la base. Cambia el canal por el que leen la respuesta:

- Boolean-based lee la respuesta en el contenido de la página (cambia o no cambia).
- Time-based la lee en el tiempo de respuesta (tarda o no tarda).

Boolean-based es más rápida y confiable cuando hay alguna diferencia visible. Time-based es más lenta, porque hay que esperar el delay en cada pregunta, y más ruidosa, porque la latencia de red mete error. Pero funciona cuando la aplicación no muestra ninguna diferencia. Es el recurso cuando no queda otro canal.

## Extraer el nombre de la base letra por letra

Acá está la pregunta concreta: la app solo muestra "Cargando..." y nada más. La misma pantalla pase lo que pase. No hay diferencia de contenido, así que el canal útil es el tiempo.

El atacante no puede pedir "decime el nombre de la base" y leerlo, porque no hay dónde leerlo. Lo que hace es reconstruir el nombre un carácter a la vez, y cada carácter lo saca a fuerza de preguntas de sí/no.

### Paso 1: aislar un carácter

Primero se queda con una sola letra del nombre. La función `SUBSTRING` (o `SUBSTR`) corta un carácter en una posición dada:

```sql
SUBSTRING(database(), 1, 1)   -- primer carácter del nombre de la base
```

`database()` devuelve el nombre de la base actual. El `SUBSTRING(..., 1, 1)` se queda con el primer carácter.

### Paso 2: convertir la letra en un número

Comparar letras directamente es incómodo. Se pasa el carácter a su código ASCII, un número entre 0 y 127:

```sql
ASCII(SUBSTRING(database(), 1, 1))
```

Si el primer carácter es "s", esto vale 115. Pero el atacante todavía no sabe que es 115. Lo va a averiguar preguntando.

### Paso 3: búsqueda binaria sobre el número

Acá está el truco que hace que esto sea práctico. En vez de probar los 128 valores posibles de a uno, el atacante parte el rango a la mitad en cada pregunta:

```sql
id=5 AND IF(ASCII(SUBSTRING(database(),1,1)) > 64, SLEEP(3), 0)
```

"¿El código del primer carácter es mayor que 64?" Si el servidor tarda tres segundos, la respuesta es sí. Si vuelve enseguida, es no. Cada pregunta descarta la mitad de los valores que quedaban:

- ¿> 64? Tarda. Sí. (está entre 65 y 127)
- ¿> 96? Tarda. Sí. (entre 97 y 127)
- ¿> 112? Tarda. Sí. (entre 113 y 127)
- ¿> 120? Vuelve rápido. No. (entre 113 y 120)
- ¿> 116? Vuelve rápido. No. (entre 113 y 116)
- ¿> 114? Tarda. Sí. (entre 115 y 116)
- ¿> 115? Vuelve rápido. No. (es 115)

Siete preguntas para fijar un carácter, en lugar de hasta 128. Eso es log₂(128). El código 115 es "s".

### Paso 4: repetir y avanzar de posición

Con la primera letra resuelta, se mueve a la segunda cambiando el índice del SUBSTRING:

```sql
ASCII(SUBSTRING(database(), 2, 1))   -- segundo carácter
ASCII(SUBSTRING(database(), 3, 1))   -- tercero
```

Y así hasta que el SUBSTRING devuelve vacío (código 0 o NULL), que es la señal de que se terminó el nombre. Juntando los caracteres 115, 104, 111, 112 queda "shop".

### Por qué se automatiza

Hacer esto a mano es inviable. Un nombre de ocho caracteres son unas 56 peticiones (8 × 7), cada una con su espera de tres segundos. Por eso nadie lo hace a mano: herramientas como sqlmap arman los payloads, miden los tiempos, corren la búsqueda binaria y reconstruyen la cadena solas. El mismo método sirve para sacar nombres de tablas, columnas y después los datos, encadenando subconsultas.

## Cómo se corta esto del lado defensivo

Toda esta familia de ataques vive de una sola causa: que la entrada del usuario termina formando parte de la consulta. Las defensas apuntan ahí.

**Consultas parametrizadas (prepared statements).** Es la defensa de fondo. El valor que manda el usuario viaja como parámetro, separado del texto SQL, así que `5 AND SLEEP(5)` se trata como el valor literal "5 AND SLEEP(5)" y nunca como código. Un ORM bien usado te da esto por defecto.

**Validación de entrada.** Si `id` tiene que ser un entero, rechazá cualquier cosa que no lo sea antes de que llegue a la base.

**Menor privilegio en la cuenta de la base.** La cuenta que usa la aplicación no debería poder leer `information_schema` entero ni funciones que no necesita. Limita hasta dónde llega el atacante si entra.

**WAF y límites de tiempo.** Un WAF frena payloads conocidos, y un timeout agresivo en las consultas le quita filo a la técnica basada en tiempo, porque un SLEEP largo se corta solo. No es defensa principal, pero ayuda.

**Monitoreo.** Una ráfaga de peticiones casi idénticas al mismo endpoint, muchas con respuestas de exactamente tres o cinco segundos, es un patrón bastante delator. Vale la pena alertar sobre eso.

Las parametrizadas son el núcleo; el resto son capas. Si la consulta nunca mezcla datos con código, el canal (sea contenido o tiempo) deja de existir, porque ya no hay inyección que leer.
