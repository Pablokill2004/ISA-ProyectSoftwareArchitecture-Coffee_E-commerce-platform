# 01 — Arquitectura de Alto Nivel

## CaféOrigen: Marketplace de Café de Comercio Directo

### 1. Contexto del Proyecto

CaféOrigen es un marketplace que conecta a productores de café (pequeños agricultores y cooperativas en Centroamérica) directamente con compradores de café de especialidad en todo el mundo. Los productores muestran sus plantaciones, publican lotes con puntajes de catación y certificaciones, y negocian ventas con tostadores, importadores y consumidores individuales, eliminando intermediarios tradicionales.

**Restricciones clave:**

- 3–5 ingenieros durante los primeros 12–18 meses  
- ~500–2,000 usuarios en el primer año, creciendo a ~20,000 para el tercer año  
- Picos de tráfico durante la temporada de cosecha (octubre–marzo)  
- Debe soportar facturación fiscal guatemalteca (FEL), pagos con escrow y notificaciones multicanal (WhatsApp, correo, SMS)  
- Presupuesto de infraestructura menor a $800/mes en el primer año  

---

### 2. Enfoque A — Microservicios con Arquitectura Basada en Eventos

Este enfoque divide el sistema en servicios desplegables de forma independiente — Catálogo, Órdenes, Pagos, Mensajería, Logística e Identidad — cada uno con su propia base de datos y comunicación principalmente mediante eventos asíncronos.

Cuando un comprador realiza una orden, el servicio de Órdenes no llama directamente al servicio de Pagos. En su lugar, publica un evento `OrderPlaced` en un bus de eventos central (como Amazon EventBridge o RabbitMQ). El servicio de Pagos consume ese evento, crea una retención de fondos mediante Stripe y publica `PaymentAuthorized`. Los servicios de Logística y Mensajería también reaccionan a estos eventos de forma independiente.

Las llamadas síncronas se reservan para operaciones sensibles a la latencia, como verificar disponibilidad de un lote.

Cada servicio se ejecuta en su propio contenedor (AWS ECS Fargate o Lambda), con su propio pipeline CI/CD, base de datos y configuración de escalado. Un API Gateway gestiona las solicitudes externas.

---

### 3. Enfoque B — Arquitectura Microkernel (Plug-in)

Este enfoque separa el sistema en un núcleo y un conjunto de plug-ins.

El núcleo contiene la lógica genérica del marketplace:
- Autenticación de usuarios  
- Motor de listado de productos  
- Máquina de estados de órdenes  
- Orquestación de pagos  
- Sistema de notificaciones  

El núcleo define interfaces, pero no conoce detalles específicos del dominio.

Los plug-ins agregan lógica específica:
- Catálogo de café (altitud, variedad, puntaje de catación)  
- Pagos escrow con Stripe  
- Facturación FEL  
- Notificaciones por WhatsApp  
- Flujo de solicitud de muestras  

Cuando se realiza una orden, el núcleo ejecuta el proceso y llama a los plug-ins correspondientes. Todo ocurre dentro de una sola aplicación.

El sistema completo se despliega como un único contenedor. Los plug-ins pueden activarse o desactivarse por configuración.

---

### 4. Comparación

| Dimensión | Microservicios + Eventos | Microkernel (Plug-in) |
|----------|--------------------------|------------------------|
| Complejidad de despliegue | Alta | Baja |
| Tamaño del equipo | Requiere varios equipos | Ideal para 3–5 ingenieros |
| Escalabilidad | Excelente | Moderada |
| Aislamiento de fallos | Fuerte | Débil |
| Velocidad inicial | Lenta | Rápida |
| Velocidad a largo plazo | Alta por equipo | Estable |
| Costo infraestructura | ~$800–1,500/mes | ~$150–300/mes |
| Complejidad operativa | Alta | Baja |
| Extensibilidad | Compleja | Simple (plug-ins) |

---

### 5. Recomendación: Microkernel

CaféOrigen debería adoptar la arquitectura microkernel por tres razones principales:

Primero, el modelo de expansión encaja perfectamente. El crecimiento es por país. Cada mercado tiene reglas distintas, pero la lógica central es la misma. El microkernel permite reutilizar el núcleo y adaptar mediante plug-ins.

Segundo, el equipo es pequeño. Mantener múltiples microservicios sería costoso en tiempo y esfuerzo. El microkernel reduce la complejidad operativa.

Tercero, el presupuesto es limitado. Microservicios excederían el presupuesto. El microkernel permite operar con costos significativamente menores.

El principal riesgo es el aislamiento de fallos, ya que un plug-in defectuoso puede afectar todo el sistema. Esto se mitiga con controles de errores y circuit breakers.

Si el sistema crece, los plug-ins pueden evolucionar hacia servicios independientes, facilitando una transición futura.
