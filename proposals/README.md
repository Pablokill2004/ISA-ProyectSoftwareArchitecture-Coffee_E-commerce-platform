# Proposals

Esta carpeta contiene propuestas de arquitectura para **CaféOrigen**, un marketplace de café de comercio directo.  
Cada documento desarrolla un aspecto específico del diseño (arquitectura, DDD, módulos y flujos).

## Quick links

- [01 — Arquitectura de Alto Nivel](./01-cafeorigen_arquitectura.md)
- [02 — Contextos Delimitados (DDD)](./02-bounded-contexts.md)
- [03 — Descomposición en Módulos y Organización de Código](./03-service-module-decomposition.md)
- [04 — Flujos de Datos e Interacciones](./04-data-flow-and-interactions.md)

---

## Descripción de archivos

### `01-cafeorigen_arquitectura.md`
Propuesta de **arquitectura de alto nivel** para CaféOrigen. Compara dos enfoques:
- **Microservicios + eventos**
- **Microkernel (plug-ins)**
y recomienda **microkernel** considerando tamaño de equipo, costos y complejidad operativa.

### `02-bounded-contexts.md`
Descomposición del dominio usando **DDD (Bounded Contexts)**. Define contextos (Identidad, Catálogo, Pedidos, Pagos, Logística, Mensajería, Reseñas), sus entidades/eventos, dependencias upstream/downstream y patrones de integración (OHS/PL, ACL, Partnership). Incluye un **mapa de contextos** en Mermaid.

### `03-service-module-decomposition.md`
Traduce los bounded contexts a una **organización de código** bajo una arquitectura **microkernel** (ejemplo con TypeScript/NestJS). Propone árbol de directorios, responsabilidades del kernel compartido y límites de importación entre módulos.

### `04-data-flow-and-interactions.md`
Describe **flujos de datos e interacciones** entre módulos mediante escenarios y **diagramas de secuencia (Mermaid)**: registro/verificación de productor, ciclo completo de pedido con escrow, pago fallido y reintento, y flujo de reseñas/reputación.
