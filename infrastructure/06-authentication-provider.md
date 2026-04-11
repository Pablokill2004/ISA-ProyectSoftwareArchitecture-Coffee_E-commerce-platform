# 06 — Authentication Provider

|Field|Value|
|-|-|
| Componente | Authentication Provider |
| Elección | Amazon Cognito |
| Fecha | 10-04-2026 |
| Autores | Pablo Siquinajay, Christian Sosa |

## 1. Responsabilidades

- Gestiona el registro, login, recuperación de contraseña y verificación de correo electrónico de los tres tipos de usuario de CaféOrigen: `COMPRADOR`, `PRODUCTOR` y `ADMIN/SOPORTE`.
- Emite tokens de acceso (*JWT*) y tokens de refresco conformes al estándar OIDC, que el `auth.guard.ts` del kernel de NestJS valida usando la JWKS pública de Cognito sin llamadas adicionales al IdP en cada petición.
- Almacena en los *custom attributes* del *User Pool* los roles del sistema (`custom:role`) que se incluyen en el payload del JWT y que el kernel usa para aplicar el modelo RBAC (`@Roles('ADMIN')`, `@Roles('PRODUCTOR')`).
- Integra el flujo de verificación de productor: cuando el módulo de Identidad marca a un usuario como `ProductorVerificado`, actualiza el atributo `custom:producerStatus` en Cognito; el token del siguiente login ya contiene el nuevo estado sin cambios en el backend.
- Provee *Hosted UI* configurable para el flujo de registro y login, permitiendo añadir federación con Google u otros proveedores sociales en el futuro sin cambios en el backend.

## 2. Por qué este proyecto lo necesita

Construir un sistema de autenticación seguro desde cero — hashing de contraseñas, gestión de sesiones, rotación de claves, protección contra ataques de fuerza bruta, cumplimiento de buenas prácticas OWASP — es una responsabilidad que excede la capacidad de un equipo de 3–5 ingenieros enfocado en el dominio del café de especialidad. Un error en la implementación propia podría exponer datos financieros de productores o credenciales de compradores, con consecuencias legales y reputacionales graves para el marketplace.

Adicionalmente, el modelo de autorización de CaféOrigen es complejo: no solo hay roles (`COMPRADOR`, `PRODUCTOR`, `ADMIN`), sino también estados de verificación (productores deben ser aprobados por un admin antes de publicar lotes) y reglas de negocio por módulo (solo el productor dueño de un lote puede modificarlo). Un proveedor gestionado OIDC permite que el kernel centralice la autenticación y exponga un `IUserContext` consistente a todos los módulos, tal como define [ADR-005](/adrs/ADR-005-authentication.md).

## 3. Elección tecnológica

Se eligió *Amazon Cognito* por su integración nativa con el ecosistema AWS del proyecto (ALB, API Gateway, Lambda), su nivel gratuito generoso (50 000 MAU gratuitos) y la compatibilidad directa con el estándar OIDC/JWT sin necesidad de plugins adicionales.

| Dimensión | Amazon Cognito | Auth0 | Keycloak self-hosted |
|---|---|---|---|
| Gestionado / Self-hosted | Totalmente gestionado (AWS) | Totalmente gestionado (SaaS) | Self-hosted (contenedor o EC2) |
| Complejidad operativa | Muy baja — sin servidores que gestionar | Muy baja — SaaS completo | Alta — actualización, HA, base de datos propia |
| Costo a nuestra escala (<800 USD/mes) | Gratis hasta 50 000 MAU/mes; luego $0.0055/MAU | Gratis hasta 7 500 MAU en plan Free; plan Essentials ~$23/mes para más | Sin costo de licencia; costo de instancia EC2/contenedor (~$15-30/mes) |
| Característica diferencial clave | Integración nativa con ALB authorizer y API Gateway JWT authorizer sin Lambda | Dashboard más amigable; Rules/Actions con código para transformar tokens | Control total sobre claims, protocolos y extensiones; ideal para entornos regulados on-premise |

## 4. Trade-offs

| Ventajas | Desventajas |
|---|---|
| 50 000 MAU gratuitos: suficiente para toda la etapa de lanzamiento del marketplace | UX de la Hosted UI básica por defecto; personalizar el aspecto visual requiere trabajo de CSS adicional |
| JWT authorizer nativo en *Amazon API Gateway* — tokens validados antes de llegar al backend ([`01-api-gateway.md`](/infrastructure/01-api-gateway.md)) sin Lambda extra | *Custom attributes* de Cognito tienen limitaciones: máximo 25 atributos, no se pueden eliminar una vez creados |
| JWKS rotación automática — el kernel valida tokens con la clave pública sin gestionar secretos | Migrar a otro IdP en el futuro requiere un flujo de migración de usuarios (Cognito no exporta contraseñas hasheadas) |
| Soporte de MFA (TOTP, SMS) activable para roles `ADMIN` y `SOPORTE` sin cambios en el backend | La integración con el módulo de Identidad (sincronizar `ProductorVerificado` en Cognito) requiere un flujo de actualización de atributos via Admin API |
| Integración directa con *AWS WAF* para protección adicional del *User Pool* | Algunos flujos avanzados (p.ej. *Magic Link login*) no están disponibles nativamente y requieren Lambda triggers |

## 5. Integración con el resto del sistema

En la topología de [`04-deployment.md`](/diagrams/04-deployment.md), *Amazon Cognito* se ubica en **Managed External Services** (o como dependencia del API Gateway en la Public Zone). No reside en la VPC privada, sino que es un servicio AWS gestionado al que el backend y el API Gateway llaman via HTTPS.

El flujo de autenticación en CaféOrigen funciona de la siguiente manera:

1. El cliente (browser/app móvil) se autentica contra el *Cognito User Pool* (via Hosted UI o SDK) y recibe un JWT de acceso.
2. El cliente envía el JWT en el header `Authorization: Bearer <token>` a *Amazon API Gateway* ([`01-api-gateway.md`](/infrastructure/01-api-gateway.md)).
3. El JWT Authorizer del API Gateway valida el token contra la JWKS pública de Cognito; si es inválido, devuelve 401 sin llegar al backend.
4. Si es válido, el API Gateway reenvía la petición al ALB ([`02-load-balancer.md`](/infrastructure/02-load-balancer.md)) con los claims del token en un header `X-User-Context`.
5. El `auth.guard.ts` del kernel extrae el `IUserContext` del header y lo pone a disposición de los módulos; los roles del token (`custom:role`) se usan para evaluar `@Roles()` decorators.
6. Cuando el módulo de Identidad aprueba un productor, llama a la Cognito Admin API para actualizar `custom:producerStatus = verified`; el próximo token del usuario ya refleja el nuevo estado.

Referencia ADR: [ADR-005](/adrs/ADR-005-authentication.md) — "Proveedor de identidad gestionado (OIDC) + RBAC centralizado. El kernel valida tokens y expone `IUserContext`; la autorización fina vive en cada módulo."
