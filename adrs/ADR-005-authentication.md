# ADR-005: Authentication & Authorization — Proveedor gestionado + RBAC por rol

|Field|Value|
|-|-|                                                    
| Status |Accepted| 
| Date |05-04-2026 | 
| Deciders | Pablo Siquinajay, Christian Sosa| 

## Contexto

### ¿Qué problemas estamos resolviendo?

CaféOrigen debe autenticar y autorizar a distintos tipos de usuarios:

- **Compradores** (tostadores, importadores, consumidores) que navegan el catálogo y crean pedidos.
- **Productores** que publican lotes, gestionan envíos y responden mensajes.
- **Admins internos** que revisan documentos, gestionan disputas, controlan reputación.


**Debemos decidir:**

- **Proveedor de identidad**: self-managed vs. managed (Auth0, Cognito, etc.).
- **Modelo de autorización**: RBAC vs. ABAC, centralizado vs. disperso.

>En palabras sencillas:
*¿Cómo sabemos quién es el usuario y qué puede o no puede hacer en el e‑commerce de café?*



### ¿Restricciones existentes?

- Restringir funciones críticas:
  - Sólo productores verificados pueden publicar lotes.
  - Sólo admins pueden aprobar productores, emitir reembolsos manuales, etc.
- Y el equipo es pequeño, no experto en seguridad.

## Opciones

### Opción A — Autenticación y gestión de identidades autogestionada

- Implementar en el propio backend:
  - Registro, hash de contraseñas, recuperación
  - Emisión y validación de JWTs(esencial para APIs REST seguras y protección de sesiones de usuario)
  - Gestión de sesiones, bloqueo de cuentas, etc.

**Ventajas**

- <ins>Control total</ins> sobre el flujo de autenticación.
- Sin costos adicionales por usuario en proveedores externos.

**Desventajas**

- Alta responsabilidad en seguridad: hashing, rotación de claves, prevención de ataques.
- Incrementa el riesgo del proyecto si no se hace con mucha disciplina.
- Equipo pequeño con foco en dominio de negocio, no en construir un IdP; Además, <ins>el equipo no es experto en seguridad tampoco.</ins>

### Opción B — Proveedor de identidad gestionado (OIDC) + RBAC centralizado

***OIDC**: protocolo de autenticación de identidad para estandarizar el proceso de autenticación y autorización de usuarios cuando inician sesión para acceder a los servicios digitales.*

***Modelo RBAC**: El control de acceso basado en roles (RBAC) es un modelo para autorizar el acceso del usuario final a sistemas, aplicaciones y datos basado en el rol predefinido del usuario.*

***IdP**: Proveedor de identidad, almacena y gestiona las identidades digitales de los usuarios.*

- Usar un proveedor de identidad gestionado (p. ej. Auth0, AWS Cognito, Azure AD B2C, etc.) para:
  - Registro/login, federación social si se requiere.
  - Emisión de tokens OIDC/JWT.
  - Almacenamiento de usuarios básicos.
- El backend de CaféOrigen:
  - Valida tokens entrantes en el **kernel** (`auth.guard.ts`).
  - Extrae claims (id de usuario, roles, atributos).
  - Aplica un **modelo RBAC**:
    - Roles: `COMPRADOR`, `PRODUCTOR`, `ADMIN`, `SOPORTE`.
    - Verificaciones adicionales (p. ej. `ProductorVerificado`) se gestionan en el módulo de Identidad.

**Ventajas**

- Reduce la superficie de riesgo en temas de seguridad.
- Alineado con un equipo pequeño que no puede construir un IdP robusto de cero.
- Integración clara con el microkernel:
  - El kernel maneja la autenticación y expone `IUserContext` a los módulos.

**Desventajas**

- Costos por MAU (monthly active users) en algunos proveedores.
- Dependencia de un tercero (lock-in parcial).
- Se debe diseñar mapeo entre claims del IdP y el modelo de usuario interno (Identidad).

### Opción C — Sin autenticación centralizada, sólo tokens API por contexto

- Cada contexto exige su propia autenticación simple (API keys, tokens compartidos, etc.).
- No hay un concepto unificado de usuario final; cada módulo maneja identidad a su manera.

**Ventajas**

- Extremadamente simple para servicios backend–backend.

**Desventajas**

- Inviable para un producto B2B/B2C con usuarios humanos.
- No satisface los requerimientos de roles, verificación de productores, etc.

## Decisión

### ¿Qué opción?

Se adopta la **Opción B — Proveedor de identidad gestionado (OIDC) + RBAC centralizado**.

### ¿Por qué se ajusta a nuestro proyecto específicamente?

- La **autenticación** de usuarios finales se delega a un IdP gestionado que soporta OIDC.
- El **kernel** del backend:
  - Valida los tokens de acceso en una capa de middleware/guard.
  - Expone un `IUserContext` con:
    - `userId` (identificador interno enlazado a `Usuario` en el módulo de Identidad).
    - `roles` (`COMPRADOR`, `PRODUCTOR`, `ADMIN`, `SOPORTE`).
    - Metadatos relevantes (país, idioma, etc. si aplica).
- La **autorización** se implementa en dos niveles:
  1. **RBAC genérico en el kernel**:
     - Decoradores o guards por rol (`@Roles('ADMIN')`, etc.).
  2. **Reglas de negocio específicas en cada módulo**:
     - Por ejemplo, el módulo de Catálogo consulta el estado de productor en Identidad antes de permitir publicar un lote.

> Es exactamente lo que un micro‑kernel busca.

## Consecuencias

**Positivas**

- Disminuye significativamente el riesgo de errores de seguridad graves en autenticación.
- Encaja con la estructura del microkernel: la autenticación vive en el núcleo, mientras que la verificación de productores y demás reglas viven en el contexto de Identidad.
- Facilita exponer APIs seguras tanto a:
  - Frontend SPA/móvil.
  - Herramientas internas (panel admin).
- Simplifica la implementación de RBAC:
  - Cada endpoint declara qué rol(es) puede acceder.
  - Las reglas finas se aplican a nivel de dominio (por ejemplo, que un productor sólo pueda modificar sus propios lotes).

**Negativas / Riesgos**

- Dependencia en un proveedor externo de identidad:
  - **Mitigación**: diseñar la integración a través de una abstracción en el kernel, de modo que el IdP pueda cambiarse con impacto acotado.
- Costos variables por volumen de usuarios:
  - **Mitigación**:Debe considerarse en el presupuesto operativo.
- Es necesario sincronizar/relacionar la identidad externa (IdP) con el modelo interno (`Usuario`):
  - **Mitigación**:Se debe diseñar un flujo claro de provisión y enlace (por ejemplo, crear/actualizar `Usuario` en Identidad al confirmar el registro desde el IdP).
