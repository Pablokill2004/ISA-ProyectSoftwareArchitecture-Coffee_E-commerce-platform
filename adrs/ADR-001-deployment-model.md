
# ADR-001: Deployment Model — Microkernel en un solo contenedor

|Field|Value|
|-|-|                                                    
| Status |Accepted| 
| Date |03-30-2026 | 
| Deciders | Pablo Siquinajay, Christian Sosa| 

## Contexto

### ¿Qué problemas estamos resolviendo?

Se define en este docuemento el ¿Porqué elegimos Micro-kernel Architecture? para el modelo de despliegue, considerando otras opciones existentes, conociento a profundidad las ventajas y desventajas de cáda uno y nuestro análisis final que colmó el veredicto final(ver la sección de [*Decisión*](#Decisión) de este documento)

### ¿Restricciones existentes?
- Equipo de **3–5 ingenieros** durante los primeros 12–18 meses.
- Volumen estimado:
  - ~500–2,000 usuarios en el primer año.
  - Crecimiento a ~20,000 usuarios para el tercer año.
  - Picos de tráfico durante la temporada de cosecha (octubre–marzo).
- Requisitos funcionales clave:
  - Facturación fiscal guatemalteca (FEL).
  - Pagos escrow con Stripe.
  - Notificaciones multicanal (WhatsApp, correo, SMS).
- Presupuesto de infraestructura **< 800 USD/mes** en el primer año.
- Arquitectura lógica ya definida como **microkernel (núcleo + plug‑ins)** en un solo proceso (`03-service-module-decomposition.md`):
  - Núcleo técnico con autenticación, orquestación de pagos/órdenes, bus de eventos interno, HTTP API.
  - Módulos plug‑in: identidad, catálogo, pedidos, pagos, logística, mensajería, reseñas.



Es necesario definir cómo se despliega físicamente este backend: ¿un monolito en un solo contenedor? ¿varios servicios independientes? ¿serverless?

## Opciones

### Opción A — Microservicios completos por bounded context

- Cada contexto (`Identidad`, `Catálogo`, `Pedidos`, `Pagos`, `Logística`, `Mensajería`, `Reseñas`) como servicio independiente, con su propio repositorio, despliegue y pipeline.
- Comunicación principalmente asíncrona mediante eventos (bus externo) + llamadas HTTP/gRPC entre servicios.
- Despliegue en orquestador de contenedores (Kubernetes o ECS) con autoescalado por servicio.

**Ventajas**

- Escalabilidad fina: se escala de forma independiente el servicio de Pagos o Logística si crece la carga.
- Aislamiento de fallos: caída de Mensajería no tumba Catálogo.
- Autonomía de equipos: distintos equipos podrían desplegar independientemente.

**Desventajas**

- **Complejidad operativa alta** para un equipo de 3–5 personas: 
  - Múltiples repositorios/pipelines/monitoreo.
  - Coordinación de versiones y contratos.
- Riesgo de exceder presupuesto de infraestructura (múltiples servicios, balanceadores, colas, etc.).
- Overhead de latencia y depuración en flujos distribuidos.
- Contradice la recomendación previa de arquitectura microkernel para la fase inicial.

### Opción B — Monolito tradicional sin separación clara de módulos

- Una sola aplicación backend con un conjunto plano de paquetes/modulos, sin límites estrictos por contexto.
- Un contenedor/proceso único.
- Una sola base de datos compartida, modelos mezclados.

**Ventajas**

- Simplicidad de despliegue y de tubería CI/CD.
- Fácil de arrancar al inicio del proyecto.
- Lo más económico que podrímos obtener.

**Desventajas**

- Fuerte **acoplamiento** entre dominios: difícil de evolucionar por países/regulaciones.
- Riesgo de “big ball of mud”: reglas de negocio de pagos, logística, reseñas, etc. mezcladas; **MUY CONFUSO**, **ARQUITECTURA DESORGANIZADA**
- Dificulta una posible migración futura a microservicios o separación física por contexto(Micro-kernel).


### Opción C — Microkernel en un solo backend desplegado como contenedor/es idénticos

- Implementar el diseño microkernel:
  - Núcleo técnico (auth, HTTP, bus de eventos interno, logging, config).
  - Módulos plug‑in internos (`/modules/identity`, `/modules/catalog`, etc.).
- Backend desplegado como:
  - **Un único artefacto** (imagen de contenedor) que contiene kernel + plug‑ins.
  - Varios contenedores idénticos (stateless) detrás de un load balancer para escalado horizontal.
- Frontend SPA/móvil consume APIs REST/GraphQL de este backend.

**Ventajas**

- Mantiene **baja complejidad operativa**: un solo servicio que desplegar/monitorear.
- Compatible con el presupuesto (infraestructura de 150–300 USD/mes es razonable).
- Permite **evolucionar** en el futuro:
  - Módulos críticos (Pagos, Mensajería) pueden extraerse a servicios independientes cuando la escala lo justifique.
- Despliegue simple usando contenedores en un servicio gestionado (ECS Fargate, App Service, etc.).

**Desventajas**

- Aislamiento de fallos débil: un bug en un plug‑in puede tumbar el proceso completo.
- Escalabilidad menos fina: todo el backend escala junto.
- El tamaño de la imagen y el tiempo de despliegue pueden crecer con los años si no se gestiona bien.

## Decisión

 ### ¿Qué opción?

Elegimos la **Opción C — Microkernel en un solo backend desplegado como contenedor/es idénticos**.

 ### ¿Por qué se ajusta a nuestro proyecto específicamente?

Porque se ajusta muy bien a las ventajas ya dichas

- Permite **evolucionar** en el futuro:
  - Módulos críticos (Pagos, Mensajería) pueden extraerse a servicios independientes cuando la escala lo justifique.

La escalabilidad en el marketplace es algo  muy probable que necesita tener una base cimentada, es decir, a diferencia de monolito, si se escala, será mucho más costoso ent tiempo-presupuesto implementarlo que un micro-kernel.

También debido a la escasa mano de obra(devs), nos vendría muy bien por la otra ventaja de **baja complejidad operativa**, lo que nos sutentará durante varios meses incluso años. Como se mencionó, si se lo amerita, ya con un mejor presupuesto y alta demanda, será más facil trasladarse a microservicios(mucho más complejo).

En despliegue:

- Se construye **una imagen de contenedor**.
- Se ejecutan **N instancias idénticas** (stateless) detrás de un load balancer.
- El estado persistente vive en bases de datos separadas por contexto (ver ADR-003).

## Consecuencias

**Positivas**

- Operación y despliegue simples: un solo servicio backend que monitorear.
- Equipo de 3–5 ingenieros puede hacerse cargo de todo el stack sin un equipo de plataforma dedicado.
- Costos de infraestructura ajustados al presupuesto inicial (<800 USD/mes).
- Mantiene el diseño modular por contextos, facilitando extensiones específicas por país/regulación vía nuevos plug‑ins.
- Facilita una futura migración de plug‑ins críticos a microservicios independientes, si el crecimiento lo exige.

**Negativas / Riesgos + Mitigación**

- Un fallo no manejado en un plug‑in puede tumbar todo el backend:
  - **Mitigación**: manejo de errores centralizado, timeouts, circuit breakers internos, tests de regresión.
- Menor elasticidad por contexto:
  - Si sólo Pagos aumenta de carga, todo el backend escala.
  - **Mitigación**: monitorear métricas por módulo y planificar extracción a servicio independiente en el futuro.
- Tamaño creciente del despliegue con el tiempo:
  - **Mitigación**: disciplina de modularización, refactor periódico y limpieza de plug‑ins obsoletos.

