# Asistente de RRHH con AI Agent y RAG: Pastelería Rica

## 1. Caso de negocio

**Proceso que se automatiza:** hoy, los empleados de Pastelería Rica
consultan a RRHH por mail sobre vacaciones, licencias y reintegros, y una
persona responde cada consulta a mano, buscando la respuesta en los
documentos de políticas.

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

**Límites del ROI:** deja de convenir si hay menos de ~20 consultas al mes
(el costo de mantenerlo supera el ahorro), o si los documentos cambian
seguido y nadie los actualiza en la base de conocimiento (el agente
respondería con información vieja).

## 3. Por qué un AI Agent con RAG y no un nodo de IA simple

- **Un nodo de IA simple** recibe un texto y devuelve otro texto. No decide
  nada ni consulta fuentes.
- **Este sistema necesita un agente** porque debe *elegir* entre tres
  acciones (responder, derivar, fuera de alcance) según el caso, y
  decidir si la información recuperada alcanza para responder.
- **Necesita RAG** porque las políticas son propias de la empresa: el
  modelo no las conoce y, sin ellas, inventaría respuestas. Con RAG
  responde solo con documentos reales y actualizables, sin reentrenar nada.
  Una consulta estática (buscar una palabra) no entiende preguntas
  formuladas de distintas maneras.

## 4. Elección del modelo (local vs. nube)

*(se completa mañana)*

## 5. Vector Store y Embeddings

*(se completa mañana)*

## 6. Manejo de errores

*(se completa mañana)*

## 7. System Prompt completo

*(se pega la versión final mañana)*

## 8. Capturas

*(se agregan mañana en /capturas)*
