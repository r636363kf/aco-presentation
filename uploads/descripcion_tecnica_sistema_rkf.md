# Sistema RKF / Web Sales — Descripción técnica

## 1. Stack

### Backend
- **Lenguaje**: Python **3.13** (entorno pyenv `rkfsystem_p313`).
- **Framework web**: **Django 5.2.7** (monolito modular).
- **ORM**: Django ORM, multi-database router.
- **Asíncrono**: **Celery 5.5** + RabbitMQ (broker) / Redis (results), `django-celery-results`.
- **Real-time**: **Django Channels 4.x** sobre **Daphne** (ASGI), `channels_redis` como capa.
- **Servidor de aplicaciones**: uWSGI (WSGI) + Daphne (ASGI/WebSockets), corriendo en paralelo.
- **Cache**: Redis (`django-redis`), 3 DBs lógicas separadas.

### Bases de datos
- **PostgreSQL** principal (`rkfbits`, default).
- Múltiples DBs adicionales en el mismo cluster:
  - `acosales` — BI / cubos de ventas.
  - `manag_cubes` — cubos administrativos.
  - `mysbits`, `ibopebits`, `ciesbits`, `wardbits` — bases por empresa del grupo.
  - `tracking_gps` — GPS de repartos.
  - `rtdcbits` — sistema externo RTDC.
- Acceso adicional vía driver a **HyperFileSQL (WinDev/Clarion)** para sistemas legacy.

### Frontend
- Templates Django renderizados server-side.
- **jQuery** + **Bootstrap 5** + **crispy-forms / crispy-bootstrap5**.
- **paramquery (pqGrid)** para grillas de datos (con un patrón propio "semodel" estandarizado en el proyecto).
- **Select2** para combos con búsqueda/populate remoto.
- **axios** para llamadas AJAX a los endpoints dinámicos.
- WebSockets nativos del navegador para canales de Channels.

### Apps móviles
- Apps nativas que consumen una API HTTP propia (ver módulo `api_user/` y métodos `api_*`).
- Sincronización offline/online (los pedidos se cargan en cola local y se sincronizan).
- Notificaciones push vía **Firebase Cloud Messaging** (`firebase-admin`).
- GPS streaming hacia el backend (DB `tracking_gps`).

### Otros componentes
- **openpyxl**, **pandas**, **numpy** — generación de Excel y procesamiento numérico.
- **reportlab** — generación de PDFs.
- **paramiko**, **pysftp** — transferencias SFTP.
- **arrow** — manipulación de fechas (preferido sobre `datetime` directo en código nuevo).
- **django-simple-history** — versionado histórico de modelos críticos.
- **django-hijack** — soporte de impersonación para soporte y debugging.
- **django-extensions**, **bpython** — utilidades de desarrollo.

## 2. Arquitectura

### Topología general

```
                ┌─────────────────────────────┐
                │  Apps móviles (vendedores,  │
                │  reposi­toras, choferes)    │  HTTPS / FCM
                └──────────────┬──────────────┘
                               │
┌────────────────────┐         │         ┌─────────────────────┐
│  Browsers backoffice│        │         │   WhatsApp / SET    │
│  (Bootstrap5 + jq) │         │         │   /  Firebase / BI  │
└──────────┬─────────┘         │         └─────────┬───────────┘
           │ HTTPS / WS        │                   │
           ▼                   ▼                   ▼
   ┌──────────────────────────────────────────────────────┐
   │              Frontend (Nginx) + TLS                  │
   └────────────┬─────────────────────────────────────────┘
                │
     ┌──────────┴──────────┐
     │                     │
     ▼                     ▼
 ┌──────────┐         ┌──────────┐
 │  uWSGI   │         │  Daphne  │  ASGI (Channels / WS)
 │ (Django) │         │  (ASGI)  │
 └────┬─────┘         └────┬─────┘
      │                    │
      └─────────┬──────────┘
                │
                ▼
      ┌─────────────────────┐
      │  Django app monolito │
      │  (web_sales project) │
      └────┬─────────┬──────┘
           │         │
           │         └──────────────┐
           ▼                        ▼
   ┌────────────────┐      ┌─────────────────┐
   │  PostgreSQL    │      │   Redis         │
   │  (rkfbits +    │      │   (cache +      │
   │   N DBs)       │      │   channels +    │
   └────────────────┘      │   celery brkr)  │
                            └─────────────────┘
                                    │
                                    ▼
                            ┌─────────────────┐
                            │  Celery workers │
                            │  + beat         │
                            └─────────────────┘
```

### Apps Django registradas (`INSTALLED_APPS`)

Apps de negocio:
- `dashboard` — login, navegación, autenticación.
- `sales_man` — ventas, circuitos, cartera, briefs, OSA, acuerdos.
- `finance_man` — cuentas, pagos, plan de cuentas.
- `warehouse_man` — bodegas, stock, repartos.
- `invoicing_man` — facturación / KUDE.
- `etrans` — facturación electrónica SIFEN.
- `buy_man` — compras / despachos.
- `imp_man` — importaciones, aduana, impresoras.
- `apps_man` — gestión de apps móviles.
- `repartos_tracking` — tracking de entregas.
- `clarion_man` — puente a sistemas Clarion.
- `stats_metrics` — cubos / KPIs / fillrate.
- `rrhh_man` — RRHH.
- `external_rtdc` — integración RTDC.
- `api_user` — endpoints API para apps móviles.
- `claude_integration` — integración con Anthropic API (workflows internos).
- `misc` — utilitarios cross-módulo.

Apps de terceros relevantes: `daphne`, `channels`, `django_celery_results`, `simple_history`, `corsheaders`, `dal` / `dal_select2` (autocomplete), `crispy_forms` / `crispy_bootstrap5`, `django_tables2`, `sorl.thumbnail`, `django_extensions`.

### Estructura típica de un módulo de negocio

```
sales_man/
├── models.py                  # Modelos Django (campos de estado estándar)
├── admin.py                   # Admin Django
├── urls.py / views.py         # Vistas tradicionales
├── forms.py / formularios.py  # Forms (incluyendo wrappers propios)
├── tables.py                  # Tablas django-tables2
├── tasks.py / *_ctask.py      # Celery tasks
├── consumers.py / routing.py  # WebSocket consumers (Channels)
├── autocomplete_light_registry.py  # Autocompletes DAL
├── sales_circuito.py          # Lógica de dominio (clases con métodos
│                              #   invocables vía 'em' / 'execute_module')
├── sales_control.py           # Otras clases de dominio
├── sales_*.py                 # Submódulos por área (acuerdo, brief, etc.)
└── migrations/
```

Patrón clave: **lógica de negocio en clases regulares de Python** (`SalesControl`, `SalesCircuito`, `MDespacho`, etc.) cuyos métodos se invocan dinámicamente por nombre desde el frontend.

### Endpoints dinámicos (`em` / `execute_module` / `operation_record`)

El frontend no llama a URLs específicas: llama a un puñado de endpoints universales que reciben en el `POST` el módulo, paquete, clase y método a ejecutar, y los argumentos.

```
POST /em/
  module=sales_man
  package=sales_control
  attr=SalesControl
  mname=export_rutero_input
  fecha_desde=2026-04-22
  fecha_hasta=2026-04-26
  ...
```

El dispatcher (`DExecution.execute_module`) importa dinámicamente, instancia la clase, llama al método pasando `query_dict=request.POST` (o `kwargs`), serializa el retorno (un `dict`) a JSON y lo devuelve. Está envuelto en un decorador `track_error` que captura excepciones y las reporta a Sentry/Glitch — por eso el código de negocio **no** debe usar `try/except` defensivo.

Variantes:
- `em` — endpoint moderno, espera `query_dict`.
- `execute_module` — endpoint legacy, mismo patrón con otra firma.
- `operation_record` — orientado a operaciones que generan registros con auditoría.

### Modelos: convenciones

Todos los modelos transaccionales heredan implícitamente el patrón:

```python
cargado_010      = BooleanField     # registrado
cargado_010_por_gecos / _fecha / _hora
anulado_040      = BooleanField     # baja lógica
anulado_040_por_gecos / _fecha / _hora / _motivo
verificado_045   = BooleanField     # verificado
aprobado_050     = BooleanField     # aprobado
procesado_060    = BooleanField     # procesado
```

Las consultas filtran típicamente por `anulado_040=False, aprobado_050=True`. Esto reemplaza un soft-delete formal y permite trazar quién hizo qué en cada estado.

### Multi-database

Los modelos se asignan a la DB correcta vía `using('acosales')`, `using('mysbits')`, etc. Hay módulos enteros (`stats_metrics`, `repartos_tracking`, `external_rtdc`) que escriben a DBs distintas. No hay `DATABASE_ROUTERS` automáticos: la asignación es explícita por método.

### Procesamiento asíncrono

- **Celery** corre tareas largas: cubos (`stats_metrics`), generación de fillrate, integraciones SIFEN, push notifications.
- **Celery beat** dispara tareas periódicas: sync de attendance, recálculos nocturnos, mantenimientos.
- Resultados se guardan en `django-db` (tabla `django_celery_results`).
- Cron del sistema operativo dispara scripts en `bin/` (e.g. `frate_make_source.sh`) que invocan management commands de Django con throttle de paralelismo (patrón `wait -n` con `MAX_JOBS`).

### Real-time (Channels)

- WebSockets para tracking GPS de choferes, notificaciones a backoffice, estado de bloques de carga en bodega, sincronización de visitas.
- Layer en Redis (`channels_redis`) — soporta múltiples instancias Daphne.
- Routing en `web_sales/routing.py` + `routing.py` por app.

### Facturación electrónica (etrans)

- Generación de XML conforme al esquema del **SIFEN** (Paraguay).
- **CDC** (Código de Control) calculado per-documento.
- Firma con certificado digital del emisor.
- Envío SOAP a la SET con consulta de estado y eventos (cancelación, inutilización).
- Formato **KUDE** generado para el cliente final.
- Cola de reintentos vía Celery cuando el SIFEN está caído.

### Integraciones legacy

- **Clarion / HyperFile (WinDev)**: módulos `clarion_man` y `claude_integration`/`claude_bi` mantienen puentes para leer/escribir hacia bases legacy. Se usa `HFSAccess` con archivos `.ini` por empresa (ACO, etc.). Datos típicos: órdenes de pago pendientes, retenciones, archivos bancarios generados.

### Autenticación y permisos

- `django.contrib.auth` con `User` y un `UserProfile` extendido (`puesto`, `gecos`, `imei`, `espersonalactivo`, etc.).
- Permisos por módulo manejados con tags propios (`{% if request.user|user_has_role:"DIRECTOR|SUPERVISOR_DE_BODEGAS" %}`).
- Auditoría in-row en cada modelo (`*_por_gecos`).
- `django-simple-history` para tablas críticas.
- `django-hijack` para soporte (impersonación).

## 3. Patrones de código que aplica el equipo

### Frontend (templates)
- Grillas vía **paramquery semodel** estandarizado (`fake:true`, `compute:true`, `method:true` para tipar columnas — ver `.claude/paramquery_semodel_estandar.md`).
- Combos: `vform.populate.se_select` (≤500 registros) o `vform.search.se_select` (>500).
- Submits con `axios.post('{% url "em" %}', fdata)` siguiendo `.claude/llamadas_axios.md`.
- Estética uniforme con `.claude/estetica_formulario.md` (cards, gaps, tamaños de fuente).

### Backend
- **No `try/except`** en métodos invocados vía `em`/`operation_record`/`execute_module` — el decorador `track_error` los reporta a Sentry y traduce el error al frontend automáticamente.
- **Imports al tope del archivo**, nunca dentro de métodos (excepto helpers privados muy específicos).
- `select_related` / `prefetch_related` agresivamente, `values_list(..., flat=True)` para lookups.
- `arrow` para fechas en código nuevo; `datetime` solo donde el legacy lo exige.
- `Decimal` explícito en cálculos monetarios; nunca `float`.
- Para queries grandes: `.iterator(chunk_size=N)` con `tqdm` para no cargar todo en memoria.

### Performance
- Optimizaciones probadas en producción incluyen:
  - Reescribir métodos ABM móviles para evitar serialización pesada cuando solo se necesitan campos planos.
  - Bulk prefetch + matching en memoria en vez de N queries por iteración.
  - Cubos pre-generados nocturnos (`gen_fillrate`) con paralelismo controlado vía bash + `wait -n`.
  - Cache de queries idempotentes en Redis.

## 4. Despliegue y operación

- **Servidores**:
  - `192.168.4.221` (cloud.aconcagua.com.py) — producción principal.
  - `192.168.1.10` (netipa) — servidor de procesamiento batch (cubos, fillrate).
  - `192.168.1.14` — servidor de Postgres con BI (Superset).
- **Frontend HTTP**: Nginx con TLS.
- **App servers**: uWSGI (WSGI) + Daphne (ASGI), supervisor o systemd.
- **Acceso remoto**: SSH a `cloud.aconcagua.com.py` puerto 10148 (DNAT vía nftables a 192.168.4.221:22).
- **Backups**: `pg_dump -Fd -j 12 -Z3` con rsync hacia equipos locales para desarrollo (script `bin/update_local_database_acosales.sh`).
- **Cron**: scripts en `stats_metrics/bin/` y `bin/` invocan management commands de Django para tareas batch.

## 5. Configuración relevante

- **Settings**: `web_sales/settings.py` (común) + `web_sales/sts_local.py` (secrets, no commiteado) + `web_sales/sts_produccion.py` / `sts_failover.py` (overrides por entorno).
- **Variables de entorno** vía helper `env(...)` propio (con default, casteo opcional).
- `MEDIA_ROOT = /var/www/html/webapps/media/` — archivos generados (Excel, PDF, KUDE, fotos).
- `STATIC_ROOT = /var/www/html/webapps/nstatic/`.
- `EMAIL_HOST = mail.aconcagua.com.py` (SSL puerto 465).
- `BROKER_URL` Celery: AMQP por defecto (RabbitMQ); resultados en `django-db`.

## 6. Métricas del proyecto

- ~60.000+ líneas de Python.
- 16 apps Django de negocio.
- 372 paquetes Python en `requirements.txt`.
- 9 bases PostgreSQL declaradas.
- 5 empresas del grupo operadas en la misma plataforma.
- 10+ años en producción con desarrollo continuo.
