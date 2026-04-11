# 04 — Message Bus / Event Queue

|Field|Value|
|-|-|
| Componente | Event Bus / Message Queue |
| Elección | Amazon SQS + Amazon SNS |
| Fecha | 10-04-2026 |
| Autores | Pablo Siquinajay, Christian Sosa |

## 1. Responsabilidades

- Transporta los **eventos de dominio críticos** que requieren durabilidad fuera del proceso: `PagoAutorizado`, `PagoFallido`, `PagoReembolsado`, `EnvioCreado`, `EnvioEntregado`, asegurando que no se pierdan aunque el contenedor backend se reinicie durante un despliegue.
- Actúa como cola de trabajo (*SQS Standard Queue*) para los reintentos automáticos de procesos de larga duración: disputas de pago, reintentos de webhook de transportistas, re-envío de notificaciones fallidas.
- Implementa la *Dead Letter Queue* (DLQ) para aislar los mensajes que fallaron tras el número máximo de intentos, permitiendo al equipo auditarlos sin bloquear el flujo principal.
- Permite la difusión *fan-out* via *SNS topics* hacia múltiples suscriptores (p.ej. el módulo de Mensajería y el de Reseñas reciben `PedidoCompletado` sin que el módulo de Pedidos conozca a sus consumidores).
- Desacopla el ritmo de producción y consumo de eventos, protegiendo la base de datos de *RDS PostgreSQL* de picos de escritura concurrente cuando múltiples pedidos se completan simultáneamente.

## 2. Por qué este proyecto lo necesita

CaféOrigen es un marketplace donde los flujos de negocio abarcan varios módulos y tardan minutos u horas: desde que un comprador paga hasta que el lote llega a su destino y se libera el escrow a favor del productor. Durante ese tiempo, pueden ocurrir fallos de red, reinicios del contenedor o timeouts con *Stripe*. Sin un broker con durabilidad, estos eventos se perderían en el bus in-memory del kernel y los pagos quedarían en estado inconsistente — un riesgo inaceptable en un sistema con dinero real.

Adicionalmente, el diseño de bounded contexts exige bajo acoplamiento entre módulos: el módulo de Pedidos no debe saber cuántos módulos reaccionan a `PedidoCompletado`. El patrón fan-out de *SNS + SQS* resuelve esto de forma nativa sin que el kernel coordine las suscripciones en código.

Esta decisión está directamente alineada con [ADR-004](/adrs/ADR-004-event-bus.md): bus in-memory para eventos internos de bajo riesgo + broker externo ligero para eventos críticos y de larga duración.

## 3. Elección tecnológica

Se eligió *Amazon SQS + SNS* como la combinación "broker ligero" preferida en [ADR-004](/adrs/ADR-004-event-bus.md) por su modelo de precios *pay-per-message* (primer millón de mensajes al mes es gratuito), su integración nativa con *ECS Fargate* y *Lambda*, y la ausencia de servidores que mantener.

| Dimensión | Amazon SQS + SNS | Amazon MSK (Kafka gestionado) | Amazon MQ (RabbitMQ gestionado) |
|---|---|---|---|
| Gestionado / Self-hosted | Totalmente gestionado (AWS) | Totalmente gestionado (AWS) | Totalmente gestionado (AWS) |
| Complejidad operativa | Muy baja — sin brokers, particiones ni clústeres | Alta — requiere dimensionar brokers, tópicos, particiones, retención | Media — gestión de vhosts, exchanges y bindings |
| Costo a nuestra escala (<800 USD/mes) | Prácticamente gratis a baja escala (primer millón msg/mes gratis; ~$0.40/millón después) | ~$200-400/mes mínimo por clúster de 3 brokers | ~$35-70/mes por instancia mq.m5.large |
| Característica diferencial clave | Fan-out nativo SNS→múltiples SQS; DLQ integrada; escala infinita sin configuración | Replay de eventos histórico; ordenamiento estricto por partición; throughput masivo | Modelo AMQP flexible; routing rules avanzado con exchanges y routing keys |

## 4. Trade-offs

| Ventajas | Desventajas |
|---|---|
| Costo casi cero a la escala inicial del marketplace | No ofrece replay de eventos histórico (a diferencia de Kafka); una vez consumido, el mensaje desaparece |
| DLQ nativa para mensajes fallidos sin código adicional en el backend | Ordering estricto solo disponible en SQS FIFO (costo mayor y throughput limitado a 300 msg/s por grupo) |
| Fan-out SNS → múltiples colas SQS desacopla productores de consumidores sin cambios en el kernel | No apto para streaming de alto volumen en tiempo real (>10k msg/s); requeriría migrar a MSK |
| Visibilidad de mensaje configurable permite reintentos automáticos sin código de retry en el backend | Latencia mínima de ~10-20 ms vs. latencia sub-ms de un bus in-memory; aceptable para eventos de negocio |
| Integración nativa con *AWS Lambda* para workers background sin levantar un contenedor dedicado | Modelo de programación basado en polling (no push), lo que requiere un ciclo de consulta activo en el worker |

## 5. Integración con el resto del sistema

En la topología de [`04-deployment.md`](/diagrams/04-deployment.md), el nodo `Message Broker (Kafka/SQS/SNS)` se ubica en el **Data Tier** de la VPC. Esta carpeta confirma que dicho nodo se implementa con *SQS + SNS*.

Los contenedores del backend publican eventos críticos al broker via el adaptador secundario del `DomainEventBus` del kernel (definido en `domain-events.ts`). El flujo es:

1. El módulo de Pedidos emite `PedidoCompletado` al `DomainEventBus` in-memory.
2. El adaptador del kernel serializa el evento y lo publica en el *SNS topic* `pedidos-eventos`.
3. *SNS* entrega el mensaje en paralelo a:
   - Cola SQS `mensajeria-queue` → módulo de Mensajería activa notificación multicanal ([`08-notification-system.md`](/infrastructure/08-notification-system.md)).
   - Cola SQS `pagos-capture-queue` → módulo de Pagos dispara la captura del escrow en *Stripe* ([`07-payment-provider.md`](/infrastructure/07-payment-provider.md)).
   - Cola SQS `resenas-queue` → módulo de Reseñas habilita la solicitud de reseña al comprador.

Los mensajes no procesados tras 3 intentos se mueven a la DLQ correspondiente para auditoría por el rol `ADMIN`.

Referencia ADR: [ADR-004](/adrs/ADR-004-event-bus.md) — "Bus interno + broker ligero. Inicialmente *SQS/RabbitMQ* puede habilitarse gradualmente para eventos de pagos y procesos de larga duración."
