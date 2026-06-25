# RAG Vault + Telegram — Setup completo

## Lo que vas a tener al final

- La vault de Obsidian vive en GitHub (sincronización automática desde Obsidian)
- Cuando alguien edita una nota, el RAG se actualiza solo en segundos
- Los 4 socios consultan al bot de Telegram en lenguaje natural
- Costo total: ~$0-2/mes

---

## Qué necesitás crear/tener

- Cuenta GitHub
- Cuenta Supabase (supabase.com)
- Cuenta n8n Cloud (n8n.io) — free tier
- Cuenta Anthropic (console.anthropic.com) para Claude
- Cuenta OpenAI (platform.openai.com) para embeddings
- Telegram instalado en el celular

---

## PARTE 1 — Vault en GitHub

### Paso 1.1 — Crear el repo en GitHub

1. Ve a github.com → New repository
2. Nombre: `rag-vault`
3. Visibility: **Private**
4. Click en "Create repository"
5. Copiá la URL del repo (ej: `https://github.com/tu-usuario/rag-vault`)

### Paso 1.2 — Instalar Obsidian Git

1. En Obsidian: **Settings → Community Plugins → Browse**
2. Buscar "Obsidian Git" → Instalar → Activar
3. Ir a las opciones del plugin (ícono de engranaje al lado)
4. Configurar:
   - **Auto pull interval**: 10 (minutos)
   - **Auto commit-and-sync interval**: 10 (minutos)
   - **Commit message**: `vault: auto-sync {{date}}`
5. Comandos para conectar el repo (hacerlo una sola vez):

En la terminal de tu computadora, desde la carpeta de la vault:
```bash
git init
git remote add origin https://github.com/tu-usuario/rag-vault.git
git add .
git commit -m "Initial vault commit"
git push -u origin main
```

Desde ahora el plugin sincroniza automáticamente cada 10 minutos. Los otros 3 socios clonan el repo y repiten el Paso 1.2 en sus Obsidian.

### Paso 1.3 — Configurar el webhook de GitHub

Esto hace que GitHub avise a n8n cada vez que alguien sube cambios.

1. En el repo de GitHub: **Settings → Webhooks → Add webhook**
2. Payload URL: (la obtenés del Workflow 1 de n8n — ver Parte 3)
3. Content type: `application/json`
4. Events: "Just the push event"
5. Click en "Add webhook"

---

## PARTE 2 — Supabase (vector store)

### Paso 2.1 — Crear proyecto

1. Ir a supabase.com → New Project
2. Nombre: `rag-vault`
3. Guardá la **Project URL** y la **anon key** (en Project Settings → API)

### Paso 2.2 — Ejecutar el SQL

En Supabase → **SQL Editor** → copiar y ejecutar esto completo:

```sql
-- Habilitar pgvector
CREATE EXTENSION IF NOT EXISTS vector;

-- Tabla de chunks
CREATE TABLE vault_chunks (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  content       TEXT NOT NULL,
  embedding     VECTOR(1536),
  file_id       TEXT NOT NULL,
  file_path     TEXT NOT NULL,
  layer         TEXT,
  category      TEXT,
  last_modified TIMESTAMPTZ,
  created_at    TIMESTAMPTZ DEFAULT NOW()
);

-- Índice vectorial
CREATE INDEX ON vault_chunks
  USING ivfflat (embedding vector_cosine_ops)
  WITH (lists = 100);

-- Tabla de control
CREATE TABLE vault_meta (
  key   TEXT PRIMARY KEY,
  value TEXT
);
INSERT INTO vault_meta VALUES ('last_indexed_at', '2020-01-01T00:00:00Z');

-- Función de búsqueda semántica
CREATE OR REPLACE FUNCTION match_vault_chunks(
  query_embedding VECTOR(1536),
  match_count     INT DEFAULT 5,
  match_threshold FLOAT DEFAULT 0.7
)
RETURNS TABLE (
  id         UUID,
  content    TEXT,
  file_path  TEXT,
  category   TEXT,
  similarity FLOAT
)
LANGUAGE SQL STABLE AS $$
  SELECT id, content, file_path, category,
    1 - (embedding <=> query_embedding) AS similarity
  FROM vault_chunks
  WHERE 1 - (embedding <=> query_embedding) > match_threshold
  ORDER BY embedding <=> query_embedding
  LIMIT match_count;
$$;
```

---

## PARTE 3 — Bot de Telegram

### Paso 3.1 — Crear el bot con BotFather

1. Abrí Telegram y buscá **@BotFather**
2. Enviá: `/newbot`
3. BotFather te va a pedir:
   - **Nombre del bot**: `Vault RAG` (o el que quieran)
   - **Username**: `vault_rag_bot` (debe terminar en `bot`)
4. BotFather te da un **token** — guardalo, lo necesitás en n8n:
   ```
   123456789:ABCdefGhIJKlmNoPQRsTUVwxyZ
   ```

### Paso 3.2 — Compartir el bot con el equipo

Simplemente compartís el link `t.me/vault_rag_bot` con los 4 socios. Cada uno lo abre y hace click en "Start". Listo, ya pueden escribirle.

---

## PARTE 4 — n8n: los dos workflows

### Configurar credentials primero

En n8n → **Settings → Credentials → Add credential**:

| Credential | Tipo en n8n | Datos |
|-----------|-------------|-------|
| GitHub | GitHub API | Personal Access Token (scope: repo) |
| Supabase | HTTP Header Auth | URL + anon key del proyecto |
| OpenAI | OpenAI API | API key de platform.openai.com |
| Anthropic | HTTP Header Auth | API key de console.anthropic.com |
| Telegram | Telegram API | El token de BotFather |

---

### Workflow 1 — Indexación automática

Crear nuevo workflow en n8n con estos nodos:

**Nodo 1 — Webhook (trigger)**
```
Tipo: Webhook
Method: POST
Path: /vault-indexar
Authentication: None
```
Copiá la URL que te da n8n y ponéla en el webhook de GitHub (Paso 1.3).

---

**Nodo 2 — HTTP Request: leer last_indexed_at**
```
Tipo: HTTP Request
Method: GET
URL: https://[tu-proyecto].supabase.co/rest/v1/vault_meta?key=eq.last_indexed_at&select=value
Headers:
  apikey: [tu-anon-key]
  Authorization: Bearer [tu-anon-key]
```

---

**Nodo 3 — Code: extraer archivos cambiados**
```javascript
// Extrae los archivos .md modificados del payload del webhook de GitHub
const commits = $input.first().json.commits || [];
const files = new Set();

for (const commit of commits) {
  [...(commit.added || []), ...(commit.modified || [])].forEach(f => {
    if (f.endsWith('.md') && (f.startsWith('02_Wiki/') || f.startsWith('01_Sources/'))) {
      files.add(f);
    }
  });
}

return [...files].map(path => ({ json: { path } }));
```

---

**Nodo 4 — HTTP Request: borrar chunks viejos**
```
Tipo: HTTP Request
Method: DELETE
URL: https://[tu-proyecto].supabase.co/rest/v1/vault_chunks
Headers:
  apikey: [tu-anon-key]
  Authorization: Bearer [tu-anon-key]
Query params:
  file_id: eq.{{ $json.file_id }}
```

---

**Nodo 5 — HTTP Request: obtener contenido del archivo**
```
Tipo: HTTP Request
Method: GET
URL: https://raw.githubusercontent.com/tu-usuario/rag-vault/main/{{ $json.path }}
(sin autenticación si el repo es público)
```

---

**Nodo 6 — Code: chunkar y preparar metadatos**
```javascript
const path = $input.first().json.path || '';
const content = $input.first().json.data || '';

const parts = path.split('/');
const layer = parts[0] === '02_Wiki' ? 'wiki' : 'sources';
const category = parts[1] || 'general';
const fileId = path.replace(/\.md$/, '').replace(/\//g, '-').toLowerCase().replace(/[^a-z0-9-]/g, '');

const chunkSize = 1800;
const overlap = 200;
const chunks = [];

for (let i = 0; i < content.length; i += chunkSize - overlap) {
  const chunk = content.slice(i, i + chunkSize).trim();
  if (chunk.length < 80) continue;
  chunks.push({ json: { content: chunk, file_id: fileId, file_path: path, layer, category } });
}

if (chunks.length === 0 && content.trim().length > 0) {
  chunks.push({ json: { content: content.trim(), file_id: fileId, file_path: path, layer, category } });
}

return chunks;
```

---

**Nodo 7 — HTTP Request: generar embeddings**
```
Tipo: HTTP Request
Method: POST
URL: https://api.openai.com/v1/embeddings
Headers:
  Authorization: Bearer [tu-openai-key]
  Content-Type: application/json
Body (JSON):
{
  "model": "text-embedding-3-small",
  "input": "{{ $json.content }}"
}
```

---

**Nodo 8 — HTTP Request: insertar en Supabase**
```
Tipo: HTTP Request
Method: POST
URL: https://[tu-proyecto].supabase.co/rest/v1/vault_chunks
Headers:
  apikey: [tu-anon-key]
  Authorization: Bearer [tu-anon-key]
  Content-Type: application/json
  Prefer: return=minimal
Body (JSON):
{
  "content": "{{ $json.content }}",
  "embedding": {{ $json.data[0].embedding }},
  "file_id": "{{ $('Code').item.json.file_id }}",
  "file_path": "{{ $('Code').item.json.file_path }}",
  "layer": "{{ $('Code').item.json.layer }}",
  "category": "{{ $('Code').item.json.category }}"
}
```

---

**Nodo 9 — HTTP Request: actualizar timestamp**
```
Tipo: HTTP Request
Method: PATCH
URL: https://[tu-proyecto].supabase.co/rest/v1/vault_meta?key=eq.last_indexed_at
Headers:
  apikey: [tu-anon-key]
  Authorization: Bearer [tu-anon-key]
  Content-Type: application/json
Body (JSON):
{ "value": "{{ new Date().toISOString() }}" }
```

Activar el workflow. Desde ahora cada push a GitHub lo dispara automáticamente.

---

### Workflow 2 — Consulta por Telegram

**Nodo 1 — Telegram Trigger**
```
Tipo: Telegram Trigger
Credential: tu Telegram credential (token del bot)
Updates: message
```

---

**Nodo 2 — HTTP Request: embedding de la pregunta**
```
Tipo: HTTP Request
Method: POST
URL: https://api.openai.com/v1/embeddings
Headers:
  Authorization: Bearer [tu-openai-key]
  Content-Type: application/json
Body (JSON):
{
  "model": "text-embedding-3-small",
  "input": "{{ $json.message.text }}"
}
```

---

**Nodo 3 — HTTP Request: similarity search en Supabase**
```
Tipo: HTTP Request
Method: POST
URL: https://[tu-proyecto].supabase.co/rest/v1/rpc/match_vault_chunks
Headers:
  apikey: [tu-anon-key]
  Authorization: Bearer [tu-anon-key]
  Content-Type: application/json
Body (JSON):
{
  "query_embedding": {{ $json.data[0].embedding }},
  "match_count": 5,
  "match_threshold": 0.7
}
```

---

**Nodo 4 — Code: construir prompt**
```javascript
const question = $('Telegram Trigger').item.json.message.text;
const chatId = $('Telegram Trigger').item.json.message.chat.id;
const chunks = $input.first().json;

let contextText = '';
let sources = [];

if (Array.isArray(chunks) && chunks.length > 0) {
  contextText = chunks.map((c, i) =>
    `[${i+1}] ${c.content}\nFuente: ${c.file_path}`
  ).join('\n\n---\n\n');
  sources = [...new Set(chunks.map(c => c.file_path))];
} else {
  contextText = 'No se encontró información relevante en la vault.';
}

const systemPrompt = `Eres el asistente de investigación del equipo. Respondes preguntas sobre RAG y los temas documentados en la vault.
Usa ÚNICAMENTE la información del contexto provisto para responder.
Si el contexto no tiene la respuesta, dilo claramente: "No encontré información sobre esto en la vault."
Al final de tu respuesta, lista las fuentes usadas con el prefijo 📄`;

const userPrompt = `CONTEXTO:
${contextText}

PREGUNTA: ${question}`;

return [{ json: { systemPrompt, userPrompt, chatId, question, sources } }];
```

---

**Nodo 5 — HTTP Request: Claude**
```
Tipo: HTTP Request
Method: POST
URL: https://api.anthropic.com/v1/messages
Headers:
  x-api-key: [tu-anthropic-key]
  anthropic-version: 2023-06-01
  Content-Type: application/json
Body (JSON):
{
  "model": "claude-3-5-haiku-20241022",
  "max_tokens": 1024,
  "system": "{{ $json.systemPrompt }}",
  "messages": [{ "role": "user", "content": "{{ $json.userPrompt }}" }]
}
```

---

**Nodo 6 — Telegram: responder**
```
Tipo: Telegram
Credential: tu Telegram credential
Operation: Send Message
Chat ID: {{ $('Code').item.json.chatId }}
Text: {{ $json.content[0].text }}
Parse Mode: Markdown
```

Activar el workflow. Los 4 socios ya pueden usar el bot.

---

## Cómo probar todo

### Probar la indexación
1. Editá cualquier nota en Obsidian
2. Esperá ~10 min (o hacé commit manual con Cmd+P → "Obsidian Git: Commit all")
3. En Supabase → Table Editor → vault_chunks: debería aparecer el archivo nuevo
4. En n8n → Executions: ver que el Workflow 1 corrió sin errores

### Probar el bot de Telegram
1. Escribile al bot: `¿Qué es HyDE?` (si tenés ese contenido en la vault)
2. Debería responder en 2-4 segundos con información de la vault
3. Si responde "no encontré información" y sí existe: bajar el `match_threshold` de 0.7 a 0.5 en el Nodo 3 del Workflow 2

### Verificar en Supabase
```sql
-- Ver cuántos chunks están indexados
SELECT category, COUNT(*) as chunks
FROM vault_chunks
GROUP BY category ORDER BY chunks DESC;
```

---

## Costos reales para 4 usuarios

| Servicio | Costo |
|----------|-------|
| GitHub | Gratis |
| Supabase | Gratis |
| n8n Cloud (free tier) | Gratis |
| Telegram Bot API | Gratis (sin límites) |
| OpenAI embeddings | < $0.10/mes |
| Claude Haiku (respuestas) | ~$1-2/mes |
| **Total** | **~$1-2/mes** |
