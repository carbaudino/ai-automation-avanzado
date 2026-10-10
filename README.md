# AI Automation Avanzado — Proyecto Integrador

**Alumna:** Carla Baudino

Este es el repositorio del proyecto integrador del curso **AI Automation Avanzado**: un agente de atención comercial y operativa para una empresa de transporte y logística (**Logística Demo S.A.**, ficticia), construido en **n8n**. Cada checkpoint parte del anterior y le suma una capacidad nueva, hasta llegar al Proyecto Final (M11).

## Índice de checkpoints

| # | Módulo | Qué suma al agente | Carpeta |
|---|---|---|---|
| 1 | Agente base y motor de razonamiento | AI Agent (Tools Agent) que califica leads y los registra en una Data Table, con guardrail de iteraciones y log de observabilidad | [`checkpoint1/`](checkpoint1/) |
| 2 | Orquestación multi-agente | Patrón Manager-Worker: router de intención, 2 Workers como sub-workflows, contratos JSON, contingencia y trazabilidad | [`checkpoint2/`](checkpoint2/) |
| 3 | Memoria persistente | Memoria de largo plazo en Airtable por `Session_ID`, inyección de contexto con delimitadores y summarization a partir de 5 mensajes | [`checkpoint3/`](checkpoint3/) |
| 4 | Integraciones reales | Canal de email: Gmail (OAuth2), HubSpot y Slack, con IF anti auto-reply, Look up antes del Create y borradores con aprobación humana | [`checkpoint4/`](checkpoint4/) |
| 5 | RAG / conocimiento organizacional | Manual de políticas parseado con LlamaParse, base de vectores con Top-K y Minimum Score calibrados, citación de fuentes y regla "No sé" | [`checkpoint5/`](checkpoint5/) |
| 6 | Voice AI (STT / TTS) | Canal de voz en Telegram: Whisper transcribe, el agente responde con RAG en 200 caracteres como máximo y ElevenLabs devuelve un audio; contingencia ante audios inválidos y purga del binario | [`checkpoint6/`](checkpoint6/) |
| 7 | Diseño de sistema agéntico vertical | Documento de diseño para Customer Service & Sales Ops en logística: orquestador + 4 especialistas, mapa de permisos, semáforo de riesgo HITL y scorecards | [`checkpoint7/`](checkpoint7/) |
| 8 | QA automatizado con AI-as-a-Judge | Juez LLM con JSON Schema sobre el canal de voz, Switch Aceptar / Corregir (Self-Healing) / Escalar (Slack Aprobar · Editar), auditoría en Airtable y test A/B de 10 corridas | [`checkpoint8/`](checkpoint8/) |
| 9 → 11 | … Proyecto Final | Próximamente | — |

**Convención del repo:** cada checkpoint vive en su propia carpeta, con una única copia de sus `.json` de n8n, su README y sus evidencias.

## Evolución de la arquitectura

```
M1  Chat ─▶ AI Agent (calificación de leads) ─▶ Data Table + log Gmail
M2  Chat ─▶ Router de intención ─▶ Worker Leads / Worker Reclamos / escalamiento humano ─▶ log
M3  + memoria Airtable (lectura antes del router, escritura y resumen después de responder)
M4  Email ─▶ IF anti auto-reply ─▶ AI Agent ─▶ HubSpot (Look up → Update/Create) ─▶ Borrador Gmail ─▶ Slack
M5  + herramienta buscar_manual_politicas (RAG) conectada al AI Agent del canal email
M6  Telegram (voz) ─▶ Whisper (STT) ─▶ IF contingencia ─▶ AI Agent + RAG (≤ 200) ─▶ ElevenLabs (TTS) ─▶ audio al chat
M7  (diseño) Orquestador ─▶ Leads · Consultas RAG · Reclamos · Auditor de churn ─▶ Semáforo de salida (verde / amarillo / rojo)
M8  AI Agent ─▶ Juez (JSON Schema) ─▶ Airtable Auditoria_QA + Switch: ACEPTADO ─▶ voz · CORREGIR ─▶ reintento · RECHAZADO ─▶ Slack
```

## Stack

| Rol | Herramienta |
|---|---|
| Orquestación | n8n (self-hosted) |
| LLM | Groq · `openai/gpt-oss-120b` (agentes) y `openai/gpt-oss-20b` (resumidor) |
| Memoria de largo plazo | Airtable (base *Memoria Agente*, tabla *Sesiones*) |
| CRM | HubSpot (Service Key con scopes de contactos) |
| Correo | Gmail (OAuth2) · SMTP para logs |
| Mensajería del equipo | Slack (bot con `chat:write`) |
| Parseo documental | LlamaParse (LlamaCloud) |
| Embeddings / vectores | Google Gemini `gemini-embedding-001` · Simple Vector Store de n8n |
| Voz | Whisper `whisper-large-v3` (Groq) para STT · ElevenLabs `eleven_multilingual_v2` para TTS |
| Canal de chat | Telegram (bot) expuesto con ngrok |

## Cómo importar cualquier checkpoint

1. En n8n, ir a **Workflows → Import from File** y elegir el `.json` de la carpeta del checkpoint.
2. Asignar las credenciales propias en cada nodo marcado en rojo. Los `.json` no incluyen secretos: solo referencian credenciales por nombre.
3. Cuando un checkpoint tiene sub-workflows, importarlos primero, elegirlos en el nodo que los llama y **publicarlos**.

---

## Checkpoint 1 — Agente Base y Motor de Razonamiento

**Archivo:** [`checkpoint1/checkpoint1_carla_baudino.json`](checkpoint1/checkpoint1_carla_baudino.json)

### Propósito operativo

Este flujo implementa la primera versión del **Asistente de Calificación de Leads**, un agente conversacional para una empresa de transporte y logística. Recibe consultas comerciales por chat e identifica si corresponden a un lead real. Si el mensaje es genérico, pide los datos de contacto que faltan. Después califica comercialmente al lead (Alto / Medio / Bajo) y registra la información de forma autónoma, sin intervención humana en el camino feliz.

### Arquitectura del flujo

- **Chat Trigger**: captura el mensaje inicial desestructurado y sostiene la conversación de varios turnos dentro de una misma sesión.
- **AI Agent (Tools Agent)**: nodo central de razonamiento, conectado al modelo vía **Groq Chat Model** (`openai/gpt-oss-120b`). Incluye:
  - **System Prompt modular** (Rol → Ámbito → Objetivo → Reglas y Escalamiento), que define el rol operativo, los datos que puede manejar y las acciones prohibidas: no cotizar precios, no inventar datos, no salirse de su ámbito comercial.
  - **Guardrail de iteraciones**: `maxIterations` en 7, dentro del rango de 5 a 10 exigido, para evitar bucles infinitos.
  - **Memoria de conversación** (`Simple Memory`), para recordar los datos aportados en turnos anteriores del mismo chat.
- **Insert row in Data table** (herramienta del agente): registra cada lead calificado (Nombre, Empresa, Email, Estado, Calificación). Su descripción de negocio le indica al modelo en qué casos exactos activarla.
- **Log Observabilidad (Gmail SMTP)**: envía por mail un reporte de cada ejecución, con los pasos intermedios del razonamiento y la respuesta final.
- **Edit Fields**: normaliza la salida para que el chat muestre la respuesta del agente y no el detalle técnico del envío del mail.

### Cómo probarlo

1. Importar el `.json` y configurar las credenciales: Groq, SMTP de Gmail (con contraseña de aplicación) y la tabla de datos de destino.
2. Enviar por el chat un mensaje genérico (por ejemplo, "hola, quiero info sobre sus servicios"): el agente debe pedir los datos que faltan.
3. Responder con nombre, empresa, email y una necesidad concreta: el agente debe calificar el lead, registrarlo y enviar el mail de observabilidad.

## Checkpoint 2 — Orquestación Multi-Agente

Un **Manager** clasifica la intención (`LEAD_COMERCIAL`, `RECLAMO_OPERATIVO` u `other`) y delega con **Execute Workflow** ("Wait For Sub-Workflow Completion" activado) en dos **Workers** independientes. Cada Worker devuelve siempre el mismo contrato `{status, worker, request_id, data | error}`, también ante fallas. Los casos dudosos se escalan a un supervisor humano y cada delegación queda registrada en un log de trazabilidad.
➡️ [Ver carpeta checkpoint2](checkpoint2/)

## Checkpoint 3 — Memoria Persistente y Summarization

Lectura de Airtable por `Session_ID` antes del router, con un IF para usuarios nuevos. El contexto se inyecta entre `[INICIO DE CONTEXTO COMPARTIDO]` y `[FIN DEL CONTEXTO COMPARTIDO]`. A partir del 6.º mensaje, un modelo económico genera un resumen JSON que sobreescribe la fila de la sesión de forma idempotente.
➡️ [Ver carpeta checkpoint3](checkpoint3/)

## Checkpoint 4 — Integraciones (Gmail + HubSpot + Slack)

Canal de email con IF anti auto-reply, Look up en HubSpot antes de crear contactos, borradores con aprobación humana y aviso en Slack con payload mínimo.
➡️ [Ver carpeta checkpoint4](checkpoint4/)

## Checkpoint 5 — RAG: Agente con Conocimiento Organizacional

El agente de email consulta el manual de políticas de la empresa, parseado con LlamaParse y fragmentado por sección, a través de la herramienta `buscar_manual_politicas` (Top-K 3, Minimum Score 0,68). Responde solo con los fragmentos recuperados, cita la fuente y dice "No sé" cuando el dato no está. Prueba ciega: 4/5 aciertos documentales y 1 falla de contención corregida en el prompt.
➡️ [Ver carpeta checkpoint5](checkpoint5/)

## Checkpoint 6 — Voice AI: Canal de Voz

El cliente le habla al bot de Telegram con notas de voz. Whisper transcribe en español, un IF descarta audios vacíos o corruptos y pide repetir sin gastar IA, y el agente responde con el manual del Checkpoint 5 en 200 caracteres como máximo. ElevenLabs convierte la respuesta en audio (Multilingual v2, stability 0,65, clarity 0,8). El audio del cliente y el generado se purgan dentro del flujo, y las ejecuciones de producción no se guardan.
➡️ [Ver carpeta checkpoint6](checkpoint6/)

## Checkpoint 7 — Diseño de un Sistema Agéntico Vertical

Informe de consultoría que especializa el agente para Customer Service & Sales Ops en transporte de cargas. Incluye un baseline de ≈ 271 h/mes y un framework de priorización con 3 vectores. Propone una red con un orquestador y 4 especialistas encapsulados, con permisos de mínimo privilegio y Context Engineering por sub-workflow. Define 8 KPIs, un semáforo de riesgo con escalado en Slack y scorecards de leads, reclamos y churn.
➡️ [Ver carpeta checkpoint7](checkpoint7/)

## Checkpoint 8 — QA Automatizado con AI-as-a-Judge

Un Juez LLM (Groq `gpt-oss-120b` con Structured Output Parser) evalúa cada respuesta del canal de voz con una rúbrica de exactitud factual de 1 a 5. Un Switch la acepta, la devuelve al agente con la crítica (máximo 2 reintentos) o la congela y la escala a Slack con los botones Aprobar / Editar. Cada veredicto queda en Airtable. En el test A/B, la Arquitectura A (120b) logró 4,4 de precisión media contra 4,0 de la B (20b), y se recomienda A.
➡️ [Ver carpeta checkpoint8](checkpoint8/)
