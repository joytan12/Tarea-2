# Issue #1: Modelado y despliegue dockerizado de Base de Datos para FlowEx (PMV 1.1)

- **Estado**: Abierta
- **Enlace GitHub**: [https://github.com/joytan12/Tarea-2/issues/1](https://github.com/joytan12/Tarea-2/issues/1)
- **Módulo**: Base de Datos (`db/`)
- **Etiquetas**: `enhancement`, `database`, `docker`, `spec-pmv`

---

## 📌 Descripción del Requerimiento
Como primer paso en el desarrollo de la plataforma logística **FlowEx (PMV 1.1)**, se requiere diseñar, estructurar e inicializar una base de datos relacional robusta que cubra de manera integral los **14 requerimientos funcionales** definidos en la especificación ([spec/requerimientos.md](../requerimientos.md)).

La base de datos debe estar **100% dockerizada**, permitiendo a cualquier desarrollador del equipo levantar el entorno de datos completo con un único comando (`docker compose up -d`), con persistencia garantizada y datos semilla (*seeds*) listos para pruebas.

---

## 🎯 Objetivos de la Tarea
1. Diseñar el modelo entidad-relación (MER) relacional para **PostgreSQL 16**.
2. Garantizar que todas las entidades, relaciones, estados y reglas de negocio de los 14 requerimientos del PMV queden modelados.
3. Crear los scripts DDL de estructura (`01_schema.sql`) y DML de datos iniciales (`02_seed.sql`).
4. Configurar el entorno en contenedores con `docker-compose.yml`, archivo `.env.example` y volumen de persistencia.
5. Documentar la estructura y comandos de operación en `db/README.md`.

---

## 📋 Alcance del Modelo de Datos (Cobertura de Requerimientos)

### 1. Seguridad, Usuarios y Roles (`RF-01`)
- Tabla `users`: Identificador, correo electrónico, hash de contraseña, nombre completo, rol (`cliente`, `admin`, `driver`), teléfono, estado activo y marcas de tiempo.
- Tabla `roles` / Enum de roles: Control de acceso para Cliente, Admin/Coordinador, Conductor y Sistema.

### 2. Clientes Corporativos y Origen de Pedidos (`RF-02`)
- Tabla `customers`: Datos de empresa/cliente (Razón Social, RUT, contacto principal, email corporativo).
- Tabla `customer_addresses`: Libreta de direcciones guardadas de retiro en bodega/sucursal (dirección, comuna, notas de acceso).
- Discriminación de canal de origen: Flag o campo `entered_by` (`cliente` directo vs `vendedor` B2B con identificación del vendedor asignado).

### 3. Zonificación, Comunas y Bloqueo de Cobertura (`RF-04`)
- Tabla `zones`: Identificador de zona tarifaria (Z-1 Santiago Centro-Oriente, Z-2 Santiago Poniente/Sur, Z-3 Costa Viña/Valparaíso, Z-4 Sur Concepción, Z-5 Sur Temuco).
- Tabla `communes`: Listado de comunas con nombre, región, zona asignada y bandera booleana `has_coverage`.
  - Debe incluir comunas bloqueadas con `has_coverage = false` (Punta Arenas, Coyhaique, Isla de Pascua, Putre).

### 4. Flota, Conductores y Centros de Distribución (`RF-06`, `RF-08`, `RF-13`, `RF-14`)
- Tabla `distribution_hubs`: Centros de distribución física (ej. Hub Central Pudahuel, dirección, comuna, capacidad operativa).
- Tabla `vehicles`: Vehículos de reparto (patente, modelo, marca, tipo de vehículo).
- Tabla `drivers`: Perfil del repartidor vinculado a `users`, vehículo asignado, zona logística preferente y teléfono de contacto en terreno.

### 5. Pedidos, Envíos y Cotización Inteligente (`RF-03`, `RF-05`, `RF-07`, `RF-10`)
- Tabla `orders` / `shipments`:
  - `tracking_number`: Identificador único visible (formato `FX-XXXX-XXXX-CL`).
  - `delivery_code`: Código PIN seguro de 4-6 dígitos alfanuméricos (`FLW-XXXX`) para validación de entrega en terreno.
  - Datos de Remitente: Nombre, teléfono, dirección, comuna de retiro.
  - Datos de Destinatario: Nombre, teléfono, correo electrónico, dirección, comuna de entrega y zona calculada.
  - Carga: `packages_count` (1-50), `package_type` (Caja, Sobre, Pallet), `weight_kg`, `declared_value`.
  - Desglose Financiero: `base_cost` ($4.500), `packages_cost` ($800/bulto), `shipping_type` (`normal`, `express`, `same_day`), `shipping_type_cost`, `insurance_cost` (1% excedente $50k), `discount_amount`, `total_cost`.
  - Estado Logístico: Enum con los 8 hitos del ciclo de vida:
    - `pending` (Pendiente Pago)
    - `paid` (Pagado)
    - `pickup_assigned` (Recogida Asignada)
    - `picked_up` (Recogido en Origen con Foto)
    - `in_hub` (En Hub CD Pudahuel)
    - `transit` (En Reparto Final)
    - `delivered` (Entregado con PIN)
    - `incident` (Incidencia / Entrega Fallida)
  - Transacción de Pago: `is_paid`, `payment_method` (`webpay`, `transfer`), `payment_transaction_id` (`TX-XXXXX`), `paid_at`.
  - Operación Terreno: `pickup_driver_id`, `picked_up_at`, `pickup_photo_url`, `pickup_notes`, `assigned_driver_id`, `delivered_at`, `incident_reason`.

### 6. Sistema de Cupones y Promociones (`RF-12`)
- Tabla `promo_codes`: Códigos (`FLOW10`, `DESCUENTO20`, `BIENVENIDA5000`, `ENVIOFREE`), tipo de descuento (`percentage`, `fixed`, `free_shipping`), valor, fecha límite de vigencia, usos máximos y estado activo.
- Relación de cupón aplicado en cada pedido.

### 7. Auditoría Inmutable (Audit Log) (`RF-09`, `RF-10`)
- Tabla `event_logs`: Bitácora inmutable *append-only* vinculada a cada envío:
  - `id`: Identificador de evento (`EV-XXXX`).
  - `order_id`: Clave foránea al pedido.
  - `timestamp`: Marca de tiempo precisa.
  - `user_id` / `user_email`: Usuario responsable.
  - `role`: Rol del ejecutor (`customer`, `admin`, `driver`, `sistema`).
  - `action`: Hito realizado (ej. "Pedido Creado", "Recogida Realizada en Origen", "Recepción en Hub CD Pudahuel", "En Reparto Final", "Entregado").
  - `details`: Descripción textual de contexto, URLs de evidencia fotográfica o incidencias.

### 8. Registro de Notificaciones (`RF-11`)
- Tabla `notifications_log`: Historial de notificaciones emitidas por correo electrónico o WhatsApp Web API (`order_created`, `in_transit`, `delivered`, `incident`) con acuse de envío.

---

## 🐳 Requerimientos de Dockerización
La solución dentro de la carpeta `db/` debe contener:
- [ ] `docker-compose.yml`:
  - Servicio PostgreSQL 16 (imagen oficial `postgres:16-alpine`).
  - Nombre de contenedor estándar (`flowex-db`).
  - Configuración mediante variables de entorno.
  - Mapeo de puerto estándar `5432:5432`.
  - Volumen persistente de datos (`flowex_pgdata`).
  - Montaje automático de scripts en `/docker-entrypoint-initdb.d/`.
  - Healthcheck configurado con `pg_isready`.
- [ ] `.env.example`: Con variables preconfiguradas para desarrollo local (`POSTGRES_DB=flowex_db`, `POSTGRES_USER=flowex_admin`, `POSTGRES_PASSWORD=flowex_secret_pass`, `POSTGRES_PORT=5432`).
- [ ] Scripts SQL:
  - `db/init/01_schema.sql`: DDL de tablas, tipos enumerados, índices en claves foráneas y números de guía (`tracking_number`).
  - `db/init/02_seed.sql`: Inserción de catálogo de comunas/zonas, cupones de prueba, clientes base, conductores y envíos demo idénticos a los del frontend.
- [ ] `db/README.md`: Guía de inicio rápido con comandos de ejecución, parada y conexión.

---

## ✅ Criterios de Aceptación (Definition of Done)
1. **Despliegue Limpio**: Ejecutar `docker compose up -d` en `db/` levanta el servicio y pasa el healthcheck a estado *healthy*.
2. **Esquema Íntegro**: Todas las tablas, claves primarias, foráneas, restricciones de unicidad (`tracking_number`, `delivery_code`) y tipos enumerados se crean automáticamente.
3. **Poblado de Datos Inicial**: Al conectar a la base de datos se comprueba la presencia de zonas, comunas cubiertas y no cubiertas, y los cupones activos acordados.
4. **Trazabilidad Inmutable**: La tabla `event_logs` permite almacenar el historial cronológico completo de cualquier orden.
