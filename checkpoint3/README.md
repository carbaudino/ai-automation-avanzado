# Checkpoint 3 · Memoria Persistente y Summarization

| Archivo | Rol |
|---|---|
| [`manager_modulo3_baudino_carla.json`](manager_modulo3_baudino_carla.json) | Manager con la capa de memoria: Buscar Memoria → ¿Usuario recurrente? → Normalizar Contexto → Router → Workers → Guardar Intercambio → ¿Más de 5 mensajes? → Resumidor / Actualizar Sesión |
| [`worker1_modulo3_baudino_carla.json`](worker1_modulo3_baudino_carla.json) | Worker 1 con el bloque de contexto compartido |
| [`worker2_modulo3_baudino_carla.json`](worker2_modulo3_baudino_carla.json) | Worker 2 con el bloque de contexto compartido |
| [`PreEntrega_Modulo3_CarlaBaudino.pdf`](PreEntrega_Modulo3_CarlaBaudino.pdf) | Documento entregado: arquitectura, prompt del resumidor y esquema de la base |

## Base de memoria (Airtable · base *Memoria Agente*, tabla *Sesiones*)

| Columna | Tipo | Uso |
|---|---|---|
| Session_ID | Single line text (clave) | Aislamiento por sesión |
| Nombre | Single line text | Se inyecta como `user_name` |
| Estado del Caso | Single select | Status de la última solicitud; también orienta al router |
| Resumen Consolidado · Datos Clave · Acción Requerida | Long text | Resumen analítico (sin transcripciones) |
| Resumen_JSON | Long text | Objeto JSON completo del resumidor |
| Cantidad_Mensajes | Number | Activa la summarization al superar 5 |
| Fecha de Actualización | Date con hora | Último cambio |

**Búsqueda:** `{Session_ID} = '{{ $json.sessionId }}'`. Resuelta queda, por ejemplo, `{Session_ID} = 'eb1e832c94bf4934860fffcaa3008a0e'`.

**Resumen:** el JSON incluye las claves pedidas (`asunto_principal`, `puntos_clave`, `accion_requerida`) más una **extensión** `nombre_usuario`, necesaria para recuperar el nombre en la memoria.
