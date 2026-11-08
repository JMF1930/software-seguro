# SSRF en AWS: el endpoint 169.254.169.254 y el robo de credenciales

## Por qué un SSRF en cloud es tan grave

Un Server-Side Request Forgery hace que el servidor de la aplicación ejecute una petición HTTP que elige el atacante. El atacante no llega a la red interna por su cuenta; usa al servidor como intermediario, y el servidor habla desde adentro, con la confianza y la ubicación de red que tiene la aplicación.

En un entorno cloud como AWS, esa posición de adentro es lo que vuelve catastrófico al SSRF. La máquina donde corre la aplicación (una instancia EC2) tiene acceso a un servicio interno que no debería ver nadie de afuera, y ese servicio entrega, entre otras cosas, las credenciales de la instancia. Si el atacante puede dirigir una petición del servidor hacia ese servicio, se lleva las llaves.

## Qué es 169.254.169.254

Esa dirección es el Instance Metadata Service (IMDS) de AWS. Es una IP link-local, del rango `169.254.0.0/16`, que por definición solo es alcanzable desde la propia máquina; no se enruta por la red, no sale a internet, y desde afuera no existe. Cada instancia EC2 la tiene disponible internamente en esa misma dirección.

El IMDS existe para que una instancia pueda consultar datos sobre sí misma sin tener credenciales cargadas a mano: su ID, su región, su configuración de red, los datos de arranque (user-data), y lo más delicado, las credenciales temporales del rol de IAM que tiene asignado. Todo eso se sirve por HTTP plano, sin autenticación, bajo el supuesto de que solo la propia instancia puede preguntar.

Ese supuesto es justo lo que rompe un SSRF. Si la aplicación hace peticiones a URLs que controla el atacante, este le pide que consulte `http://169.254.169.254/...`, y la instancia, obediente, le devuelve lo que el servicio responde.

## Qué busca extraer el atacante

El premio está en las credenciales del rol de IAM. El recorrido es este.

Primero, el atacante pregunta qué rol tiene la instancia:

```
GET http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

Eso devuelve el nombre del rol asociado, por ejemplo `mi-app-rol-ec2`. Con ese nombre, pide las credenciales:

```
GET http://169.254.169.254/latest/meta-data/iam/security-credentials/mi-app-rol-ec2
```

Y el servicio responde con un JSON que contiene las credenciales temporales del rol:

```json
{
  "AccessKeyId": "ASIA...",
  "SecretAccessKey": "...",
  "Token": "...",
  "Expiration": "2026-11-08T18:00:00Z"
}
```

Esas tres piezas (el access key, el secret key y el token de sesión) son credenciales válidas de AWS. El atacante las carga en su propia máquina, por ejemplo como variables de entorno del AWS CLI, y a partir de ahí actúa ante la API de AWS como si fuera la instancia, con todos los permisos que el rol tenga concedidos. No tuvo que vulnerar AWS; usó una credencial legítima que el propio servicio le entregó.

Por qué esto es tan peligroso: el rol suele tener más permisos de los estrictamente necesarios, porque asignar privilegios mínimos es trabajoso y muchos equipos lo dejan amplio. Con esas credenciales, según el rol, el atacante puede listar y leer buckets de S3, consultar bases de datos, leer secretos de Secrets Manager, levantar o apagar instancias, y en los peores casos escalar hacia el control de la cuenta entera. Un SSRF que parecía una molestia menor se convierte en una brecha de toda la infraestructura.

Además del rol, el atacante puele mirar el user-data, que a veces guarda scripts de arranque con secretos incrustados (claves, tokens, contraseñas que alguien dejó ahí por comodidad):

```
GET http://169.254.169.254/latest/user-data
```

## IMDSv1 contra IMDSv2

Lo que acabo de describir es IMDSv1, el esquema original, donde una sola petición GET alcanza para obtener las credenciales. Eso es exactamente lo que un SSRF puede forzar, porque un SSRF típico solo te deja provocar un GET a una URL.

AWS introdujo IMDSv2 justamente para cortar este vector. Cambia el acceso a un esquema orientado a sesión: antes de poder leer nada, hay que pedir un token con una petición `PUT`, y después usar ese token en una cabecera en cada lectura:

```
PUT http://169.254.169.254/latest/api/token
  con la cabecera: X-aws-ec2-metadata-token-ttl-seconds: 21600
```

El token que devuelve se manda luego así:

```
GET http://169.254.169.254/latest/meta-data/...
  con la cabecera: X-aws-ec2-metadata-token: <token>
```

Esto frena al SSRF por dos razones. Primero, la mayoría de los SSRF solo permiten GET, y acá hace falta un PUT para arrancar, cosa que muchas aplicaciones vulnerables no pueden forzar. Segundo, hay que agregar una cabecera HTTP personalizada, y un SSRF rara vez controla las cabeceras de la petición que provoca. IMDSv2 agrega además un límite de saltos de red (hop limit) en el campo TTL del paquete, puesto por defecto en 1, lo que impide que la respuesta del metadata atraviese un proxy o un contenedor hacia afuera.

Por eso la defensa principal es obligar a IMDSv2 en todas las instancias y deshabilitar IMDSv1, de modo que una sola petición GET ya no alcance para sacar credenciales.

## Cómo se defiende, en capas

Forzar IMDSv2 es el primer paso y el más efectivo contra este ataque puntual, pero conviene sumar capas, porque IMDSv2 mitiga la explotación del metadata, no el SSRF en sí.

Privilegio mínimo en el rol de IAM. Si el rol de la instancia solo puede hacer lo justo que la aplicación necesita, las credenciales robadas valen mucho menos. Es la diferencia entre que el atacante lea un bucket específico o que controle la cuenta.

Arreglar el SSRF en el código. La causa es que la aplicación hace peticiones a URLs que vienen del usuario sin validarlas. La defensa es validar el destino contra una lista blanca de dominios permitidos, resolver el nombre y bloquear direcciones internas y link-local (todo `169.254.0.0/16`, `127.0.0.0/8`, los rangos privados), y no seguir redirecciones hacia esas direcciones.

Controles de red. Reglas de salida (egress) que impidan que la instancia inicie conexiones hacia donde no debe, y a nivel de host, bloquear el acceso al metadata desde los procesos que no lo necesitan.

## En una línea

Un SSRF en AWS es grave porque deja que el atacante haga que el servidor consulte 169.254.169.254, el servicio de metadata interno, y de ahí extraiga las credenciales temporales del rol de IAM de la instancia, con las que actúa ante la API de AWS con los permisos de la aplicación. La defensa central es obligar IMDSv2, que exige un PUT y una cabecera que un SSRF común no puede forjar, sumado a roles de privilegio mínimo y a validar los destinos de las peticiones salientes.
