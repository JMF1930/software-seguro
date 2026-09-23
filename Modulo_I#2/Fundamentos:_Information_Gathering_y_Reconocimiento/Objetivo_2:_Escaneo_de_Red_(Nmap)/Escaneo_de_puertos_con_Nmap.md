# Escaneo de puertos con la aplicación Nmap

## 1. Objetivo

Realizar un reconocimiento de puertos TCP abiertos sobre un host remoto utilizando **[Nmap](https://nmap.org/)** (*Network Mapper*), la herramienta de referencia para escaneo de redes y auditoría de seguridad. El objetivo es determinar qué puertos están accesibles públicamente, qué servicios corren detrás de ellos y qué versión de software exponen, tanto por línea de comandos como mediante su interfaz gráfica (Zenmap).

Host de prueba principal: **`bugcrowd.com`**, complementado con un segundo objetivo (`icnet-srl.com.ar`) para contrastar resultados.

## 2. Escaneo de descubrimiento de puertos (`-p-` + `--open`)

### 2.1. Comando

```bash
nmap -p- --open --min-rate 5000 -n bugcrowd.com
```

### 2.2. Explicación de los parámetros

| Flag | Función |
|---|---|
| `-p-` | Escanea el rango completo de puertos TCP (1–65535). |
| `--open` | Muestra únicamente los puertos que están abiertos (u *open\|filtered*), omitiendo los cerrados/filtrados del listado principal. |
| `--min-rate <number>` | Fuerza a Nmap a enviar paquetes a una tasa mínima (en este caso, 5000 paquetes/segundo), acelerando el escaneo. |
| `-n` | Deshabilita la resolución DNS inversa, reduciendo el tiempo de escaneo al no consultar registros PTR. |

### 2.3. Resultado

```text
$ nmap -p- --open --min-rate 5000 -n bugcrowd.com
Nmap scan report for bugcrowd.com (151.101.66.132)
Host is up (0.29s latency).
Other addresses for bugcrowd.com (not scanned): 151.101.194.132 151.101.2.132 151.101.130.132
Not shown: 65533 filtered tcp ports (no-response)
PORT    STATE SERVICE
80/tcp  open  http
443/tcp open  https

Nmap done: 1 IP address (1 host up) scanned in 28.34 seconds
```

Sobre los 65.535 puertos TCP analizados, únicamente **80/tcp** (`http`) y **443/tcp** (`https`) se encuentran abiertos, lo cual es consistente con una superficie expuesta mínima y bien administrada (solo el tráfico web habilitado hacia Internet).

![Escaneo completo de puertos sobre bugcrowd.com con --open](./assets_nmap/01-bugcrowd-p-open.png)

## 3. Detección de servicios y versiones (`-sV`)

Agregando el flag **`-sV`** al comando anterior, Nmap intenta identificar el servicio y la versión de software que corre detrás de cada puerto abierto, mediante el envío de sondas (*service/version detection probes*) y el análisis de las respuestas (*fingerprinting*).

### 3.1. Comando

```bash
nmap -p- -sV --open --min-rate 5000 -n bugcrowd.com
```

| Flag | Función |
|---|---|
| `-sV` | Realiza sondeo de los puertos abiertos para determinar el servicio y su versión. |

### 3.2. Resultado

```text
$ nmap -p- -sV --open --min-rate 5000 -n bugcrowd.com
PORT    STATE SERVICE     VERSION
80/tcp  open  http-proxy  Varnish
443/tcp open  ssl/https
```

Se identifica que el puerto 80 está detrás de un proxy de caché **Varnish**, y que el puerto 443 sirve tráfico TLS/HTTPS. Nmap no logró huellear con certeza el servicio del puerto 443 ("*1 service unrecognized despite returning data*"), por lo que ofrece la huella cruda (*service fingerprint*, bloque `SF-Port443...`) para que, opcionalmente, se envíe a la base de datos pública de Nmap.

![Detección de servicios y versión sobre bugcrowd.com con -sV](./assets_nmap/02-bugcrowd-sV.png)

### 3.3. Segundo ejemplo — `icnet-srl.com.ar`

Se repite el mismo escaneo sobre un segundo dominio para contrastar resultados:

```bash
nmap -p- -sV --open --min-rate 5000 -n icnet-srl.com.ar
```

```text
$ nmap -p- -sV --open --min-rate 5000 -n icnet-srl.com.ar
Nmap scan report for icnet-srl.com.ar (216.198.79.1)
PORT   STATE SERVICE VERSION
80/tcp  open  http     Vercel
443/tcp open  ssl/https Vercel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 131.47 seconds
```

En este caso, la huella de servicio permite identificar con precisión que el sitio está alojado sobre la plataforma **Vercel** (se observan encabezados HTTP como redirecciones 308 permanentes y el header `server: Vercel`), tanto en el puerto 80 como en el 443.

![Detección de servicios y versión sobre icnet-srl.com.ar con -sV](./assets_nmap/03-icnet-sV.png)

## 4. Referencia rápida de parámetros utilizados

| Flag | Descripción |
|---|---|
| `-n` / `-R` | Nunca resolver DNS inverso / siempre resolverlo (por defecto: a veces). |
| `-p <rango>` | Escanea únicamente los puertos indicados. Ejemplos: `-p22`, `-p1-65535`, `-p U:53,111,137,T:21-25,80,139,8080,S:9`. |
| `--min-rate <número>` | Envía paquetes a una tasa no menor a `<número>` por segundo. |
| `-sV` | Sondea los puertos abiertos para determinar servicio y versión. |
| `--open` | Muestra solo los puertos abiertos (o posiblemente abiertos). |

## 5. Escaneo mediante interfaz gráfica — Zenmap

Como alternativa a la línea de comandos, se utilizó **[Zenmap](https://nmap.org/zenmap/)**, la interfaz gráfica oficial de Nmap, aplicando el perfil predefinido **"Quick scan"**, que internamente ejecuta:

```bash
nmap -T4 -F <objetivo>
```

| Flag | Descripción |
|---|---|
| `-T4` | Plantilla de temporización agresiva (*aggressive timing*), prioriza velocidad de ejecución. |
| `-F` | Escaneo rápido (*fast mode*): reduce el número de puertos escaneados a los 100 más comunes según `nmap-services`. |

### 5.1. `bugcrowd.com`

```text
nmap -T4 -F bugcrowd.com
Nmap scan report for bugcrowd.com (151.101.194.132)
Not shown: 98 filtered tcp ports (no-response)
PORT    STATE SERVICE
80/tcp  open  http
443/tcp open  https

Nmap done: 1 IP address (1 host up) scanned in 2.54 seconds
```

![Escaneo rápido (Quick scan) de bugcrowd.com desde Zenmap](./assets_nmap/04-zenmap-bugcrowd.png)

### 5.2. `icnet-srl.com.ar`

```text
nmap -T4 -F icnet-srl.com.ar
Nmap scan report for icnet-srl.com.ar (216.198.79.1)
Not shown: 98 filtered tcp ports (no-response)
PORT    STATE SERVICE
80/tcp  open  http
443/tcp open  https

Nmap done: 1 IP address (1 host up) scanned in 3.32 seconds
```

![Escaneo rápido (Quick scan) de icnet-srl.com.ar desde Zenmap, con ambos hosts ya relevados en el panel izquierdo](./assets_nmap/05-zenmap-icnet.png)

El escaneo rápido, al limitarse a los 100 puertos más comunes y usar temporización agresiva, entrega resultados equivalentes al escaneo completo en una fracción del tiempo (≈2-3 segundos vs. ≈28-130 segundos), lo cual lo hace apto para reconocimientos preliminares, mientras que el escaneo con `-p-` sigue siendo necesario para garantizar cobertura sobre la totalidad del espacio de puertos.

## 6. Conclusiones

- Ambos dominios analizados (`bugcrowd.com` e `icnet-srl.com.ar`) exponen únicamente los puertos **80/tcp** y **443/tcp**, reflejando una superficie de ataque acotada al tráfico web estándar.
- El flag `-sV` es clave para pasar de un simple listado de puertos a información de *fingerprinting* de la pila tecnológica (Varnish, Vercel), dato relevante tanto para tareas de auditoría/pentesting como para reconocimiento de infraestructura de terceros.
- Zenmap resulta útil como capa de visualización y para perfiles de escaneo predefinidos, aunque para escaneos exhaustivos (todos los puertos, detección de versión) la línea de comandos ofrece mayor control sobre los parámetros y el rendimiento del escaneo.
