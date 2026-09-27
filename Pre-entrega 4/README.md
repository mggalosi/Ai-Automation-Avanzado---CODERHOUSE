# Pre-entrega 4 · Integraciones Avanzadas: Bucle Ludoteca

**Checkpoint:** Sincronización del Cerebro Agéntico con Ecosistemas de Negocio
**Alumno:** Matias Galosi
**Archivo entregable:** checkpoint4_matias_galosi.json

Agente de soporte por email para **Bucle Ludoteca**, una tienda de juegos de mesa de Berisso. Continúa el proyecto integrador de los módulos anteriores: toma el agente con prompt modular (M1), la taxonomía de intenciones y el log de auditoría (M2) y la memoria de largo plazo por cliente (M3), y los conecta con tres herramientas externas reales mediante **OAuth2**.

| Rol en el caso de negocio | Herramienta |
|---|---|
| Casilla de soporte al cliente | **Gmail** |
| CRM de la tienda | **HubSpot** |
| Canal del equipo de operaciones | **Slack** (`#operaciones`) |
| Catálogo, memoria y log (heredados de M2/M3) | Google Sheets |

---

## Arquitectura

```
[Gmail Trigger - Soporte]  (solo correos dirigidos a la casilla de soporte)
        │
        ▼
① [IF - ¿Es auto-reply?] ── Sí ─▶ [Stop - Auto-reply ignorado]   → corta el bucle infinito
        │ No
        ▼
④ [Set - Limpiar payload] → [IF - ¿Payload válido?] ── No ─▶ [Set] → [Slack - Alertas]   → evita el 400
        │ Sí
        ▼
[Leer Memoria] + [Cruzar Memoria por Session_ID] → [Normalizar Memoria]
        │
        ▼
[Agente Soporte Email]  (tool: Catálogo · Structured Output Parser · máx. 5 iteraciones)
        │
        ▼
② [HubSpot - Buscar contacto] → [IF - ¿Contacto existe?]
        ├ Sí → [HubSpot - Actualizar contacto]
        └ No → [HubSpot - Crear contacto]                          → evita el 409
        │
        ▼
③ [Gmail - Crear Borrador (HITL)]  (sin envío: queda para aprobación humana)
        │
        ▼
[Set - Payload mínimo Slack] → [Slack - Notificar Operaciones]
        │
        ▼
[Guardar Log] → [Guardar Memoria]

Errores del Agente, HubSpot, Gmail, Log o Memoria → [Set - Payload Slack error] → [Slack - Alertas]
```
---

## 1. Conectividad segura y mínimo privilegio

Las tres integraciones se autentican por **OAuth2**. No se usan API keys, bots tokens ni Private App Tokens.


### Operaciones por conector

| Conector | Lectura (pasado) | Escritura (futuro) |
|---|---|---|
| Gmail | Gmail Trigger: lee los correos entrantes a la casilla de soporte | Create Draft: guarda el borrador de respuesta |
| HubSpot | Search: busca el contacto por email | Create or Update: crea o actualiza el contacto |
| Slack | No lee nada | Post message: publica en `#operaciones` |

### Scopes

| Conector | Scopes que usa el flujo | Qué se hizo para acotarlos |
|---|---|---|
| **Slack** | `chat:write` (bot) | La app se creó desde un manifest con **un único scope**. El bot no puede leer canales, historial ni usuarios, ni recibir mensajes directos. El canal se configura **por ID** y no "desde lista", así no hace falta `channels:read`. |
| **HubSpot** | `crm.objects.contacts.read`, `crm.objects.contacts.write` | App privada (instalable solo en la cuenta propia) con esos dos scopes como **requeridos**. La credencial nativa de n8n exige además una lista fija de scopes (companies, deals, owners, lists, forms): se declararon como **opcionales** para que la autorización no falle, y el flujo no los usa. |
| **Gmail** | Lectura de correo y creación de borradores | La credencial nativa de n8n pide un set de scopes fijo, más amplio que lo que usa el flujo. El riesgo se acota en el diseño: **no hay ningún nodo de envío de correo en el lienzo** (ver punto 3). |


### Otras medidas
- **Allowed HTTP Request Domains:** cada credencial solo puede usarse contra su propio dominio (`api.hubapi.com` para HubSpot, `slack.com` para Slack).
- **Sin secretos en el JSON:** el archivo exportado solo referencia las credenciales por ID y nombre; los tokens quedan cifrados en la instancia de n8n.
- **Filtro de entrada:** el Gmail Trigger solo toma correos dirigidos a la dirección de soporte (`to:` en el filtro de búsqueda), no toda la bandeja.

---

## 2. Contención de bucles y mitigación de errores

### ① IF anti auto-reply (bucle infinito)
Va **inmediatamente después del trigger**. Descarta el correo si se cumple **cualquiera** de estas condiciones (sin distinguir mayúsculas):

- **Asunto:** `auto-reply`, `autoreply`, `automatic reply`, `out of office`, `undeliverable`, `delivery status notification`, `respuesta automática`, `fuera de la oficina`.
- **Remitente:** `no-reply`, `noreply`, `mailer-daemon`, `postmaster`.

Si coincide, el correo termina en `Stop - Auto-reply ignorado` sin llegar al agente, al CRM ni a Slack.

### ④ Set de limpieza + validación (Error 400)
- El `Set - Limpiar payload` deja solo los campos que el flujo necesita: email (extraído limpio del remitente, en minúsculas), nombre, asunto, cuerpo (recortado a 4000 caracteres) e ID del hilo. Descarta HTML, adjuntos, headers y demás campos del correo.
- El `IF - ¿Payload válido?` verifica que el email tenga **formato válido** y que el cuerpo **no esté vacío**. Si falla, el correo no llega al agente ni a HubSpot (que respondería 400) y se avisa en Slack.

### ② Look up antes del Create (Error 409)
Antes de escribir en HubSpot, `HubSpot - Buscar contacto` busca por email (máximo 1 resultado, siempre devuelve salida aunque no haya coincidencia). El `IF - ¿Contacto existe?` decide:
- **Existe:** Actualizar contacto.
- **No existe:** Crear contacto.

Así se evitan contactos duplicados en el CRM.

### Manejo de errores
- **Reintentos:** 3 intentos con 2 s de espera en todos los nodos que llaman a servicios externos.
- **Salida de error:** el agente, HubSpot, Gmail y las escrituras en Sheets tienen salida de error. Ante una falla, `Slack - Alertas` publica el nodo y el ID de la ejecución para revisarla.
- **Límite del agente:** máximo 5 iteraciones; salida estructurada validada por un Structured Output Parser con taxonomía cerrada (`ATENCION_COMERCIAL`, `RECOMENDACION_JUEGOS`, `ESCALAR_HUMANO`).

---

## 3. Gobernanza operativa e interfaz HITL

### ③ Gmail - Crear Borrador (HITL)
La única salida de correo del workflow usa la operación **Create Draft**. **No hay ningún nodo que envíe correos.**

- El borrador queda con **destinatario** (el cliente) y **dentro del hilo original**, así el operador solo tiene que revisarlo y hacer clic en *Enviar*.
- El agente nunca confirma compras, pagos, envíos ni reembolsos: deriva esos casos al equipo.
- Cada borrador genera un aviso en Slack, *"Borrador pendiente de aprobación"*, con prioridad **ALTA** cuando la intención es `ESCALAR_HUMANO` (reclamos, compras concretas, casos fuera de ámbito).

### Set de payload mínimo antes de Slack
Antes de cada nodo de Slack hay un `Set` que arma un único campo `texto`, con campos truncados (asunto a 120 caracteres, resumen y acción a 200). A Slack no llega el cuerpo completo del correo, ni binarios, ni objetos del agente.

---

## Memoria y trazabilidad (continuidad con M2/M3)

- **Memoria por cliente:** hoja `Memoria` indexada por `Session_ID` = email del remitente. El agente recibe el resumen previo del cliente dentro de un bloque delimitado `[INICIO DE CONTEXTO COMPARTIDO] ... [FIN DEL CONTEXTO COMPARTIDO]`, marcado como dato y no como instrucción. En cada correo actualiza la memoria (`appendOrUpdate` por `Session_ID`, sin duplicar filas).
- **Log de auditoría:** una fila por correo procesado en la hoja `Log` (timestamp, cliente, intención, respuesta del agente, estado).
- **Anti prompt injection:** el correo se pasa al agente entre delimitadores `[INICIO DEL CORREO] ... [FIN DEL CORREO]`, con la regla explícita de tratar su contenido como dato.

---

## Test de regresión manual

| # | Caso | Resultado esperado |
|---|---|---|
| 1 | Consulta de disponibilidad y precio desde un remitente nuevo | Contacto **creado** en HubSpot, borrador con precios del catálogo, aviso en Slack, filas en Log y Memoria |
| 2 | Segundo correo desde el mismo remitente | Contacto **actualizado** (sin duplicar), cliente marcado como recurrente, borrador que usa la memoria previa |
| 3 | Correo con asunto "Out of office" | Termina en `Stop - Auto-reply ignorado`; no se crea contacto, borrador ni aviso |


