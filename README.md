# Asistente de RRHH con AI Agent y RAG: Pastelería Rica

Sistema de automatización en n8n que responde consultas de empleados sobre
políticas de RRHH (vacaciones, licencias y reintegros) usando un AI Agent
con RAG (Retrieval-Augmented Generation).

## Contenido del repositorio

- `Proyecto final (2).json`: workflow principal exportado desde n8n.
- `Carga de documentos RAG.json`: workflow que carga las políticas en el Vector Store.
- `Proyecto final version ultima.JPG`: captura del canvas del workflow.
- `CapturaRAG.JPG`: captura del RAG / ejecución.
- `email recibido.JPG`: mail recibido por el empleado.

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

**Por qué escala mejor que el proceso manual:** el costo manual crece de forma
lineal con las consultas (más consultas = más horas de RRHH), mientras que el
costo del sistema casi no cambia: cada consulta adicional cuesta centavos de API.

**Límites del ROI:** deja de convenir con menos de ~20 consultas al mes (el
costo de mantenerlo supera el ahorro) o si los documentos cambian seguido y
nadie actualiza la base de conocimiento (el agente respondería con
información desactualizada).

## 3. Por qué un AI Agent con RAG y no un nodo de IA simple

- Un **nodo de IA simple** recibe un texto y devuelve otro texto: no decide
  nada ni consulta fuentes.
- Este sistema necesita un **agente** porque debe elegir entre tres acciones
  (`responder`, `derivar`, `fuera_de_alcance`), decidir cuándo consultar la
  herramienta de políticas y evaluar si la información recuperada alcanza
  para responder.
- Necesita **RAG** porque las políticas son propias de la empresa: el modelo
  no las conoce y sin ellas inventaría respuestas. Con RAG responde solo con
  documentos reales y actualizables, sin reentrenar nada. Una consulta
  estática (búsqueda por palabra) no entiende preguntas formuladas de
  distintas maneras.

## 4. Arquitectura del workflow

1. **Webhook** (entrada 1): recibe `nombre`, `email` y `pregunta`.
2. **Google Sheets, hoja `Empleados`** (entrada 2): busca por email los datos
   del empleado (antigüedad, área) para personalizar la respuesta.
3. **IF**: valida que el empleado exista antes de llamar al agente.
4. **AI Agent** con Anthropic Chat Model, Structured Output Parser y
   Supabase Vector Store como herramienta de consulta.
5. **Switch**: según `accion`, enruta a:
   - `responder` → Gmail (respuesta al empleado)
   - `derivar` → Google Sheets, hoja `Consultas derivadas` (registro para RRHH)
   - `fuera_de_alcance` → Gmail (aviso al empleado)
   - *Fallback* → Google Sheets (derivación de acciones no reconocidas)

## 5. Elección del modelo (local vs. nube)

**Modelo del agente:** Claude (Anthropic) en la nube.

- **Costo:** con ~200 consultas/mes el gasto por API es bajo y no justifica
  mantener infraestructura propia (servidor con GPU para Ollama).
- **Privacidad:** las consultas son sobre políticas generales y no se envían
  datos sensibles (solo nombre, antigüedad y área). Si se manejaran datos
  personales delicados, convendría un modelo local con Ollama.
- **Independencia de proveedores:** es la desventaja de la nube; se mitiga
  porque n8n permite cambiar de modelo sin rehacer el flujo.

## 6. Vector Store y Embeddings

- **Vector Store:** Supabase (PostgreSQL con extensión `pgvector`), por su
  plan gratuito y su integración nativa con n8n.
- **Embeddings:** Google Gemini, que genera vectores de **3072 dimensiones**.
  Se usa un proveedor distinto al del agente porque Anthropic no ofrece
  modelo de embeddings.
- **Aprendizaje técnico:** la dimensión de la columna `embedding` de la tabla
  debe coincidir con la del modelo de embeddings; si no, la inserción falla
  (error "expected 768 dimensions, not 3072").
- Los documentos se dividen con un Recursive Character Text Splitter
  (chunk 500, overlap 50) y se cargan con un workflow aparte.

## 7. Manejo de errores

1. **Empleado no encontrado (dato ambiguo o inexistente):** el nodo Google
   Sheets lee la hoja `Empleados` con "Always Output Data". Un nodo **IF**
   verifica que el email exista. Si no existe, el flujo no llega al agente y
   registra el caso en la hoja `Consultas derivadas` con el motivo
   "Empleado no encontrado en la base".

2. **Falla del agente (Supabase, API del modelo o respuesta inválida):**
   el AI Agent usa *On Error: Continue (using error output)*, por lo que un
   fallo del Vector Store o del modelo no corta el flujo y el caso queda
   registrado para RRHH. Además, el **Switch** tiene una salida *Fallback*
   que captura cualquier acción distinta de las tres permitidas y la deriva
   a una persona.

3. **Límites en el system prompt:** ante ambigüedad o falta de información
   en las políticas, el agente tiene instrucción explícita de derivar en
   vez de inventar.

## 8. System Prompt completo

```
Eres el asistente de RRHH de Pastelería Rica. Respondes consultas de
empleados con tono cordial, claro y profesional, en español.

DATOS QUE RECIBES:
- Datos del empleado (nombre, antigüedad, área), provenientes de la hoja Empleados.
- La consulta del empleado.
- Una herramienta que busca en las políticas de RRHH (vacaciones, licencias y reintegros).

ALCANCE - PUEDES:
- Responder sobre vacaciones, licencias y reintegros usando SOLO la
  información devuelta por la herramienta de políticas.
- Personalizar la respuesta con la antigüedad del empleado.

NO PUEDES:
- Inventar datos, plazos o montos que no estén en las políticas.
- Aprobar o rechazar solicitudes.
- Opinar sobre temas legales, salariales o conflictos personales.
- Responder temas ajenos a RRHH.

DECISIÓN (elige exactamente una acción):
- "responder": la herramienta devolvió información suficiente para contestar.
- "derivar": el caso es personal o sensible, o las políticas no contienen la
  información necesaria.
- "fuera_de_alcance": la consulta no es un tema de RRHH.

REGLAS ANTE LA DUDA:
- Si hay ambigüedad o información incompleta, elige "derivar". Nunca adivines.
- Consulta siempre la herramienta de políticas antes de responder.
- Si la herramienta no devuelve resultados, elige "derivar".

FORMATO DE SALIDA (JSON):
- accion: "responder", "derivar" o "fuera_de_alcance"
- respuesta: texto breve y claro para el empleado
- motivo: una frase explicando por qué elegiste esa acción
```

## 9. Capturas

**Canvas del workflow:**

![Canvas del workflow](capturas/canvas.png)

**Ejecución de prueba exitosa:**

![Ejecución exitosa](capturas/ejecucion.png)

**Mail recibido por el empleado:**

![Mail recibido](capturas/mail.png)
