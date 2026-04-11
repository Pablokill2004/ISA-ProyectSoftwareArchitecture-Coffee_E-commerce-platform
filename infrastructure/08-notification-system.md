# 08 — Notification System

|Field|Value|
|-|-|
| Componente | Notification System |
| Elección | Amazon SES (email) + Twilio (WhatsApp/SMS) + Firebase Cloud Messaging (push) |
| Fecha | 10-04-2026 |
| Autores | Pablo Siquinajay, Christian Sosa |

## 1. Responsabilidades

- Entrega **notificaciones transaccionales por correo electrónico** (confirmación de pedido, estado de envío, resultado de verificación de productor, solicitud de reseña) usando *Amazon SES* desde plantillas HTML gestionadas en el módulo de Mensajería.
- Envía **mensajes de WhatsApp y SMS** via *Twilio* para notificaciones de alta urgencia que requieren atención inmediata del productor (nuevo pedido recibido, disputa abierta) o del comprador (pedido listo para recoger, pago fallido).
- Distribuye **notificaciones push** a los dispositivos móviles de compradores y productores usando *Firebase Cloud Messaging* (FCM), activadas por eventos de dominio (`PedidoConfirmado`, `EnvioEntregado`, `ResenaRecibida`).
- Actúa como el **contexto terminal downstream** del sistema: el módulo de Mensajería consume eventos del bus *SQS/SNS* ([`04-message-bus.md`](/infrastructure/04-message-bus.md)) y no publica eventos hacia ningún otro módulo, simplificando el grafo de dependencias.
- Registra en la base de datos del esquema `messaging` (en *RDS PostgreSQL* — [`05-primary-database.md`](/infrastructure/05-primary-database.md)) el historial de notificaciones enviadas, el canal usado y el estado de entrega, para auditoría y diagnóstico de fallos.

## 2. Por qué este proyecto lo necesita

CaféOrigen conecta productores guatemaltecos — muchos de ellos pequeños agricultores con acceso primario a WhatsApp — con compradores internacionales que operan desde apps web y móviles. Una sola estrategia de notificación (solo email, o solo push) dejaría fuera a una parte del mercado objetivo.

Los productores prefieren WhatsApp como canal principal de comunicación de negocios en Guatemala; forzarlos a instalar la app móvil de CaféOrigen solo para recibir alertas de nuevos pedidos crearía fricciones que reducirían la participación activa del lado de la oferta del marketplace. Los compradores internacionales, por su parte, esperan confirmaciones instantáneas por email y notificaciones push si tienen la app instalada.

Sin un sistema de notificaciones multicanal, los eventos críticos del ciclo de vida del pedido (`PagoCapturado`, `EnvioCreado`, `EnvioEntregado`) quedarían visibles solo al hacer login activo en la plataforma, aumentando el tiempo de respuesta a problemas y reduciendo la confianza en el marketplace.

## 3. Elección tecnológica

Se eligió la combinación *Amazon SES + Twilio + Firebase Cloud Messaging* para cubrir los tres canales requeridos por el dominio. *Amazon SES* reemplaza a *SendGrid* (mencionado en el diagrama original) para alinear el stack de email con AWS y reducir el número de proveedores externos. *Twilio* es el único canal WhatsApp Business API disponible a escala de startup en Guatemala. *Firebase Cloud Messaging* es el estándar de facto para push en Android y iOS sin servidor propio.

| Dimensión | Amazon SES + Twilio + FCM | Amazon SNS + Pinpoint puro | SendGrid + Twilio + OneSignal |
|---|---|---|---|
| Gestionado / Self-hosted | Tres servicios SaaS gestionados | Totalmente gestionado dentro de AWS | Tres servicios SaaS externos fuera de AWS |
| Complejidad operativa | Media — tres SDKs distintos gestionados por el módulo de Mensajería via ACLs | Baja — todo en AWS; pero *Pinpoint* tiene curva de configuración para WhatsApp | Baja para cada servicio; complejidad en coordinar tres proveedores no-AWS |
| Costo a nuestra escala (<800 USD/mes) | SES: $0.10/1 000 emails; Twilio WhatsApp: ~$0.005/msg; FCM: gratuito | SNS email: $2/100k; Pinpoint SMS: ~$0.00645/SMS; sin WhatsApp nativo en la región | SendGrid gratis hasta 100 emails/día (limitado); OneSignal gratis hasta 10k suscriptores |
| Característica diferencial clave | SES: alta deliverability; Twilio: WhatsApp Business API oficial; FCM: push Android/iOS sin servidor | Integración nativa AWS; pero Pinpoint no cubre WhatsApp Business API en Guatemala de forma nativa | SendGrid tiene mayor reputación de deliverability que SES en algunos mercados; pero introduce 3 proveedores no-AWS |

## 4. Trade-offs

| Ventajas | Desventajas |
|---|---|
| Cubre los tres canales requeridos por el dominio (email, WhatsApp/SMS, push) sin servidor propio | Tres SDKs distintos (*@aws-sdk/client-ses*, *twilio*, *firebase-admin*) que el módulo de Mensajería encapsula via tres ACLs separados |
| *Amazon SES* en misma región AWS reduce latencia de entrega y simplifica la gestión de credenciales via IAM | *SES* requiere pasar por proceso de verificación de dominio y solicitar salida del sandbox de producción (retraso inicial de 1-3 días hábiles) |
| *Twilio* WhatsApp Business API es la ruta oficial y más documentada para startups en Latinoamérica | *Twilio* añade un proveedor externo adicional fuera de AWS; su disponibilidad no está ligada a la del stack de AWS |
| *Firebase Cloud Messaging* es gratuito y cubre Android e iOS con un único SDK sin servidor de push propio | Dependencia de *Google Firebase*; si Google depreca FCM v1, requiere actualización del módulo de Mensajería |
| El módulo de Mensajería es el único consumidor de eventos; cualquier fallo en notificaciones no afecta a los demás módulos | Reintentos fallidos de notificaciones deben gestionarse explícitamente en el módulo (o via DLQ de SQS) para evitar notificaciones duplicadas |

## 5. Integración con el resto del sistema

En la topología de [`04-deployment.md`](/diagrams/04-deployment.md), los tres proveedores de notificación se ubican en **Managed External Services** dentro del nodo `Mensajería & Logística`. El módulo de Mensajería del microkernel se comunica con ellos exclusivamente a través de ACLs (`ses.adapter.ts`, `twilio.adapter.ts`, `fcm.adapter.ts`), alineado con el patrón de integración documentado en [`/proposals/03-service-module-decomposition.md`](/proposals/03-service-module-decomposition.md).

El flujo de notificación completo para el evento `PedidoConfirmado` es:

1. El módulo de Pedidos publica `PedidoConfirmado` al *SNS topic* `pedidos-eventos` via el broker *SQS/SNS* ([`04-message-bus.md`](/infrastructure/04-message-bus.md)).
2. *SNS* entrega el mensaje a la cola SQS `mensajeria-queue`.
3. El módulo de Mensajería consume el evento y determina el canal preferido del destinatario según su perfil en *RDS* ([`05-primary-database.md`](/infrastructure/05-primary-database.md)).
4. Si el destinatario es un productor con WhatsApp configurado: el ACL `twilio.adapter.ts` envía el mensaje via *Twilio* WhatsApp Business API.
5. Si el destinatario es un comprador con app móvil: el ACL `fcm.adapter.ts` envía la notificación push via *Firebase Cloud Messaging*.
6. Adicionalmente para todos: el ACL `ses.adapter.ts` envía el email de confirmación via *Amazon SES*.
7. El resultado (éxito o fallo por canal) se registra en el esquema `messaging` de *RDS PostgreSQL*.

Los tokens FCM y las preferencias de canal del usuario se almacenan en caché transitoria en *ElastiCache for Redis* ([`03-cache.md`](/infrastructure/03-cache.md)) para evitar consultas repetidas a la base de datos en rafagas de notificaciones.

Nota sobre desviación del diagrama: el diagrama original en [`04-deployment.md`](/diagrams/04-deployment.md) menciona *SendGrid* como proveedor de email. Esta carpeta actualiza esa elección a *Amazon SES* para alinear el stack de email con AWS, reducir el número de proveedores externos y aprovechar la gestión de credenciales unificada via IAM. El módulo de Mensajería abstrae este cambio detrás del ACL `ses.adapter.ts`; el resto del sistema no se ve afectado.

Referencia ADR: [ADR-004](/adrs/ADR-004-event-bus.md) — el módulo de Mensajería está definido como "contexto terminal downstream: suscriptor de todos los eventos relevantes del dominio; no publica eventos propios."
