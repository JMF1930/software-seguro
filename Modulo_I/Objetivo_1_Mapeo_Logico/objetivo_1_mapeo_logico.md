
Diagrama de flujo: Cliente → Servidor → Base de Datos → Servidor Remoto
Diagrama de flujo que muestra paso a paso cómo viaja un dato desde que el usuario interactúa con la interfaz web (Frontend), pasa por el procesamiento del servidor (Backend) y termina interactuando con la Base de Datos, incluyendo la comunicación con un servidor remoto.
> Convertido desde `Diagrama_de_flujo.drawio` a Mermaid, compatible con la vista de Markdown de GitHub (no requiere plugins externos).
1. Arquitectura general (contenedores Docker)
```mermaid
flowchart LR
    Cliente["🖥️ Cliente"]

    subgraph Servidor["Servidor"]
        direction TB
        subgraph DockerA["Contenedor de Docker"]
            direction TB
            FrontA["Front"]
            BackA["Back"]
            BDA["BD"]
        end
    end

    subgraph Remoto["Servidor remoto"]
        direction TB
        subgraph DockerB["Contenedor de Docker"]
            direction TB
            FrontB["Front"]
            BackB["Back"]
            BDB["BD"]
        end
    end

    Cliente -- "Consulta HTTPS Request 1 / 2 / 3" --> FrontA
    FrontA -- "Consulta HTTPS Response 1 / 2 / 4" --> Cliente

    FrontA <--> BackA
    BackA <--> BDA

    BackA -- "Solicitud" --> DockerB
    DockerB -- "Respuesta" --> FrontA
```
2. Secuencia de la petición (paso a paso)
```mermaid
sequenceDiagram
    actor Cliente
    participant Front as Front (Servidor)
    participant Back as Back (Servidor)
    participant BD as BD (Servidor)
    participant Remoto as Servidor remoto

    Cliente->>Front: Request 1: Start-challenge
    Front-->>Cliente: Response 1: Index

    Cliente->>Front: Request 2: ChallengeId 1
    Front-->>Cliente: Response 2

    Cliente->>Front: Request 3
    Front-->>Cliente: Response 4

    Front->>Back: Reenvía la solicitud
    Back->>BD: Consulta / actualización de datos
    BD-->>Back: Resultado de la consulta

    Back->>Remoto: Solicitud 
    Remoto-->>Back: Respuesta del servidor remoto

    Back-->>Front: Datos procesados
    Front-->>Cliente: Respuesta final renderizada
```
3. Referencia de mensajes (leyenda)
Paso	Mensaje	Descripción
1	Request 1	Start-challenge — el cliente inicia el reto/solicitud contra el Front.
1	Response 1	Index — el servidor responde con la página/índice inicial.
2	Request 2	ChallengeId: 1 — el cliente envía el identificador del reto.
2	Response 2	Respuesta asociada al Request 2.
3	Request 3 / Response 4	Intercambio adicional entre Cliente y Front.
4	Solicitud HTTPS o Consulta SQL	Comunicación interna Back ↔ BD y Back ↔ Servidor remoto.
---
Notas de la conversión
Los colores originales del `.drawio` (azul, naranja, verde) se agruparon aquí por tipo de flujo: azul/naranja = tráfico Cliente ↔ Front, verde = tráfico Back ↔ BD / Servidor remoto.
Ambos diagramas usan sintaxis Mermaid, que GitHub renderiza de forma nativa dentro de archivos `.md` (sin necesidad de imágenes exportadas ni plugins).
Si necesitas editar la disposición visual (posiciones exactas, colores exactos), lo ideal es mantener el `.drawio` como fuente y este `.md` como versión "documentada" para el repositorio.
