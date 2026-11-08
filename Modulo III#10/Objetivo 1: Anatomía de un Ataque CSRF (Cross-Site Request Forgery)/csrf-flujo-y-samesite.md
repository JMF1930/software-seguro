# CSRF: el flujo del ataque y cómo lo frena SameSite

## Qué es, en una frase

Un ataque CSRF (Cross-Site Request Forgery) hace que el navegador de una víctima autenticada envíe, sin que ella lo sepa, una petición a un sitio donde tiene sesión abierta. El atacante no roba la sesión ni la contraseña; usa la sesión que ya está activa para que el sitio ejecute una acción en nombre de la víctima.

## El flujo paso a paso

Tomemos el caso típico: la víctima tiene sesión abierta en su banco, `banco.com`.

1. La víctima inicia sesión en `banco.com`. El servidor le manda una cookie de sesión, y el navegador la guarda asociada a ese dominio. La sesión queda viva mientras la cookie no expire.

2. Sin cerrar esa sesión, la víctima abre otra pestaña y entra a un sitio del atacante, `malicioso.com`. Llegó ahí por un correo, un enlace en un foro, un anuncio, lo que sea.

3. La página de `malicioso.com` contiene, escondido, código que dispara una petición a `banco.com`. Puede ser un formulario que se autoenvía con JavaScript:

```html
<form action="https://banco.com/transferir" method="POST" id="f">
  <input type="hidden" name="destino" value="cuenta-del-atacante">
  <input type="hidden" name="monto" value="5000">
</form>
<script>document.getElementById("f").submit();</script>
```

O algo tan simple como una imagen, si la acción se puede hacer por GET:

```html
<img src="https://banco.com/transferir?destino=atacante&monto=5000">
```

4. El navegador de la víctima procesa esa página y, al encontrar el formulario o la imagen, lanza la petición a `banco.com`. Acá está el punto central: el navegador adjunta automáticamente las cookies de `banco.com` a cualquier petición dirigida a `banco.com`, sin importar desde qué página se originó. Así que la petición forjada viaja con la cookie de sesión válida de la víctima.

5. `banco.com` recibe una petición con una cookie de sesión legítima, bien formada, y no tiene cómo saber que no la originó la víctima a propósito. La procesa como auténtica. La transferencia se hace.

La víctima nunca vio nada. No tipeó un monto ni confirmó una transferencia. Su navegador, de fondo, mandó la petición que el atacante preparó.

## Por qué ocurre: la autoridad ambiental de las cookies

La causa de raíz está en cómo los navegadores manejan las cookies. Una cookie se envía de forma automática en toda petición que vaya a su dominio, sin importar qué sitio generó esa petición. El navegador no pregunta "¿esta petición nació en banco.com o en otra página?". Solo ve que va dirigida a `banco.com` y adjunta la cookie correspondiente.

Esto se llama a veces autoridad ambiental: la credencial (la cookie) está siempre presente en el ambiente y se aplica sola, sin que nadie la invoque explícitamente. Es cómodo, porque hace que no tengas que volver a loguearte en cada clic. Y es justo lo que el CSRF explota: el atacante no necesita la cookie, ni leerla ni copiarla. Le alcanza con provocar una petición hacia el sitio, porque sabe que el navegador va a poner la cookie por su cuenta.

Por eso el `HttpOnly` no sirve de nada contra CSRF. Ese flag impide que el JavaScript lea la cookie, pero el CSRF nunca la lee; deja que el navegador la mande solo. El problema no es el acceso a la cookie, es el envío automático.

La condición para que el ataque funcione es que la acción dependa únicamente de la cookie para autorizarse. Si el servidor exige además algún dato que `malicioso.com` no puede conocer ni adivinar, el ataque se cae. De ahí salen las defensas.

## Cómo lo mitiga el atributo SameSite

`SameSite` es un atributo de la cookie que cambia justamente la regla del paso 4: le dice al navegador cuándo puede adjuntar esa cookie en peticiones que vienen de otro sitio. Rompe el supuesto sobre el que se apoya todo el CSRF, el envío automático entre sitios.

El servidor lo setea al emitir la cookie:

```
Set-Cookie: session=abc123; HttpOnly; Secure; SameSite=Lax
```

Tiene tres valores.

### SameSite=Strict

El navegador manda la cookie solo si la petición se origina en el mismo sitio que la cookie. Si la petición nace en cualquier otro origen, la cookie no se adjunta, punto. En el ejemplo del banco, la petición del formulario de `malicioso.com` llegaría a `banco.com` sin cookie de sesión, así que el servidor la ve como no autenticada y la rechaza. El CSRF queda cortado.

El costo es de usabilidad. Como la cookie tampoco viaja cuando llegás a `banco.com` desde un enlace externo (un mail, un resultado de Google), la primera carga te aparece como deslogueado aunque tengas sesión. Para un banco puede estar bien; para un sitio donde esperás que seguir un enlace te deje ya adentro, molesta.

### SameSite=Lax

Es el equilibrio, y hoy es el valor por defecto en los navegadores modernos cuando una cookie no declara `SameSite`. Permite que la cookie viaje en peticiones de otro sitio solo si se cumplen dos condiciones a la vez: que sea una navegación de nivel superior (la víctima hace clic en un enlace y la barra de direcciones cambia a `banco.com`) y que use un método seguro, es decir GET. Para todo lo demás, la cookie no se adjunta.

Esto frena el CSRF clásico porque las acciones que cambian estado (transferir, borrar, cambiar el email) deberían ir por POST, y con `Lax` la cookie no acompaña a un POST que viene de otro sitio. El formulario autoenviado de `malicioso.com`, al ser POST entre sitios, llega sin cookie. Al mismo tiempo, como sí permite el GET de una navegación de nivel superior, seguir un enlace a `banco.com` desde un mail te deja logueado, y la experiencia no se rompe.

El matiz importante de `Lax`: deja pasar peticiones GET entre sitios. Si tu aplicación tiene acciones que modifican estado accesibles por GET (un `/eliminar?id=5` que borra con solo visitarlo), `Lax` no te protege contra eso, porque el GET de nivel superior sí lleva la cookie. Es una razón más para no poner nunca acciones con efectos detrás de un GET.

### SameSite=None

Desactiva la protección: la cookie viaja en todas las peticiones entre sitios, como antes de que SameSite existiera. Los navegadores exigen que venga junto con `Secure` (solo por HTTPS). Se usa cuando de verdad necesitás la cookie en un contexto cross-site legítimo, por ejemplo un servicio embebido en iframes de otros dominios o una API que varios orígenes consumen. Si lo elegís, volvés a necesitar otra defensa contra CSRF.

## SameSite no reemplaza a los tokens

`SameSite=Lax` o `Strict` mitigan el CSRF de forma radical y son la primera línea sensata, pero la recomendación de defensa en profundidad es combinarlos con tokens anti-CSRF. El token es un valor aleatorio, único por sesión o por petición, que el servidor incrusta en sus formularios y vuelve a verificar al recibir la acción. Como `malicioso.com` no puede leer ni adivinar ese token (la política de mismo origen le impide ver el contenido de las páginas de `banco.com`), no puede incluirlo en su petición forjada, y el servidor la rechaza por token ausente o inválido.

Los dos mecanismos cubren huecos distintos. SameSite falla si hay acciones de estado por GET, si alguna configuración obliga a `SameSite=None`, o frente a algún caso borde de navegadores viejos que no lo respetan. El token no depende del comportamiento del navegador con las cookies. Tenerlos juntos obliga al atacante a vencer dos barreras a la vez.

## En una línea

El CSRF funciona porque el navegador adjunta la cookie de sesión a toda petición dirigida al sitio, sin importar desde dónde se originó, así que una página maliciosa puede disparar acciones autenticadas sin tocar la cookie. `SameSite=Strict` corta el envío entre sitios por completo, `SameSite=Lax` lo corta para los métodos que cambian estado (POST) y deja pasar la navegación normal por GET, y en ambos casos conviene sumar tokens anti-CSRF como segunda capa.
