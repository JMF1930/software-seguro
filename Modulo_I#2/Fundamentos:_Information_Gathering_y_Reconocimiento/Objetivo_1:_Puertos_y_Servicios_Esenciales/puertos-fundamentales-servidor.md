# Puertos fundamentales de un servidor y sus servicios por defecto

Los puertos son "puertas de entrada" lógicas que identifican, dentro de una misma dirección IP, a qué servicio o aplicación va dirigido el tráfico de red. Van del 0 al 65535, y el rango 0–1023 se conoce como **"well-known ports"** (puertos bien conocidos), reservados por convención (IANA) para servicios estándar de internet. A continuación se detallan los solicitados.

## Puerto 20 — FTP (Data)
Usado por **FTP (File Transfer Protocol)** para la **transferencia de datos** en sí (el contenido de los archivos). Trabaja en conjunto con el puerto 21, que maneja el control de la sesión. En modo activo, el servidor abre esta conexión hacia el cliente para enviar/recibir los archivos.

## Puerto 21 — FTP (Control)
También parte de **FTP**, pero se encarga del **canal de control**: autenticación (usuario/contraseña), comandos como listar directorios, cambiar de carpeta, iniciar una subida o descarga, etc. Es un protocolo antiguo que transmite las credenciales en texto plano, por lo que hoy se considera inseguro si no se combina con una capa de cifrado adicional (FTPS) o se reemplaza directamente por SFTP.

## Puerto 22 — SSH
**SSH (Secure Shell)** permite el acceso remoto cifrado a la terminal de un servidor, además de servir como transporte seguro para transferencia de archivos (SCP, SFTP) y túneles. Es el reemplazo moderno y seguro de Telnet y de FTP para tareas de administración remota, ya que cifra tanto las credenciales como el tráfico.

## Puerto 23 — Telnet
**Telnet** permite acceso remoto a una terminal, igual que SSH, pero **sin ningún tipo de cifrado**: usuario, contraseña y todos los comandos viajan en texto plano. Por esta razón está prácticamente en desuso para producción y se lo considera un riesgo de seguridad significativo si está expuesto.

## Puerto 25 — SMTP
**SMTP (Simple Mail Transfer Protocol)** es el protocolo estándar para el **envío** de correo electrónico entre servidores de mail (server-to-server) y desde clientes hacia el servidor de salida. No debe confundirse con la recepción/lectura de correo, que usan otros protocolos (como POP3 o IMAP).

## Puerto 53 — DNS
**DNS (Domain Name System)** traduce nombres de dominio (como `www.ejemplo.com`) a direcciones IP, y viceversa. Usa tanto UDP (para consultas simples y rápidas, la mayoría del tráfico DNS) como TCP (para transferencias de zona o respuestas que exceden el tamaño de un datagrama UDP).

## Puerto 80 — HTTP
**HTTP (Hypertext Transfer Protocol)** es el protocolo base de la web, usado para solicitar y transferir páginas, recursos y datos entre navegador y servidor. El tráfico viaja **sin cifrar**, por lo que hoy en día se usa mayormente para redirigir automáticamente hacia HTTPS (puerto 443).

## Puerto 110 — POP3
**POP3 (Post Office Protocol v3)** se usa para la **descarga** de correo electrónico desde el servidor hacia el cliente. Su funcionamiento clásico descarga los mensajes al dispositivo y, por defecto, los elimina del servidor, a diferencia de IMAP (puerto 143), que mantiene el correo sincronizado en el servidor.

## Puerto 443 — HTTPS
**HTTPS** es HTTP funcionando sobre una capa de cifrado **TLS/SSL**. Es el estándar actual para cualquier sitio web que maneje datos sensibles (logins, formularios, pagos, etc.), ya que protege confidencialidad e integridad del tráfico y autentica al servidor mediante certificados digitales.

## Puerto 3306 — MySQL
Puerto por defecto del motor de base de datos **MySQL** (y su fork **MariaDB**), usado para que las aplicaciones (o clientes de administración) se conecten al servidor de base de datos y ejecuten consultas SQL. Por buenas prácticas de seguridad, este puerto normalmente **no debería estar expuesto a internet**, sino accesible únicamente desde la red interna o el propio servidor de aplicación.

## Tabla resumen

| Puerto | Servicio | Función principal | Cifrado por defecto |
|---|---|---|---|
| 20 | FTP (Data) | Transferencia de archivos | No |
| 21 | FTP (Control) | Autenticación y comandos de FTP | No |
| 22 | SSH | Acceso remoto seguro / túneles | Sí |
| 23 | Telnet | Acceso remoto a terminal | No |
| 25 | SMTP | Envío de correo electrónico | No (salvo extensiones como STARTTLS) |
| 53 | DNS | Resolución de nombres de dominio | No |
| 80 | HTTP | Tráfico web sin cifrar | No |
| 110 | POP3 | Descarga de correo electrónico | No (salvo variante POP3S) |
| 443 | HTTPS | Tráfico web cifrado (HTTP + TLS) | Sí |
| 3306 | MySQL | Conexión al motor de base de datos | No (salvo configuración explícita con SSL) |

## Observación general

Vale la pena notar un patrón: varios de estos protocolos "clásicos" (FTP, Telnet, SMTP básico, POP3, DNS) fueron diseñados en una época de internet donde la seguridad no era una prioridad de diseño, por lo que transmiten datos en texto plano por defecto. La tendencia moderna es reemplazarlos o envolverlos en una capa cifrada: SSH en vez de Telnet, SFTP/FTPS en vez de FTP, HTTPS en vez de HTTP, IMAPS/POP3S en vez de sus versiones sin cifrar, y SMTP con STARTTLS. Desde el punto de vista de seguridad, cualquiera de estos puertos "inseguros" que aparezca abierto y expuesto a internet en un análisis de un servidor debería considerarse una superficie de ataque a revisar.
