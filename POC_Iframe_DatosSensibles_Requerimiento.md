# POC — Iframe de Visualización de Datos Sensibles de Tarjeta

## Contexto

Credibanco necesita un componente iframe PCI DSS v4.0 compliant que permita a los tarjetahabientes visualizar de forma segura los datos sensibles de su tarjeta prepago (PAN completo, fecha de vencimiento, CVV2) desde el Portal Tarjetahabiente, sin que el portal padre (dominio de la entidad emisora) tenga acceso a estos datos.

Este iframe es parte de la plataforma Prepago On-Prem y se integra con el Portal Tarjetahabiente existente (Angular 17+), el backend Spring Boot, y el HSM Thales payShield para descifrado de datos.

### Diagrama de referencia
- Archivo: `Prepago/Iframe_Visualizacion_DatosSensibles.drawio`
- Flujo de PIN relacionado: `Prepago/pinpad_pci_drawio_6_mejorado.drawio`

---

## Arquitectura de la POC

### Componentes a construir

```
┌─────────────────────────────────────────────────────────────────┐
│ Portal TH (Angular)          │ Iframe Card Data (Angular)      │
│ http://localhost:4200         │ http://localhost:4300            │
│ Dominio A — FUERA del CDE    │ Dominio B — DENTRO del CDE      │
│                               │                                  │
│ • Botón "Ver datos tarjeta"  │ • Recibe postMessage(cardId)    │
│ • Carga iframe               │ • Solicita datos al backend     │
│ • postMessage { cardId,      │ • Renderiza PAN/exp/CVV2        │
│   sessionToken }             │ • Protecciones anti-copia       │
│ • Recibe notificación        │ • Auto-oculta en 30s            │
│   CARD_DATA_HIDDEN           │ • Borra DOM y memoria           │
└───────────┬───────────────────┴──────────────┬──────────────────┘
            │ postMessage (sin datos sensibles) │ HTTPS REST
            │                                    │
┌───────────┴────────────────────────────────────┴──────────────────┐
│ Backend Card Data API (Spring Boot 3 + Java 21)                   │
│ http://localhost:8080                                              │
│                                                                    │
│ • POST /api/card-data/request                                     │
│   - Valida sessionToken                                           │
│   - Valida MFA (simulado en POC)                                  │
│   - Genera viewToken (UUID, TTL 30s, 1 uso)                      │
│   - Rate limit: 3 views/tarjeta/hora                              │
│                                                                    │
│ • GET /api/card-data/view/{viewToken}                             │
│   - Consume viewToken (1 solo uso)                                │
│   - Descifra PAN del HSM Simulator                                │
│   - Retorna { pan, exp, cvv2 }                                   │
│   - Headers: Cache-Control: no-store, no-cache                    │
│   - CORS: solo origin del iframe                                  │
│                                                                    │
│ • HSM Simulator (módulo interno para POC)                         │
│   - Simula descifrado AES-256 del PAN                             │
│   - En producción: Thales payShield cmd M2/M4                     │
│                                                                    │
│ • Base de datos: H2 in-memory (POC)                               │
│   - Tabla: card (card_id, pan_encrypted, exp_date, cvv2_encrypted)│
│   - Tabla: view_token (token, card_id, used, created_at, ttl)     │
│   - Tabla: view_audit_log (card_id, ip, user_agent, timestamp)    │
└───────────────────────────────────────────────────────────────────┘
```

---

## User Stories

### US-01: Portal carga iframe en dominio separado
**Como** tarjetahabiente autenticado en el portal,
**quiero** presionar "Ver datos de mi tarjeta",
**para que** se cargue un iframe seguro donde pueda ver mis datos.

**Criterios de aceptación:**
- El portal carga un iframe apuntando a `localhost:4300/card-view`
- El iframe tiene atributo `sandbox="allow-scripts allow-same-origin"`
- El portal envía `postMessage({ cardId, sessionToken, action: 'VIEW_CARD_DATA' })` al iframe
- El portal valida `event.origin` en las respuestas del iframe
- El portal NUNCA recibe PAN, CVV2 ni fecha de vencimiento en ningún mensaje

### US-02: Iframe valida origen y solicita datos
**Como** iframe de datos sensibles,
**quiero** validar que el mensaje viene del portal autorizado,
**para** evitar que un sitio malicioso solicite datos de tarjeta.

**Criterios de aceptación:**
- El iframe valida `event.origin === 'http://localhost:4200'` (configurable)
- El iframe valida `event.source === window.parent`
- Si el origin no coincide, ignora el mensaje y loguea el intento
- El iframe hace POST `/api/card-data/request` con `{ cardId, sessionToken }`
- El backend retorna un `viewToken` (UUID, TTL 30s, un solo uso)
- El iframe hace GET `/api/card-data/view/{viewToken}` para obtener los datos

### US-03: Backend valida sesión y genera viewToken
**Como** backend de datos sensibles,
**quiero** validar la sesión del usuario y generar un token de visualización efímero,
**para** asegurar que solo usuarios autenticados vean datos de tarjeta.

**Criterios de aceptación:**
- Valida que `sessionToken` sea válido (en POC: token hardcoded o JWT simple)
- Valida que `cardId` pertenezca al usuario de la sesión
- Genera `viewToken` UUID con TTL 30 segundos y flag `used = false`
- Rate limiting: máximo 3 visualizaciones por tarjeta por hora (en POC: contador en memoria)
- Si rate limit excedido, retorna HTTP 429
- Log de auditoría: `card_id`, timestamp, IP (sin PAN ni CVV2 en logs)

### US-04: Backend descifra y entrega datos sensibles
**Como** backend de datos sensibles,
**quiero** descifrar el PAN y CVV2 almacenados y entregarlos al iframe,
**para** que el tarjetahabiente pueda ver sus datos de forma segura.

**Criterios de aceptación:**
- Valida que `viewToken` exista, no esté usado, y no haya expirado (TTL 30s)
- Marca `viewToken` como usado (un solo uso — idempotente)
- Descifra PAN con HSM Simulator (AES-256-GCM en POC)
- Descifra CVV2 con HSM Simulator
- Retorna JSON: `{ pan, exp, cvv2 }`
- Headers obligatorios:
  - `Cache-Control: no-store, no-cache, must-revalidate`
  - `Pragma: no-cache`
  - `X-Content-Type-Options: nosniff`
  - `X-Frame-Options: DENY` (el endpoint no debe cargarse en iframe directo)
- CORS: `Access-Control-Allow-Origin: http://localhost:4300` (solo el iframe)
- Si viewToken ya fue usado o expiró: HTTP 403
- PAN y CVV2 NUNCA se loguean (ni en debug)

### US-05: Iframe renderiza datos con PAN enmascarado por defecto
**Como** tarjetahabiente,
**quiero** ver mis datos de tarjeta con el PAN enmascarado inicialmente,
**para** tener control sobre cuándo revelo el número completo.

**Criterios de aceptación:**
- Vista inicial muestra:
  - Número de tarjeta: `**** **** **** 1234` (solo últimos 4 visibles)
  - Fecha de vencimiento: `12/28` (visible, no es dato PCI sensible por sí solo)
  - CVV2: `•••` (oculto por defecto)
- Botón "👁 Ver número completo" revela el PAN full
- Botón "👁 Ver CVV" revela el CVV2
- Cada revelación es independiente (puede ver PAN sin ver CVV y viceversa)
- Los datos NO se almacenan en localStorage, sessionStorage, ni cookies

### US-06: Protecciones anti-copia en el iframe
**Como** sistema de seguridad,
**quiero** dificultar la copia de datos sensibles del iframe,
**para** reducir el riesgo de exfiltración de datos de tarjeta.

**Criterios de aceptación:**
- CSS: `user-select: none; -webkit-user-select: none`
- JS: `oncopy`, `oncut`, `onpaste`, `oncontextmenu`, `ondragstart` → `return false`
- El texto de PAN/CVV2 no es seleccionable con el mouse
- Click derecho deshabilitado dentro del iframe
- Arrastrar texto deshabilitado
- Nota: screenshots no se pueden prevenir (limitación del navegador)

### US-07: Auto-ocultar datos después de 30 segundos
**Como** sistema de seguridad,
**quiero** que los datos sensibles se oculten automáticamente después de 30 segundos,
**para** minimizar el tiempo de exposición.

**Criterios de aceptación:**
- Timer de 30 segundos inicia al revelar PAN o CVV2 (el que se revele primero)
- Indicador visual de countdown (barra de progreso o texto "Se ocultará en Xs")
- Al expirar:
  1. PAN vuelve a `**** **** **** 1234`
  2. CVV2 vuelve a `•••`
  3. Variables JS sobreescritas: `pan = '0'.repeat(16); cvv = '000'`
  4. Nodos DOM con datos sensibles eliminados y recreados vacíos
- Envía `postMessage({ action: 'CARD_DATA_HIDDEN', reason: 'TIMEOUT' })` al portal padre
- Para ver de nuevo: requiere nuevo viewToken (nueva llamada al backend)

### US-08: Manejo de errores
**Como** tarjetahabiente,
**quiero** ver mensajes claros cuando algo falla,
**para** saber qué hacer.

**Criterios de aceptación:**
- viewToken expirado → "Sesión expirada. Presione 'Ver datos' nuevamente"
- viewToken ya usado → "Esta solicitud ya fue procesada"
- Rate limit excedido → "Ha alcanzado el límite de visualizaciones. Intente en 1 hora"
- Error de red → "Error de conexión. Intente nuevamente"
- Backend no disponible → "Servicio no disponible temporalmente"
- Ningún mensaje de error expone datos técnicos internos

---

## Stack Técnico para la POC

| Componente | Tecnología |
|---|---|
| Portal TH (simulado) | Angular 17+ standalone, puerto 4200 |
| Iframe Card Data | Angular 17+ standalone, puerto 4300 |
| Backend API | Java 21 + Spring Boot 3.2+ + Spring Security |
| Base de datos | H2 in-memory (POC) → Oracle 19c en producción |
| HSM Simulator | Módulo Java con `javax.crypto` AES-256-GCM |
| Rate limiting | Caffeine cache in-memory (POC) → Redis en producción |
| Build | Maven multi-module o proyectos separados |

---

## Estructura de proyecto sugerida

```
poc-card-data-iframe/
├── portal-th/                    # Angular 17+ (puerto 4200)
│   └── src/app/
│       ├── card-view/            # Componente con botón + iframe
│       └── app.component.ts
│
├── iframe-card-data/             # Angular 17+ (puerto 4300)
│   └── src/app/
│       ├── card-display/         # Componente de visualización
│       ├── services/
│       │   └── card-data.service.ts
│       └── guards/
│           └── origin-validator.ts
│
└── backend-card-data/            # Spring Boot 3 (puerto 8080)
    └── src/main/java/
        ├── controller/
        │   └── CardDataController.java
        ├── service/
        │   ├── CardDataService.java
        │   ├── ViewTokenService.java
        │   └── HsmSimulator.java
        ├── model/
        │   ├── Card.java
        │   └── ViewToken.java
        ├── config/
        │   ├── CorsConfig.java
        │   └── SecurityConfig.java
        └── repository/
            ├── CardRepository.java
            └── ViewTokenRepository.java
```

---

## Datos de prueba (seed)

```sql
-- Tarjetas de prueba (PAN cifrado con AES-256-GCM, llave en HSM Simulator)
INSERT INTO card (card_id, pan_encrypted, last_four, exp_date, cvv2_encrypted, status)
VALUES (1, '{encrypted}', '1234', '12/28', '{encrypted}', 'ACTIVE');

INSERT INTO card (card_id, pan_encrypted, last_four, exp_date, cvv2_encrypted, status)
VALUES (2, '{encrypted}', '5678', '06/27', '{encrypted}', 'ACTIVE');

-- PAN en claro para referencia (SOLO en este doc, nunca en código):
-- Card 1: 4532 0123 4567 1234
-- Card 2: 5412 7534 9821 5678
```

---

## Criterios de éxito de la POC

1. El portal padre (localhost:4200) NUNCA tiene acceso al PAN, CVV2 ni datos sensibles
2. Los datos sensibles solo existen dentro del DOM del iframe (localhost:4300)
3. Same-origin policy impide que el portal lea el DOM del iframe (verificable en DevTools)
4. El PAN se muestra enmascarado por defecto y se revela solo con acción explícita
5. Los datos se auto-ocultan después de 30 segundos
6. El viewToken es de un solo uso y expira en 30 segundos
7. Rate limiting funciona (4ta solicitud en 1 hora retorna 429)
8. Los headers de no-cache están presentes en la respuesta del backend
9. Las protecciones anti-copia funcionan (no se puede seleccionar/copiar texto)
10. Los logs del backend NO contienen PAN ni CVV2

---

## Qué NO incluye la POC (diferido a producción)

- Autenticación real SSO/MFA (simulada con token hardcoded)
- HSM Thales real (simulado con javax.crypto AES-256-GCM)
- CVV2 dinámico vía Pomelo API (se usa CVV2 estático cifrado)
- Oracle 19c (se usa H2 in-memory)
- Redis para rate limiting (se usa Caffeine in-memory)
- CSP headers reales con SRI hashes (se documenta la configuración)
- Certificados TLS reales (se usa HTTP en localhost para la POC)
- OpenShift deployment (se ejecuta local con `ng serve` y `mvn spring-boot:run`)

---

## Referencia PCI DSS v4.0

| Requisito | Cómo se cumple en la POC |
|---|---|
| Req 3.3 | PAN enmascarado por defecto, revelación temporal 30s |
| Req 3.4 | No localStorage/sessionStorage/cookies/cache |
| Req 4.2.1 | TLS entre iframe y backend (HTTP en POC, TLS en prod) |
| Req 6.4.3 | Iframe en dominio separado, CSP documentado |
| Req 7.2 | viewToken de un solo uso, rate limiting |
| Req 8.3.6 | MFA simulado (real en producción) |
| Req 12.3.1 | TTL documentado: viewToken 30s, visualización 30s |
