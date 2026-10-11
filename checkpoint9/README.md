# Checkpoint 9 · Gobernanza, monitoreo financiero y trazabilidad (traceId)

**Alumna:** Carla Baudino · **Curso:** AI Automation Avanzado · **Entrega:** [`PreEntrega_Modulo9_CarlaBaudino.pdf`](PreEntrega_Modulo9_CarlaBaudino.pdf)

Plan de gobernanza del canal de voz supervisado del Checkpoint 8. Tiene tres partes:

- **Finanzas:** KPIs, ROI y TCO.
- **Trazabilidad:** un `traceId` que viaja por todos los sub-workflows, con logs JSON en Airtable.
- **Ética y compliance:** mapa de riesgos, prompt de objetividad y privacidad del proveedor.

## Archivos

| Archivo | Qué es |
|---|---|
| [`checkpoint9_canal_voz_supervisado.json`](checkpoint9_canal_voz_supervisado.json) | El canal de voz con Juez del M8. Ahora genera el `traceId`, lo pasa al RAG y al Juez, y escribe `traceId` + `Log_JSON` en Airtable. Su prompt suma la sección **OBJETIVIDAD Y EQUIDAD**. |
| [`checkpoint9_juez.json`](checkpoint9_juez.json) | El sub-workflow Juez del M8: recibe el `traceId` y lo devuelve en su veredicto. |
| [`checkpoint9_base_conocimiento_rag.json`](checkpoint9_base_conocimiento_rag.json) | El sub-workflow RAG del Checkpoint 5, actualizado: recibe el `traceId` y lo devuelve junto con los fragmentos. |
| [`auditoria_qa_traceid.csv`](auditoria_qa_traceid.csv) | Exportación de `Auditoria_QA` con las columnas `traceId` y `Log_JSON`. |

## Trazabilidad

![Recorrido del traceId](evidencias/01_recorrido_traceid.png)

- **Formato:** `TR-<execution.id>-<message_id de Telegram>`. Se genera en *Purgar audio del cliente*.
- **Recorrido:** viaja como input explícito a la herramienta `buscar_manual_politicas` (sub-workflow RAG) y al sub-workflow Juez. Los dos lo devuelven en su salida.
- **Registro:** termina en `Auditoria_QA`, en la columna `traceId`, y en el mensaje de Slack cuando el caso se escala.
- **Campos del `Log_JSON`:** `traceId`, `timestamp`, `executionId`, `workflow`, `nodeName`, `canal`, `intento`, `toolName`, `toolOutput`, `respuesta`, `systemPromptSnapshot`, `confidenceScore` (exactitud / 5), `veredicto` y `categoriaError`.

| traceId | Intento | Veredicto | toolOutput |
|---|---|---|---|
| TR-354-63 | 0 | CORREGIR (2) · DATO_NO_RESPALDADO | Sección 4, score 0,68 |
| TR-354-63 | 1 | ACEPTADO (5) | Sección 4, score 0,74 |

**Auditoría retroactiva:** cruzar el log de Airtable con la sub-ejecución del RAG por el mismo traceId (TR-348-59) reveló un error de orden de ejecución en los logs. Se corrigió moviendo *Fila de auditoría* por encima del Switch, y la corrección se verificó con TR-354-63.

![Airtable](evidencias/02_airtable_traceid.png)

## Números clave (detalle en el PDF)

| | Valor |
|---|---|
| Resolución autónoma (piloto) | 83 % (5 de 6) |
| Ahorro estimado | USD 945 por mes (≈ 118 h) |
| TCO mensual | USD 146 actual · USD 48,51 optimizado |
| ROI mensual | 547 % |
| Mayor costo | La voz (ElevenLabs), no los tokens |

## Cómo importarlo

1. Importá primero los dos sub-workflows: `checkpoint9_base_conocimiento_rag.json` y `checkpoint9_juez.json`. **Publicalos** y ejecutá la ingesta del manual.
2. Importá el canal. En *buscar_manual_politicas* y en *Evaluar respuesta (Juez)*, elegí los sub-workflows con **From list**.
3. En Airtable, agregá a `Auditoria_QA` las columnas `traceId` (texto) y `Log_JSON` (texto largo).
