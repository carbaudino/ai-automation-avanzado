# Checkpoint 6 · Canal de Voz (Telegram + Whisper + ElevenLabs)

**Alumna:** Carla Baudino · **Curso:** AI Automation Avanzado · **Archivo:** [`checkpoint6_carla_baudino.json`](checkpoint6_carla_baudino.json) · **Entrega:** [`PreEntrega_Modulo6_CarlaBaudino.pdf`](PreEntrega_Modulo6_CarlaBaudino.pdf)

Al proyecto integrador (agente de atención de Logística Demo S.A.) se le suma un **canal de voz**. El cliente le manda una nota de voz al bot de Telegram. Whisper la transcribe, el AI Agent responde consultando el manual de políticas del Checkpoint 5 (RAG) y ElevenLabs convierte la respuesta en un audio que vuelve al mismo chat.

## Flujo

```
Telegram Trigger
   │
Descargar nota de voz (Get a file) ── error ──┐
   │                                          │
Renombrar a .ogg                              │
   │                                          │
Oídos del agente - Whisper (STT) ── error ────┤
   │                                          │
¿Transcripción válida? ── false ──────────────┴──▶ Contingencia - pedir repetir (texto)
   │ true
Purgar audio del cliente          (descarta el binario: desde acá solo viaja texto)
   │
Cerebro - AI Agent (voz)          (Groq + memoria por chat + herramienta buscar_manual_politicas)
   │
Texto para voz (≤ 200)            (limpia markdown y recorta a 200 caracteres)
   │
Parámetros de voz (marca)
   │
Voz del agente - ElevenLabs (TTS) ── error ──▶ Fallback texto (TTS caído)
   │
Enviar nota de voz (Telegram)     (Send Audio con el binario data)
   │
Purgar audio generado
```

## Cómo se cumple cada requisito

| Requisito | Nodo |
|---|---|
| Trigger de mensajería que recibe voz | `Telegram Trigger` + `Descargar nota de voz` (el trigger solo trae el `file_id`; este nodo baja el audio al binario `data`) |
| Whisper después del trigger, binario `data`, idioma `es` | `Oídos del agente - Whisper (STT)`: `whisper-large-v3` en Groq, `file` = binario `data`, `language` = `es`, `temperature` = 0 |
| Texto de Whisper al AI Agent (Tools Agent) | `Cerebro - AI Agent (voz)` recibe `texto_cliente` |
| Respuestas de máximo 200 caracteres | Sección CONTENCIÓN FINANCIERA del System Prompt + `Texto para voz (≤ 200)`, que lo garantiza aunque el modelo se pase |
| ElevenLabs Multilingual con Stability y Clarity | `Parámetros de voz (marca)`: `eleven_multilingual_v2`, stability 0,65, similarity_boost 0,8, style 0 |
| Envío del audio por Telegram | `Enviar nota de voz (Telegram)`: Send Audio con el binario `data` de ElevenLabs |
| Contingencia ante audio corrupto o vacío | `¿Transcripción válida?`: texto de 4 caracteres o más **y** duración de 1 s o más; si no, `Contingencia - pedir repetir` |
| Destrucción del audio binario | `Purgar audio del cliente` y `Purgar audio generado` (solo pasan los campos de texto) + Settings sin guardar ejecuciones de producción |

### Desvíos respecto de los nodos sugeridos

- **Whisper por Groq vía HTTP Request** en lugar del nodo de OpenAI: es el mismo modelo (`whisper-large-v3`) y el proyecto ya usa Groq como proveedor.
- **ElevenLabs vía HTTP Request** en lugar del nodo de comunidad, para no depender de un paquete externo en n8n self-hosted.
- **`Renombrar a .ogg`**: Telegram guarda las notas de voz como `.oga` y la API de Groq rechaza esa extensión, aunque el contenido es el mismo (Ogg/Opus).
- **Send Audio** en lugar de Send Voice: ElevenLabs devuelve MP3 y Send Voice exige Ogg/Opus.

## Seguridad y compliance

- **Audio en memoria:** n8n corre con `N8N_DEFAULT_BINARY_DATA_MODE=default`, así que el audio no se escribe en disco.
- **Purga temprana:** apenas Whisper devuelve el texto, el audio del cliente sale del flujo. El MP3 generado se descarta después de enviarlo.
- **Sin historial:** *Save failed / successful production executions* = **Do not save**.
- **Mínimo privilegio en ElevenLabs:** la API key está restringida a *De texto a voz*, sin acceso a los demás endpoints, y tiene un tope de 2.000 créditos.
- **Costo acotado:** con 200 caracteres como máximo, cada respuesta consume como máximo 200 créditos de ElevenLabs. Las contingencias responden en texto y no gastan IA ni TTS.
- El `.json` no incluye secretos: las credenciales se referencian por nombre.

## Pruebas (08/10/2026, Telegram en producción)

| Prueba | Respuesta del agente | Resultado |
|---|---|---|
| Voz: "¿A cuántos grados llevan la fruta?" | "Transportamos la fruta en rango refrigerado, entre 2 y 8 grados Celsius." (audio) | ✅ Dato del manual, sección 5 |
| Voz: "¿Cuánto tarda un envío a Mendoza?" | "El envío a Mendoza tarda entre 72 y 96 horas hábiles." (audio) | ✅ Dato del manual, sección 4 (Cuyo) |
| Voz: "¿Cuánto cuesta un viaje a Córdoba?" | "No sé. Le puedo derivar a un asesor para que le informe el precio." | ✅ El manual no tiene precios |
| Audio breve sin consulta clara | "Disculpe, ¿podría repetir su consulta en una sola frase, por favor?" (audio) | ✅ Pide repetir |
| Mensaje de texto "Hola" | "No pude entender bien el audio. ¿Me lo puede repetir en una nota de voz corta, sin ruido de fondo?" | ✅ Contingencia, sin costo de IA ni de TTS |

### Evidencia

**Ejecución completa en producción**
![Lienzo](evidencias/01_lienzo_ejecucion_exitosa.png)

**Whisper (STT): binario `data`, idioma `es`**
![Whisper](evidencias/02_whisper_stt.png)

**System Prompt con la regla de 200 caracteres**
![System Prompt](evidencias/03_system_prompt_200_caracteres.png)

**Herramienta RAG del Checkpoint 5**
![Herramienta RAG](evidencias/04_herramienta_rag.png)

**ElevenLabs: parámetros de voz, nodo y API key restringida**
![Parámetros de voz](evidencias/05_parametros_de_voz.png)
![ElevenLabs URL y auth](evidencias/06_elevenlabs_url_y_auth.png)
![ElevenLabs body y respuesta](evidencias/07_elevenlabs_body_y_respuesta.png)
![API key restringida](evidencias/08_api_key_restringida.png)

**Envío por Telegram**
![Send Audio](evidencias/09_send_audio.png)

**Contingencia**
![IF contingencia](evidencias/10_if_contingencia.png)
![Agente pide repetir](evidencias/11_agente_pide_repetir.png)

**Settings: ejecuciones de producción sin guardar**
![Settings](evidencias/12_settings_do_not_save.png)

**Chat de Telegram y prueba de Mendoza**
![Chat Telegram](evidencias/13_chat_telegram.png)
![Prueba Mendoza](evidencias/14_prueba_mendoza.png)

## Cómo importarlo

1. Tener importado y **publicado** el sub-workflow `Checkpoint 5 - Base de Conocimiento RAG` (carpeta [`checkpoint5/`](../checkpoint5/)), y ejecutar su ingesta. El vector store vive en memoria, así que la ingesta se repite cada vez que n8n se reinicia.
2. En n8n: **Workflows → Import from File** y elegir `checkpoint6_carla_baudino.json`.
3. Asignar las credenciales: Telegram (los 5 nodos de Telegram), Groq (Whisper y Chat Model) y **Header Auth** para ElevenLabs, con Name `xi-api-key` y la API key en Value.
4. En `buscar_manual_politicas`, en **Workflow → From list**, elegir `Checkpoint 5 - Base de Conocimiento RAG`. El ID cambia en cada instalación.
5. En `Parámetros de voz (marca)`, revisar el `voice_id`. Viene con George, una voz predeterminada que funciona en el plan gratuito.
6. Telegram necesita una URL pública HTTPS (por ejemplo, ngrok con dominio estático en `WEBHOOK_URL`). Después, **publicar** el workflow.
