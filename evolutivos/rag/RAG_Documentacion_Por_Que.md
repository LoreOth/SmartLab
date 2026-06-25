# RAG Vault — Por qué cada paso

## Flujo 1: Actualización

**Obsidian Git (plugin)**
El equipo edita en Obsidian como siempre. El plugin sube los cambios a GitHub automáticamente cada 10 minutos. Sin esto, los cambios quedarían solo en la computadora local y el RAG nunca se enteraría.

**GitHub como repositorio central**
n8n vive en la nube y no puede leer archivos de computadoras locales. GitHub es el puente: guarda la vault en un lugar accesible desde cualquier servicio externo.

**Webhook de GitHub → n8n**
En lugar de que n8n revise GitHub cada X minutos (polling), GitHub avisa a n8n en el instante en que alguien sube un cambio. Más eficiente y la actualización del RAG es inmediata.

**Borrar chunks viejos antes de re-indexar**
Cada archivo puede generar varios chunks. Si alguien edita una nota y no borramos los chunks anteriores, Supabase termina con la versión vieja Y la nueva al mismo tiempo. Cuando alguien consulta, el RAG devuelve contenido contradictorio. El borrado garantiza que solo existe la versión actual.

**Chunkar el texto**
Los modelos de embeddings tienen un límite de tokens por entrada. Además, recuperar una nota entera como contexto es impreciso: si la nota tiene 5 temas, el LLM recibe información irrelevante. Los chunks son fragmentos pequeños y temáticamente cohesivos que permiten recuperar exactamente la parte relevante.

**Generar embeddings (OpenAI)**
Un embedding convierte texto en un vector numérico que representa su significado semántico. Esto permite buscar por concepto ("¿qué es la alucinación?") y encontrar resultados aunque usen palabras distintas ("hallucination in LLMs"). Sin embeddings, solo habría búsqueda por palabras exactas.

**Guardar en Supabase con metadatos (file_id, file_path, category)**
El vector solo sirve para encontrar el chunk. Los metadatos sirven para dos cosas: el `file_id` permite borrar selectivamente al actualizar, y el `file_path` permite citar la fuente en la respuesta para que el usuario sepa de dónde viene la información.

---

## Flujo 2: Consulta

**Telegram Trigger**
Recibe el mensaje del usuario y lo pasa al workflow. Telegram es gratuito, sin límites ni aprobaciones burocráticas, y los 4 socios pueden usarlo desde el celular sin instalar nada nuevo.

**Embeber la pregunta (mismo modelo)**
Para comparar la pregunta con los chunks almacenados, ambos deben estar en el mismo "idioma vectorial". Si la indexación usó `text-embedding-3-small`, la consulta también debe usarlo. Modelos distintos producen vectores incomparables.

**Búsqueda por similitud en Supabase**
Compara el vector de la pregunta contra todos los chunks y devuelve los 5 más similares semánticamente. El umbral de 0.7 filtra resultados poco relevantes. Esto es lo que hace que el sistema "entienda" la pregunta en lugar de buscar palabras textuales.

**Construir el prompt con contexto**
El LLM no tiene acceso directo a Supabase. Se le pasa el contexto explícitamente: los 5 chunks recuperados más la pregunta del usuario. El LLM solo puede responder con lo que está en ese contexto — esto previene que invente información (alucinaciones).

**Claude — Generar respuesta**
Toma el prompt con el contexto y genera una respuesta en lenguaje natural, citando las fuentes. Usamos `claude-3-5-haiku` porque es rápido (~1 segundo), suficientemente inteligente para Q&A sobre documentos, y muy barato.

**Responder en Telegram**
Envía la respuesta de vuelta al chat del usuario que preguntó, incluyendo las fuentes (file_path de los chunks usados) para que el equipo pueda ir a la nota original en Obsidian si quiere profundizar.
