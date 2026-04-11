# 05 — Primary Database

|Field|Value|
|-|-|
| Componente | Primary Database |
| Elección | Amazon RDS for PostgreSQL |
| Fecha | 10-04-2026 |
| Autores | Pablo Siquinajay, Christian Sosa |

## 1. Responsabilidades

- Persiste el estado de todos los bounded contexts del dominio en **esquemas lógicamente separados** dentro de la misma instancia RDS: `identity`, `catalog`, `orders`, `payments`, `logistics`, `messaging`, `reviews`.
- Garantiza la **integridad transaccional ACID** para operaciones de negocio críticas: la transición de estado de un pedido (`CREADO → PAGO_RETENIDO → CONFIRMADO`) es atómica y no puede quedar en estado intermedio ante un fallo del proceso.
- Almacena los **outbox events** de los módulos de Pagos y Pedidos en una tabla dedicada (`outbox`) por esquema, implementando el *outbox pattern* que coordina la escritura en base de datos con la publicación al broker *SQS/SNS* ([`04-message-bus.md`](/infrastructure/04-message-bus.md)) en la misma transacción.
- Provee capacidades de búsqueda y filtrado avanzado del catálogo mediante índices parciales y consultas `JSONB` sobre los atributos variables de los lotes de café (variedad, origen, proceso, cupping score).
- Mantiene la traza de auditoría de las operaciones de pago (monto, moneda, estado de escrow, referencia *Stripe*) en el esquema `payments`, necesaria para la conciliación fiscal FEL Guatemala.

## 2. Por qué este proyecto lo necesita

La estrategia *database-per-context* adoptada en [ADR-003](/adrs/ADR-003-database-strategy.md) exige que cada bounded context posea su propio espacio de datos sin acceso directo de otro módulo. En CaféOrigen esto es especialmente relevante porque el dominio de Pagos maneja datos financieros sensibles (montos, referencias *Stripe*, información fiscal) que no deben estar en el mismo esquema que el catálogo de lotes o los mensajes de chat.

Al mismo tiempo, con un equipo de 3–5 ingenieros y presupuesto limitado, no es viable operar siete bases de datos independientes desde el primer día. *Amazon RDS for PostgreSQL* permite implementar los siete esquemas en una única instancia gestionada, lo que cumple con los principios DDD de aislamiento lógico mientras mantiene la simplicidad operativa de un solo motor de base de datos que parchear, escalar y respaldar.

## 3. Elección tecnológica

Se eligió *Amazon RDS for PostgreSQL* por ser el motor relacional gestionado de AWS más maduro, con soporte de esquemas múltiples, tipos `JSONB` para los atributos variables de lotes de café, y modelo de precios predecible en instancias pequeñas.

| Dimensión | Amazon RDS for PostgreSQL | Amazon Aurora PostgreSQL | PostgreSQL self-hosted en EC2 |
|---|---|---|---|
| Gestionado / Self-hosted | Totalmente gestionado (AWS) | Totalmente gestionado (AWS, compatible PostgreSQL) | Self-hosted |
| Complejidad operativa | Baja — backups, parches y failover automáticos | Baja — similar a RDS pero con clúster Aurora | Muy alta — HA manual con replicación, WAL shipping, patroni |
| Costo a nuestra escala (<800 USD/mes) | ~$30-50/mes (db.t3.medium, Single-AZ) | ~$70-100/mes mínimo (modo Serverless v2 más accesible, pero costo variable) | ~$20-30/mes EC2 t3.medium + gestión manual |
| Característica diferencial clave | Compatibilidad total con PostgreSQL; costo más bajo del stack AWS; RDS Proxy disponible | Read replicas hasta 15; replicación Aurora más rápida; Serverless v2 escala a cero | Control total sobre configuración, extensiones y versión exacta de PostgreSQL |

## 4. Trade-offs

| Ventajas | Desventajas |
|---|---|
| Backups automáticos con retención de 7-35 días; restauración a punto en el tiempo (PITR) incluida | Costos fijos incluso con tráfico bajo; no escala a cero como Aurora Serverless v2 |
| Multi-AZ opcional — réplica standby en otra zona de disponibilidad para failover en < 60 segundos | Rendimiento máximo limitado vs. Aurora (throughput de escritura ~20-30% menor) |
| RDS Proxy disponible para gestionar pool de conexiones cuando el número de réplicas ECS escala | Costo adicional para habilitar Multi-AZ (~doble del precio de Single-AZ) |
| Soporte nativo de `JSONB`, índices GIN y búsqueda de texto completo para atributos del catálogo | Todos los contextos en una misma instancia: un problema de rendimiento en un esquema puede afectar a los demás |
| Compatible con *TypeORM* y *Prisma* sin cambios, tal como lo define `03-service-module-decomposition.md` | Migración futura a instancias separadas por contexto requiere mover esquemas, aunque es menos traumático que separar microservicios completos |

## 5. Integración con el resto del sistema

En la topología de [`04-deployment.md`](/diagrams/04-deployment.md), *Amazon RDS for PostgreSQL* ocupa el **Data Tier** de la VPC, representado por los tres nodos lógicos `DB Core Domains`, `DB Supporting` y `DB Identidad & Pagos`. Físicamente, en la etapa actual estos tres nodos corresponden a esquemas separados en una misma instancia RDS; lógicamente, cada bounded context solo accede a su propio esquema a través del *repository pattern* de su módulo NestJS.

Los accesos desde el backend son siempre vía ORM (*TypeORM*) usando las credenciales del contexto correspondiente, nunca mediante consultas `JOIN` entre esquemas. Esto preserva el aislamiento DDD y facilita extraer un contexto a su propia instancia en el futuro con un cambio de *connection string*.

El *outbox pattern* (documentado en [ADR-004](/adrs/ADR-004-event-bus.md) como mitigación de riesgo) usa transacciones de RDS: el módulo de Pedidos escribe el cambio de estado del pedido **y** el registro en la tabla `orders.outbox` en la misma transacción, garantizando que el broker *SQS* solo reciba el evento si la base de datos lo confirmó.

El módulo de Pagos accede al esquema `payments` que incluye las referencias de *Stripe Connect* ([`07-payment-provider.md`](/infrastructure/07-payment-provider.md)) y los datos necesarios para emitir la factura FEL Guatemala.

Referencia ADR: [ADR-003](/adrs/ADR-003-database-strategy.md) — "Base de datos por contexto, implementada como esquemas separados en un mismo clúster/motor gestionado."
