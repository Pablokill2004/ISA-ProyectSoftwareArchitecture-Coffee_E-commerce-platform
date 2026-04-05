# Diagramas de Arquitectura — CaféOrigen

Este directorio contiene los diagramas de arquitectura requeridos en la **Sección 3: Architecture Diagrams** de `Proyect_Instructions.md`. Todos los diagramas están en formato **Mermaid** para que puedan renderizarse directamente en GitHub.

## Índice

1. [`01-system-context.md`](01-system-context.md)  
   Diagrama de **contexto del sistema (C4 Level 1)**: muestra CaféOrigen como una caja única, los actores humanos y los sistemas externos (Stripe, SAT FEL, transportistas, Twilio/SendGrid, etc.), y las interacciones principales.

2. [`02-bounded-context-map.md`](02-bounded-context-map.md)  
   **Mapa de contextos delimitados (DDD Context Map)**: muestra los 7 bounded contexts definidos en `proposals/02-bounded-contexts.md`, su clasificación (Core / Supporting / Generic) y los patrones de integración (OHS/PL, Partnership, ACL, Conformista).

3. [`03-data-flow.md`](03-data-flow.md)  
   **Diagramas de secuencia** para los flujos clave descritos en `proposals/04-data-flow-and-interactions.md`:
   - Registro de usuario y verificación de productor.
   - Pedido completo de un lote (compra con escrow y entrega).
   - Pago fallido y reintento (incluye ruta de fallo).

4. [`04-deployment.md`](04-deployment.md)  
   Diagrama de **despliegue / infraestructura**: muestra la topología física y lógica de CaféOrigen con una arquitectura **microkernel** desplegada como un único backend, bases de datos por contexto, message broker y servicios gestionados en la nube.

## Estilo y convenciones

- Todos los diagramas usan **≤ 10 nodos principales** para mantener la legibilidad.
- Las flechas están **etiquetadas** con el tipo de interacción (HTTP, Webhook, Evento, etc.) y, cuando es relevante, con el payload principal.
- Los diagramas son **consistentes** con:
  - `01-cafeorigen_arquitectura.md` (decisión de microkernel vs microservicios).
  - `02-bounded-contexts.md` (bounded contexts y patrones de integración).
  - `03-service-module-decomposition.md` (microkernel con módulos plug‑in).
  - `04-data-flow-and-interactions.md` (flujos de datos).

Cada archivo incluye una **explicación escrita** de las decisiones más importantes.