# Directory Listing habilitado: qué ocurre técnicamente y cómo mitigarlo

## 1. Qué es el Directory Listing

Cuando un servidor web (Apache, Nginx, IIS) recibe una petición hacia una **ruta de directorio** (no hacia un archivo específico), su comportamiento depende de una configuración puntual:

- **Comportamiento esperado/seguro**: el servidor busca dentro de ese directorio un **archivo de índice** predefinido (`index.html`, `index.php`, `default.aspx`, etc.) y lo sirve automáticamente como respuesta. Si ese archivo de índice no existe, y la directiva de Directory Listing está **deshabilitada**, el servidor responde con un error **403 Forbidden** — niega el acceso al listado, sin revelar nada sobre el contenido del directorio.
- **Comportamiento inseguro (Directory Listing habilitado)**: si no hay archivo de índice y la directiva está **activa**, el servidor, en lugar de negar el acceso, **genera dinámicamente una página HTML que lista todos los archivos y subdirectorios contenidos en esa ruta**, con enlaces clicables a cada uno.

## 2. Qué ocurre técnicamente cuando el atacante navega a ese directorio

### El mecanismo interno
Esta directiva existe a nivel de configuración del servidor web:
- En **Apache**, se controla mediante la directiva `Options +Indexes` (habilitado) u `Options -Indexes` (deshabilitado), configurable globalmente o por directorio (en `httpd.conf`, `apache2.conf`, o en un archivo `.htaccess` local).
- En **Nginx**, se controla con la directiva `autoindex on;` (habilitado) o `autoindex off;` (deshabilitado, que es el valor por defecto de Nginx desde el inicio, a diferencia de versiones antiguas de Apache donde a veces venía activado).

Cuando la directiva está en `on`/`+Indexes`, el servidor usa un módulo interno (`mod_autoindex` en Apache, el módulo `ngx_http_autoindex_module` en Nginx) que **lee el contenido real del sistema de archivos en esa ruta** y lo traduce a una página HTML generada al vuelo — típicamente una tabla simple con columnas de nombre, fecha de modificación y tamaño de cada archivo/carpeta.

### La secuencia concreta del lado del atacante
1. Durante la fase de enumeración (reconocimiento activo), el atacante prueba rutas conocidas o descubre directorios mediante fuzzing de rutas (herramientas como `dirb`, `gobuster`, `ffuf`), o incluso mediante Google Dorking como vimos antes (`intitle:"index of"`).
2. Al acceder a una de esas rutas que no tiene archivo índice, en lugar de un 403, recibe un **200 OK** con el listado completo del directorio.
3. El atacante ahora tiene **visibilidad completa de la estructura de archivos** en ese punto del servidor — nombres exactos, extensiones, tamaños, fechas de modificación (que pueden indicar, por ejemplo, qué archivo fue tocado más recientemente, sugiriendo relevancia).
4. Puede navegar recursivamente por subdirectorios listados, haciendo clic en cada carpeta, ampliando el mapa de todo lo que el servidor expone sin querer.
5. Finalmente, descarga directamente cualquier archivo del listado con una simple petición `GET`, sin necesidad de adivinar su nombre exacto — el servidor se lo mostró gratuitamente.

Esto convierte lo que normalmente sería un proceso de reconocimiento lento y basado en conjeturas (probar nombres de archivo uno por uno) en un **mapa completo y preciso del sistema de archivos**, entregado directamente por el propio servidor.

## 3. Qué tipo de archivos sensibles suelen quedar expuestos

En la práctica, los directorios con Directory Listing habilitado suelen contener, por descuido del equipo de desarrollo/operaciones, archivos que nunca debieron estar dentro del directorio público (document root) del servidor web:

### a) Backups y copias de seguridad
`backup.zip`, `site_backup_2024.tar.gz`, `database_dump.sql`, `wp-config.php.bak` — el mismo tipo de archivo que analizamos antes al hablar de Google Dorking y de Security Misconfiguration. Estos backups frecuentemente contienen **la base de datos completa** o **archivos de configuración con credenciales**.

### b) Archivos de configuración
`.env`, `config.php.old`, `web.config.bak`, `settings.ini` — suelen contener credenciales de base de datos, claves de API, secretos de sesión (`SECRET_KEY`), tokens de servicios externos (pasarelas de pago, servicios de email).

### c) Código fuente sin compilar/transpilar
Archivos `.php`, `.py`, `.java` o `.cs` que, si se descargan en lugar de ejecutarse (algo que puede ocurrir si están fuera del alcance de ejecución del intérprete, o en una carpeta mal configurada), revelan directamente la lógica de negocio y, potencialmente, vulnerabilidades de código (consultas SQL construidas de forma insegura, validaciones ausentes) — retomando lo que analizamos sobre stack traces: ver el código fuente real es, en cierto modo, un nivel de divulgación aún mayor que ver solo un stack trace puntual.

### d) Archivos de control de versiones
Carpetas `.git/` o `.svn/` expuestas permiten, en el peor caso, reconstruir **todo el historial de commits** del proyecto, incluyendo credenciales que fueron agregadas y luego "eliminadas" en un commit posterior (pero que siguen existiendo en el historial de git, recuperables).

### e) Logs del servidor o de la aplicación
`error.log`, `access.log`, `debug.log` — pueden contener rutas internas, trazas de errores detalladas (conectando con el informe anterior sobre Stack Traces), direcciones IP de usuarios, o incluso credenciales que quedaron registradas accidentalmente por un log mal diseñado.

### f) Documentos internos y archivos de datos
Hojas de cálculo, documentos PDF o Word con información interna de la empresa, listas de usuarios exportadas en CSV, certificados (`.pem`, `.key`, `.pfx`) con claves privadas.

## 4. Por qué esto conecta directamente con los temas ya analizados

Vale la pena notar que este hallazgo no vive aislado — es consistente con varios conceptos de esta misma serie de análisis:
- Es un ejemplo de manual de **Security Misconfiguration**, la misma categoría que vimos con el `wp-config.php.bak` expuesto.
- El operador de Google Dorking **`intitle:"index of"`**, que mencionamos al hablar de reconocimiento pasivo, es exactamente la técnica usada para *encontrar* servidores con esta configuración insegura desde un buscador, sin necesidad de interactuar directamente con el sitio.
- El impacto recae principalmente sobre el pilar de **Confidencialidad** de la tríada CIA (se expone información que debía permanecer restringida), aunque como vimos, también puede escalar hacia Integridad o Disponibilidad si entre los archivos filtrados hay credenciales que permiten modificar datos o tomar control del sistema.

## 5. Cómo se mitiga

### a) Deshabilitar explícitamente el listado de directorios
La medida más directa y fundamental:
- **Apache**: `Options -Indexes` en la configuración global o por directorio.
- **Nginx**: `autoindex off;` (que además es el valor por defecto, por lo que este riesgo suele aparecer más por configuraciones heredadas o mal replicadas que por el comportamiento nativo del servidor).

### b) Asegurar que todo directorio público tenga su archivo de índice
Colocar un `index.html` (incluso vacío o con un mensaje genérico) en cada directorio servido públicamente, de modo que, aunque por algún motivo el listado quedara habilitado, el servidor siempre encuentre primero ese archivo antes de intentar generar el listado.

### c) Nunca almacenar archivos sensibles dentro del document root
La mitigación estructuralmente más sólida: los backups, archivos de configuración, código fuente sin servir, y cualquier archivo que no esté destinado a ser descargado públicamente **no deberían existir, en ningún momento, dentro de la carpeta que el servidor web expone** (`/var/www/html`, `wwwroot`, etc.). Deberían almacenarse fuera del árbol público, en rutas a las que el servidor web no tenga ningún mapeo de URL.

### d) Reglas explícitas de denegación por tipo de archivo o patrón
Como capa de defensa adicional (defensa en profundidad), bloquear explícitamente el acceso vía HTTP a extensiones y patrones sensibles, incluso si accidentalmente quedaran dentro del document root:
```apache
# Apache (.htaccess o configuración del VirtualHost)
<FilesMatch "\.(bak|sql|env|log|git|config)$">
    Require all denied
</FilesMatch>
```
```nginx
# Nginx
location ~* \.(bak|sql|env|log)$ {
    deny all;
}
```

### e) Automatización y escaneo de configuración
Conectando con lo que analizamos sobre mitigación de Security Misconfiguration en entornos cloud: usar **Infrastructure as Code** con configuraciones de servidor ya endurecidas (hardened) por defecto, y escaneos automatizados periódicos (tanto internos como mediante herramientas de pentesting) que detecten si algún directorio quedó expuesto con listado habilitado, antes de que lo encuentre un atacante o un Google Dork.

### f) Principio de mínimo privilegio en el sistema de archivos
Asegurar que el usuario/proceso con el que corre el servidor web tenga permisos de lectura únicamente sobre lo estrictamente necesario, reduciendo qué podría llegar a listarse o servirse incluso en caso de una mala configuración puntual.

## 6. Conclusión

El Directory Listing habilitado convierte al propio servidor web en un cómplice involuntario del reconocimiento del atacante: en lugar de tener que adivinar o fuerza-brutear nombres de archivo, el servidor entrega directamente un mapa completo y veraz de su estructura de archivos. El riesgo real no está en la directiva en sí —que puede tener usos legítimos, como un repositorio de descargas públicas intencional— sino en la combinación letal de tenerla activa **junto con** archivos sensibles (backups, configuraciones, código fuente, control de versiones) alojados dentro del directorio público. La mitigación más robusta combina deshabilitar la directiva a nivel de servidor, garantizar que los archivos sensibles nunca residan en esa ubicación en primer lugar, y reforzar ambas medidas con reglas explícitas de denegación y verificación automatizada periódica de la configuración.
