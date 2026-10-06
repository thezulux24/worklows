# Workflows de Automatización (n8n)

Catálogo de flujos de automatización exportados desde n8n.

| Archivo | Nombre en n8n | Trigger / Webhook | Función Principal |
| :--- | :--- | :--- | :--- |
| [`msd-deal-event.json`](./msd-deal-event.json) | MSD - Deal Event | `POST /deal-event` | Procesa eventos de negocios de HubSpot y genera logs de conversión para Google Ads en Supabase. |
| [`msd-attribution-capture.json`](./msd-attribution-capture.json) | MSD - Attribution Capture | `POST /attribution-capture` | Captura atribución de leads (click IDs, UTMs, GA4) enviada desde GTM a Supabase. |
| [`ai-library-ingest.json`](./ai-library-ingest.json) | AI Library - Roam Ingest | `POST /roam-ingest` | Ingesta de mensajes de Roam, validación HMAC y encolado de enlaces en Data Tables de n8n. |
| [`prospecta-lead-sophie.json`](./prospecta-lead-sophie.json) | Prospecta - Lead Sophie | `POST /prospecta-lead` | Normaliza leads del bot Sophie y realiza upsert/creación en HubSpot CRM. |
| [`ai-library-queue-api.json`](./ai-library-queue-api.json) | AI Library - Queue API | `POST /queue-next`<br>`POST /queue-update` | Motor FIFO de despacho, recuperación de tareas zombis y actualización de estados en cola. |

---

📄 **Plan de Migración:** Consulta el documento completo [PLAN-MIGRACION-VPS.md](./PLAN-MIGRACION-VPS.md) para ver la estrategia de migración hacia scripts independientes en un VPS.