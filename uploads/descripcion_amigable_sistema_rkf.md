# Sistema RKF / Web Sales — Descripción amigable

## ¿Qué es?

**Web Sales** (también llamado RKF / "rkfbits" internamente) es el ERP propietario del **Grupo Aconcagua**, una distribuidora mayorista que opera en Paraguay. Es un sistema de producción con más de 10 años de evolución continua que conecta toda la cadena del negocio: desde que un vendedor sale a la calle con su celular hasta que la mercadería llega al punto de venta y la factura electrónica se emite a la SET.

No es un producto enlatado: está hecho a la medida de las operaciones de Aconcagua y de las empresas del grupo (Aconcagua, DyT, MyS, IBO, CIES).

## ¿Qué problema resuelve?

Un distribuidor mayorista en Paraguay necesita coordinar muchas piezas a la vez:

- **Vendedores en ruta** que visitan cientos de farmacias, despensas, mayoristas y supermercados cada día.
- **Pedidos** que se levantan en la calle, muchas veces sin internet, y deben llegar al backoffice en tiempo y forma.
- **Stock** distribuido en varias bodegas, entre pallets cerrados y unidades sueltas.
- **Repartos** con camiones cuya capacidad y ruta hay que optimizar.
- **Facturación electrónica** obligatoria por ley (SET / SIFEN), con firma digital y CDC por cada documento.
- **Cuentas por cobrar y pagar**, créditos, cobranzas y conciliaciones bancarias.
- **Compras** al exterior y al mercado local, con despachos de aduana.
- **Acuerdos comerciales** con cadenas y proveedores (góndolas, exhibiciones, OSA).
- **Tareas en punto de venta**: reposición, control de vencimientos, fotos de exhibición, relevamientos.

Web Sales es el cerebro que une todo eso en un solo flujo.

## ¿Qué ofrece, módulo por módulo?

### 🛒 Ventas (sales_man)
- Catálogo de productos con precios por canal y precios negociados por cliente.
- Cartera de clientes y puntos de venta.
- **Circuitos** y rutas de vendedores y reposi­toras: planificación semanal, diaria, quincenal, mensual e intercalada.
- Toma de pedidos en app móvil con sincronización online/offline.
- **Tareas en PDV**: levantamiento de acuerdos, OSA (On Shelf Availability), control de vencimientos, fotos georreferenciadas.
- Briefs comerciales, comparativas de premios y comisiones.
- Análisis de ventas por zona / vendedor / producto / canal.

### 💰 Finanzas (finance_man)
- Cuentas corrientes de clientes y proveedores.
- Documentos: facturas, notas de crédito y débito, recibos.
- Pagos y egresos a proveedores con control de cuentas bancarias.
- Conciliación bancaria.
- Plan de cuentas y centros de costo.
- Bloqueos de venta por mora.

### 📦 Almacén (warehouse_man)
- Múltiples bodegas con áreas y celdas físicas.
- Stock dual: pallets cerrados + unidades sueltas.
- Ajustes de inventario, movimientos entre bodegas, comprobaciones de stock.
- **Bloques de carga** y armado de cajas para reparto.
- Asignación de boxes a camiones, control de salidas.
- Trazabilidad por lote y vencimiento.

### 🚚 Repartos y tracking (repartos_tracking, transport_man)
- Generación de hojas de reparto.
- Asignación a camiones y choferes.
- Tracking GPS en tiempo real.
- App del chofer para confirmación de entregas.

### 🧾 Facturación (invoicing_man + etrans)
- Emisión de facturas, notas y recibos.
- Generación automática del **CDC** (Código de Control de la SET).
- Firma digital de XML y envío al **SIFEN**.
- Formato **KUDE** (factura electrónica visual).
- Manejo de eventos: cancelaciones, inutilizaciones, consultas de estado.
- Cumplimiento tributario completo Paraguay.

### 🛍️ Compras (buy_man)
- Solicitudes y proformas de compra.
- Aprobaciones y negociaciones con proveedores.
- Recepción y control de mercadería.

### 🛬 Importaciones / Despachos (imp_man)
- Despachos de aduana, puertos, medios de transporte.
- Cálculo de aranceles, IVA renta y tasas de cambio.
- Costos por importación.

### 📊 Analítica e indicadores (stats_metrics)
- Cubos de datos para Superset / BI.
- **Fillrate** (cumplimiento de pedidos) histórico de 5 años.
- KPIs de ventas, visitas, tareas, OSA, vencimientos.
- Métricas de productividad de vendedores.

### 👷 Recursos humanos (rrhh_man)
- Personal, asistencia, justificaciones.
- Integración con marcadores biométricos.

### 🤝 Trade marketing (trade_man)
- Acuerdos comerciales, exhibiciones, espacios negociados.
- Relevamientos en góndola y reportes de ejecución.

## Lo que hace bien

- **Mobile-first real**: la app de ventas funciona aunque el vendedor esté sin señal en una ruta del Chaco; sincroniza cuando vuelve a tener internet.
- **Tiempo real donde importa**: tracking GPS y notificaciones por WebSocket; push a celulares vía Firebase.
- **Multi-empresa**: una sola plataforma corre las 5 empresas del grupo con bases de datos separadas.
- **Cumplimiento tributario completo**: integración nativa con la SET/SIFEN, no es un agregado.
- **Auditoría profunda**: cada registro guarda quién lo cargó, quién lo verificó, quién lo aprobó y cuándo (campos `cargado_010`, `verificado_045`, `aprobado_050`, `anulado_040`, `procesado_060`).
- **Histórico**: cambios en datos críticos quedan versionados con `django-simple-history`.

## Quién lo usa

- **Vendedores y reposi­toras** en la calle, vía app móvil.
- **Choferes** que reciben hojas de reparto en su celular.
- **Operadores de bodega** que arman pedidos, mueven stock y cierran cajas.
- **Backoffice administrativo**: facturación, cobranzas, pagos.
- **Gerencia y trade marketing**: dashboards y reportes de ejecución.
- **Contabilidad**: registros financieros y cumplimiento tributario.
- **Auditoría interna**: revisiones de circuito, OSA, acuerdos.

## Integraciones externas

- **SET / SIFEN** — Sistema de facturación electrónica de Paraguay (SOAP).
- **Firebase Cloud Messaging** — Notificaciones push a apps móviles.
- **WhatsApp Bot** — Notificaciones automatizadas a clientes y personal.
- **Sistemas legacy Clarion / HyperFile (WinDev)** — Convivencia con módulos antiguos durante la migración progresiva.
- **Apache Superset** — Dashboards de BI sobre los cubos generados.
- **Marcadores biométricos** — Asistencia de personal.
- **Bancos** — Archivos de pago y conciliación bancaria.

## Estado actual

- **Producción activa** desde hace más de una década.
- **Branch principal de desarrollo**: `develop`.
- **Stack moderno**: Django 5.2, Python 3.13, PostgreSQL.
- Migración progresiva desde sistemas Clarion/HyperFile heredados.
- Iteraciones recientes en menús nuevos para apps móviles, integración WhatsApp y mejoras en facturación de etiquetados.
