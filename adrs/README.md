# README.md

# Architecture Decision Records (ADRs) — CaféOrigen

Este directorio captura las decisiones arquitectónicas importantes: el contexto que la motivó, las opciones evaluadas, la decisión tomadas, consecuencias, etc.

## Tabla de ADRs

1. [**ADR-001 – Deployment Model (Microkernel en un solo contenedor)**](/adrs/ADR-001-deployment-model.md)
   - Se define el modelo de despliegue del backend de CaféOrigen como una aplicación monolítica modular (microkernel + plug‑ins) en uno o varios contenedores idénticos detrás de un load balancer.

2. [**ADR-002 – Communication Style (Sincrónico HTTP + eventos internos)**](/adrs/ADR-002-communication-style.md)
   - Se decide cómo se comunican los módulos/bounded contexts dentro del microkernel y qué se expone al frontend, combinando HTTP sincrónico con eventos de dominio internos.

3. [**ADR-003 – Database Strategy (Database-per-context en un solo cluster)**](/adrs/ADR-003-database-strategy.md)
   - Se establece una estrategia de bases de datos separadas por contexto (esquemas o bases lógicas) dentro de un mismo motor/cluster, balanceando independencia de modelo con simplicidad operativa.

4. [**ADR-004 – Event / Message Bus (Bus de eventos interno basado en la aplicación + broker ligero)**](/adrs/ADR-004-event-bus.md)
   - Se define el uso de un bus de eventos de dominio en memoria dentro del backend, combinado con un broker ligero (por ejemplo, *RabbitMQ*) sólo cuando se requiere durabilidad y desacoplamiento con sistemas externos.

5. [**ADR-005 – Authentication & Authorization (Proveedor gestionado + RBAC por rol de usuario)**](/adrs/ADR-005-authentication.md)
   - Se decide usar un proveedor de identidad gestionado para autenticación (OIDC) y un modelo de autorización basado en roles (comprador, productor, admin, soporte), centralizado en el kernel.
