# 02 — Load Balancer

|Field|Value|
|-|-|
| Componente | Load Balancer |
| Elección | AWS Application Load Balancer (ALB) |
| Fecha | 10-04-2026 |
| Autores | Pablo Siquinajay, Christian Sosa |

## 1. Responsabilidades

- Distribuye el tráfico HTTP/2 entrante desde el *Amazon API Gateway* entre las N réplicas stateless del microkernel (contenedores *NestJS* en *AWS ECS Fargate*) usando el algoritmo round-robin o least-connections.
- Ejecuta health checks periódicos hacia el endpoint `/health` de cada contenedor; elimina automáticamente del pool las réplicas que no responden, sin intervención del equipo.
- Permite *connection draining* durante despliegues: las conexiones activas se completan antes de retirar el contenedor antiguo, garantizando cero interrupciones durante releases.
- Enruta tráfico WebSocket de manera transparente usando sticky sessions opcionales para la conexión del módulo de Mensajería, sin romper los frames WS en medio de una sesión de chat.
- Actúa como frontera de red entre la Public Zone y la VPC privada (App Tier), asegurando que los contenedores del backend nunca estén expuestos directamente a Internet.

## 2. Por qué este proyecto lo necesita

El diseño del microkernel exige contenedores stateless que escalen horizontalmente ([ADR-001](/adrs/ADR-001-deployment-model.md)). Sin un load balancer, escalar a dos o más réplicas del backend requeriría gestionar el enrutamiento manualmente o exponerlas directamente, lo que rompería la premisa de stateless y complicaría los despliegues sin tiempo de inactividad.

En el contexto de CaféOrigen, los picos de tráfico son predecibles (lanzamiento de lotes de café de especialidad, campañas estacionales de cosecha) pero de magnitud desconocida para un equipo de 3–5 ingenieros. El ALB absorbe ese escalado sin código adicional en el backend: basta con ajustar el *desired count* de tareas en *ECS Fargate* y el balanceador las incorpora al pool en segundos.

## 3. Elección tecnológica

Se eligió *AWS Application Load Balancer* por ser el componente nativo de AWS para balanceo de carga en capa 7 (HTTP/HTTPS/WebSocket), integrado directamente con *ECS Fargate* y *API Gateway*, con soporte de health checks, connection draining y métricas en *CloudWatch* sin configuración adicional.

| Dimensión | AWS ALB | NGINX self-hosted | HAProxy self-hosted |
|---|---|---|---|
| Gestionado / Self-hosted | Totalmente gestionado (AWS) | Self-hosted en EC2/contenedor | Self-hosted en EC2/contenedor |
| Complejidad operativa | Muy baja — configurable por consola o IaC | Alta — HA requiere al menos 2 instancias + keepalived | Alta — similar a NGINX; curva de config más pronunciada |
| Costo a nuestra escala (<800 USD/mes) | ~$16/mes fijo + $0.008 por LCU; predecible | Costo de 2 instancias EC2 t3.small (~$30-60/mes) + gestión | Similar a NGINX en EC2 |
| Característica diferencial clave | Integración nativa con ECS + target groups + WebSocket sin plugins | Máxima flexibilidad y control de routing rules | Rendimiento muy alto en TCP/L4; más potente en L4 que en L7 |

## 4. Trade-offs

| Ventajas | Desventajas |
|---|---|
| Cero servidores que mantener; AWS garantiza disponibilidad multi-AZ | Vendor lock-in; reglas de routing propietarias del ALB |
| Health checks y deregistration delay configurables sin tocar el backend | Menor flexibilidad que NGINX para lógica de routing avanzada (reescritura de URLs complejas) |
| Integración directa con *ECS Fargate* target groups via service discovery | Coste fijo mínimo (~$16/mes) incluso sin tráfico |
| Métricas de latencia, bytes transferidos y errores 5xx disponibles en *CloudWatch* sin agentes extra | Límite de 100 reglas de listener por ALB (no es un problema a nuestra escala) |
| Connection draining nativo facilita despliegues blue/green sin pérdida de conexiones | No soporta L4 (TCP puro); si en el futuro se necesita protocolo binario, requeriría NLB adicional |

## 5. Integración con el resto del sistema

En la topología de [`04-deployment.md`](/diagrams/04-deployment.md), el *AWS ALB* se ubica como puente entre la **Public Zone** y el **App Tier** de la VPC privada. Recibe tráfico exclusivamente desde *Amazon API Gateway* ([`01-api-gateway.md`](/infrastructure/01-api-gateway.md)) y lo distribuye entre los contenedores `Backend Container 1` y `Backend Container 2` (y cualquier réplica adicional que *ECS Auto Scaling* levante).

Los contenedores del microkernel están registrados en un *ECS Fargate* target group; cuando el módulo de *ECS Auto Scaling* levanta una nueva tarea, el ALB la incorpora automáticamente al pool tras superar el health check en `/health`. Durante un despliegue, el *connection draining* de 30 segundos (configurable) garantiza que los pedidos en curso (`PedidoRealizado`, `PagoAutorizado`) no sean interrumpidos a mitad de transacción.

Referencia ADR: coherente con [ADR-001](/adrs/ADR-001-deployment-model.md) — "varios contenedores idénticos del backend detrás de un load balancer; el backend es stateless; el estado persistente vive en las bases de datos y en el broker de mensajes".
