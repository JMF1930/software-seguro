# Por qué las consultas parametrizadas cierran la inyección SQL

## El error de raíz de las listas negras

Bloquear palabras como `SELECT` o `UNION`, o escapar comillas, ataca los síntomas. La causa de la inyección SQL es una sola: el dato del usuario y el código SQL viajan en el mismo string, y la base recibe esa mezcla y la interpreta entera como un solo programa.

Cuando escapás comillas, seguís armando ese string mezclado; apenas agregás una capa que intenta adivinar qué parte es dato. Y adivinar es evadible. Hay codificaciones (Unicode, hex), comentarios que parten palabras, juegos con el conjunto de caracteres (el clásico problema del `GBK` y las comillas, o un `\` que se come tu escape), y funciones que no necesitan comillas ni la palabra prohibida. Una lista negra solo conoce lo que alguien ya pensó en prohibir. El atacante solo necesita una variante que no esté en la lista.

Las consultas parametrizadas atacan la causa. Separan el dato del código antes de que la base vea nada, y lo hacen a nivel de protocolo, no de string.

## Las dos fases: prepare y execute

Una consulta parametrizada parte la operación en dos pasos que hasta viajan en mensajes distintos del protocolo cliente-servidor.

### Fase 1: prepare

El cliente le manda a la base la plantilla de la consulta, con huecos marcados donde van a ir los datos. Los huecos son los placeholders (`?` en MySQL y JDBC, `$1`, `$2` en PostgreSQL, `:nombre` en otros):

```sql
SELECT * FROM usuarios WHERE nombre = ? AND pass = ?
```

La base recibe esto y hace todo el trabajo de interpretación ahora, con los huecos todavía vacíos:

- Pasa el parser: analiza la gramática SQL del string.
- Arma el árbol sintáctico de la consulta (qué es un SELECT, dónde está el WHERE, cuáles son las columnas).
- Resuelve tablas y columnas, y arma el plan de ejecución.

Al terminar esta fase, la estructura de la consulta queda congelada. La base ya decidió que esto es un SELECT sobre `usuarios` con dos comparaciones en el WHERE. Esa decisión está tomada, y los datos todavía no llegaron.

### Fase 2: execute

Recién ahora el cliente manda los valores, en un mensaje separado:

```
param 1 = admin' OR '1'='1
param 2 = loquesea
```

La base toma cada valor y lo ata (bind) al hueco correspondiente del plan que ya compiló. El punto central: estos valores entran como contenido de un campo, nunca vuelven a pasar por el parser de SQL. La base no reinterpreta la gramática. El plan ya está hecho; lo único que falta es rellenar los huecos con datos tratados como datos.

## Por qué esto neutraliza cualquier carácter

Mirá qué pasa con `admin' OR '1'='1` como valor del parámetro 1.

En una consulta armada por concatenación, ese string se pegaría dentro del texto SQL, la comilla cerraría el literal, y el `OR '1'='1` pasaría a ser parte de la lógica del WHERE. Clásico bypass de login.

En una consulta parametrizada, ese mismo string llega en la fase de execute, después de que el parser ya cerró. La base lo ata al hueco como el valor literal completo del campo `nombre`. O sea, busca un usuario cuyo nombre sea, carácter por carácter, `admin' OR '1'='1`, comilla incluida. No existe ningún usuario así, la consulta no devuelve filas, y el login falla. La comilla nunca tuvo poder de cerrar nada, porque para cuando apareció ya no había ningún parser escuchando.

Y acá está el motivo por el que da igual qué mande el usuario. La seguridad no viene de inspeccionar los caracteres y decidir si son peligrosos. Viene de cuándo entra el dato. Entra después de que la estructura quedó fija, así que no hay carácter, codificación ni palabra reservada que pueda cambiar una estructura que ya está cerrada. `SELECT`, `UNION`, `--`, `;`, mil comillas: todo eso es texto dentro de un campo. No hay nada que evadir, porque no hay una lista de prohibidos que esquivar. El canal por el que se cuela una inyección (meter código donde se espera un dato) directamente no existe.

La diferencia con el escaping es temporal. El escaping procesa el dato y lo mete igual en el string de código, confiando en que lo transformó bien. La parametrización nunca lo mete en el código; lo manda por otro carril.

## Dónde sí seguís siendo vulnerable

Las parametrizadas resuelven el 100% de la inyección sobre los datos, pero conviene saber qué no cubren, porque es donde los equipos se confían de más.

Los placeholders solo valen para datos, no para identificadores ni palabras clave. No podés parametrizar el nombre de una tabla, de una columna, ni la dirección de un `ORDER BY`:

```sql
SELECT * FROM usuarios ORDER BY ? ASC   -- el ? NO sirve para el nombre de columna
```

Si tu aplicación arma el nombre de la columna o de la tabla a partir de entrada del usuario, estás concatenando código de nuevo, y la inyección vuelve. Para esos casos la defensa es otra: validar contra una lista blanca de valores permitidos (los nombres de columna que de verdad existen y tienen sentido), no parametrizar.

El segundo agujero es usar parametrizadas mal. Si construís el string concatenando y después se lo pasás a la API de prepared statement, no ganaste nada; el daño ya está hecho antes del bind. La protección existe solo si el valor del usuario entra por el placeholder, nunca por el texto de la consulta.

## En una línea

Una lista negra pregunta "¿este carácter es peligroso?" y siempre se le escapa alguno. Una consulta parametrizada no pregunta nada sobre el contenido: fija la estructura de la consulta antes de recibir el dato, y le entrega el dato a la base por un canal donde ya no se interpreta como SQL. Mientras el dato entre por el placeholder y no por el string, no hay carácter que pueda convertirse en código.
