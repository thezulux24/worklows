# Plan de Migración: De n8n a Microservicios en VPS con Coolify
> **Documento Técnico de Arquitectura e Implementación — LATINOVATION**  
> *Plataforma:* VPS gestionado con **Coolify** (PaaS self-hosted sobre Docker & Traefik).  
> *Objetivo:* Reemplazar flujos visuales de n8n por un microservicio autónomo en Node.js/TypeScript desplegado automáticamente mediante GitOps en Coolify.

---

## 1. La Ventaja Estratégica de Tener Coolify

Tener **Coolify** en el VPS simplifica enormemente la migración y la operación a largo plazo, ya que elimina por completo la gestión manual de servidores (evitando configurar PM2 a mano, certbot de Let's Encrypt o archivos de Nginx en la terminal):

* **Despliegues Automáticos con Git (GitOps):** Cada `git push` a la rama `main` en GitHub compila y despliega automáticamente la nueva versión del servicio sin tiempo de inactividad (*zero-downtime rolling update*).
* **SSL y Dominio Automáticos:** Coolify administra Traefik internamente. Con solo escribir el dominio (ej. `https://automations.latinovation.com`), aprovisiona y renueva los certificados SSL de Let's Encrypt de forma automática.
* **Gestión Segura de Variables de Entorno:** Los secretos (`DATABASE_URL`, API tokens, webhooks secrets) se gestionan desde el panel web de Coolify, cifrados y fuera del código fuente.
* **Servicios Auxiliares en 1 Clic (Redis / PostgreSQL):** Si se requiere una cola ultrarrápida con Redis (BullMQ) para el sistema de `ai-library`, Coolify permite levantar un contenedor de Redis en la misma red privada de Docker con un solo clic.
* **Monitoreo y Reinicio:** Supervisión nativa de consumo de CPU/RAM, logs en tiempo real y reinicio automático si un contenedor falla.

---

## 2. Análisis Detallado de las Automatizaciones Actuales

A continuación se detalla la función, endpoints, dependencias y mejoras de cada flujo al correr como microservicio en Coolify:

---

### A. [`msd-deal-event.json`](./msd-deal-event.json) — *Atribución de Negocios para Google Ads*
* **Trigger:** Webhook `POST /deal-event` (header `x-msd-deal-secret` o parámetro `_secret`).
* **Propósito y Funcionamiento Actual:**
  1. Recibe notificaciones de HubSpot ante cambios de estado de negocios (`deal_created`, `deal_won`, `deal_lost`).
  2. Valida la clave secreta.
  3. Normaliza email y teléfono generando hashes **SHA-256** para las Conversiones Mejoradas de Google Ads.
  4. Consulta en PostgreSQL (Supabase) la tabla `lead_attribution` (últimos 90 días) y `msd_locations`.
  5. Clasifica el método de coincidencia (`gclid+enhanced`, `gbraid`, `wbraid`, `enhanced_only`).
  6. Inserta el registro en `conversion_upload_log`.
* **Mejoras al Migrar a Coolify:**
  * **Secretos Protegidos:** El token `1a8847cc...` se traslada a las variables de entorno de Coolify (`MSD_DEAL_SECRET`).
  * **Pool Concurrente:** Conexión persistente mediante `pg.Pool`, reduciendo la latencia de consulta a menos de 10 ms.
  * **Reintentos Automáticos:** Tolerancia a microcortes de red con reintentos exponenciales hacia Supabase.

---

### B. [`msd-attribution-capture.json`](./msd-attribution-capture.json) — *Captura de Atribución Web desde GTM*
* **Trigger:** Webhook `POST /attribution-capture` (enviado desde Google Tag Manager en modo `fetch no-cors`).
* **Propósito y Funcionamiento Actual:**
  1. Captura datos de marketing cada vez que un usuario envía un formulario web.
  2. Parsea el cuerpo enviado como texto plano por restricciones de GTM `no-cors`.
  3. Valida el secreto `_secret` y hashea email y teléfono con SHA-256.
  4. Extrae click IDs (`gclid`, `gbraid`, `wbraid`, `fbclid`, `msclkid`, `ttclid`), GA4 IDs, UTMs, URLs y metadatos del navegador.
  5. Inserta en la tabla `lead_attribution` de Supabase con `ON CONFLICT (hs_submission_guid) DO NOTHING`.
* **Mejoras al Migrar a Coolify:**
  * **Absorción de Picos:** Al ejecutarse en un contenedor ligero en Coolify, puede absorber cientos de peticiones simultáneas de campañas publicitarias sin saturar memoria.
  * **Validación de Tipos con Zod:** Sanitización y validación estricta de todos los parámetros UTM y click IDs antes de tocar la base de datos.

---

### C. [`prospecta-lead-sophie.json`](./prospecta-lead-sophie.json) — *Sincronización de Leads de Sophie con HubSpot*
* **Trigger:** Webhook `POST /prospecta-lead` (Header Auth: `x-prospecta-secret`).
* **Propósito y Funcionamiento Actual:**
  1. Recibe leads capturados por el asistente virtual "Sophie".
  2. Limpia y normaliza el contacto: detecta leads de prueba (`qa+...`, `@example.com`, prefijo `[PRUEBA]`), separa nombres y apellidos, y limpia el número de teléfono.
  3. Decisión condicional:
     - **Con email:** Llama a HubSpot para hacer un *Upsert* (actualización o creación).
     - **Sin email:** Realiza una llamada HTTP REST directa a `/crm/v3/objects/contacts` de HubSpot para crearlo solo con teléfono.
  4. Responde `200` ante éxito o `502` si HubSpot falla.
* **Mejoras al Migrar a Coolify:**
  * **SDK Oficial (`@hubspot/api-client`):** Reemplazo de nodos visuales por el cliente oficial con tipado completo.
  * **Protección Contra Rate Limits:** Uso de limitadores de tasa en memoria (`p-queue`) para evitar exceder los límites de peticiones de HubSpot (100 req / 10s).
  * **Pruebas Automatizadas:** Pruebas unitarias en el pipeline de build para validar que las expresiones regulares de normalización funcionen antes de desplegar.

---

### D. [`ai-library-ingest.json`](./ai-library-ingest.json) — *Ingesta de Mensajes y URLs desde Roam*
* **Trigger:** Webhook `POST /roam-ingest`.
* **Propósito y Funcionamiento Actual:**
  1. Responde de inmediato `HTTP 200 { ok: true }` para no bloquear al cliente emisor.
  2. En segundo plano consulta la clave de firma en la Data Table interna de n8n `fnIg37MSJjgeoH3c`.
  3. Valida la firma criptográfica HMAC.
  4. Filtra por canal y verifica duplicados.
  5. Extrae URLs del texto mediante expresiones regulares.
  6. Encola los enlaces en la tabla de tareas `sAM4ZqvSEJcEblJv`.
* **Mejoras al Migrar a Coolify:**
  * **Cero Tablas Propietarias de n8n:** La cola se gestiona en PostgreSQL o en una instancia de Redis levantada en el mismo Coolify.
  * **Desacople Real:** La respuesta inmediata `200 OK` y el posterior encolado asíncrono se ejecutan de forma nativa en Node.js sin consumir memoria extra.

---

### E. [`ai-library-queue-api.json`](./ai-library-queue-api.json) — *Motor de Despacho y Gestión de Cola FIFO*
* **Triggers:**
  - `POST /queue-next` (petición de la siguiente tarea disponible).
  - `POST /queue-update` (actualización de estado: `completed`, `failed`, `skipped`).
* **Propósito y Funcionamiento Actual:**
  1. **En `/queue-next`:** Recupera tareas zombis (si llevan más de 1 hora en `processing`, las regresa a `pending`). Busca la tarea más antigua en `pending`, la marca en `processing` y la retorna al worker.
  2. **En `/queue-update`:** Valida y aplica el cambio de estado de la tarea.
* **Mejoras al Migrar a Coolify (Impacto Crítico):**
  * **Prevención Absoluta de Condiciones de Carrera:** En n8n, dos peticiones concurrentes a `/queue-next` pueden tomar la misma tarea porque las Data Tables no soportan bloqueos atómicos.  
    En el microservicio con PostgreSQL se implementa:
    ```sql
    SELECT * FROM ai_library_queue 
    WHERE estado = 'pending' 
    ORDER BY ts ASC 
    LIMIT 1 
    FOR UPDATE SKIP LOCKED;
    ```
    O alternativamente, mediante **BullMQ sobre Redis** (disponible en 1 clic en Coolify).

---

## 3. Arquitectura del Proyecto para Coolify

La aplicación se estructura como un servicio web estándar en **Node.js + TypeScript** empaquetado en un contenedor Docker optimizado:

```text
automation-service/
├── src/
│   ├── server.ts              # Servidor Express o Fastify
│   ├── config/
│   │   └── env.ts             # Validación de variables de entorno con Zod
│   ├── db/
│   │   └── pool.ts            # Pool de conexiones a PostgreSQL
│   ├── middlewares/
│   │   └── auth.ts            # Middlewares de validación de secrets y HMAC
│   ├── routes/
│   │   ├── msd.routes.ts      # Endpoints /deal-event y /attribution-capture
│   │   ├── sophie.routes.ts   # Endpoint /prospecta-lead
│   │   └── queue.routes.ts    # Endpoints /roam-ingest, /queue-next, /queue-update
│   ├── services/
│   │   ├── hubspot.service.ts # Interacción con HubSpot API
│   │   └── queue.service.ts   # Lógica atómica de colas
│   └── utils/
│       └── normalizers.ts     # Hashes SHA-256 y parseo de payloads
├── .env.example
├── Dockerfile                 # Multi-stage build optimizado
├── package.json
└── tsconfig.json
```

---

## 4. Configuración del Dockerfile para Coolify

Coolify compila repositorios automáticamente usando un `Dockerfile` multi-stage. Esto produce una imagen de producción mínima (alrededor de 120 MB en total):

```dockerfile
# Etapa 1: Construcción
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Etapa 2: Entorno de Producción
FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production

COPY package*.json ./
RUN npm ci --only=production

COPY --from=builder /app/dist ./dist

USER node
EXPOSE 3000

CMD ["node", "dist/server.js"]
```

---

## 5. Proceso de Despliegue en Coolify Paso a Paso

El despliegue con Coolify se realiza en 4 pasos sencillos:

```mermaid
flowchart LR
    A["Repositorio GitHub<br>(Push a main)"] -->|Webhook automático| B["Coolify en VPS"]
    B -->|Compila Dockerfile| C["Contenedor Docker<br>(~50 MB RAM)"]
    D["Traefik (Coolify)"] -->|SSL Automático Let's Encrypt| C
    E["Clientes / Webhooks<br>(HubSpot, GTM, Roam)"] -->|HTTPS seguro| D
```

1. **Crear el Repositorio:** Subir el código de `automation-service` a GitHub (público o privado).
2. **Crear la Aplicación en Coolify:**
   - En el panel de Coolify, ir a **Projects** -> **New Resource** -> **Private / Public Repository (GitHub)**.
   - Seleccionar el repositorio y la rama `main`.
   - Tipo de build: `Dockerfile`.
   - Puerto de destino: `3000`.
3. **Asignar Dominio y SSL:**
   - En la sección **Domains**, configurar el subdominio deseado:  
     `https://automations.latinovation.com` (o el dominio configurado en tu DNS).
   - Traefik gestionará el certificado SSL automáticamente.
4. **Configurar Variables de Entorno en Coolify:**
   - En la pestaña **Environment Variables**, definir los secretos sin exponerlos en Git:
     ```env
     PORT=3000
     DATABASE_URL=postgresql://user:password@pooler.supabase.com:6543/postgres
     MSD_DEAL_SECRET=1a8847cc503b42b7cd27d6dbfab74f60b10abfa1b3e78f6ba414b9e4e76df3a8
     MSD_ATTRIBUTION_SECRET=9263794e1e4b3b7183395e490a7732b4653e8f501d760ac67e5ac57760546c01
     PROSPECTA_WEBHOOK_SECRET=tu_secreto_aqui
     HUBSPOT_ACCESS_TOKEN=pat-na1-...
     ROAM_SIGNING_SECRET=tu_secreto_roam
     AI_LIBRARY_QUEUE_TOKEN=kVXgkT3qk6N67eSW
     ```
5. **Hacer clic en "Deploy":** Coolify compilará la imagen y el servicio quedará activo inmediatamente.

---

## 6. Integración de Nuevas Automatizaciones en Coolify

Para seguir sumando flujos sin depender de n8n:

* **Nuevos Webhooks:** Basta con crear un nuevo archivo de ruta (ej. `src/routes/stripe.routes.ts`), registrarlo en `server.ts` y hacer `git push`. Coolify redespliega en segundos.
* **Tareas Programadas (Cron Jobs):**
  - **Opción A (En código):** Usar `node-cron` dentro del mismo microservicio.
  - **Opción B (Nativo de Coolify):** Coolify incluye una función nativa de **Cron Jobs** por servicio donde se pueden ejecutar comandos programados o peticiones internas a la API.
* **Procesos con Redis / BullMQ:** Si el volumen de tareas de la cola crece, se puede crear un servicio Redis en Coolify (1 clic) y conectarlo mediante el nombre del contenedor interno de Docker (`redis:6379`), sin costos de red externa.

---

## 7. Comparativa Final: n8n vs. Microservicio en Coolify

| Criterio | n8n en Docker | Microservicio en Coolify |
| :--- | :--- | :--- |
| **Consumo de Memoria** | 500 MB – 1.5 GB RAM | **40 – 80 MB RAM** |
| **Tiempo de Respuesta** | 150 – 400 ms | **5 – 20 ms** |
| **Despliegues** | Manuales vía UI / Importación JSON | **Automáticos con `git push` (GitOps)** |
| **Manejo de SSL y Dominios** | Requiere configuración externa | **Nativo y automático con Traefik** |
| **Concurrencia en Colas** | Susceptible a condiciones de carrera | **Bloqueos atómicos seguros (`SKIP LOCKED`)** |
| **Gestión de Secretos** | Hardcoded en nodos de código | **Panel cifrado de variables en Coolify** |
| **Escalabilidad** | Limitada a nodos soportados | **Ilimitada (librerías npm, Python, IA)** |
