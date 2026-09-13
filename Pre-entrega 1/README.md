# Checkpoint 1 — Agente base de Bucle Ludoteca

Primera versión del agente de IA para **Bucle Ludoteca** (tienda de juegos de mesa, Berisso), construida en n8n como motor de razonamiento inicial del proyecto integrador. Este `.json` es la base que se va a importar y extender en los módulos siguientes (M2 en adelante).

## Objetivo del módulo

Configurar y validar el `AI Agent Node` en modo Tools Agent: system prompt estructurado, guardrail de iteraciones máximas, una tool acoplada lateralmente y un log de observabilidad hacia un canal externo.

## Arquitectura

```
[Chat Trigger] → [AI Agent - Bucle Ludoteca] → [Gmail - Log de Auditoría]
                        ↑ ai_languageModel
                [OpenAI Chat Model (gpt-4o)]
                        ↑ ai_tool
                [Gmail Tool - Notificar Equipo]
```

## Componentes

| Nodo | Tipo | Rol |
|---|---|---|
| `Chat Trigger` | `@n8n/n8n-nodes-langchain.chatTrigger` | Punto de entrada de la consulta del cliente |
| `AI Agent - Bucle Ludoteca` | `@n8n/n8n-nodes-langchain.agent` (Tools Agent) | Motor de razonamiento — system prompt modular, `maxIterations: 5` |
| `OpenAI Chat Model` | `@n8n/n8n-nodes-langchain.lmChatOpenAi` (`gpt-4o`) | Modelo de lenguaje del agente |
| `Gmail Tool - Notificar Equipo` | `n8n-nodes-base.gmailTool` (conexión `ai_tool`) | Tool lateral: notifica por mail cuando hay que escalar a un humano (pedido de compra o reclamo) |
| `Gmail - Log de Auditoría` | `n8n-nodes-base.gmail` (nodo secuencial) | Observabilidad: envía siempre un mail de "Tarea completada" con la consulta y la respuesta del agente |

## System Prompt

Estructura modular Rol → Ámbito → Objetivo → Reglas y Escalamiento. Define el rol de "Agente de Atención Comercial de Bucle Ludoteca", acota el ámbito a consultas de venta (catálogo, precios, horarios/ubicación), prohíbe confirmar compras en nombre del negocio, y fija el límite de 5 iteraciones de razonamiento.

## Cómo importar y probar

1. Importar `checkpoint1_matias_galosi.json` en n8n.
2. Asignar credenciales propias en los nodos que las requieren: `OpenAI Chat Model` (API key de OpenAI) y los dos nodos de Gmail (credencial OAuth2 de Gmail, conectada a la misma cuenta).
3. Confirmar que el campo `sendTo` de ambos nodos de Gmail apunte al mail donde se quieren recibir las notificaciones.
4. Abrir el Chat Trigger y probar con al menos dos casos:
   - Una consulta puramente informativa (ej. "¿qué horario tienen?") — el agente debe responder sin activar la tool de Gmail.
   - Un pedido de compra concreto (ej. "quiero comprar el juego Catan, ¿cómo pago?") — el agente debe activar la tool de Gmail y además llegar el mail de log.
5. Revisar el panel de ejecución: recorrido en verde, sin errores de sintaxis en las expresiones.

## Criterios de evaluación cubiertos

| Criterio | Peso | Cómo se cumple |
|---|---|---|
| Observabilidad y log externo de finalización | 35% | `Gmail - Log de Auditoría` se ejecuta siempre después del agente, con asunto "Tarea completada..." |
| Arquitectura de Interfaces No-Code | 30% | La tool de Gmail está conectada por `ai_tool` (lateral), nunca como nodo secuencial del lienzo principal |
| Guardrails Financieros y Observabilidad | 35% | `maxIterations: 5` (dentro del rango 5-10 exigido) + log automático de auditoría |

## Notas

- Las notificaciones están dirigidas por el momento al Gmail personal del responsable del emprendimiento, no a un mail propio del negocio (todavía no existe esa casilla separada).
- El alcance de este módulo es solo venta: no incluye reservas, alquiler ni eventos (se evaluará sumarlos en un módulo de integraciones más adelante).
