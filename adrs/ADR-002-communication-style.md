# ADR-002: Communication Style — HTTP sincrónico + eventos de dominio internos
|Field|Value|
|-|-|                                                    
| Status |Proposed| 
| Date |31-03-2026 | 
| Deciders | Pablo Siquinajay, Christian Sosa| 

## Contexto

### ¿Qué problemas estamos resolviendo?
**Debemos decidir:**
- Qué interacciones serán **sincrónicas** (HTTP, invocaciones directas vía servicios de aplicación).
- Qué interacciones serán **asíncronas** (eventos de dominio publicados en el bus interno o en un broker).

### ¿Restricciones existentes?
En CaféOrigen, la arquitectura microkernel se organiza en:

- Núcleo técnico compartido (HTTP, autenticación, bus de eventos interno).
- Módulos plug‑in por bounded context:
  - Identidad, Catálogo, Pedidos, Pagos, Logística, Mensajería, Reseñas.

Los flujos descritos en `04-data-flow-and-interactions.md` (registro de productor, pedido completo, pago fallido, reseñas) muestran interacciones entre estos contextos, los cuales <ins> se deben seguir</ins>:

- **Frontend ↔ Backend**: Registro/login, gestión de perfil, navegación de catálogo, creación de pedidos, reseñas, etc.
- **Entre módulos internos**:
  - Identidad → Catálogo/Pedidos (aprobación de productores).
  - Pedidos ↔ Pagos ↔ Logística ↔ Mensajería.
  - Pedidos → Reseñas → Catálogo.

Todos estos flujos deben ser validados con la mejor opción mejorando la optimización, eficiencia y correcto uso de las llamadas **sincrónicas**/**asíncronas** 

## Opciones

### Opción A — Todo sincrónico (solo HTTP/llamadas directas)

- El frontend llama controladores HTTP del backend.
- Dentro del backend, los módulos se comunican mediante:
  - Llamadas directas a servicios de aplicación (inyección de dependencias).
  - Sin uso de eventos de dominio como mecanismo principal de integración.

**Ventajas**

- Modelo mental sencillo: request–response en todas las direcciones.
- Depuración fácil: la pila de llamadas muestra todo el flujo.
- Con inyección de dependencias mejora la mantenibilidad en la selección del tipo de dependencia.

**Desventajas**

- Alto acoplamiento entre módulos.
- Difícil modelar procesos de larga duración (ej. disputa, reintento de pagos) sin un mecanismo explícito de eventos/estados.
- No refleja los patrones de integración ya descritos (OHS/PL, ACL, Partnership).

### Opción B — Todo asíncrono (event-driven para casi todo)

- Incluso interacciones sencillas (por ejemplo, obtener el resumen de un productor) se harían sólo mediante eventos o colas.
- El frontend dispararía comandos que publican eventos, y el backend respondería vía polling o websockets.

**Ventajas**

- Alto desacoplamiento, flexible a largo plazo.
- Facilita escalar hacia un modelo completamente distribuido en el futuro.

**Desventajas**

- **Sobre-ingeniería** para el volumen y equipo actual; es decir que la aplicación de este diseño excede lo necesario para cumplir su propósito, lo que puede resultar en costos adicionales y dificultades operativas.
- Aumenta latencia y complejidad para operaciones CRUD simples.
- Rompe la expectativa clásica del frontend (necesita respuestas inmediatas).

### Opción C — Modelo híbrido: HTTP sincrónico para comandos/consultas de corto ciclo + eventos de dominio para procesos de larga duración

- **Frontend ↔ Backend**:
  - Principalmente **HTTP REST** (y opcionalmente WebSockets para chat/notificaciones en tiempo real).
- **Dentro del backend (entre módulos)**:
  - Llamadas sincrónicas para consultas simples y validaciones de consistencia inmediata (por ejemplo, verificar disponibilidad de un lote).
  - **Eventos de dominio internos** (publicados en el bus del kernel) para:
    - Cambios de estado importantes de negocio (`PedidoRealizado`, `PagoAutorizado`, `ProductorVerificado`, `ReputacionActualizada`, etc.).
    - Procesos de larga duración (escrow, disputas, reintentos de pago).

**Ventajas**

- Mantiene latencia baja para operaciones críticas de UX (búsqueda de lotes, creación de pedidos).
- Modela de forma natural los procesos descritos en `04-data-flow-and-interactions.md`.
- Mantiene bajo el acoplamiento entre contextos, usando eventos como contrato.

**Desventajas**

- Añade complejidad moderada (gestión de eventos, idempotencia).
- Exige disciplina para no abusar de llamadas directas entre módulos.

## Decisión

 ### ¿Qué opción?

Se adopta la **Opción C — Modelo híbrido**:

 ### ¿Por qué se ajusta a nuestro proyecto específicamente?

Porque permite ajustar de manera medible y respectiva a flujos respectivos en vez de todo un tipo(ej. asíncrono) a un sistema con varios de los siguientes modulos y flujos: 
- **Frontend ↔ Backend**
  - APIs **REST** como interfaz principal:
    - `/auth/register`, `/auth/login`, `/productores`, `/lotes`, `/pedidos`, `/reseñas`, etc.(**Sincrónico**)
  - Canales **WebSocket** en el contexto de **Mensajería** para chat y notificaciones en tiempo real.(**Asíncrono**)
- **Entre módulos del backend**:
  - **Sincrónico (invocaciones directas o consultas HTTP internas)** cuando:
    - Se necesita una respuesta inmediata para validar un comando.
    - Ejemplo: Pedidos → Catálogo valida disponibilidad de `LoteCafe` antes de crear `Pedido`.
  - **Asíncrono (eventos de dominio internos)** como mecanismo principal de integración:
    - Identidad publica `ProductorVerificado` → Catálogo habilita lotes para ese productor.
    - Pedidos publica `PedidoRealizado` → Pagos crea `TransaccionPago` y `RetencionEscrow`.
    - Pagos publica `PagoAutorizado` → Pedidos cambia estado a `PAGO_RETENIDO`.
    - Pedidos publica `PedidoCompletado` → Reseñas habilita ventana de reseña → Catálogo ajusta ranking.

## Consecuencias

**Positivas**

- Mantiene buena UX (operaciones clave siguen modelo request–response).
- El diseño de flujos de `04-data-flow-and-interactions.md` encaja directamente con esta decisión, reforzando consistencia entre documentos.
- Eventos de dominio documentan claramente la evolución de los agregados de negocio.
- El bus de eventos interno puede, en el futuro, redirigirse a un broker externo si se extraen algunos plug‑ins a microservicios.

**Negativas / Riesgos**

- Es necesario definir políticas de **idempotencia** y **manejo de errores** para consumidores de eventos:
  - Reintentos, DLQ (cuando se use broker), logging consistente.
- Posibilidad de abuso de llamadas sincrónicas entre módulos, incrementando acoplamiento:
  - **Mitigación**: regla explícita de “integración primaria por eventos”; las llamadas directas se limitan a consultas/validaciones, no a orquestación de procesos.
- Los desarrolladores deben comprender bien qué se hace sincrónico y qué se hace asíncrono para evitar diseños inconsistentes.
