# ADR-003: Database Strategy — Database-per-context en un solo cluster

|Field|Value|
|-|-|                                                    
| Status |Proposed| 
| Date |03-31-2026 | 
| Deciders | Pablo Siquinajay, Christian Sosa| 

## Contexto

### ¿Qué problemas estamos resolviendo?

CaféOrigen tiene 7 bounded contexts principales:

1. Identidad
2. Catálogo
3. Pedidos
4. Pagos
5. Logística
6. Mensajería
7. Reseñas

Cada módulo tiene su propio modelo de dominio (`entities`, `value-objects`, `events`), como se detalla en `03-service-module-decomposition.md`. 

El problema viene siendo que debemos definir <ins>cómo se organizará la persistencia</ins>:

- ¿Una sola base de datos compartida (“big schema”)?
- ¿Una base por servicio/contexto?
- ¿Motores distintos según el tipo de dato (polyglot persistence)?

### ¿Restricciones existentes?

- Equipo pequeño, sin un DBA dedicado.
- Presupuesto limitado de infraestructura.
- Necesidad de integridad razonable (órdenes, pagos, FEL).
- Flujo de datos principalmente dentro de un único backend (microkernel).

## Opciones

### Opción A — Base de datos única compartida por todos los contextos

- Un solo motor (por ejemplo, PostgreSQL) con un único esquema donde todos los módulos escriben y leen.
- Tablas mezcladas de `Identidad`, `Catálogo`,`Pagos`, etc.

**Ventajas**

- Operación muy sencilla (un solo motor, un solo endpoint).
- Transacciones multi-contexto triviales (dentro de la misma base).

**Desventajas**

- **Alto acoplamiento de datos**:
  - Si la BD cae, todo el sistema se desploma.
  - Dificultad para cambiar modelos de un contexto sin afectar a otros.
- Escalado más complejo a largo plazo (migrar un solo contexto a otra base es costoso).
- Tolerancia a fallos prácticamente nulo.

### Opción B — Database-per-context en un mismo motor/cluster

- Utilizar un mismo motor de base de datos (por ejemplo, PostgreSQL gestionado), pero:
  - Un **esquema por contexto** (`identity`, `catalog`, `orders`, `payments`, etc.).
  - O bien múltiples bases lógicas dentro del mismo cluster.
- Cada módulo accede solo a su esquema/base; no se permite acceso directo a datos de otros contextos.

**Ventajas**

- Aísla modelos de datos por contexto, alineado con DDD.
- Mantiene **simplicidad operativa** (un motor gestionado, backups centralizados).
- Permite futuras migraciones granulares (extraer Pagos a un motor dedicado, por ejemplo).
- Facilita aplicar políticas distintas (índices, particiones) por contexto.

**Desventajas**

- Introduce cierta complejidad en configuración de conexiones (múltiples esquemas/bases).
- Transacciones cruzando contextos se vuelven más complicadas;
  - Se debe favorecer consistencia eventual mediante eventos en lugar de transacciones distribuidas.

### Opción C — Polyglot persistence (motores distintos por contexto)

- Elegir el “mejor” tipo de base para cada contexto:
  - Relacional para Pagos y Pedidos.
  - Documental para Catálogo.
  - Time-series para logs/observabilidad.
  - Key-value para sesiones, etc.

**Ventajas**

- Modelo optimizado por caso de uso:
  - Búsquedas de catálogo rápidas en un motor de búsqueda.
  - Pagos fuertemente consistentes en relacional.

**Desventajas**

- Mayor complejidad operativa (múltiples motores que administrar, monitorear, respaldar).
- Equipo pequeño podría no tener la capacidad y experiencia para gestionar tantos sistemas con diferentes políticas(Relacional vs Documental vs Time-Series).
- Costos de infraestructura y de mantenimiento más altos, difícil de justificar en primeras fases.

## Decisión

 ### ¿Qué opción?

Se adopta la **Opción B — Database-per-context en un mismo motor/cluster**.

 ### ¿Por qué se ajusta a nuestro proyecto específicamente?

- Se utilizará un **motor relacional gestionado** (por ejemplo, PostgreSQL en la nube) como base principal.
- Cada contexto tendrá:
  - Un **esquema** propio (`identity`, `catalog`, `orders`, `payments`, `logistics`, `messaging`, `reviews`).
  - O, si el proveedor lo recomienda, una base lógica separada dentro del mismo cluster.
- Las integraciones entre contextos se basarán en:
  - Eventos de dominio [ver ADR-002](ADR-002-communication-style.md#decisión).
  - DTOs publicados (“published language”) expuestos por APIs internas, no por lecturas directas de tablas de otros contextos. [ver más sobre "published language”](/proposals/02-bounded-contexts.md#contexto-1-identidad-y-verificación) en el campo
  *Patrón de Integración*

## Consecuencias

**Positivas**

- Conserva la **separación de modelos** y respeta los bounded contexts definidos.
- <ins>Simplifica</ins> la operación en comparación con [polyglot persistence](#opción-c--polyglot-persistence-motores-distintos-por-contexto):
  - Un cluster, un tipo de base principal.
- Permite, en el futuro, migrar contextos específicos (por ejemplo, Catálogo a un motor de búsqueda adicional) sin rediseñar todo.
- Mitiga la tentación de usar la [BD como un todo](#opción-a--base-de-datos-única-compartida-por-todos-los-contextos).

**Negativas / Riesgos**

- Las transacciones que abarcan varios contextos deberán rediseñarse como procesos con **consistencia eventual**:
  - Por ejemplo: Pedido + Pago + Logística se coordinan por eventos. Ya que todos estos se involucran necesariamente para un mismo fin; *Una transacción del cliente con el sistema al momento de pagar por un producto de café*.
- Puede ser necesario configurar pools de conexión separados por esquema/base:
  - **Mitigación**: usar una librería ORM con buen soporte multischema (por ejemplo, TypeORM con múltiples conexiones o esquemas).
- El equipo debe tener disciplina para no romper la regla de “cada contexto sólo accede a su esquema”.