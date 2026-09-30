# Especificación de Requerimientos de Software (ERS)
## Plataforma de Gestión Logística FlowEx (PMV 1.1)

Basado en la plataforma y especificación interactiva desplegada en [https://flowex-front.vercel.app/](https://flowex-front.vercel.app/).

---

## 1. Descripción General del Proyecto

**FlowEx** es una plataforma web para la gestión integral de operaciones logísticas de última milla y distribución regional (PMV 1.1). Cubre el ciclo logístico completo desde la recepción de la solicitud de despacho, el cobro y la recogida en origen con evidencia fotográfica, la consolidación en Centro de Distribución (Hub Central), hasta el despacho final en terreno validado mediante código PIN seguro y trazabilidad pública en tiempo real.

---

## 2. Actores y Roles del Sistema

El sistema define cuatro roles principales de interacción:

1. **Cliente Registrado (`cliente`)**:
   - Empresas (B2B) o personas naturales que solicitan envíos.
   - Disponen de datos persistentes (direcciones frecuentes de retiro, razón social, RUT/teléfono).
   - Consultan pedidos en tránsito, gestionan pagos de envíos pendientes y reciben códigos seguros (PIN).

2. **Administrador / Coordinador de Operaciones (`admin`)**:
   - Operadores de tráfico, despachadores y auditores (ej. Ricardo Barría - Auditor General).
   - Acceso al panel operativo centralizado, zonificación automática, asignación de conductores y consola de control de estados.
   - Capacidad para registrar envíos en nombre de clientes corporativos bajo el rol de "Vendedor / Ejecutivo B2B".

3. **Conductor / Repartidor Terreno (`driver`)**:
   - Conductores de flota (ej. Roberto Gómez - Unidad Sprinter KJL-942, Camila Rojas, Juan Pablo Valenzuela).
   - Disponen de consola móvil liviana con dos fases operativas: **Ruta de Recogida en Orígenes** y **Ruta de Reparto a Destinos**.
   - Capturan fotografías de recepción en origen, cambian estados en terreno y solicitan el PIN seguro al destinatario.

4. **Público / Destinatario Final (Sin autenticación)**:
   - Cualquier persona o cliente que posea un número de guía (`FX-XXXX-XXXX-CL`).
   - Accede al portal público de trazabilidad en vivo, consulta el historial de eventos del paquete y contacta soporte o al repartidor vía WhatsApp.

5. **Sistema (`Sistema`)**:
   - Procesador automático de transacciones de pago (Webpay, Transferencia), motor de cálculo tarifario, generación de rutas y disparador de notificaciones.

---

## 3. Matriz de Requerimientos Funcionales (14 Requerimientos PMV)

A continuación se detalla la especificación de los 14 requerimientos funcionales obligatorios del sistema:

### RF-01: Autenticación con Roles Diferenciados
- **Descripción**: El sistema debe proveer autenticación y control de acceso basado en roles (`cliente`, `admin`, `driver`).
- **Comportamiento**:
  - Al seleccionar o cambiar de rol, el sistema actualiza la sesión activa, los menús de navegación lateral y los privilegios en pantalla.
  - La sesión y rol activo se persisten localmente (`localStorage`).
  - Cada rol visualiza únicamente las vistas y botones pertinentes a su operativa.
- **Criterio de Aceptación**: Los usuarios no autorizados no pueden ejecutar acciones de asignación de conductores ni alteración de auditoría reservadas para administración.

---

### RF-02: Registro y Persistencia de Cliente + Identificador de Origen
- **Descripción**: El sistema debe persistir perfiles de clientes recurrentes e identificar el canal de origen del pedido.
- **Detalle de Datos**:
  - Perfil del cliente: ID cliente (`CUST-001`), Razón Social / Nombre (`Importaciones Santiago S.A.`), correo electrónico, teléfono y libreta de direcciones frecuentes (ej. Sucursal Providencia, Bodega Central Pudahuel).
  - Selector de origen del pedido: Permite conmutar entre **Cliente Directo** (`enteredBy: 'cliente'`) y **Vendedor / Ejecutivo B2B** (`enteredBy: 'vendedor'`, registrando el nombre del ejecutivo responsable).
- **Criterio de Aceptación**: En cada nuevo pedido, las direcciones y remitentes precargados se completan automáticamente según el perfil del cliente en sesión.

---

### RF-03: Ingreso de Pedido (Simple y Masivo) & Cálculo de Seguro
- **Descripción**: Formulario dinámico para la cotización y registro de envíos individuales o en lote con cálculo automático de flete y póliza de seguro.
- **Campos del Formulario**:
  - **Destinatario**: Nombre completo, correo electrónico, teléfono y dirección física.
  - **Comuna de Destino**: Selector con validación reactiva de cobertura territorial.
  - **Carga**: Cantidad de bultos (1 a 50), tipo de paquete (caja chica/mediana/grande, sobre seguro, pallet), peso aproximado.
  - **Tipo de Envío**: Normal Standard (48h), Express Priority (24h), Mismo Día / Same Day.
  - **Valor Declarado ($ CLP)**: Valor monetario de las mercancías contenidas.
  - **Seguro Obligatorio**: Cálculo automático de seguro de carga.
    - Tasa base: `$1,000 CLP`.
    - Si el valor declarado supera `$50,000 CLP`, se añade el **1% sobre el monto excedente**:
      $$\text{Seguro} = 1000 + \max(0, (\text{Valor Declarado} - 50000) \times 0.01)$$
  - **Modalidad Masiva (Batch)**: Permite ingresar una lista con múltiples destinatarios en una única transacción consolidada.
- **Criterio de Aceptación**: La interfaz muestra en tiempo real el desglose de cotización (subtotal flete, recargo por tipo de servicio, seguro y descuentos).

---

### RF-04: Zonificación Automática por Comuna y Bloqueo de Cobertura
- **Descripción**: Motor de zonificación logística que categoriza instantáneamente la comuna seleccionada y restringe zonas sin cobertura.
- **Matriz de Cobertura y Zonas**:
  - **Zona Santiago Centro / Oriente (Z-1)**: Santiago, Providencia, Las Condes, Ñuñoa, Vitacura, La Reina.
  - **Zona Santiago Poniente / Sur / Norte (Z-2)**: Maipú, Pudahuel, San Bernardo, La Florida, Quilicura.
  - **Zona Costa (Z-3)**: Viña del Mar, Valparaíso, Concón.
  - **Zona Sur Concepción (Z-4)**: Concepción, Talcahuano, San Pedro de la Paz.
  - **Zona Sur Temuco (Z-5)**: Temuco, Padre Las Casas.
  - **Zonas Bloqueadas (Sin Cobertura)**: Punta Arenas, Coyhaique, Isla de Pascua, Putre.
- **Criterio de Aceptación**: Si el usuario elige una comuna sin cobertura, se deshabilita el botón de pago y se muestra una alerta visual informando la indisponibilidad de despacho.

---

### RF-05: Pasarela de Pago Integrada y Gestión de Envíos Pendientes
- **Descripción**: Módulo de recaudación embebido con soporte para pagos inmediatos o diferidos.
- **Funcionalidades**:
  - Métodos disponibles: **Webpay Plus** (Tarjetas de Débito/Crédito) y **Transferencia Bancaria Directa**.
  - Si el pago es exitoso: El pedido pasa a estado `paid` (Pagado) y genera un identificador de transacción (`TX-WEBPAY-XXXXX`).
  - Si el pago no se completa inmediatamente: El pedido se almacena con estado `pending` (Pendiente de Pago).
  - En la sección **Mis Envíos**, los pedidos se dividen en dos pestañas: *Pendientes por Pagar* (con botón directo "Pagar Ahora") y *Pagados & En Tránsito*.
- **Criterio de Aceptación**: Un pedido no puede ingresar a la ruta de recogida ni de despacho si su estado de pago es `pending`, debiendo liquidarse previamente.

---

### RF-06: Generación Automática de Ruta del Conductor
- **Descripción**: Consola de despacho que consolida y secuencia los envíos asignados a cada conductor para su jornada de trabajo.
- **Criterios de Agrupación**:
  - Filtro exclusivo por el identificador del conductor (`DRV-01`, `DRV-02`, `DRV-03`).
  - Agrupación de paquetes en dos fases:
    1. **Ruta de Recogida (Origins)**: Retiros en instalaciones del remitente.
    2. **Ruta de Reparto (Destinations)**: Entregas finales domiciliarias.
  - Estimación de paradas, volumen de bultos y distancia estimada en kilómetros.
- **Criterio de Aceptación**: El conductor puede consultar su itinerario con dirección, teléfono de contacto y secuencia optimizada de paradas.

---

### RF-07: Operaciones Terreno & Cambio de Estado Móvil con PIN de Seguridad
- **Descripción**: Interfaz móvil optimizada para conductores que permite registrar hitos de entrega en tiempo real sin recargar mapas pesados.
- **Funcionalidades**:
  - Botones de acción directa: "Recogido en Origen", "En Hub CD", "En Reparto Final", "Entregado", "Incidencia / Entrega Fallida".
  - **Captura de Foto de Recogida**: Al retirar en origen, el driver sube evidencia fotográfica del estado de los bultos junto a observaciones.
  - **Validación de Código Seguro PIN**: Para marcar un paquete como "Entregado", el conductor debe solicitar al receptor su código PIN secreto (`FLW-XXXX`).
  - Acceso directo para contactar al cliente o destinatario por llamada o WhatsApp.
- **Criterio de Aceptación**: No se permite la entrega sin el registro del código PIN o justificación de incidencia en caso de entrega fallida.

---

### RF-08: Panel de Administración Operativo Centralizado
- **Descripción**: Panel de control limpio y ágil para el equipo de operaciones sin dashboards analíticos redundantes.
- **Características**:
  - Tabla operativa de pedidos con vista de Tracking, Remitente, Destinatario, Zonificación Automática, Estado de Pago, Driver Asignado y Estado Logístico.
  - Filtros directos por estado del pedido (Todos, Pendiente Pago, Pagado, Recogido, En Hub, En Reparto, Entregado, Incidencia).
  - Filtros por estado financiero (Todos, Solo Pagados, Solo Pendientes).
  - Selectores en línea para asignación o reasignación rápida de conductores por zona.
- **Criterio de Aceptación**: Las asignaciones y cambios de estado ejecutados en la tabla impactan de inmediato en la bitácora y en la vista del conductor.

---

### RF-09: Registro Inmutable de Eventos (Audit Log)
- **Descripción**: Cada acción o cambio de estado realizado en el sistema debe registrarse permanentemente en una bitácora vinculada al paquete.
- **Estructura del Registro de Evento**:
  - `id`: Identificador único del evento (`EV-XXXX`).
  - `timestamp`: Fecha y hora exacta de la acción.
  - `user`: Identificador o correo del usuario que ejecutó la acción.
  - `role`: Rol del ejecutor (`customer`, `admin`, `driver`, `Sistema`).
  - `action`: Nombre del hito o evento (ej. "Pedido Creado", "Pago Confirmado", "Recogida Realizada en Origen", "Recepción en Hub CD Pudahuel", "Salida del Hub & Asignación Ruta Final", "En Reparto Final", "Entregado").
  - `details`: Descripción detallada de las circunstancias, fotos asociadas, descuentos o notas.
- **Criterio de Aceptación**: El historial de auditoría no puede ser editado ni eliminado por ningún usuario.

---

### RF-10: Vista Externa Pública de Seguimiento (Trazabilidad en Vivo)
- **Descripción**: Portal público accesible para destinatarios finales mediante el número de guía (`FX-XXXX-XXXX-CL`) sin necesidad de inicio de sesión.
- **Datos Visibles**:
  - Estado actual y barra de progreso de entrega.
  - Código PIN de seguridad (`FLW-XXXX`) para que el destinatario se lo entregue al conductor.
  - Datos de origen, destino, bultos y seguro cubierto.
  - Datos del conductor y vehículo asignado (ej. Roberto Gómez - Sprinter KJL-942).
  - Foto de recogida en origen.
  - Línea de tiempo completa basada estrictamente en el **Log de Eventos (RF-09)**.
- **Criterio de Aceptación**: La página de tracking público debe reflejar de manera inmediata cualquier cambio realizado por el conductor o la administración.

---

### RF-11: Notificaciones Automáticas Multicanal (Email & WhatsApp)
- **Descripción**: Generación y despacho de notificaciones automáticas ante eventos clave en la vida del envío.
- **Canales y Disparadores**:
  - **Eventos Gatillantes**:
    1. `order_created`: Creación y confirmación del pedido (envía número de guía y PIN seguro).
    2. `in_transit`: Salida a reparto final (informa conductor en camino).
    3. `delivered`: Entrega exitosa.
    4. `incident`: Ocurrencia de fallo o entrega no lograda.
  - **Correo Electrónico**: Notificaciones estructuradas con remitente, asunto y cuerpo descriptivo.
  - **WhatsApp**: Enlace directo preformateado con API de WhatsApp (`https://api.whatsapp.com/send?phone=...&text=...`) conteniendo el mensaje y URL directa de tracking.
- **Criterio de Aceptación**: Cada notificación enviada queda indexada en el historial del paquete con marca de tiempo.

---

### RF-12: Sistema de Cupones y Códigos Promocionales
- **Descripción**: Módulo de promociones y descuentos aplicables en el formulario de cotización y en el checkout.
- **Catálogo de Códigos y Reglas de Negocio**:
  | Código Promo | Tipo de Descuento | Valor / Regla | Descripción |
  |---|---|---|---|
  | `FLOW10` | Porcentual | 10% | 10% de descuento en el costo total del envío |
  | `DESCUENTO20` | Porcentual | 20% | 20% de descuento promocional sobre el total |
  | `BIENVENIDA5000` | Monto Fijo | $5.000 CLP | Descuento directo de $5.000 CLP |
  | `ENVIOFREE` | Flete Base Gratis | $4.500 CLP | Bonificación del 100% de la tarifa base ($4.500 CLP) |
- **Reglas**:
  - Validación insensible a mayúsculas/minúsculas.
  - El monto descontado no puede superar el valor total del envío (el precio resultante nunca será negativo).
- **Criterio de Aceptación**: Al aplicar un cupón válido se recalcula la cotización en vivo y se detalla el descuento en el resumen y en el log de auditoría.

---

### RF-13: Ciclo Logístico Integrado (Recogida $\rightarrow$ Hub CD $\rightarrow$ Reparto)
- **Descripción**: Soporte para la cadena logística completa desde el origen hasta el consumidor final.
- **Etapas Obligatorias**:
  1. **Generación**: Cliente registra el pedido y paga la tarifa.
  2. **Asignación de Recogida**: Se asigna conductor para retiro en bodega del cliente.
  3. **Recogida en Origen**: Driver retira el paquete, toma fotografía y sube confirmación.
  4. **Ingreso a Hub CD**: Paquete ingresa al Centro de Distribución Pudahuel (`in_hub`), donde se clasifica y etiqueta por zona de destino.
  5. **Despacho a Reparto Final**: Se asigna a conductor de la zona geográfica correspondiente (`transit`).
  6. **Entrega Final**: Conductor entrega el paquete al destinatario tras validar el PIN seguro (`delivered`).
- **Criterio de Aceptación**: Cada etapa modifica secuencialmente el estado del paquete e inserta un registro en el log de auditoría.

---

### RF-14: Optimización y Enrutamiento de Recogida (Dual-Routing)
- **Descripción**: Algoritmo y módulo de enrutamiento dual para organizar paradas de recolección previas al arribo al centro de distribución.
- **Funcionalidades**:
  - Vista segregada entre recolecciones pendientes y despachos finales.
  - Ordenamiento secuencial de puntos de retiro en clientes corporativos (remitentes) con indicación de bultos a retirar y teléfono de contacto en bodega.
  - Enlace al conductor con la ubicación del Hub Central Pudahuel para la consolidación final de la carga recogida.
- **Criterio de Aceptación**: El conductor visualiza de manera separada su itinerario matutino de recolección y su ruta vespertina de reparto final.

---

## 4. Tarifas y Modelo de Cotización

El costo total del envío se calcula con la siguiente fórmula matemática:

$$\text{Costo Total} = \max(0, (\text{Flete Base} + \text{Costo Bultos} + \text{Recargo Servicio} + \text{Seguro}) - \text{Descuento})$$

### Desglose de Parámetros:
1. **Flete Base**: `$4,500 CLP`.
2. **Costo por Bulto**: $\text{Bultos} \times \$800\text{ CLP}$.
3. **Recargo por Tipo de Envío**:
   - `normal` (Standard 48h): `$0 CLP`.
   - `express` (Priority 24h): `$3,000 CLP`.
   - `same_day` (Mismo Día): `$5,000 CLP`.
4. **Seguro de Mercancía**:
   - Para valores declarados $\le \$50,000\text{ CLP}$: `$1,000 CLP`.
   - Para valores declarados $> \$50,000\text{ CLP}$: $\$1,000 + (\text{Valor Declarado} - 50000) \times 0.01$.

---

## 5. Máquina de Estados del Paquete

| Código Estado | Etiqueta UI | Descripción | Acciones Permitidas |
|---|---|---|---|
| `pending` | Pendiente Pago | Pedido ingresado sin confirmación de pago | Pagar en línea, Cancelar |
| `paid` | Pagado | Pago procesado exitosamente; listo para programar retiro | Asignar driver de recogida |
| `pickup_assigned` | Recogida Asignada | Conductor asignado para retirar en origen | Iniciar ruta de retiro |
| `picked_up` | Recogido (Foto) | Conductor retiró el paquete en origen con fotografía | Trasladar a Centro de Distribución |
| `in_hub` | En Hub CD | Paquete recepcionado y clasificado en Hub Central Pudahuel | Asignar conductor de reparto final |
| `transit` | En Reparto Final | Paquete en vehículo de entrega rumbo al destinatario | Entregar con PIN, Registrar Incidencia |
| `delivered` | Entregado | Paquete entregado satisfactoriamente al cliente | Finalizado |
| `incident` | Incidencia | Falla en entrega (dirección incorrecta, cliente ausente) | Reintentar despacho, Devolver a Hub |

---

## 6. Requerimientos No Funcionales (RNF)

- **RNF-01 (Diseño Responsivo y Móvil)**: La vista de conductor (`/driver/daily`) debe ser ultraliviana, optimizada para pantallas táctiles de dispositivos móviles y conexiones 3G/4G en terreno.
- **RNF-02 (Trazabilidad e Inmutabilidad)**: Los eventos en el registro de auditoría (`eventLogs`) deben ser de solo adición (*append-only*), sin posibilidad de alteración histórica.
- **RNF-03 (Seguridad en la Entrega)**: El código PIN de entrega debe generarse de forma criptográficamente pseudoaleatoria (`FLW-XXXX`) y solo revelarse al destinatario final y al portal del cliente.
- **RNF-04 (Validación de Integridad en Frontend)**: Bloqueo reactivo inmediato en formularios ante comunas no cubiertas y campos obligatorios incompletos.
- **RNF-05 (Rendimiento)**: Carga instantánea de tablas de administración y páginas de tracking sin dependencias de servicios externos pesados de renderizado cartográfico.
