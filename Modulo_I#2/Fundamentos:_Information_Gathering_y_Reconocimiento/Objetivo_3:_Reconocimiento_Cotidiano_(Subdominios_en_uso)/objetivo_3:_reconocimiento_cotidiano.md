Objetivo 3 — Reconocimiento Cotidiano: Subdominios en uso
1. Consigna
La arquitectura web basada en subdominios está presente en la gran mayoría de los servicios que se consumen a diario. El ejercicio consiste en identificar, sobre aplicaciones reales de uso cotidiano, al menos cinco (5) servicios que operen bajo un subdominio, documentando en cada caso el dominio raíz, el subdominio específico utilizado y la URL completa correspondiente.
2. Metodología
Para el reconocimiento se utilizó Subfinder (v2.16.0), una herramienta de enumeración pasiva de subdominios desarrollada por ProjectDiscovery. La herramienta consulta múltiples fuentes públicas (motores de búsqueda, certificados TLS, DNS históricos, etc.) sin realizar tráfico directo contra los servidores objetivo, por lo que es apta para tareas de reconocimiento no intrusivo (passive reconnaissance).
Sintaxis utilizada en cada consulta:
```bash
subfinder -d <dominio_raiz>
```
Sobre cada dominio raíz analizado se seleccionó, del listado de subdominios devuelto, aquel que se utiliza de forma habitual en el día a día.
3. Resultados
#	Aplicación / Servicio	Dominio raíz	Subdominio en uso	URL completa	Subdominios totales hallados
1	ICNet SRL (sitio corporativo)	`icnet-srl.com.ar`	`www`	`https://www.icnet-srl.com.ar`	1
2	Plataforma de capacitación — Universidad Blas Pascal	`ubp.edu.ar`	`miubp-ext`	`https://miubp-ext.ubp.edu.ar`	123
3	Webmail corporativo — Claro Argentina	`claro.com.ar`	`cp.cloud`	`https://cp.cloud.claro.com.ar`	328
4	Portal de proveedores — IPLAN	`iplan.com.ar`	`www.iplanproveedores`	`https://www.iplanproveedores.iplan.com.ar`	81
5	Plataforma de capacitación — EPEC	`epec.com.ar`	`aula`	`https://aula.epec.com.ar`	21
> En cada URL, la porción resaltada en **negrita** dentro de la columna "Subdominio en uso" es la que antecede al dominio raíz (`<subdominio>.<dominio-raiz>`).
4. Detalle por servicio
4.1. ICNet SRL — `www.icnet-srl.com.ar`
Búsqueda con un único resultado. Se utiliza a diario como canal de acceso a la información institucional y de contacto con potenciales clientes.
```text
$ subfinder -d icnet-srl.com.ar
www.icnet-srl.com.ar
[INF] Found 1 subdomains for icnet-srl.com.ar in 7 seconds 440 milliseconds
```
![Enumeración de subdominios de icnet-srl.com.ar](./assets/01-icnet-srl.png)
---
4.2. Universidad Blas Pascal — `miubp-ext.ubp.edu.ar`
Institución educativa con una superficie de subdominios considerablemente mayor (123 en total). El subdominio `miubp-ext` corresponde a la plataforma externa de capacitación utilizada habitualmente.
```text
$ subfinder -d ubp.edu.ar
...
miubp-ext.ubp.edu.ar
...
[INF] Found 123 subdomains for ubp.edu.ar in 10 seconds 850 milliseconds
```
![Enumeración de subdominios de ubp.edu.ar — parte 1](./assets/02-ubp-edu-1.png)
![Enumeración de subdominios de ubp.edu.ar — parte 2](./assets/03-ubp-edu-2.png)
![Enumeración de subdominios de ubp.edu.ar — parte 3 (resultado final: 123 subdominios)](./assets/04-ubp-edu-3.png)
---
4.3. Claro Argentina — `cp.cloud.claro.com.ar`
El dominio con mayor cantidad de subdominios detectados (328). El subdominio `cp.cloud` se utiliza para la gestión de las casillas de correo alojadas en el servidor de Claro.
```text
$ subfinder -d claro.com.ar
...
cp.cloud.claro.com.ar
...
[INF] Found 328 subdomains for claro.com.ar in 55 seconds 137 milliseconds
```
![Enumeración de subdominios de claro.com.ar — parte 1](./assets/05-claro-1.png)
![Enumeración de subdominios de claro.com.ar — parte 2 (resultado final: 328 subdominios)](./assets/06-claro-2.png)
---
4.4. IPLAN — `www.iplanproveedores.iplan.com.ar`
Se hallaron 81 subdominios. El subdominio `www.iplanproveedores` se utiliza para acceder a la plataforma de certificación de trabajos frente al cliente.
```text
$ subfinder -d iplan.com.ar
...
www.iplanproveedores.iplan.com.ar
...
[INF] Found 81 subdomains for iplan.com.ar in 2 seconds 269 milliseconds
```
![Enumeración de subdominios de iplan.com.ar — parte 1](./assets/07-iplan-1.png)
![Enumeración de subdominios de iplan.com.ar — parte 2 (resultado final: 81 subdominios)](./assets/08-iplan-2.png)
---
4.5. EPEC — `aula.epec.com.ar`
Búsqueda con 21 subdominios detectados. El subdominio `aula` se utiliza para acceder a la plataforma de capacitación de la distribuidora eléctrica.
```text
$ subfinder -d epec.com.ar
...
aula.epec.com.ar
...
[INF] Found 21 subdomains for epec.com.ar in 7 seconds 479 milliseconds
```
![Enumeración de subdominios de epec.com.ar (resultado final: 21 subdominios)](./assets/09-epec.png)
5. Conclusiones
Los cinco servicios relevados confirman que el esquema `<subdominio>.<dominio-raiz>` es el estándar de facto para segmentar funcionalidades (correo, e-learning, portales de proveedores, sitios corporativos) dentro de un mismo dominio raíz.
La cantidad de subdominios expuestos varía fuertemente según el tamaño y la superficie digital de la organización: desde 1 (ICNet SRL) hasta 328 (Claro Argentina).
Un volumen alto de subdominios (como el de `claro.com.ar` o `ubp.edu.ar`) incrementa la superficie de ataque de la organización, por lo que este tipo de reconocimiento pasivo es también una técnica habitual en auditorías de seguridad y pruebas de penetración (OSINT / reconnaissance).
6. Formato de entrega
Enlace al presente archivo `.md`, con el listado de aplicaciones, subdominios en uso y URL completa detallado en la sección 3. Resultados.
