# AI Automation Avanzado — Proyecto Integrador

Repositorio del proyecto integrador del curso **AI Automation Avanzado**. Contiene el flujo de n8n que se va ampliando módulo a módulo a lo largo de la cursada, partiendo siempre de la versión del checkpoint anterior.

## Checkpoint 1 — Agente Base y Motor de Razonamiento

**Archivo:** `checkpoint1_carla_baudino.json`

### Propósito operativo

Este flujo implementa la primera versión del **Asistente de Calificación de Leads**, un agente conversacional pensado para una empresa de transporte y logística. Su función es recibir consultas comerciales entrantes por chat, identificar si corresponden a un lead real, pedir los datos de contacto faltantes cuando el mensaje es genérico, calificar comercialmente al lead (Alto / Medio / Bajo) y registrar esa información de forma autónoma, sin intervención humana en el camino feliz.

### Arquitectura del flujo

- **Chat Trigger**: captura el mensaje inicial desestructurado del usuario y sostiene la conversación de varios turnos dentro de una misma sesión.
- **AI Agent (Tools Agent)**: nodo central de razonamiento, en modo *Tools Agent*, conectado a un modelo de lenguaje vía **Groq Chat Model** (`openai/gpt-oss-120b`). Incluye:
  - **System Prompt modular** (Rol → Ámbito → Objetivo → Reglas y Escalamiento) que define el rol operativo del agente, qué datos puede manejar y qué acciones tiene explícitamente prohibidas (no cotizar precios, no inventar datos, no salirse de su ámbito comercial).
  - **Guardrail de iteraciones**: `maxIterations` fijado en 7 (dentro del rango de 5 a 10 exigido), para blindar el flujo contra bucles lógicos infinitos.
  - **Memoria de conversación** (`Simple Memory`, buffer de ventana), para que el agente recuerde los datos que el usuario ya aportó en turnos anteriores del mismo chat.
- **Insert row in Data table** (herramienta lateral, no secuencial): conectada como extensión del agente vía el puerto *Tool*. Registra cada lead calificado en una tabla de datos nativa de n8n (Nombre, Empresa, Email, Estado, Calificación). Tiene una descripción de negocio extensa que le indica al modelo en qué casos exactos debe activarla de forma autónoma.
- **Log Observabilidad (Gmail SMTP)**: nodo final de notificación que envía por mail un reporte de auditoría de cada ejecución, incluyendo tanto los pasos intermedios de razonamiento del agente (qué herramienta invocó, con qué datos, y qué observó) como la respuesta final — actuando como reporte automático de supervisión humana.
- **Edit Fields**: normaliza la salida final para que la interfaz de chat muestre la respuesta conversacional del agente y no el detalle técnico del envío del mail.

### Cómo probarlo

1. Importar el `.json` en una instancia de n8n.
2. Configurar las credenciales propias: Groq (modelo de lenguaje), SMTP de Gmail (con contraseña de aplicación) y la tabla de datos de destino.
3. Abrir el chat de test y enviar un mensaje genérico (ej. "hola, quiero info sobre sus servicios"): el agente debe responder pidiendo los datos faltantes.
4. Responder con nombre, empresa, email y una necesidad concreta: el agente debe calificar el lead, registrarlo en la tabla y enviar el mail de observabilidad con el detalle del razonamiento.

### Roadmap del proyecto integrador

Este flujo es la base que se va a ir ampliando en los próximos módulos del curso:

- **M2** — Multi-agente (Manager + Workers como sub-workflows)
- **M3** — Memoria y contexto persistente (Airtable por Session_ID)
- **M4** — Integraciones reales (CRM / Calendario / Workspace vía OAuth2)
- **M5** — RAG / base documental (Vector store)
- **M6** — Voz (STT / TTS)
- ... hasta el **Proyecto Final Integrador (M11)**
