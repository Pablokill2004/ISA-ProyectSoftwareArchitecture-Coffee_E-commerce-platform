# 01 — System Context (C4 Level 1)

Este diagrama muestra a **CaféOrigen** como un solo sistema de backend (arquitectura **microkernel** con plug‑ins), los actores humanos y los sistemas externos con los que interactúa. No se muestran componentes internos ni bounded contexts; el foco es el “big picture”.

## Diagrama de contexto del sistema

```mermaid
C4Context
title CaféOrigen - System Context (Level 1)

Person(Producer, "Productor de café", "Pequeños agricultores y cooperativas que publican lotes de café.")
Person(Buyer, "Comprador de café de especialidad", "Tostadores, cafeterías y compradores que adquieren lotes de café.")
Person(Admin, "Administrador de CaféOrigen", "Equipo interno que verifica productores, gestiona disputas y monitorea la plataforma.")

System_Boundary(CafeOrigenBoundary, "CaféOrigen Platform (Microkernel)") {
  System(CafeOrigenBackend, "Backend CaféOrigen", "Aplicación backend monolítica modular (microkernel + plug‑ins) que orquesta identidad, catálogo, pedidos, pagos, logística, mensajería y reseñas.")

  System_Ext(FrontendWeb, "Web / Mobile SPA", "Aplicación web/móvil (por ejemplo, React/Next.js) que consumen las APIs de CaféOrigen.")
}

System_Ext(Stripe, "Stripe Connect", "Proveedor de pagos y escrow.")
System_Ext(SATFEL, "SAT FEL", "Servicio de facturación electrónica guatemalteca.")
System_Ext(Shippers, "Transportistas / AfterShip / Flexport", "APIs de logística y tracking de envío.")
System_Ext(TwilioSendgrid, "Twilio / SendGrid / Firebase", "Proveedores de mensajería (WhatsApp/SMS/email/push).")

Rel(Producer, FrontendWeb, "Publica lotes, conversa con compradores, gestiona pedidos", "HTTPS")
Rel(Buyer, FrontendWeb, "Busca lotes, realiza pedidos, paga y deja reseñas", "HTTPS")
Rel(Admin, FrontendWeb, "Verifica productores, revisa pedidos y disputas", "HTTPS")

Rel(FrontendWeb, CafeOrigenBackend, "Llama APIs REST/GraphQL para todas las operaciones", "HTTPS/JSON")

Rel(CafeOrigenBackend, Stripe, "Autorización/captura de pagos escrow; recepción de webhooks de eventos de pago", "HTTPS (REST/Webhooks)")
Rel(CafeOrigenBackend, SATFEL, "Emite facturas electrónicas FEL", "HTTPS (REST)")
Rel(CafeOrigenBackend, Shippers, "Crea envíos y recibe actualizaciones de tracking", "HTTPS (REST/Webhooks)")
Rel(CafeOrigenBackend, TwilioSendgrid, "Envía mensajes y notificaciones multicanal", "HTTPS (REST APIs)")
```

## Explicación

- **Un solo backend (CaféOrigenBackend)**  
  El backend se despliega como **un único proceso / contenedor** que implementa una arquitectura **microkernel**:  
  - El núcleo orquesta peticiones, autenticación, eventos internos y configuración.  
  - Los plug‑ins implementan los bounded contexts: Identidad, Catálogo, Pedidos, Pagos, Logística, Mensajería y Reseñas.

- **Actores humanos**  
  - **Productor**: crea su perfil, envía documentos para verificación y publica lotes.  
  - **Comprador**: explora el catálogo, realiza pedidos, paga vía escrow y deja reseñas.  
  - **Administrador**: verifica productores, revisa disputas de pedidos/pagos y monitorea la operación.

- **Sistemas externos**  
  - **Stripe Connect** para pagos escrow.  
  - **SAT FEL** para facturación fiscal obligatoria en Guatemala.  
  - **Transportistas / AfterShip / Flexport** para logística y tracking.  
  - **Twilio / SendGrid / Firebase** para mensajería y notificaciones.

Este nivel de contexto enfatiza que, aunque a futuro algunos plug‑ins podrían extraerse como microservicios, en la fase actual el sistema se opera como una **plataforma única** con integraciones bien encapsuladas hacia el exterior.