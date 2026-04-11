# Infraestructura — CaféOrigen

## Propósito

Esta carpeta documenta la **justificación, alternativas y trade-offs** de cada componente de infraestructura de CaféOrigen. Complementa los otros tres pilares de documentación del repositorio:

- **`/adrs`** — Decisiones arquitectónicas globales (modelo de despliegue, base de datos, bus de eventos, autenticación).
- **`/diagrams`** — Topología visual del sistema (especialmente [`04-deployment.md`](/diagrams/04-deployment.md), que muestra cómo encajan físicamente los componentes aquí descritos).
- **`/proposals`** — Propuestas de diseño y descomposición de módulos.

Los archivos de `/adrs` responden *qué* arquitectura adoptamos y *por qué* a nivel de patrón. Esta carpeta responde *qué herramienta concreta* cubre cada rol de infraestructura, *por qué esa herramienta sobre sus alternativas* y *cómo se conecta a los bounded contexts del dominio*.

## Cómo leer esta carpeta

- Si buscas la **topología** de red, zonas y flujo de datos → empieza por [`/diagrams/04-deployment.md`](/diagrams/04-deployment.md).
- Si buscas la **razón de un patrón arquitectónico** (por qué microkernel, por qué database-per-context) → ve a [`/adrs`](/adrs/README.md).
- Si buscas **qué producto/servicio AWS** usamos para un rol concreto, sus alternativas y sus trade-offs → estás en el lugar correcto.

Cada archivo de esta carpeta describe **un componente** con cinco secciones fijas: Responsabilidades, Por qué este proyecto lo necesita, Elección tecnológica (tabla comparativa), Trade-offs y cómo se Integra con el resto del sistema.

## Índice

- [01 — API Gateway](01-api-gateway.md) — Punto de entrada público para REST y WebSocket
- [02 — Load Balancer](02-load-balancer.md) — Distribución de carga entre réplicas del microkernel
- [03 — Cache](03-cache.md) — Caché en memoria para sesiones, catálogo y rate limiting
- [04 — Message Bus / Event Queue](04-message-bus.md) — Bus de eventos de dominio + cola durable para procesos críticos
- [05 — Primary Database](05-primary-database.md) — Persistencia relacional por bounded context (esquemas separados)
- [06 — Authentication Provider](06-authentication-provider.md) — Proveedor OIDC gestionado + RBAC centralizado en el kernel
- [07 — Payment Provider](07-payment-provider.md) — Pasarela de pagos escrow para transacciones productor-comprador
- [08 — Notification System](08-notification-system.md) — Notificaciones multicanal: email, WhatsApp/SMS y push

## Restricciones transversales

Todas las decisiones de esta carpeta están condicionadas por las siguientes restricciones del proyecto, documentadas en [`/proposals/01-cafeorigen_arquitectura.md`](/proposals/01-cafeorigen_arquitectura.md):

| Restricción | Valor |
|-|-|
| Presupuesto de infraestructura (año 1) | < 800 USD/mes |
| Tamaño del equipo | 3–5 ingenieros |
| Proveedor de cloud principal | AWS |
| Modelo de despliegue | Microkernel único, contenedores stateless |
| Cumplimiento fiscal | SAT FEL Guatemala (Pagos) |
| Horizonte de evolución | Migración gradual a microservicios posible sin rediseño |

Cuando una alternativa más potente existe pero supera estas restricciones (p.ej. *Kafka* sobre *SQS*, *Aurora Serverless* sobre *RDS*), se documenta como "Alternativa B" con su justificación de descarte.

## Convenciones

- Fecha de redacción: 10-04-2026 (formato DD-MM-YYYY).
- Autores: Pablo Siquinajay, Christian Sosa.
- Terminología técnica en inglés únicamente cuando el proyecto ya la usa en inglés (*stateless*, *microkernel*, *bounded context*, *ACL*, *outbox pattern*, *OIDC*, *RBAC*).
- Sin emojis en prosa.
- Nombres de productos en *cursiva* dentro del texto corrido.
- Identificadores de código (`auth.guard.ts`, `DomainEventBus`) en backticks.
- Todos los enlaces son absolutos desde la raíz del repositorio.
