[README (2).md](https://github.com/user-attachments/files/29935712/README.2.md)
# Odoo QC Fail Alert → Telegram (n8n)

Notificación en tiempo casi real a operarios de planta cuando un Quality Check falla en una orden de fabricación, con deduplicación para evitar spam de alertas repetidas sobre el mismo fallo.

Este proyecto es la continuación directa del caso [`odoo-quality-control-hard-stop`](enlace-al-otro-repo): una vez resuelto el bloqueo real de producción ante un Quality Check fallido, el siguiente problema operativo es **quién se entera, y cuándo**. Un bloqueo silencioso en el ERP no sirve de nada si el encargado de turno no lo ve hasta la revisión de la tarde.

## Caso de negocio

En almacén y planta real, un Quality Check fallido que se queda "dentro del ERP" pierde su valor si nadie lo ve a tiempo. Con 25 años de operativa de almacén, la experiencia es consistente: el operario que necesita saberlo está en planta, no delante de una pantalla de Odoo. Telegram es el canal donde ya está mirando.

El objetivo funcional: en cuanto un Quality Check pasa a estado `fail`, alguien lo sabe en menos de 5 minutos sin tener que entrar a Odoo a comprobarlo.

## Problema técnico de partida

Odoo Online en plan **Standard** bloquea el acceso XML-RPC/JSON-RPC externo fuera del periodo de trial. Esto descarta de raíz cualquier conector nativo de Odoo en n8n (que dependen de esos protocolos) y obliga a construir la integración a mano con nodos **HTTP Request** llamando directamente al endpoint `/jsonrpc`.

Es un hallazgo que otros consultores en el mismo plan se van a encontrar tarde o temprano — está documentado aquí con la solución de workaround completa.

## Arquitectura

```
Schedule Trigger (cada 5 min)
      ↓
HTTP Request (JSON-RPC → quality.check, domain: quality_state = fail)
      ↓
Split Out (un item por Quality Check fallido)
      ↓
Get row(s)  (Data Table: estado de alertas ya enviadas)
      ↓
If (¿ya se avisó de este Quality Check?)
   ├── false → Send a text message (Telegram) → Upsert row(s) (marca alerted = true)
   └── true  → (no acción, evita alerta duplicada)
```

**Captura del canvas completo:**
`screenshots/01-canvas-completo.png`

## Deduplicación

Sin control de estado, cada ejecución del Schedule Trigger (cada 5 min) volvería a alertar sobre el mismo Quality Check mientras siga en `fail`, generando spam.

La solución usa una Data Table (`quality_check_alert_state`) con las columnas `qualityCheckId` y `alerted`:

1. **Get row(s)** busca si el `qualityCheckId` ya tiene una fila registrada.
2. **If** evalúa `alerted is not equal to true`. Si la condición es falsa (ya se avisó), el flujo cae en la rama False y no se envía nada.
3. Si es la primera vez, se envía el Telegram y **Upsert row(s)** guarda `alerted: true` con match por `qualityCheckId`, cerrando el ciclo.

**Capturas:**
- `screenshots/02-http-request-jsonrpc.png` — configuración del nodo HTTP Request y ejemplo de respuesta JSON-RPC de Odoo
- `screenshots/03-if-threshold.png` — condición de deduplicación y ejemplo de item cayendo en la rama False
- `screenshots/04-upsert-dedup.png` — mapeo de columnas y condición de match del Upsert

## Resultado

**Captura:**
`screenshots/05-telegram-alerta.png`

Mensaje real generado en producción, incluyendo producto, orden de fabricación, centro de trabajo y responsable de la verificación — toda la trazabilidad que antes exigía entrar a Odoo, ahora en un mensaje.

## Limitaciones conocidas

- **Cadencia fija de 5 minutos, no push real-time.** El Schedule Trigger consulta Odoo por polling; un fallo de QC puede tardar hasta 5 minutos en generar alerta. Para tiempo real estricto haría falta un webhook desde Odoo, descartado en este caso por el sandbox de Automation Rules bloqueando `import requests` (ver `odoo-quality-control-hard-stop`).
- **Sin manejo de errores 5xx.** Si el endpoint JSON-RPC de Odoo devuelve un error de servidor, el workflow no tiene lógica de reintento ni notificación de fallo silencioso.
- **Autenticación:** la clave de API de Odoo se pasa dentro del body JSON-RPC. El JSON exportado en este repo sustituye la key real por un placeholder (`API_KEY_HERE`) — debe configurarse una key propia antes de reutilizar el workflow.

## Stack

- Odoo Online (SaaS, plan Standard)
- n8n (self-hosted, Docker)
- Telegram Bot API

---

Parte de una serie documentando casos reales de consultoría funcional Odoo + automatización con n8n.
