# M4 — Sincronización del Cerebro Agéntico con Ecosistemas de Negocio

**Proyecto:** Agente de Triaje Odontológico · Odontología Sonrisas
**Curso:** AI Automation Avanzado (Coderhouse)
**Autor:** Javier Echalecu

## Archivos de este módulo

| Archivo | Rol |
|---|---|
| `checkpoint4_javier_echalecu.json` | **Manager** — extiende el de M3 con entrada multicanal (chat + correo) e integraciones |
| `Worker_CRM_Email.json` | **Worker nuevo** — sincroniza HubSpot, crea borrador de respuesta, notifica a Slack |
| `Worker_Agendamiento3.json` | Sin cambios desde M3 |
| `Worker_Urgencia3.json` | Sin cambios desde M3 |

Son **4 workflows independientes de n8n**, no un solo lienzo con sub-flujos.

## Cómo importar

1. Lienzo vacío → *Import from File* → `Worker_CRM_Email.json`. Publicar el workflow.
2. Lienzo vacío → *Import from File* → `checkpoint4_javier_echalecu.json`.
3. Abrir `Execute Workflow - Worker_CRM_Email` → reseleccionar el workflow del dropdown → recargar el mapeo de los 10 inputs (`email`, `nombre_pila`, `apellido`, `telefono`, `motivo`, `categoria`, `asunto`, `message_id_header`, `cuerpo_borrador`, `resumen_slack`).
4. Confirmar que `Execute Workflow - Worker_Agendamiento` y `Execute Workflow - Worker_Urgencia` sigan apuntando a `Worker_Agendamiento3` / `Worker_Urgencia3` (los de M3, ya existentes en la instancia, no incluidos como novedad acá).
5. Asignar credenciales (ver tabla de abajo) y **Publicar** el Manager. `Worker_Agendamiento3` y `Worker_Urgencia3` también deben estar publicados — n8n exige que todo sub-workflow referenciado lo esté.

## Los 4 nodos que evalúa la rúbrica

| # | Criterio | Nodo real | Workflow |
|---|---|---|---|
| ① | IF anti auto-reply | `IF - ¿Es Auto-Reply o Ruido Automático?` | Manager |
| ② | Look up antes del Create (anti-409) | `HubSpot - Buscar Contacto (Look up)` + `IF - ¿Contacto Existe en CRM?` | Worker_CRM_Email |
| ③ | Create Draft (HITL) | `Code - Crear Borrador vía IMAP APPEND (HITL)` | Worker_CRM_Email |
| ④ | Set de limpieza de payload (anti-400) | `Set - Payload Mínimo para CRM y Correo` | Manager |

## Arquitectura: qué se agregó, qué no se tocó

El correo entra como **segundo canal del Manager existente** (`IMAP Trigger`), no como workflow paralelo — converge con el `Chat Trigger` en `Set - Extraer Session_ID`, el normalizador multicanal. De ahí en adelante, la memoria, el Router, el Switch y el bloque de resumen heredados de M3 funcionan exactamente igual, sin saber por qué canal entró el mensaje.

El bloque de integraciones (HubSpot + borrador + Slack) vive en el `Worker_CRM_Email`, un cuarto Worker en lienzo separado que habla con el Manager mediante el mismo contrato `{status, data, estado_caso_sugerido}` que ya usan los Workers de M2/M3 — por eso el bloque de escritura de memoria lo absorbe sin modificarse.

## Desvíos documentados respecto a la letra de la consigna

La consigna pide OAuth2 para las tres integraciones. En este entorno no fue posible completar dos de esos flujos con el mecanismo exacto que se pedía. Se optó por alternativas funcionalmente equivalentes, declaradas explícitamente:

### 1. Gmail — IMAP + App Password, no OAuth2

No fue viable completar el flujo OAuth2 de Google Cloud en este entorno. Se resolvió con IMAP genérico + App Password.

- **Qué se pierde:** scopes granulares de Google (`gmail.readonly`/`gmail.compose` sin `gmail.send`). Una App Password habilita el buzón completo (IMAP+SMTP).
- **Qué se mantiene:** la garantía de "esto no puede enviar correo" pasa de la capa de autorización del proveedor a la capa de **diseño del workflow** — en ningún punto del sistema existe un nodo o una línea de código con un verbo IMAP/SMTP de envío. El único verbo de escritura implementado es `APPEND` a la carpeta de Borradores, auditable revisando el código del `Code - Crear Borrador vía IMAP APPEND (HITL)`.
- **Mitigación adicional:** la App Password vive en variables de entorno del contenedor Docker (`IMAP_USER`, `IMAP_APP_PASSWORD`), nunca en el JSON del workflow.
- Como no existe un nodo nativo de n8n para crear borradores vía IMAP genérico (el nodo Gmail sí lo tiene, pero exige OAuth2), el `APPEND` se implementó a mano con el módulo `tls` nativo de Node dentro de un Code node: `LOGIN → APPEND → LOGOUT`, sin librerías externas.

### 2. HubSpot — Private App / Service Key, no OAuth2 de tres pasos

Las "aplicaciones privadas" de HubSpot (a diferencia de las apps públicas) no ofrecen el flujo OAuth2 de tres pasos con Redirect URL — emiten un token/clave estático que se pega directo en la credencial de n8n.

- Scopes acotados igual de fino: `crm.objects.contacts.read` + `crm.objects.contacts.write`, nada más.
- Sin acceso a deals, tickets, companies ni export de la base.

**Resumen:** de las 3 integraciones, **Slack** es OAuth2 puro y cumple la letra exacta de la consigna. Gmail y HubSpot cumplen el espíritu del criterio (conectividad segura, mínimo privilegio verificable, autenticación fuera del código) por vías alternativas, explicadas y no ocultas — mismo criterio de honestidad que se usó en M3 para justificar Gemini en vez de GPT-4o-mini.

## Matriz de mínimo privilegio

| Sistema | Rol | Lectura (pasado) | Escritura (futuro) | Autenticación | Lo que NO puede hacer |
|---|---|---|---|---|---|
| **Gmail** | Casilla de soporte | IMAP Trigger: INBOX, sin adjuntos | APPEND a Drafts | IMAP + App Password | **Enviar correos** — garantía de diseño, cero nodos de envío en el sistema |
| **HubSpot** | CRM de la clínica | Search por email, 4 propiedades | Create/Update de contacto | Private App / Service Key | Deals, tickets, companies, export |
| **Slack** | Canal de operaciones | `channels:read` | `chat:write` | OAuth2 | Leer historial del canal, subir archivos |

## Compuertas de contención

- **① Anti auto-reply**, 11 reglas OR sobre asunto y remitente (incluye auto-referencia de la propia casilla) — corta la ejecución en 16ms sin tocar LLM, HubSpot ni Airtable.
- **② Look up antes del alta en CRM** — decide entre alta completa y actualización incremental. Protege campos cargados por humanos de ser pisados por inferencias del LLM en visitas repetidas del mismo paciente.
- **④ Validación de payload en dos capas** (Manager y Worker) — email contra regex, nombre partido en firstname/lastname, teléfono descartado si no tiene ≥6 dígitos, textos truncados.

## Gobernanza HITL

`Code - Crear Borrador vía IMAP APPEND (HITL)` implementa exclusivamente el verbo `APPEND`. Ninguna respuesta redactada por el modelo sale de la organización sin que un operador la abra en Gmail → Borradores, la revise y presione Enviar manualmente.

## Testing realizado — evidencia end-to-end

| # | Escenario | Resultado |
|---|---|---|
| 1 | Mail con asunto `Out of office: ...` | Ejecución de 16ms, terminó en `NoOp - Bucle Cortado`. Cero llamadas a LLM/HubSpot/Airtable |
| 2 | Remitente `noreply@...` | Mismo mecanismo que el 1 (no re-testeado individualmente) |
| 3 | Paciente nuevo por correo, con nombre/teléfono/urgencia | Router→URGENCIA→Worker_Urgencia (Slack+Sheets) ; look up sin resultados→contacto **creado** en HubSpot ; borrador creado en Gmail ; fila nueva en Airtable con `Session_ID = email:<remitente>` |
| 4 | Mismo paciente, segundo correo | Look up **encuentra** el contacto→rama Actualizar (Slack confirma "CRM: contacto actualizado") ; HubSpot sigue con 1 solo contacto, no se duplicó ; **misma fila** de Airtable, contador incrementado a 2 |
| 5 | Mensaje por Chat Trigger (regresión) | Memoria, Router y Workers de M3 funcionan igual ; `Session_ID` de formato chat, aislado del `Session_ID` de email ; no se invocó HubSpot ni Gmail |

Un `503 Service Unavailable` transitorio de Google Sheets apareció durante el Escenario 3, capturado por `continueOnFail` sin interrumpir la orquestación — mismo patrón de robustez ya documentado en M3 con la API de Gemini.

## Modelos y credenciales

- Gemini: `gemini-3.5-flash-lite`, heredado de M3 sin cambios.
- Airtable, Google Sheets, Slack (Manager antiguo): credenciales heredadas de M3.
- Slack OAuth2, HubSpot Private App, IMAP: credenciales nuevas de este módulo, no reutilizables desde M3.
