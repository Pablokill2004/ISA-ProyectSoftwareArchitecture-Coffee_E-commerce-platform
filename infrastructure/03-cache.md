# 03 — Cache

|Field|Value|
|-|-|
| Componente | Cache |
| Elección | Amazon ElastiCache for Redis |
| Fecha | 10-04-2026 |
| Autores | Pablo Siquinajay, Christian Sosa |

## 1. Responsabilidades

- Almacena en memoria las sesiones de usuario (token claims deserializados, estado de verificación del productor) para que el `auth.guard.ts` del kernel no consulte la base de datos en cada petición autenticada.
- Cachea las páginas de resultados del catálogo de lotes de café (listados por origen, variedad, cupping score) con TTL corto (~60 segundos), reduciendo la carga sobre el esquema `catalog` de *RDS PostgreSQL* durante picos de tráfico.
- Implementa *rate limiting* distribuido por IP y por `userId` para los endpoints de creación de pedidos y de chat, usando contadores atómicos de Redis que funcionan correctamente con las N réplicas stateless del microkernel.
- Mantiene el estado de disponibilidad de lotes (`stock` en memoria) durante el flujo de reserva de pedido (`CREADO → PAGO_RETENIDO`) para evitar condiciones de carrera entre compradores concurrentes antes de confirmar en base de datos.
- Sirve como almacén de *idempotency keys* de corta duración para las operaciones de pago, evitando cargos duplicados si el cliente reintenta la petición por timeout de red.

## 2. Por qué este proyecto lo necesita

El microkernel de CaféOrigen despliega múltiples réplicas stateless del backend ([ADR-001](/adrs/ADR-001-deployment-model.md)). Esto implica que el estado compartido entre réplicas — sesiones de usuario, contadores de rate limiting, disponibilidad de stock — no puede vivir en memoria del proceso; necesita un almacén externo de baja latencia al que todas las réplicas accedan por igual.

Sin cache, cada petición autenticada al módulo de Catálogo o Pedidos implicaría al menos una consulta SQL para validar el token y otra para leer el catálogo, multiplicando la carga en *RDS PostgreSQL*. En un lanzamiento de lote de café de especialidad con alta demanda simultánea, esto saturaria las conexiones de base de datos y degradaría la experiencia para todos los compradores — un escenario crítico para la reputación del marketplace.

## 3. Elección tecnológica

Se eligió *Amazon ElastiCache for Redis* por ser el servicio gestionado AWS de menor fricción operativa para Redis, con soporte de Multi-AZ, failover automático y compatibilidad total con el cliente *ioredis* de NestJS.

| Dimensión | Amazon ElastiCache (Redis) | Redis self-hosted en EC2 | ElastiCache for Memcached |
|---|---|---|---|
| Gestionado / Self-hosted | Totalmente gestionado (AWS) | Self-hosted | Totalmente gestionado (AWS) |
| Complejidad operativa | Muy baja — parches, backups y failover automáticos | Alta — HA con Sentinel o Cluster manual | Baja, pero sin persistencia ni estructuras de datos avanzadas |
| Costo a nuestra escala (<800 USD/mes) | ~$15-25/mes (cache.t3.micro, single-AZ) | ~$10-15/mes EC2 t3.micro + gestión manual | Similar a ElastiCache Redis en precios |
| Característica diferencial clave | Estructuras de datos avanzadas (Sorted Sets para ranking, Streams para eventos ligeros) + AOF persistence opcional | Control total de configuración y versión de Redis | Sencillo y rápido, pero sin sorted sets, hashes avanzados ni persistencia |

## 4. Trade-offs

| Ventajas | Desventajas |
|---|---|
| Failover automático Multi-AZ; si el nodo primario falla, la réplica promueve en < 60 segundos | Costo fijo mínimo incluso con tráfico bajo (~$15/mes para t3.micro) |
| Estructuras de datos nativas (Sorted Sets, Hashes, Counters) facilitan rate limiting e idempotency keys sin código extra | Vendor lock-in; ElastiCache no expone el nodo Redis directamente, lo que complica herramientas de administración estándar fuera de la VPC |
| Compatible con *ioredis* y *@nestjs/cache-manager* sin configuración especial | Single-AZ en etapa inicial; habilitar Multi-AZ duplica el costo del nodo |
| Métricas nativas en *CloudWatch* (CacheHits, CacheMisses, CurrConnections) sin agentes adicionales | No reemplaza la base de datos; los TTL cortos exigen diseño cuidadoso de invalidación de caché cuando un productor actualiza un lote |

## 5. Integración con el resto del sistema

En la topología de [`04-deployment.md`](/diagrams/04-deployment.md), el nodo `Cache / Redis` se ubica en el **Data Tier** de la VPC privada. Ambas réplicas del backend (`Backend Container 1` y `Backend Container 2`) lo consultan via GET/SET en el mismo punto final de ElastiCache, garantizando coherencia de estado entre réplicas.

Los módulos que interactúan con la caché directamente son:

- **Identidad** — cachea el perfil de usuario y los claims del token OIDC de *Cognito* ([`06-authentication-provider.md`](/infrastructure/06-authentication-provider.md)) para evitar validaciones repetidas en el kernel.
- **Catálogo** — cachea listados y páginas de resultados con TTL corto; invalida la entrada cuando el productor actualiza el estado de un lote.
- **Pedidos** — usa contadores atómicos de Redis para gestionar la reserva optimista de stock antes de confirmar en *RDS PostgreSQL* ([`05-primary-database.md`](/infrastructure/05-primary-database.md)).
- **Pagos** — almacena *idempotency keys* de *Stripe Connect* ([`07-payment-provider.md`](/infrastructure/07-payment-provider.md)) con TTL de 24 horas para prevenir cargos duplicados.

No existe un ADR dedicado al caché; esta elección es consecuencia directa de [ADR-001](/adrs/ADR-001-deployment-model.md) (estado compartido entre réplicas stateless) y de las restricciones de presupuesto del proyecto.
