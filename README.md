# SmartLab - Flujos n8n por cliente

Este repositorio organiza los flujos de n8n por cliente para mantenerlos versionados y fáciles de compartir.

## Estructura propuesta

```text
clientes/
  tetris/
    README.md
    Extractor_de_Facturas_WhatsApp.json
    Extractor_Facturas_Telegram_Mistral_Config.json
    Extractor_Facturas_Telegram_v3_DosSteps.json
```

## Cómo añadir un nuevo cliente

1. Crear una carpeta nueva dentro de `clientes/` con el nombre del cliente.
2. Agregar los JSON de los flujos y un README breve por cliente.
3. Mantener nombres claros y descriptivos para cada workflow.

## Publicar en GitHub

Una vez creado el repositorio vacío en GitHub, estos comandos dejan el contenido listo para subir:

```bash
git remote add origin https://github.com/<tu-usuario>/<tu-repo>.git
git push -u origin main
```

Este formato permite escalar el repositorio a medida que vayan llegando nuevos clientes y nuevos flujos.
