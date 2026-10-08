# Checkpoint 5 · RAG: Agente con Conocimiento Organizacional

El agente de email del Checkpoint 4 ahora responde las consultas de políticas con el **Manual de Políticas y Procedimientos de Servicio (MAN-SC-001 v3.1)** de Logística Demo S.A., un documento ficticio creado para el curso.

| Archivo | Rol |
|---|---|
| [`checkpoint5_carla_baudino.json`](checkpoint5_carla_baudino.json) | Agente de email con la herramienta `buscar_manual_politicas` y el prompt RAG |
| [`checkpoint5_base_conocimiento_rag.json`](checkpoint5_base_conocimiento_rag.json) | Sub-workflow con dos carriles: A · ingesta del manual y B · herramienta de búsqueda (Top-K + Minimum Score) |
| [`manual_politicas_limpio.md`](manual_politicas_limpio.md) | Salida de LlamaParse ya limpia, que es lo que se indexa |
| [`Manual_Politicas_Servicio_Logistica_Demo.pdf`](Manual_Politicas_Servicio_Logistica_Demo.pdf) | Documento original |
| [`PreEntrega_Modulo5_CarlaBaudino.pdf`](PreEntrega_Modulo5_CarlaBaudino.pdf) | Documento entregado: las 5 piezas de la rúbrica |

## Pipeline

```
LlamaParse (Cost-Effective) ─▶ limpieza (encabezados, tabla partida, títulos, HTML→Markdown)
   ─▶ 1 fragmento por sección (13) ─▶ embeddings Gemini ─▶ Simple Vector Store
AI Agent ─▶ buscar_manual_politicas ─▶ Top-K = 3 ─▶ Minimum Score ≥ 0,68 ─▶ fragmentos con sección y score
```

## Calibración

| Parámetro | Valor | Por qué |
|---|---|---|
| Top-K | 3 | El fragmento correcto salió 1.º en las 4 preguntas con respuesta; el contexto queda acotado a ~750 tokens |
| Minimum Score | 0,68 | Fragmentos correctos: 0,73–0,80. Ruido: ≤ 0,65. Con 0,68 llega solo el fragmento útil |

## Prueba ciega (5 preguntas por email)

| # | Pregunta | Resultado |
|---|---|---|
| 1 | ¿A cuántos grados llevan la fruta fresca? | ✅ 2–8 °C (sección 5.1) |
| 2 | Cajas rotas adentro, detectadas 2 días después | ✅ Daño oculto, hasta 72 h (sección 6) |
| 3 | Aviso a la mañana que no cargamos a la tarde | ✅ 50 % de la tarifa (sección 9) |
| 4 | ¿Cuánto tarda hasta Bariloche? | ❌ **Falla de contención:** respondió "Patagonia, 96–120 h" deduciendo con conocimiento externo que Bariloche está en Río Negro. Como la localidad no aparece en los fragmentos, la respuesta esperada era "No sé" |
| 5 | ¿Retiran los domingos? | ✅ "No sé" (sin fragmentos sobre el umbral) |

### Métricas (corregidas tras la devolución docente)

| Métrica | Resultado |
|---|---|
| Aciertos documentales (respuesta 100 % respaldada por los fragmentos) | **4/5 (80 %)** |
| Fallas de contención (inferencia con conocimiento externo) | **1/5 (20 %)**, la pregunta 4 |
| Aplicación correcta de la regla "No sé" | 1/1 |
| Recuperación Top-1 correcta | 4/4 preguntas con respuesta en el manual |

**Acción correctiva aplicada:** se agregó al System Prompt la regla *"No infieras ni completes datos que no estén escritos en los fragmentos, aunque los conozcas por otro lado (por ejemplo, ubicar una ciudad en una provincia o zona). En ese caso respondé 'No sé'"*. La misma regla se aplica al agente de voz del Checkpoint 6, que usa la misma herramienta.

**Verificación:** con la regla nueva se repitió la pregunta 4 y el agente respondió *"No sé el tiempo exacto de entrega de un camión hasta Bariloche. Le derivamos su consulta al área correspondiente…"*, con `fuentes` vacío. La falla de contención quedó corregida.

![Pregunta 4 corregida: No sé](evidencias/05_correccion_bariloche_no_se.png)

## Evidencias
![Parseo en LlamaParse](evidencias/01_llamaparse_titulos_tablas.png)
![Parámetros RAG](evidencias/02_parametros_rag.png)
![Verificación del Minimum Score](evidencias/03_verificacion_min_score.png)
![Regla "No sé"](evidencias/04_regla_no_se.png)

## Cómo importarlo
1. Importar `checkpoint5_base_conocimiento_rag.json`, asignar la credencial de Gemini en los dos nodos de embeddings, **publicarlo** y ejecutar **Ejecutar Ingesta** (13 fragmentos).
2. Importar `checkpoint5_carla_baudino.json` y, en `buscar_manual_politicas`, elegir el sub-workflow desde la lista. Asignar las credenciales de Gmail, Airtable, HubSpot, Slack y Groq.
3. La base de vectores vive en memoria: después de reiniciar n8n hay que volver a ejecutar la ingesta.
