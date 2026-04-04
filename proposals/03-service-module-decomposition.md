# 03 — Descomposición en Módulos y Organización de Código

## 1. Visión General

CaféOrigen adopta una **arquitectura microkernel (plug-in)**:

- **Núcleo (kernel)**: concentra la lógica genérica de marketplace:
  - Autenticación / autorización
  - Registro de usuarios y perfiles básicos
  - Máquina de estados de pedidos
  - Orquestación de pagos (sin detalles de Stripe/FEL)
  - Orquestación de envíos (sin detalles de transportistas)
  - Orquestación de notificaciones (sin detalles de Twilio/SendGrid)
  - Infraestructura compartida (HTTP API, eventos de dominio internos, configuración, logging)

- **Módulos plug‑in**: implementan los **7 contextos delimitados** definidos en `02-bounded-contexts (1).md`:
  - Identidad
  - Catálogo
  - Pedidos
  - Pagos
  - Logística
  - Mensajería
  - Reseñas

Cada plug‑in es un **módulo interno independiente** dentro de un único deploy (un solo contenedor / proceso), pero con límites de contexto claros.

---

## 2. Árbol de directorios propuesto

Suponiendo un backend en TypeScript/Node con NestJS (u otro framework modular similar), la estructura lógica sería:

```text
/backend
│
├── 📁 src
│   │
│   ├── 🧩 app (Application Bootstrap)
│   │   ├── main.ts
│   │   └── app.module.ts
│   │
│   ├── ⚙️ config (Global Configuration)
│   │   ├── config.module.ts
│   │   └── configuration.ts
│   │
│   ├── 🔗 shared (Shared Kernel)
│   │   ├── 🧠 kernel
│   │   │   ├── kernel.module.ts
│   │   │   ├── domain-events.ts
│   │   │   ├── use-case-bus.ts
│   │   │   ├── 🔐 auth
│   │   │   │   ├── auth.guard.ts
│   │   │   │   └── current-user.decorator.ts
│   │   │   ├── 💾 persistence
│   │   │   │   ├── database.module.ts
│   │   │   │   └── orm.config.ts
│   │   │   └── 📨 messaging
│   │   │       ├── event-publisher.ts
│   │   │       └── event-subscriber.ts
│   │   └── 🧰 common
│   │       ├── errors/
│   │       ├── utils/
│   │       └── types/
│   │
│   └── 📦 modules (Bounded Contexts)
│       │
│       ├── 👤 identity
│       │   ├── identity.module.ts
│       │   ├── ⚙️ application/
│       │   │   ├── commands/
│       │   │   ├── queries/
│       │   │   └── services/
│       │   ├── 🧠 domain/
│       │   │   ├── entities/
│       │   │   │   ├── Usuario.ts
│       │   │   │   ├── PerfilProductor.ts
│       │   │   │   └── DocumentoVerificacion.ts
│       │   │   ├── value-objects/
│       │   │   └── events/
│       │   │       ├── UsuarioRegistrado.ts
│       │   │       └── ProductorVerificado.ts
│       │   └── 🛠️ infrastructure/
│       │       ├── repositories/
│       │       ├── controllers/
│       │       └── mappers/
│       │
│       ├── 📦 catalog
│       │   ├── catalog.module.ts
│       │   ├── ⚙️ application/
│       │   ├── 🧠 domain/
│       │   │   ├── entities/
│       │   │   │   ├── LoteCafe.ts
│       │   │   │   ├── ImagenLote.ts
│       │   │   │   └── NotaCatacion.ts
│       │   │   └── events/
│       │   │       ├── LotePublicado.ts
│       │   │       └── LoteAgotado.ts
│       │   └── 🛠️ infrastructure/
│       │
│       ├── 📝 orders
│       │   ├── orders.module.ts
│       │   ├── ⚙️ application/
│       │   ├── 🧠 domain/
│       │   │   ├── entities/
│       │   │   │   ├── Pedido.ts
│       │   │   │   ├── SolicitudMuestra.ts
│       │   │   │   └── HistorialEstadoPedido.ts
│       │   │   └── events/
│       │   │       ├── PedidoRealizado.ts
│       │   │       ├── PedidoConfirmado.ts
│       │   │       └── PedidoCancelado.ts
│       │   └── 🛠️ infrastructure/
│       │
│       ├── 💳 payments
│       │   ├── payments.module.ts
│       │   ├── ⚙️ application/
│       │   ├── 🧠 domain/
│       │   │   ├── entities/
│       │   │   │   ├── TransaccionPago.ts
│       │   │   │   ├── RetencionEscrow.ts
│       │   │   │   └── Factura.ts
│       │   │   └── events/
│       │   │       ├── PagoAutorizado.ts
│       │   │       ├── PagoCapturado.ts
│       │   │       └── PagoFallido.ts
│       │   └── 🛠️ infrastructure/
│       │       └── acl/ (Anti‑Corruption Layer)
│       │           ├── stripe/
│       │           │   └── stripe-payment-provider.ts
│       │           └── sat-fel/
│       │               └── sat-fel-client.ts
│       │
│       ├── 🚚 logistics
│       │   ├── logistics.module.ts
│       │   ├── 🧠 domain/
│       │   │   ├── entities/
│       │   │   │   ├── Envio.ts
│       │   │   │   ├── EventoEnvio.ts
│       │   │   │   └── Transportista.ts
│       │   │   └── events/
│       │   │       ├── EnvioCreado.ts
│       │   │       └── EstadoEnvioActualizado.ts
│       │   └── 🛠️ infrastructure/
│       │       └── acl/
│       │           ├── aftership/
│       │           ├── flexport/
│       │           └── manual/
│       │
│       ├── 💬 messaging
│       │   ├── messaging.module.ts
│       │   ├── 🧠 domain/
│       │   │   ├── entities/
│       │   │   │   ├── Conversacion.ts
│       │   │   │   ├── Mensaje.ts
│       │   │   │   └── RegistroNotificacion.ts
│       │   │   └── events/
│       │   │       ├── MensajeEnviado.ts
│       │   │       ├── NotificacionEntregada.ts
│       │   │       └── NotificacionFallida.ts
│       │   └── 🛠️ infrastructure/
│       │       └── acl/
│       │           ├── twilio/
│       │           ├── sendgrid/
│       │           └── firebase/
│       │
│       └── ⭐ reviews
│           ├── reviews.module.ts
│           ├── 🧠 domain/
│           │   ├── entities/
│           │   │   ├── Resena.ts
│           │   │   ├── ReputacionProductor.ts
│           │   │   └── RespuestaResena.ts
│           │   └── events/
│           │       ├── ResenaEnviada.ts
│           │       └── ReputacionActualizada.ts
│           └── 🛠️ infrastructure/
│
└── 📁 test/
    └── ...
```

> Nota: El frontend (SPA móvil/web) consumiría las APIs expuestas por estos módulos, pero el foco de este documento es el backend.

---

## 3. Qué posee, qué expone y a qué bounded context mapea cada módulo

### 3.1. Núcleo (`/src/app/shared/kernel`)

- **Posee**
  - Infraestructura compartida: logging, manejo de errores, configuración.
  - <ins>Bus de eventos de dominio interno.</ins>
  - Abstracciones para autenticación/autorización.
  - Integración con el framework HTTP (ruteo, middlewares, etc.).
- **Expone**
  - Contratos de eventos de dominio (`DomainEvent` base).
  - Interfaces para publicadores/suscriptores de eventos.
  - Interfaces de seguridad (`IUserContext`, guardas de autorización).
- **Bounded contexts**
  - No es un bounded context de negocio. Es un **kernel técnico** que soporta a todos los contextos de dominio.

---

### 3.2. Módulo de Identidad (`/modules/identity`)

- **Posee**
  - Entidades: `Usuario`, `PerfilProductor`, `DocumentoVerificacion`.
  - Reglas de negocio de registro, login, verificación de productores y suspensión de cuentas.
  - Workflow de revisión de documentos.
- **Expone**
  - Endpoints HTTP para:
    - Registro / login / refresco de tokens.
    - Gestión de perfil y envío de documentos para verificación.
  - Eventos de dominio:
    - `UsuarioRegistrado`
    - `ProductorVerificado`
    - `ProductorRechazado`
    - `UsuarioSuspendido`
  - Un conjunto de DTOs publicados (Published Language):
    - `ResumenUsuarioDTO`
    - `EstadoProductorDTO`
- **Bounded context**
  - Mapea al **Contexto 1: Identidad y Verificación** definido en el documento 02.

---

### 3.3. Módulo de Catálogo (`/modules/catalog`)

- **Posee**
  - Entidades: `LoteCafe`, `ImagenLote`, `NotaCatacion`.
  - Reglas de negocio de creación, publicación, agotamiento y archivado de lotes.
  - Lógica de búsqueda y filtros.
- **Expone**
  - Endpoints HTTP:
    - Crear/editar/publicar lotes.
    - Listar/buscar lotes con filtros.
    - Gestionar notas de catación.
  - Eventos:
    - `LotePublicado`
    - `LoteActualizado`
    - `LoteAgotado`
    - `LoteArchivado`
    - `NotaCatacionAgregada`
  - DTOs de consulta de lotes (para front y otros módulos como Mensajería).
- **Bounded context**
  - Mapea al **Contexto 2: Catálogo**.

---

### 3.4. Módulo de Pedidos (`/modules/orders`)

- **Posee**
  - Entidades: `Pedido`, `SolicitudMuestra`, `HistorialEstadoPedido`.
  - Máquina de estados de pedido y lógica de transición.
  - Reglas de validación de disponibilidad contra el catálogo (a través de eventos/consultas controladas).
- **Expone**
  - Endpoints HTTP:
    - Crear pedidos.
    - Confirmar, cancelar, ver detalle y listar pedidos.
    - Flujo de solicitudes de muestras.
  - Eventos:
    - `PedidoRealizado`
    - `PedidoConfirmado`
    - `PedidoCancelado`
    - `PedidoEnviado`
    - `PedidoEntregado`
    - `PedidoCompletado`
    - `PedidoEnDisputa`
    - `MuestraSolicitada`
- **Bounded context**
  - Mapea al **Contexto 3: Pedidos**.

---

### 3.5. Módulo de Pagos (`/modules/payments`)

- **Posee**
  - Entidades: `TransaccionPago`, `RetencionEscrow`, `Factura`.
  - Lógica de escrow, captura, fallos, reembolsos y generación de facturas.
  - ACLs hacia Stripe y SAT FEL.
- **Expone**
  - Endpoints HTTP (limitados, típicamente solo para administración y webhooks).
  - Eventos:
    - `PagoAutorizado`
    - `PagoCapturado`
    - `PagoFallido`
    - `PagoReembolsado`
    - `FacturaEmitida`
  - Interfaces internas para manejar webhooks externos de Stripe/FEL.
- **Bounded context**
  - Mapea al **Contexto 4: Pagos**.

---

### 3.6. Módulo de Logística (`/modules/logistics`)

- **Posee**
  - Entidades: `Envio`, `EventoEnvio`, `Transportista`.
  - Reglas de generación de envíos, actualización de estados y manejo de excepciones de logística.
  - ACLs hacia AfterShip/Flexport/mensajería manual.
- **Expone**
  - Endpoints HTTP:
    - Consulta de tracking.
    - Configuración de transportistas (para admins).
  - Eventos:
    - `EnvioCreado`
    - `EstadoEnvioActualizado`
    - `EnvioEntregado`
    - `ExcepcionEnvio`
- **Bounded context**
  - Mapea al **Contexto 5: Logística**.

---

### 3.7. Módulo de Mensajería (`/modules/messaging`)

- **Posee**
  - Entidades: `Conversacion`, `Mensaje`, `RegistroNotificacion`.
  - Lógica de chat y enrutamiento de notificaciones.
  - ACLs hacia Twilio, SendGrid, Firebase.
- **Expone**
  - Endpoints HTTP/WebSocket:
    - API de chat entre comprador y productor.
    - APIs para listar conversaciones / mensajes.
  - Eventos:
    - `MensajeEnviado`
    - `NotificacionEntregada`
    - `NotificacionFallida`
- **Bounded context**
  - Mapea al **Contexto 6: Mensajería y Notificaciones**.

---

### 3.8. Módulo de Reseñas (`/modules/reviews`)

- **Posee**
  - Entidades: `Resena`, `ReputacionProductor`, `RespuestaResena`.
  - Cálculo de reputación y agregados por productor.
- **Expone**
  - Endpoints HTTP:
    - Crear reseñas.
    - Consultar reputación de productores.
  - Eventos:
    - `ResenaEnviada`
    - `ReputacionActualizada`
- **Bounded context**
  - Mapea al **Contexto 7: Reseñas y Reputación**.

---

## 4. Cómo la descomposición refuerza los límites de contexto

1. **Separación física por módulo**
   - Cada contexto tiene su propio árbol `/domain`, `/application`, `/infrastructure`.
   - **Regla de imports**: código de un contexto no importa clases de dominio de otro contexto directamente.  
     - Solo se permite:
       - Importar contratos de eventos (`PedidoRealizadoEvent`, `PagoAutorizadoEvent`) definidos en módulos publicados.
       - Importar DTOs publicados (`ResumenUsuarioDTO`, `EstadoProductorDTO`) desde Identidad.

2. **Eventos como integración primaria**
   - Los flujos entre contextos siguen los patrones descritos en el documento 02:
     - OHS/PL entre Identidad → Catálogo/Pedidos/Mensajería.
     - Partnership entre Catálogo ↔ Pedidos.
     - ACL hacia sistemas externos desde Pagos, Logística, Mensajería.
   - Esto evita dependencias de capa de dominio entre módulos.

3. **APIs internas vs públicas**
   - **APIs públicas (externas)**: controladores HTTP del backend (`controllers`) que exponen endpoints REST/GraphQL al frontend.
   - **APIs internas (entre módulos)**:
     - Contratos de eventos de dominio.
     - Interfaces de “servicios de aplicación” compartidos SOLO a través de interfaces en el kernel o DTOs publicados.

4. **Plug‑ins de infraestructura**
   - Integraciones externas (Stripe, SAT FEL, AfterShip, Twilio, etc.) se encapsulan en subdirectorios `/acl` dentro de su contexto.
   - Si se cambia de proveedor, se reemplaza el adaptador dentro de ese contexto sin afectar al resto.

---

## 5. Proceso / despliegue

- **Un solo proceso / contenedor**:
  - Todos los módulos (`identity`, `catalog`, `orders`, `payments`, `logistics`, `messaging`, `reviews`) viven en el mismo proceso de backend.
  - Esto es consistente con la recomendación de **microkernel** para un equipo de 3–5 ingenieros y presupuesto limitado.
- **Escalamiento horizontal**:
  - Se despliega más de un contenedor idéntico del backend (stateless), detrás de un load balancer.
  - Estado compartido en bases de datos por contexto (pueden ser esquemas separados).

- Si el sistema crece en complejidad/escala, algunos plug‑ins (por ejemplo, Pagos o Mensajería) podrían extraerse como microservicios independientes, conservando las mismas fronteras de contexto definidas aquí.
