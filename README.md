# Asistente de RRHH con AI Agent y RAG: Pastelería Rica

Sistema de automatización en n8n que responde consultas de empleados sobre
políticas de RRHH (vacaciones, licencias y reintegros) usando un AI Agent
con RAG (Retrieval-Augmented Generation).

## Contenido del repositorio

- `workflow-agente-rag.json`: workflow principal exportado desde n8n.
- `workflow-carga-documentos.json`: workflow que carga las políticas en el Vector Store.
- `capturas`: capturas sueltas del canvas y de las ejecuciones exitosas.

---

## 1. Caso de negocio

**Proceso que se automatiza:** hoy los empleados de Pastelería Rica consultan
a RRHH por mail sobre vacaciones, licencias y reintegros, y una persona
responde cada consulta a mano, buscando la respuesta en los documentos de
políticas.

**Por qué es buen candidato:** es repetitivo, las respuestas están en
documentos escritos y solo una parte de los casos requiere criterio humano.

## 2. ROI estimado

| Concepto | Valor estimado |
|---|---|
| Consultas por mes | 200 |
| Tiempo por consulta (manual) | 10 min |
| Total manual | ~33 h/mes |
| Costo por hora | $15 |
| **Costo actual** | **~$500/mes** |
| Consultas resueltas por el agente | 70% |
| **Ahorro mensual** | **~$350** |
| Costo de implementación (único) | 8 h x $15 = $120 |
| Costo mensual (API + n8n) | ~$30 |
| **Recupero de la inversión** | **~1 mes** |

**Límites del ROI:** deja de convenir con menos de ~20 consultas al mes (el
costo de mantenerlo supera el ahorro) o si los documentos cambian seguido y
nadie actualiza la base de conocimiento (el agente respondería con
información desactualizada).

## 3. Por qué un AI Agent con RAG y no un nodo de IA simple

- Un **nodo de IA simple** recibe un texto y devuelve otro texto: no decide
  nada ni consulta fuentes.
- Este sistema necesita un **agente** porque debe elegir entre tres acciones
  (`responder`, `derivar`, `fuera_de_alcance`) y decidir si la información
  recuperada alcanza para responder.
- Necesita **RAG** porque las políticas son propias de la empresa: el modelo
  no las conoce y sin ellas inventaría respuestas. Con RAG responde solo con
  documentos reales y actualizables, sin reentrenar nada. Una consulta
  estática (búsqueda por palabra) no entiende preguntas formuladas de
  distintas maneras.

## 4. Arquitectura del workflow

1. **Webhook** (entrada 1): recibe `nombre`, `email` y `pregunta`.
2. *(pendiente)* **Segunda entrada**: [Google Sheets / Gmail Trigger].
3. **AI Agent** con Anthropic Chat Model, Structured Output Parser y
   Supabase Vector Store como herramienta de consulta.
4. **Switch**: según `accion`, enruta a:
   - `responder` → Gmail (respuesta al empleado)
   - `derivar` → Google Sheets (registro para RRHH)
   - `fuera_de_alcance` → Gmail (aviso al empleado)

## 5. Elección del modelo (local vs. nube)

**Modelo del agente:** Claude (Anthropic) en la nube.

- **Costo:** con ~200 consultas/mes el gasto por API es bajo y no justifica
  mantener infraestructura propia.
- **Privacidad:** las consultas son sobre políticas generales; no se envían
  datos sensibles. Si se manejaran datos personales delicados, convendría
  un modelo local con Ollama.
- **Independencia de proveedores:** es la desventaja de la nube; se mitiga
  porque n8n permite cambiar de modelo sin rehacer el flujo.

## 6. Vector Store y Embeddings

- **Vector Store:** Supabase (PostgreSQL con extensión `pgvector`), por su
  plan gratuito y su integración nativa con n8n.
- **Embeddings:** Google Gemini. Genera vectores de **3072 dimensiones**.
- **Aprendizaje técnico:** la dimensión de la columna `embedding` de la tabla
  debe coincidir con la del modelo de embeddings; si no, la inserción falla.
- Los documentos se dividen con un Recursive Character Text Splitter
  (chunk 500, overlap 50).

## 7. Manejo de errores

*(pendiente: se completa al implementarlo)*

1. **[Punto 1]**: [qué puede fallar y cómo se
