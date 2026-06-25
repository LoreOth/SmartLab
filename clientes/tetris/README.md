# Cliente Tetris

Esta carpeta reúne los flujos de n8n del cliente Tetris relacionados con la extracción de facturas desde WhatsApp y Telegram.

## Archivos incluidos

- `Extractor_de_Facturas_WhatsApp.json`: flujo para recibir mensajes por WhatsApp, validar sesión y completar el procesamiento de la factura.
- `Extractor_Facturas_Telegram_Mistral_Config.json`: flujo para Telegram con integración de Mistral y configuración configurable.
- `Extractor_Facturas_Telegram_v3_DosSteps.json`: flujo para Telegram con un proceso en dos pasos basado en obra y tipo.

## Diagramas conceptuales

### Flujo general de WhatsApp

```mermaid
flowchart TD
  A[Recibir mensaje] --> B[Preparar entrada]
  B --> C[Consultar estado del usuario]
  C --> D{¿Sesión activa?}
  D -->|No| E[Limpiar estado expirado]
  D -->|Sí| F[Router de estado]
  E --> G[Mensaje de sesión expirada]
  F --> H{¿Tiene foto?}
  H -->|Sí| I[Procesar factura]
  H -->|No| J[Solicitar documento]
  I --> K[Responder al usuario]
  J --> K
```

### Flujo general de Telegram

```mermaid
flowchart TD
  A[Recibir mensaje de Telegram] --> B[Preparar entrada]
  B --> C[Consultar estado del usuario]
  C --> D{¿Sesión activa?}
  D -->|No| E[Limpiar estado]
  D -->|Sí| F[Router de estado]
  F --> G{¿Tiene foto?}
  G -->|No| H[Solicitar imagen]
  G -->|Sí| I[Clasificar obra y tipo]
  I --> J[Procesar factura]
  H --> J
  J --> K[Responder al usuario]
```

Estos diagramas son una vista conceptual de la lógica de cada flujo y pueden ajustarse a medida que se agreguen nuevas versiones.
