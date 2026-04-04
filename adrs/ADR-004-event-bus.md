# ADR-004: Event / Message Bus — Bus interno de dominio + broker ligero

|Field|Value|
|-|-|                                                    
| Status |Accepted| 
| Date |04-04-2026 | 
| Deciders | Pablo Siquinajay, Christian Sosa| 


## Contexto

### ¿Qué problemas estamos resolviendo?
Los flujos de CaféOrigen están fuertemente basados en eventos de negocio:

- Registro y verificación de productor (`UsuarioRegistrado`, `ProductorVerificado`, `ProductorRechazado`).
- Creación y ciclo de vida de pedidos (`PedidoRealizado`, `PedidoConfirmado`, `PedidoCancelado`, `PedidoCompletado`).
- Pagos escrow (`PagoAutorizado`, `PagoCapturado`, `PagoFallido`, `PagoReembolsado`).
- Logística de envíos (`EnvioCreado`, `EstadoEnvioActualizado`, `EnvioEntregado`).
- Reseñas y reputación (`ResenaEnviada`, `ReputacionActualizada`).

En [`03-service-module-decomposition.md`](/proposals/03-service-module-decomposition.md#3-qué-posee-qué-expone-y-a-qué-bounded-context-mapea-cada-módulo) se define un **bus de eventos de dominio interno** en el kernel:

```text
│   ├── 🔗 shared (Shared Kernel)
│   │   ├── 🧠 kernel
│   │   │   ├── kernel.module.ts <--
│   │   │   ├── domain-events.ts <--
│   │   │   ├── use-case-bus.ts <--
```
Debemos decidir la tecnología y el alcance del bus:

- ¿Sólo in-memory dentro del proceso?
- ¿Un broker persistente (*RabbitMQ*, *Kafka*, *cloud-managed queue*)?
- ¿Mixto?

### ¿Restricciones existentes?
- Microkernel desplegado como uno o varios contenedores idénticos.
- Volúmenes de mensajes moderados (marketplace en crecimiento, no un sistema de streaming masivo).
- Presupuesto limitado.
- Procesos de negocio de minutos/horas (p.ej. disputas, reintentos de pago).

## Opciones

### Opción A — Sólo bus de eventos in-memory dentro del proceso

- Implementar un event bus simple en el kernel:
  - Handlers registrados en memoria.
  - Publicación/suscripción dentro de un mismo proceso.
- En despliegue con múltiples instancias, cada instancia maneja sus propios eventos.

**Ventajas**

- Implementación extremadamente sencilla.
- Latencia mínima.
- Sin costos adicionales de infraestructura.

**Desventajas**

- **No hay durabilidad**:
  - Si el proceso se cae, se pierden eventos no procesados.
- En despliegues con múltiples instancias, se pierde coordinación global:
  - Un evento puede ser manejado sólo por la instancia que lo publicó, lo que puede ser suficiente en algunos casos pero no en otros.
- Difícil integrar sistemas externos (ej. colas para jobs programados, reintentos de largo plazo).

### Opción B — Broker externo robusto (*Kafka*)

- Introducir *Kafka* como plataforma central de streaming de eventos.
- Todos los módulos publican/consumen desde topics en *Kafka*.

**Ventajas**

- Alta durabilidad, ordenamiento, replays, escalabilidad.
- Excelente para escenarios de streaming de alto volumen.

**Desventajas**

- Operación compleja para un equipo pequeño.
- Overkill para el volumen y caso de uso actual.
- Aumenta los costos de infraestructura y la carga cognitiva.

### Opción C — Bus interno + broker ligero (*RabbitMQ* o cola gestionada)

- Mantener el **bus in-memory** del kernel como mecanismo de publicación/suscripción dentro del proceso.
- Cuando un tipo de evento necesita:
  - Durabilidad (no perder eventos en reinicios).
  - Integración con sistemas externos.
  - Procesamiento asíncrono de larga duración.
- Entonces se configura un **adaptador** hacia un broker ligero o cola gestionada:
  - Ejemplo: RabbitMQ, SQS, Pub/Sub, etc.
- El núcleo arma una abstracción para productores/consumidores; la implementación se puede cambiar por configuración.

**Ventajas**

- Mantiene la simplicidad para la mayoría de los eventos de dominio internos.
- Permite agregar **durabilidad selectiva** donde es necesario:
  - Jobs de reintento de pagos.
  - Procesos de logística con integraciones externas.
- Escalable a futuro sin imponer complejidad a todo el sistema desde el día 1.

**Desventajas**

- Añade una capa de indirección (hay que decidir qué eventos van al broker).
- Se deben diseñar estrategias de idempotencia y DLQ para esos eventos durables.

## Decisión

### ¿Qué opción?

Se adopta la **Opción C — Bus interno + broker ligero**.

### ¿Por qué se ajusta a nuestro proyecto específicamente?

- El **kernel** proveerá:
  - Una interfaz de `DomainEventBus` in-memory para la mayoría de los eventos.
  - Posibilidad de registrar **publicadores secundarios** que envían eventos críticos a un broker externo cuando se habilite.
- Inicialmente, se prioriza el uso del bus in-memory; la integración con un broker gestionado (por ejemplo, *SQS*/*RabbitMQ*) puede habilitarse gradualmente para:
  - Eventos de pagos (`PagoAutorizado`, `PagoFallido`, `PagoReembolsado`, etc.).
  - Procesos de larga duración (disputas, reintentos).
  - Integraciones con terceros (notificaciones, logística).

## Consecuencias

**Positivas**

- Mantiene bajo el costo y la complejidad operativa al inicio.
- Respeta el diseño del microkernel (`kernel.messaging`) definido en [`03-service-module-decomposition.md`](/proposals/03-service-module-decomposition.md#37-módulo-de-mensajería-modulesmessaging).
- Permite evolucionar hacia una arquitectura con mayor durabilidad y desacoplamiento cuando el negocio lo requiera.
- Evita un lock-in temprano en una tecnología pesada como *Kafka*, manteniendo una capa de abstracción.

**Negativas / Riesgos**

- Mientras sólo se use el bus in-memory:
  - Riesgo de pérdida de eventos en caídas de proceso o despliegues.
  - **Mitigación**: usar transacciones de base de datos + outbox pattern cuando se integren procesos críticos.
- La incorporación de un broker externo más adelante requerirá trabajo adicional:
  - Diseño de topologías, colas, DLQs, políticas de reintento.
- - **Mitigación**: Requiere disciplina para documentar claramente qué eventos son “fire-and-forget” y cuáles requieren durabilidad.

---
