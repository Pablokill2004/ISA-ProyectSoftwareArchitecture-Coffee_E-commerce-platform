# 02 — Bounded Context Map (DDD)


Este diagrama resume los **7 contextos delimitados** definidos en `proposals/02-bounded-contexts.md`, agrupados por tipo de dominio (**Core / Supporting / Generic**) y etiquetando cada relación con su **patrón de integración**.  
Para mantener el diagrama legible (≤10 nodos principales), los distintos sistemas externos se agrupan en dos nodos lógicos:  
- **Sistemas de Pagos y Facturación** (Stripe, SAT FEL).  
- **Sistemas de Mensajería y Logística** (Twilio, SendGrid, Firebase, transportistas).

## Diagrama de mapa de contextos

```mermaid
flowchart TB
    %% Agrupaciones por tipo de dominio
    subgraph CoreDomain["Core Domain"]
        Pedidos["Pedidos"]
        Catalogo["Catálogo"]
        Reseñas["Reseñas y Reputación"]
    end

    subgraph Supporting["Supporting Subdomains"]
        Logistica["Logística"]
        Mensajeria["Mensajería y Notificaciones"]
    end

    subgraph Generic["Generic Subdomains"]
        Identidad["Identidad y Verificación"]
        Pagos["Pagos"]
    end

    %% Relaciones internas con patrones de integración

    %% Identidad como upstream via OHS/PL
    Identidad -->|"OHS/PL"| Catalogo
    Identidad -->|"OHS/PL"| Pedidos
    Identidad -->|"OHS/PL"| Mensajeria

    %% Partnership entre Catálogo y Pedidos
    Catalogo <-->|"Partnership"| Pedidos

    %% Pedidos downstream hacia Pagos y Logística (OHS/PL)
    Pedidos -->|"OHS/PL"| Pagos
    Pedidos -->|"OHS/PL"| Logistica

    %% Feedback de Pagos y Logística hacia Pedidos (OHS/PL)
    Pagos -->|"OHS/PL"| Pedidos
    Logistica -->|"OHS/PL"| Pedidos

    %% Eventos de Pedidos hacia Mensajería y Reseñas
    Pedidos -->|"OHS/PL"| Mensajeria
    Pedidos -->|"OHS/PL"| Reseñas

    %% Reseñas influye en Catálogo (Partnership)
    Reseñas -->|"Partnership"| Catalogo

    %% Sistemas externos agrupados (menos nodos)
    ExternalPayments["Sistemas de Pagos y Facturación\n(Stripe, SAT FEL)"]
    ExternalCommsLog["Sistemas de Mensajería y Logística\n(Twilio, SendGrid, Firebase, Transportistas)"]

    Pagos -.->|"ACL / Conformista"| ExternalPayments
    Logistica -.->|"ACL"| ExternalCommsLog
    Mensajeria -.->|"ACL"| ExternalCommsLog
```

## Leyenda

- **Tipos de dominio**
  - **Core Domain**:  
    - **Pedidos**: corazón del modelo de negocio (flujo de compra, estados de pedido, disputas).  
    - **Catálogo**: discovery y gestión de lotes de café de especialidad.  
    - **Reseñas y Reputación**: confianza y reputación, clave para el valor del marketplace.
  - **Supporting Subdomains**:  
    - **Logística**: envíos y tracking, crítico pero no único del negocio de CaféOrigen.  
    - **Mensajería**: chat y notificaciones que dan soporte al flujo principal.
  - **Generic Subdomains**:  
    - **Identidad**: autenticación, verificación de productores; dominio genérico pero con reglas propias del contexto.  
    - **Pagos**: lógica de pagos y facturación, basada fuertemente en servicios externos (Stripe, SAT FEL).

- **Patrones de integración**
  - **OHS/PL (Open Host Service / Published Language)**:  
    Un contexto expone una interfaz estable y un lenguaje publicado (DTOs, eventos) que otros contextos consumen.  
    Ejemplo: Identidad → Catálogo/Pedidos/Mensajería con `ResumenUsuarioDTO`, `EstadoProductorDTO`.
  - **Partnership**:  
    Dos contextos co‑evolucionan y requieren coordinación cercana.  
    Ejemplo: Catálogo ↔ Pedidos (disponibilidad de lotes y creación/cancelación de pedidos).  
    Ejemplo: Reseñas → Catálogo (reputación afecta ranking de lotes).
  - **ACL (Anti-Corruption Layer)**:  
    Capa de traducción que protege el modelo de dominio de las peculiaridades de APIs externas.  
    En el diagrama esto se refleja en las relaciones hacia:  
    - **Sistemas de Pagos y Facturación** (Stripe, SAT FEL).  
    - **Sistemas de Mensajería y Logística** (Twilio, SendGrid, Firebase, transportistas).
  - **Conformista**:  
    CaféOrigen debe ajustarse al modelo de un tercero sin negociación, como SAT FEL para la facturación fiscal (incluido dentro del nodo agrupado de Pagos y Facturación).

## Explicación

Este mapa refuerza:

- La decisión de arquitectura **microkernel**: cada bounded context se implementa como un **módulo plug‑in** dentro de un solo backend, pero mantiene **fronteras claras de dominio**.
- La orientación **event-driven interna** descrita en `02-bounded-contexts.md` y `04-data-flow-and-interactions.md`:  
  - Pedidos actúa como **emisor central** de eventos de negocio (`PedidoRealizado`, `PedidoConfirmado`, etc.).  
  - Pagos, Logística, Mensajería y Reseñas reaccionan a esos eventos sin acoplarse directamente a las entidades de Pedidos.
- Los contextos **Generic** y **Supporting** son buenos candidatos a ser extraídos como servicios independientes en una fase futura, sin romper el lenguaje ubícuo ni los límites actuales.
- La **agrupación de sistemas externos en el diagrama** (Pagos/Facturación por un lado, Mensajería/Logística por otro) refleja cómo se encapsulan detrás de ACLs técnicas dentro de los contextos de Pagos, Logística y Mensajería, sin cambiar la semántica de los bounded contexts de negocio.