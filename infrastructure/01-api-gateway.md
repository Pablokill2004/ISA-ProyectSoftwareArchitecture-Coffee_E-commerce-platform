# 01 — API Gateway

|Field|Value|
|-|-|
| Componente | API Gateway |
| Elección | Amazon API Gateway (HTTP API + WebSocket API) |
| Fecha | 10-04-2026 |
| Autores | Pablo Siquinajay, Christian Sosa |

## 1. Responsabilidades

- Expone los endpoints REST del microkernel (`/api/v1/...`) al exterior como una fachada HTTPS gestionada, sin requerir que cada contenedor del backend gestione TLS directamente.
- Establece y mantiene conexiones WebSocket persistentes para el chat en tiempo real del módulo de Mensajería (`/proposals/03-service-module-decomposition.md`), enrutando mensajes hacia las réplicas del backend.
- Aplica *throttling* y cuotas por API key para proteger el sistema ante picos de tráfico no autenticado (p.ej. scraping del catálogo de lotes).
- Valida los tokens OIDC emitidos por *Amazon Cognito* mediante un *authorizer* Lambda nativo antes de que la petición llegue al kernel de NestJS, reduciendo la carga de autenticación en el backend.
- Proporciona un punto de entrada único y versionable, lo que permite introducir la ruta `/api/v2/...` sin tocar el código del microkernel.

## 2. Por qué este proyecto lo necesita

CaféOrigen expone dos protocolos distintos al mismo tiempo: REST para todas las operaciones de dominio (crear pedidos, publicar lotes, consultar estados) y WebSocket para el chat comprador-productor que vive en el módulo de Mensajería. Sin un API Gateway, cada réplica stateless del microkernel tendría que gestionar TLS, throttling y autenticación de tokens de forma redundante, lo que aumentaría la complejidad del código de aplicación y la superficie de ataque.

Además, el mercado de café de especialidad tiene flujos de acceso irregulares: cuando un lote de variedad excepcional se publica en el catálogo, el tráfico puede multiplicarse en minutos. El throttling gestionado a nivel de API Gateway protege el backend y la base de datos de estos picos sin necesidad de sobreprovisionar instancias permanentemente, lo cual es crítico con un presupuesto de infraestructura por debajo de los 800 USD/mes.

## 3. Elección tecnológica

Se eligió *Amazon API Gateway* (modalidad HTTP API) por su integración nativa con el resto del ecosistema AWS que usa el proyecto, su modelo de precios por llamada (sin coste fijo), y la disponibilidad de WebSocket API en el mismo servicio, lo que evita añadir un componente separado para el chat de Mensajería.

| Dimensión | Amazon API Gateway | Kong | NGINX self-hosted |
|---|---|---|---|
| Gestionado / Self-hosted | Totalmente gestionado (AWS) | Gestionado (Kong Konnect) o self-hosted | Self-hosted en EC2/contenedor |
| Complejidad operativa | Muy baja — sin servidores que mantener | Media — requiere instancia de control plane | Alta — parcheado, HA, config manual |
| Costo a nuestra escala (<800 USD/mes) | ~$1/millón de llamadas HTTP; muy bajo a baja escala | Tier gratuito limitado; costo de instancia EC2 adicional | Solo costo de instancia EC2 (~$15-30/mes t3.small) |
| Característica diferencial clave | WebSocket API nativo + authorizer Cognito sin código extra | Plugin ecosystem muy rico para lógica avanzada | Control total sobre configuración de red y SSL |

## 4. Trade-offs

| Ventajas | Desventajas |
|---|---|
| Sin servidores que mantener ni parches de seguridad | Vendor lock-in con AWS; migrar a otro gateway requiere reconfiguración |
| WebSocket API y HTTP API en un único servicio | Latencia adicional de ~1-2 ms por ser un proxy gestionado externo |
| Authorizer nativo con *Cognito* elimina validación de tokens en el kernel | Límites de tamaño de payload (10 MB) y tiempo de conexión WebSocket (2 horas) |
| Escalado automático sin configuración — absorbe picos del catálogo | Coste puede escalar linealmente en campañas de alto tráfico sin presupuesto fijo |
| Trazabilidad integrada con *AWS CloudWatch* y *X-Ray* | Debugging de WebSocket más complejo que HTTP estándar |

## 5. Integración con el resto del sistema

En la topología descrita en [`04-deployment.md`](/diagrams/04-deployment.md), *Amazon API Gateway* ocupa la **Public Zone** como el único punto de entrada desde Internet hacia la VPC interna. Recibe el tráfico HTTPS del `Browser / App Móvil` y lo enruta hacia el *Application Load Balancer* (componente [`02-load-balancer.md`](/infrastructure/02-load-balancer.md)), que a su vez distribuye la carga entre las réplicas del microkernel en el App Tier.

Para tráfico WebSocket, el API Gateway mantiene la sesión del cliente y reenvía los frames al backend a través del ALB. El módulo de Mensajería del kernel gestiona el estado de conexión en el nivel de aplicación.

El *authorizer* de *Amazon Cognito* ([`06-authentication-provider.md`](/infrastructure/06-authentication-provider.md)) se conecta directamente al API Gateway; las peticiones que no portan un JWT válido son rechazadas con HTTP 401 antes de llegar al App Tier, aliviando la guardia `auth.guard.ts` del kernel para los endpoints que requieren autenticación.

Referencia ADR: esta elección no está cubierta por un ADR propio, pero es coherente con [ADR-001](/adrs/ADR-001-deployment-model.md) (stateless backend + load balancer) y [ADR-005](/adrs/ADR-005-authentication.md) (validación de tokens en el kernel centralizado).
