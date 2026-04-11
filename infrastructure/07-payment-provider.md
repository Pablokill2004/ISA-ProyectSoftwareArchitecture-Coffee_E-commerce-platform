# 07 — Payment Provider

|Field|Value|
|-|-|
| Componente | Payment Provider |
| Elección | Stripe Connect |
| Fecha | 10-04-2026 |
| Autores | Pablo Siquinajay, Christian Sosa |

## 1. Responsabilidades

- Crea y gestiona **cuentas conectadas de productor** (*Stripe Connect Express*) que permiten recibir pagos directamente en la cuenta bancaria del productor guatemalteco sin que CaféOrigen maneje fondos de terceros.
- Ejecuta **retenciones de pago escrow** (`PaymentIntent` en estado `capture_method: manual`) cuando el comprador confirma un pedido: el dinero queda retenido hasta que se confirma la entrega, protegiendo a ambas partes de la transacción.
- Realiza la **captura del monto retenido** cuando el módulo de Logística confirma la entrega (`EnvioEntregado`), liberando los fondos al productor menos la comisión de la plataforma configurada como `application_fee_amount`.
- Gestiona los flujos de **reembolso y disputa**: si el comprador abre una disputa, *Stripe* actúa como árbitro inicial; el módulo de Pagos puede emitir reembolsos parciales o totales via la API de *Stripe*.
- Emite los datos de transacción (monto, moneda, referencia, fecha) que el módulo de Pagos usa para generar las facturas FEL Guatemala a través de la integración con SAT via ACL.

## 2. Por qué este proyecto lo necesita

CaféOrigen opera como **marketplace de dos lados**: los productores guatemaltecos reciben el pago de compradores internacionales (tostadores, importadores). Esta estructura implica que el dinero no es simplemente un cobro al comprador, sino una transferencia a un tercero (el productor) con comisión de plataforma, lo que requiere una arquitectura de pagos *split* que una pasarela estándar no ofrece.

*Stripe Connect* es uno de los pocos proveedores que ofrece este flujo (*marketplace payments* con escrow manual) de forma nativa, incluyendo el proceso de verificación KYC de las cuentas del productor, el manejo de divisas y la emisión de informes fiscales — tareas que construir desde cero implicaría meses de desarrollo adicional y una complejidad regulatoria que el equipo de 3–5 ingenieros no puede asumir.

AWS no ofrece un equivalente funcional a *Stripe Connect* para pagos escrow entre productores y compradores, por lo que es la única excepción justificada al criterio "100% AWS" de este proyecto.

## 3. Elección tecnológica

Se eligió *Stripe Connect* por ser el estándar de la industria para marketplaces con pagos a terceros, con soporte de escrow manual, cuentas Express para productores sin código de registro propio, y una API bien documentada compatible con el patrón ACL del módulo de Pagos.

| Dimensión | Stripe Connect | PayPal Commerce Platform | Adyen for Platforms |
|---|---|---|---|
| Gestionado / Self-hosted | SaaS totalmente gestionado | SaaS totalmente gestionado | SaaS totalmente gestionado |
| Complejidad operativa | Baja — SDK TypeScript oficial; webhooks con firma HMAC; testing con modo test nativo | Media — API REST documentada pero menos cohesiva; flujos de pago más verbosos | Alta — requiere proceso de onboarding enterprise; documentación más compleja |
| Costo a nuestra escala (<800 USD/mes) | 2.9% + $0.30 por transacción (tarjeta); sin cuota mensual fija | 3.49% + $0.49 por transacción (tarjeta estándar); comisión similar a Stripe | Negociable por volumen; no disponible para startups pequeñas sin acuerdo previo |
| Característica diferencial clave | `PaymentIntent` con `capture_method: manual` para escrow nativo; cuentas Express con KYC automático | Familiaridad del comprador con PayPal; ideal si el segmento objetivo prefiere PayPal sobre tarjeta | Mejor para volúmenes >$1M/mes y mercados internacionales complejos; excesivo para etapa inicial |

## 4. Trade-offs

| Ventajas | Desventajas |
|---|---|
| Escrow manual nativo (`capture_method: manual`) sin lógica extra en el módulo de Pagos | Comisión por transacción (2.9% + $0.30) es el mayor costo variable del sistema; no existe alternativa más barata con las mismas capacidades |
| Cuentas Express para productores con KYC automático por *Stripe*; el equipo no gestiona verificación de identidad financiera | Dependencia total de un proveedor externo para los cobros; una interrupción de *Stripe* detiene las ventas del marketplace |
| SDK TypeScript oficial (*stripe-node*) con tipado completo; integración directa con el módulo de Pagos via ACL | Restricciones por país para las cuentas conectadas; Guatemala está en lista de países soportados pero con límites de retiro |
| Webhooks con firma HMAC para validar eventos (`payment_intent.succeeded`, `charge.dispute.created`) en el módulo de Pagos | Los fondos de productores pueden estar retenidos hasta 7 días hábiles en el primer ciclo de pago de una cuenta Express nueva |
| Portal de *Stripe Dashboard* para que el equipo monitoree y gestione disputas sin código adicional | Las tarifas de Stripe no incluyen la integración con SAT FEL Guatemala; esa facturación fiscal sigue siendo responsabilidad del módulo de Pagos via ACL |

## 5. Integración con el resto del sistema

En la topología de [`04-deployment.md`](/diagrams/04-deployment.md), *Stripe* se ubica en **Managed External Services** dentro del nodo `Pagos & Facturación`. El módulo de Pagos del backend se comunica con *Stripe* exclusivamente a través de un ACL (`stripe.adapter.ts`) que encapsula el SDK *stripe-node*, aislando el resto del sistema de cambios en la API de *Stripe*.

El flujo de pagos en el ciclo de vida de un pedido es:

1. **Retención**: cuando el módulo de Pedidos emite `PedidoCreado`, el módulo de Pagos crea un `PaymentIntent` en *Stripe* con `capture_method: manual`. El comprador confirma el pago; *Stripe* emite `payment_intent.amount_capturable_updated`. El módulo de Pagos publica `PagoAutorizado`.
2. **Captura**: cuando el módulo de Logística publica `EnvioEntregado`, el módulo de Pagos recibe el evento via *SQS* ([`04-message-bus.md`](/infrastructure/04-message-bus.md)) y ejecuta `paymentIntent.capture()` en *Stripe*, transfiriendo el monto al productor menos la comisión. El módulo de Pagos publica `PagoCapturado`.
3. **Reembolso**: si hay disputa, el módulo de Pagos llama a `refund.create()` en *Stripe* y publica `PagoReembolsado`.
4. **Facturación**: los datos de la captura se envían via ACL al endpoint SAT FEL Guatemala para emitir la factura electrónica requerida por la ley guatemalteca.

Las *idempotency keys* de todas las llamadas a *Stripe* se almacenan en *ElastiCache for Redis* ([`03-cache.md`](/infrastructure/03-cache.md)) con TTL de 24 horas para prevenir cargos duplicados en reintentos.

Referencia ADR: este componente está referenciado en [ADR-001](/adrs/ADR-001-deployment-model.md) (integración externa via ACL) y en los flujos de datos de [`/proposals/04-data-flow-and-interactions.md`](/proposals/04-data-flow-and-interactions.md).
