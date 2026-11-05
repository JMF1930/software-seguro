# Manipulación de parámetros: cantidad negativa en un carrito de compras

**Asignatura:** Ciberseguridad — Lógica de negocio y validación de entrada
**Fecha:** 05/11/2026
**Enfoque:** Defensivo. Entender la falla para validarla correctamente del lado del servidor.

---

## 1. El escenario

En un e-commerce, al agregar un producto al carrito se envía una petición parecida a:

```http
POST /api/carrito/agregar
Content-Type: application/json

{ "producto_id": 777, "cantidad": 1 }
```

Interceptando la petición (con un proxy o las DevTools) se cambia `"cantidad": 1` por `"cantidad": -5`. El servidor lo procesa sin objetar y calcula el total como `precio × cantidad`. Con una cantidad negativa, el subtotal se vuelve negativo y el total del carrito baja en lugar de subir; en una implementación mal hecha, eso se traduce en **saldo a favor del atacante** en vez de un cobro.

## 2. Por qué ocurre (análisis técnico)

Esta es una vulnerabilidad de **lógica de negocio** combinada con **validación de entrada insuficiente**. La raíz es la misma que ya vimos en IDOR y en `alg: none`: **el servidor confía en un dato que controla el cliente y lo usa sin validarlo.**

Puntos técnicos concretos:

1. **El cliente controla `cantidad`.** Todo lo que viaja en la petición (body, query, headers) es manipulable. Los controles de la interfaz —un selector que solo deja elegir 1, 2, 3…— se ejecutan en el navegador y el atacante los saltea enviando la petición directamente. La validación del cliente es usabilidad, no seguridad.

2. **Ausencia de validación de dominio en el servidor.** El campo `cantidad` tiene un dominio válido evidente: entero y mayor o igual a 1. El backend vulnerable no comprueba ni el **tipo** ni el **rango**, y acepta un entero negativo.

3. **El valor entra directo en una operación aritmética y financiera.** `total = precio × cantidad` con `cantidad = -5` da un subtotal de `-6000`. Si la pasarela de pago o el monedero interpreta un total negativo como un abono, el sistema "paga" al atacante. El signo negativo convierte una compra en un ingreso.

4. **Confianza transitiva.** A menudo el valor manipulado contamina pasos posteriores: actualización de stock (que podría *incrementarse*), cálculo de impuestos, puntos de fidelidad, etc. Un solo campo no validado se propaga por toda la lógica.

### Salida real de la demo (`demo_cantidad.py`)

```
=== Carrito VULNERABLE (cantidad = -5) ===
Total: -6000 <- saldo negativo: el sistema 'paga' al atacante

=== Carrito CORRECTO ===
cantidad=    -5 -> RECHAZADO (cantidad invalida: debe ser >= 1)
cantidad=     0 -> RECHAZADO (cantidad invalida: debe ser >= 1)
cantidad=   2.5 -> RECHAZADO (cantidad invalida: debe ser un entero)
cantidad=   '3' -> RECHAZADO (cantidad invalida: debe ser un entero)
cantidad=   999 -> RECHAZADO (cantidad invalida: excede el maximo por pedido)
cantidad=     2 -> total = 2400
```

El mismo valor malicioso que el carrito vulnerable procesa (`-5`) es rechazado por el validador correcto antes de llegar a la aritmética.

## 3. Cómo debe validar el desarrollador (lado del servidor)

La regla de oro: **validar siempre en el servidor, sobre una lista blanca (lo permitido), rechazando todo lo demás.** El campo `cantidad` debe pasar por estas comprobaciones, en orden, antes de usarse en cualquier cálculo:

1. **Tipo estricto — debe ser un entero.** Rechazar strings (`"3"`), decimales (`2.5`), booleanos, `null` y ausencia del campo. Si llega como texto, convertir de forma segura y fallar si no es convertible.

2. **Límite inferior — estrictamente positiva.** `cantidad >= 1`. Esto es lo que corta de raíz el ataque: un entero negativo o cero se rechaza. (El `0`, según el caso, puede tratarse como "eliminar del carrito", pero nunca como cantidad de compra.)

3. **Límite superior — tope de negocio y stock real.** Un máximo razonable por pedido (p. ej. 10) y nunca más que el stock disponible. Esto evita abusos y desbordamientos lógicos. Conviene además acotar contra el rango del tipo entero para descartar valores absurdamente grandes.

4. **Recalcular el total en el servidor, con el precio del servidor.** Nunca confiar en un precio o subtotal que venga del cliente. El backend toma el `producto_id`, busca el **precio en su propia base de datos** y calcula `total = precio_servidor × cantidad_validada`. El cliente no debe poder influir en el precio ni en el total, solo proponer qué y cuánto.

5. **Defensa en profundidad.** Reforzar con tipos sin signo o restricciones a nivel de base de datos (`CHECK (cantidad >= 1)`), validación de esquema en la capa de API (JSON Schema, DTOs tipados) y una comprobación final antes de cobrar que rechace cualquier total ≤ 0.

### Pseudocódigo de la validación

```python
def validar_cantidad(valor, stock_disponible, max_por_pedido=10):
    if not isinstance(valor, int) or isinstance(valor, bool):
        raise ValueError("cantidad invalida: debe ser un entero")
    if valor < 1:
        raise ValueError("cantidad invalida: debe ser >= 1")
    if valor > max_por_pedido:
        raise ValueError("cantidad invalida: excede el maximo por pedido")
    if valor > stock_disponible:
        raise ValueError("cantidad invalida: excede el stock disponible")
    return valor

# El total SIEMPRE con el precio del servidor y la cantidad ya validada:
cantidad = validar_cantidad(peticion["cantidad"], stock=producto.stock)
total = producto.precio_en_bd * cantidad   # precio nunca viene del cliente
```

## 4. Conclusión

La falla no está en la aritmética, sino en **procesar un dato no confiable sin validar su tipo ni su rango** y en dejar que el cliente influya en un cálculo financiero. La corrección es validación estricta del lado del servidor —entero, `>= 1`, con tope superior— y recálculo del total con datos propios del servidor. Es el mismo principio de todo el trabajo anterior: **el servidor nunca debe confiar en la entrada del cliente; debe imponer sus propias reglas.**

## Referencias

- OWASP — Input Validation Cheat Sheet
- OWASP — Business Logic Vulnerabilities / Testing for Business Logic
- OWASP Top 10 — A04:2021 Insecure Design
- CWE-20: Improper Input Validation; CWE-840: Business Logic Errors
