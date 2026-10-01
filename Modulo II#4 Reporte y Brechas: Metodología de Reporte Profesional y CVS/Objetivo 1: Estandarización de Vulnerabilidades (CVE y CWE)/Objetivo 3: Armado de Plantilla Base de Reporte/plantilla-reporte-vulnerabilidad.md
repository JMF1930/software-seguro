# 🛡️ Reporte de Vulnerabilidad

> **Plantilla profesional de reporte de seguridad**
> Diseñada para comunicar un hallazgo de forma clara tanto a nivel directivo como técnico.

---

## 📋 Metadatos del Reporte

| Campo | Valor |
|---|---|
| **ID del Ticket** | `VULN-YYYY-NNN` |
| **Título del Hallazgo** | _Nombre corto y descriptivo de la vulnerabilidad_ |
| **Severidad** | 🔴 Crítica / 🟠 Alta / 🟡 Media / 🔵 Baja / ⚪ Informativa |
| **Estado** | Abierto / En remediación / Re-test / Cerrado |
| **Activo afectado** | _Sistema, aplicación, URL, host o componente_ |
| **Entorno** | Producción / Staging / Desarrollo |
| **Autor (analista)** | _Nombre / handle_ |
| **Fecha de detección** | `YYYY-MM-DD` |
| **Fecha del reporte** | `YYYY-MM-DD` |
| **Clasificación** | 🔒 Confidencial |

---

## 1. 📈 Resumen Ejecutivo
> **Audiencia: directivos y responsables de negocio. Sin jerga técnica.**

**¿Qué encontramos?**
_En 2-3 oraciones, en lenguaje llano: qué falla existe y en qué sistema._

**¿Por qué importa? (impacto al negocio)**
_Traducir el riesgo técnico a consecuencias de negocio: pérdida de datos de clientes,
interrupción del servicio, multas regulatorias, daño reputacional, fraude financiero._

**¿Qué tan grave es?**
_Severidad en una frase + puntaje CVSS. Ej.: "Severidad Crítica (CVSS 9.8).
Un atacante externo sin credenciales podría acceder a la base de datos de clientes."_

**¿Qué hay que hacer?**
_La acción principal recomendada, resumida. Ej.: "Aplicar el parche del proveedor y
rotar credenciales. Esfuerzo estimado: bajo. Plazo recomendado: inmediato."_

| Riesgo | Probabilidad | Impacto | Urgencia de acción |
|---|---|---|---|
| _Breve_ | Alta/Media/Baja | Alto/Medio/Bajo | Inmediata / 30 días / Planificada |

---

## 2. 🔧 Detalles Técnicos
> **Audiencia: equipo técnico, desarrolladores, administradores de sistemas.**

### 2.1 Descripción de la vulnerabilidad
_Explicación técnica detallada de la falla: qué la causa, cómo funciona,
qué componente/versión/configuración está afectada._

### 2.2 Clasificación
| Campo | Valor |
|---|---|
| **Tipo (CWE)** | `CWE-NNN` — _Nombre_ |
| **Categoría OWASP** | _Ej.: A03:2021 – Injection_ |
| **CVE (si aplica)** | `CVE-YYYY-NNNNN` |

### 2.3 Activos y alcance afectado
- **Host / URL / Endpoint:** `ejemplo.com/api/v1/...`
- **Versión del software:** `_componente v.X.Y.Z_`
- **Parámetro / función vulnerable:** `_parámetro, campo o método_`
- **Alcance:** _¿Un sistema aislado o múltiples? ¿Datos sensibles involucrados?_

### 2.4 Causa raíz
_El origen real del problema: falta de validación de entrada, dependencia
desactualizada, configuración por defecto insegura, error de lógica, etc._

---

## 3. 📊 Métricas CVSS
> **Puntaje objetivo y reproducible del riesgo técnico (CVSS v3.1).**

| Métrica | Puntaje | Severidad |
|---|---|---|
| **CVSS Base Score** | `0.0` | 🔴/🟠/🟡/🔵 |

**Vector CVSS:**
```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H
```

### Desglose del vector (Base)
| Métrica | Valor elegido | Justificación |
|---|---|---|
| **Attack Vector (AV)** | Network / Adjacent / Local / Physical | _¿Desde dónde se explota?_ |
| **Attack Complexity (AC)** | Low / High | _¿Requiere condiciones especiales?_ |
| **Privileges Required (PR)** | None / Low / High | _¿Qué nivel de acceso necesita el atacante?_ |
| **User Interaction (UI)** | None / Required | _¿Hace falta que una víctima haga algo?_ |
| **Scope (S)** | Unchanged / Changed | _¿El impacto sale del componente vulnerable?_ |
| **Confidentiality (C)** | None / Low / High | _Impacto en confidencialidad_ |
| **Integrity (I)** | None / Low / High | _Impacto en integridad_ |
| **Availability (A)** | None / Low / High | _Impacto en disponibilidad_ |

> 💡 Calculadora oficial: https://www.first.org/cvss/calculator/3.1

---

## 4. 🧪 Prueba de Concepto (PoC)
> **Reproducción paso a paso. Debe permitir a cualquier técnico replicar el hallazgo.**

### 4.1 Requisitos previos
- _Herramientas necesarias (Burp Suite, curl, navegador, etc.)_
- _Credenciales o nivel de acceso de partida (si aplica)_
- _Estado inicial del sistema_

### 4.2 Pasos de reproducción

**Paso 1 — _Descripción de la acción_**
```bash
# Comando o petición exacta
curl -X POST "https://ejemplo.com/endpoint" -d "parametro=valor"
```
_Qué se observa tras este paso._

**Paso 2 — _Descripción de la acción_**
```http
POST /login HTTP/1.1
Host: ejemplo.com
...
```
_Resultado observado._

**Paso 3 — _Explotación / confirmación_**
_Acción final que confirma la vulnerabilidad._

### 4.3 Evidencia
> _Capturas, logs o salidas que demuestran el éxito. Reemplazar por evidencia real._

```
[ Salida, respuesta del servidor, o datos extraídos que prueban el impacto ]
```

![Evidencia](ruta/a/captura.png)

### 4.4 Resultado esperado vs. obtenido
| Esperado (sistema seguro) | Obtenido (sistema vulnerable) |
|---|---|
| _Acceso denegado / error controlado_ | _Acceso concedido / dato expuesto_ |

> ⚠️ **Nota ética:** la PoC se ejecuta únicamente sobre sistemas autorizados,
> con el mínimo impacto necesario para demostrar el riesgo. No se extraen datos
> reales de clientes ni se degrada el servicio.

---

## 5. ✅ Recomendaciones de Remediación
> **Audiencia: equipo técnico que implementará la corrección.**

### 5.1 Solución recomendada (prioritaria)
_La corrección definitiva. Ej.: "Actualizar la librería a la versión X.Y.Z" o
"Implementar consultas parametrizadas en el endpoint afectado."_

```diff
- query = "SELECT * FROM users WHERE id = " + user_input   # vulnerable
+ query = "SELECT * FROM users WHERE id = ?"               # parametrizado
+ cursor.execute(query, (user_input,))
```

### 5.2 Mitigaciones temporales (si no se puede corregir ya)
- _Regla de WAF, bloqueo de IP, deshabilitar función, restringir acceso de red._

### 5.3 Buenas prácticas de fondo (prevención)
- _Validación de entrada centralizada, principio de mínimo privilegio,
  gestión de dependencias, revisiones de código, hardening._

### 5.4 Plan de acción
| Acción | Responsable | Prioridad | Plazo | Estado |
|---|---|---|---|---|
| _Aplicar parche_ | _Equipo_ | Alta | _YYYY-MM-DD_ | Pendiente |
| _Verificar (re-test)_ | _Analista_ | Alta | _YYYY-MM-DD_ | Pendiente |

### 5.5 Verificación (re-test)
_Cómo se validará que la corrección funciona: repetir la PoC y confirmar que
el sistema ahora responde de forma segura._

---

## 📚 Referencias
- _CVE / Advisory del proveedor_
- _CWE: https://cwe.mitre.org/_
- _OWASP: https://owasp.org/_
- _Documentación técnica relacionada_

---

## 📝 Historial de cambios
| Versión | Fecha | Autor | Cambio |
|---|---|---|---|
| 1.0 | `YYYY-MM-DD` | _Analista_ | Creación del reporte |

---
<div align="center">
<sub>🔒 Documento confidencial — uso interno / cliente autorizado únicamente.</sub>
</div>
