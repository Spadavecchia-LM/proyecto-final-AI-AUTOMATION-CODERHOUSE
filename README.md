# Presupuestos por WhatsApp — Workflow n8n + Airtable + IA

Workflow de n8n (`proyecto final`) que recibe pedidos de clientes por WhatsApp, arma un presupuesto consultando el catálogo de Airtable con un agente de IA (Claude Sonnet 5), lo deja pendiente de aprobación humana por Gmail, y responde al cliente por WhatsApp según el resultado.

**Link a la base completa:** _[https://airtable.com/invite/l?inviteId=invDTR89ug0z4XlRT&inviteToken=dc7d9d94a0627ef21bbe989f1fe26ac161589190d1119cb35d32f686b53d6517&utm_medium=email&utm_source=product_team&utm_content=transactional-alerts)]_


## Índice
- [Arquitectura general](#arquitectura-general)
- [Flujo paso a paso](#flujo-paso-a-paso)
- [Bases de Airtable](#bases-de-airtable)
- [Agente de IA — reglas de negocio](#agente-de-ia--reglas-de-negocio)
- [Credenciales requeridas](#credenciales-requeridas)
- [Manejo de errores y casos borde](#manejo-de-errores-y-casos-borde)

## Arquitectura general

```
WhatsApp Trigger
      │
      ▼
   AI Agent (Claude Sonnet 5) ──[tool]── ARTICULOS (Airtable, tabla "Artículos")
      │                        ──[memoria]── Simple Memory (por wa_id)
      │                        ──[output parser]── Structured Output Parser
      │
      ├── (total > 0) ──► Guarda presupuesto en Airtable (Estado: "Procesado por IA")
      │                         │
      │                         ▼
      │                    Gmail sendAndWait → aprobación humana (APROBAR / RECHAZAR)
      │                         │
      │                    ┌────┴────┐
      │                 Aprobado   Rechazado
      │                    │            │
      │           Estado="Aprobado   Estado="Rechazado
      │            por humano"        por humano"
      │                    │            │
      │           Envía presupuesto  Notifica al admin
      │           final por WhatsApp por Telegram
      │
      └── (total = 0 / no matchea) ──► Responde por WhatsApp que no pudo procesar el pedido
```

## Flujo paso a paso

1. **WhatsApp Trigger**: recibe el mensaje entrante del cliente.
2. **AI Agent**: interpreta el pedido en lenguaje informal, consulta la tabla `Artículos` de Airtable (una sola vez por conversación, vía tool `ARTICULOS`) y arma el presupuesto ítem por ítem. Usa `Simple Memory` para mantener contexto por número de WhatsApp (`wa_id`) y un `Structured Output Parser` para forzar una salida JSON con `usuario`, `articulos`, `total` y `mensaje`.
3. **¿El presupuesto es mayor a 0?** (nodo IF): filtra pedidos vacíos o sin match en el catálogo.
   - Si **no** → responde al cliente que no pudo procesar la solicitud.
4. **Guarda el presupuesto en Airtable** (tabla `Presupuestos`), con Estado inicial `Procesado por IA`.
5. **Aprobación humana por Gmail** (`sendAndWait`, tipo *double approval*): se envía un mail con el detalle del presupuesto y dos botones (APROBAR / RECHAZAR).
6. **IF (aprobado == true)**:
   - **Aprobado** → actualiza Estado a `Aprobado por humano` y envía el presupuesto final al cliente por WhatsApp, con formato y vigencia de 48 hs.
   - **Rechazado** → actualiza Estado a `Rechazado por humano` y notifica al admin por Telegram.

## Bases de Airtable

Base: **proyecto final AI Automation**

| Tabla | Uso en el workflow | Link |
|---|---|---|
| **Artículos** | Catálogo consultado por el AI Agent (tool `ARTICULOS`, operación `search`) para matchear cada ítem del pedido y calcular precios. | _[https://airtable.com/appYGsKIp7kY6uGPM/shrMMPyHQTU69VTG1]_ |
| **Presupuestos** | Registro de cada presupuesto generado, con campos `N° de presupuesto`, `Fecha de creación`, `Cliente`, `Artículos`, `Total` y `Estado` (`Pendiente` / `Procesado por IA` / `Aprobado por humano` / `Rechazado por humano`). | _[https://airtable.com/appYGsKIp7kY6uGPM/shr7qjPmo18Qg9Kol]_ |

**Link a la base completa:** _[https://airtable.com/invite/l?inviteId=invDTR89ug0z4XlRT&inviteToken=dc7d9d94a0627ef21bbe989f1fe26ac161589190d1119cb35d32f686b53d6517&utm_medium=email&utm_source=product_team&utm_content=transactional-alerts)]_

> Los `base_id` (`appYGsKIp7kY6uGPM`) y `table_id` de cada tabla ya están cacheados en el JSON; si se reimporta el workflow en otra cuenta de Airtable, van a necesitar volver a mapearse.

## Agente de IA — reglas de negocio

Modelo: **Claude Sonnet 5** (`@n8n/n8n-nodes-langchain.lmChatAnthropic`), con `retryOnFail: true` y `maxTries: 3`.

Resumen del system prompt del `AI Agent`:
- Interpreta pedidos informales tipo *"5 packs de coca, 10 lavandinas de 1 litro"*.
- Consulta la tool de Airtable **una sola vez** por pedido y reutiliza esa lista en el contexto de la conversación.
- Matchea cada ítem por descripción + presentación + marca, priorizando la coincidencia más específica.
- Si hay ambigüedad entre dos artículos igual de plausibles, no adivina: la marca para preguntar.
- Si no encuentra un artículo, lo marca como NO ENCONTRADO y no lo cotiza.
- Nunca expone el catálogo completo en la respuesta, solo los ítems pedidos.
- Cálculo: `subtotal = precio × cantidad` (respetando unidad de venta / unidades por bulto), `total = suma de subtotales` redondeado a 2 decimales.
- Estilo de respuesta: español rioplatense, breve, sin emojis (salvo que el usuario los use primero), formato WhatsApp (`*negrita*`) sin markdown.

## Credenciales requeridas

| Servicio | Nodo(s) | Nota |
|---|---|---|
| WhatsApp Business API | Trigger + 3 nodos de envío | Incluye `phoneNumberId` y normalización de `wa_id` (saca el prefijo `549` para Argentina) |
| Anthropic API | Anthropic Chat Model | Modelo Claude Sonnet 5 |
| Airtable (OAuth2) | ARTICULOS, guardado/actualización de Presupuestos | Misma base para las 2 tablas |
| Gmail (OAuth2) | Aprobación humana | Envía a una casilla fija |
| Telegram | Notificación de rechazo | `chatId` fijo (admin) |

## Manejo de errores y casos borde

- Si el `AI Agent` falla, reintenta hasta 3 veces (`onError: continueErrorOutput`); si sigue fallando, el workflow responde al mismo usuario que no pudo procesar el pedido.
- Si el total del presupuesto no es mayor a 0 (sin ítems matcheados), se responde por WhatsApp indicando que no se pudo procesar y no se llega a la instancia de aprobación humana.
- El rechazo humano no notifica al cliente por WhatsApp, solo al admin por Telegram.


