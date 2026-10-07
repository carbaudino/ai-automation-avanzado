# Checkpoint 2 · Orquestación Multi-Agente (Manager-Worker)

| Archivo | Rol |
|---|---|
| [`manager_modulo2_baudino_carla.json`](manager_modulo2_baudino_carla.json) | Manager: Chat Trigger → Router de Intención (Text Classifier) → Sets de contrato → Execute Workflow → log de trazabilidad (Gmail SMTP) → respuesta |
| [`worker1_modulo2_baudino_carla.json`](worker1_modulo2_baudino_carla.json) | Worker 1 · Calificación de Leads (el agente del Checkpoint 1 convertido en sub-workflow) |
| [`worker2_modulo2_baudino_carla.json`](worker2_modulo2_baudino_carla.json) | Worker 2 · Redacción de la derivación de reclamos a Operaciones |
| [`preentrega_modulo2_baudino_carla.pdf`](preentrega_modulo2_baudino_carla.pdf) | Documento entregado: capturas, esquema de datos y criterio de enrutamiento |

## Contrato de datos

| Dirección | Campo | Tipo | Obligatorio |
|---|---|---|---|
| Manager → Worker | `request_id` | string | Sí |
| Manager → Worker | `mensaje` | string | Sí |
| Manager → Worker 1 | `session_id` | string | Sí (memoria del worker) |
| Worker → Manager | `status` | `success` · `error` | Sí |
| Worker → Manager | `worker`, `request_id` | string | Sí |
| Worker → Manager | `data` | object | Solo si `status = success` |
| Worker → Manager | `error` | string | Solo si `status = error` |

Ejemplo de respuesta de contingencia:
```json
{ "status": "error", "worker": "worker_redaccion_reclamos", "request_id": "1852", "error": "Request timed out" }
```

## Cómo importarlo
1. Importar primero los dos Workers y guardarlos.
2. Importar el Manager y, en los nodos "Delegar - Worker 1 Leads" y "Delegar - Worker 2 Reclamos", elegir cada Worker desde la lista (Wait For Sub-Workflow Completion activado).
3. Asignar las credenciales de Groq, SMTP y la Data Table.
