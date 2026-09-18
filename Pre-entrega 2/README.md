# Bucle Ludoteca — Orquestación Multi-Agente (Manager-Worker)

Checkpoint 2 del curso **AI Automation Avanzado** (Coderhouse). Extiende el agente único de atención comercial del Checkpoint 1 hacia una arquitectura **Manager-Worker**: un agente Router clasifica la intención de cada consulta de un cliente de Bucle Ludoteca (tienda de juegos de mesa real, Berisso) y la delega a un agente especialista (Worker), implementado como sub-workflow independiente de n8n.

El canal de entrada/salida es un bot de Telegram real, y ambos Workers consultan como tool un catálogo real en Google Sheets — no hay datos hardcodeados en los prompts.

## Qué resuelve

- **Router de Intenciones**: clasifica cada mensaje en una taxonomía cerrada de 3 categorías (AI Agent + Structured Output Parser), separando la decisión probabilística (LLM) del enrutamiento determinista (Switch).
- **Worker 1 — Atención Comercial**: responde consultas de catálogo, precio, disponibilidad y horarios. Puede escalar por su cuenta a un humano vía Gmail si detecta un pedido de compra concreto o un caso fuera de su ámbito, además del escalamiento que ya decide el Router — dos capas de escalamiento independientes.
- **Worker 2 — Recomendador de Juegos**: recomienda 2-3 juegos del mismo catálogo real según jugadores, edad, categoría o tiempo disponible, inferido del campo Resumen de la planilla cuando está disponible.
- **Auditoría**: cada ejecución del Manager, sea resuelta por un Worker o escalada, queda registrada en un Google Sheet con el payload enviado, la respuesta recibida y el resultado final entregado al cliente.

## Arquitectura

```
[Telegram Trigger] → [Contrato de Entrada] → [Router de Intenciones] → [Switch]
   ATENCION_COMERCIAL   → [Execute Workflow: Worker 1] ─┐
   RECOMENDACION_JUEGOS → [Execute Workflow: Worker 2] ─┼→ [Log de Auditoria (Sheets)] → [Telegram: Enviar Respuesta]
   ESCALAR_HUMANO / error de cualquier Worker → [Gmail: Notificar Supervisor] ─┘
```

El contrato de datos entre Manager y Workers es intencionalmente mínimo: `{ session_id, mensaje_usuario }` de entrada, y `{ status: "success"|"error", data: {...} }` de salida — esto desacopla al Manager de los detalles internos de cada Worker y permite que cualquier error de un Worker caiga en el mismo camino de escalamiento humano sin lógica distinta por rama.

## Archivos de este repositorio

| Archivo | Contenido |
|---|---|
| `manager_modulo2_galosi_matias.json` | Workflow del Manager (Router, Switch, logging, respuesta al cliente) |
| `worker1_atencion_comercial_modulo2_galosi_matias.json` | Sub-workflow Worker 1 |
| `worker2_recomendador_juegos_modulo2_galosi_matias.json` | Sub-workflow Worker 2 |
| `preentrega_modulo2_galosi_matias.pdf` | Documentación completa: prompts, diagramas, tabla de validación y troubleshooting |

## Validación

Los 3 casos de la taxonomía fueron probados de punta a punta contra el canal real de Telegram:

| Caso | Mensaje de ejemplo | Resultado |
|---|---|---|
| Atención Comercial | "¿Tenés Catan? ¿A cuánto sale?" | El Worker consultó el catálogo real y respondió correctamente que no está disponible, sin inventar un precio |
| Recomendación | "Busco un juego para jugar en familia con chicos, no muy largo" | Recomendación coherente pese a que la planilla no tiene columnas explícitas de jugadores/edad/duración |
| Escalamiento | "Quiero hacer un reclamo, un pedido no me llegó" | Se disparó el mail al supervisor y el cliente recibió la confirmación de derivación |

## Incidencias encontradas durante el desarrollo

Documentar el proceso de debugging fue parte deliberada de este checkpoint. Las cuatro más relevantes:

1. **El bot no respondía en Telegram** aunque el Log se completaba: faltaba el nodo de salida de Telegram al migrar desde un Chat Trigger de prueba.
2. **El Log solo guardaba el timestamp**: el nodo de Telegram estaba conectado antes del Log, y el output de un nodo de acción reemplaza el `$json` completo por la respuesta de su propia API — se resolvió reordenando a Log → Telegram.
3. **El Worker de Recomendación fallaba con un error de tipos** (`session_id` llegaba como Number donde se esperaba String) — se resolvió activando "Attempt to convert types" en el Execute Workflow.
4. **Los reclamos se clasificaban como consulta comercial en vez de escalar**: la taxonomía del prompt del Router incluía "reclamos" dentro de ATENCION_COMERCIAL — no era un error del modelo, clasificaba tal cual se le pedía. Se corrigió la definición de las categorías.

Detalle completo de causa raíz y solución de cada una en el PDF adjunto.

## Contexto del curso

Este módulo es el segundo hito de un proyecto integrador que evoluciona checkpoint a checkpoint: M1 (agente único) → **M2 (Manager-Worker, este repo)** → M3 (memoria persistente por sesión) → M4 (integraciones de calendario/CRM) → M5 (RAG sobre el catálogo) → M6 (voz), hasta un concierge integral para Bucle Ludoteca.

---
Matías Gabriel Galosi — AI Automation Avanzado, Coderhouse
