# Requerimientos — Onboarding de Tarjetahabiente

Documento puntual del flujo de onboarding (pre-registro + activación de cuenta en el portal) para tarjetahabientes de la plataforma prepago Credibanco.

**Fuentes consolidadas:**
- `Prepago/Requerimientos Técnicos (12).docx` — capítulos "Portal Web Prepago - Back (Onboarding)" y "Portal Web Prepago - Front (Onboarding)" (Etapas 2 y 4 de Fase 1 MVP).
- `Prepago/Onboarding_Tarjetahabiente.drawio` — diagrama de proceso por fases (swimlanes).
- `Prepago/PrepagoUnificadoArquitectura_V_1_5.drawio` — pestaña "Onboarding Tarjetahabiente" (vista C4 L2).
- `Prepago/docs/decisiones-arquitectura.md` — ADR-016, ADR-023, ADR-024.
- `Prepago/docs/requerimientos-ciclo-vida-tarjeta-y-portal-th.md` — sección 6.

> Última actualización: 14/05/2026
> Estado: **Vigente** — Fase 1 MVP, Etapa 4

---

## Tabla de contenido

1. [Resumen y alcance](#1-resumen-y-alcance)
2. [Disparadores del onboarding](#2-disparadores-del-onboarding)
3. [Flujo funcional por fases](#3-flujo-funcional-por-fases)
4. [Componentes y responsabilidades](#4-componentes-y-responsabilidades)
5. [Contratos de API](#5-contratos-de-api)
6. [Modelo de datos](#6-modelo-de-datos)
7. [Reglas de negocio](#7-reglas-de-negocio)
8. [Estados y máquinas de estado](#8-estados-y-máquinas-de-estado)
9. [Casos de error y manejo](#9-casos-de-error-y-manejo)
10. [Seguridad y cumplimiento](#10-seguridad-y-cumplimiento)
11. [Tarjetas innominadas (exclusión)](#11-tarjetas-innominadas-exclusión)
12. [Notificaciones y plantillas](#12-notificaciones-y-plantillas)
13. [Auditoría](#13-auditoría)
14. [Performance y SLAs](#14-performance-y-slas)
15. [Pendientes](#15-pendientes)
16. [Referencias cruzadas](#16-referencias-cruzadas)

---

## 1. Resumen y alcance

### 1.1 Objetivo

Permitir que un tarjetahabiente (TH) cuya tarjeta nominada fue creada en el sistema active su cuenta en el Portal Tarjetahabiente, asignando una contraseña tras verificar su identidad mediante cédula y OTP enviada al correo registrado.

### 1.2 Alcance funcional (Fase 1 MVP)

**Incluye:**
- Pre-registro disparado automáticamente al crear una tarjeta nominada.
- Email transaccional al TH con link único (UUID v4).
- Verificación de identidad (tipo y número de documento contra el cliente asociado a la tarjeta).
- Generación, envío y validación de OTP de 6 dígitos.
- Asignación de contraseña por parte del TH.
- Creación del usuario en SSO Keycloak (Realm CARDHOLDER).
- Confirmación visual y redirección al Portal TH.
- Restablecimiento y cambio de contraseña.
- Login con número de documento + contraseña.
- Reenvío del link de pre-registro cuando cambia el correo del TH.

**Excluye:**
- Tarjetas innominadas — se rigen por `Activacion_PIN_Innominada.drawio` (ADR-010).
- Onboarding masivo o por archivo — solo se dispara por creación API o por archivo NXXX nominado.
- Flujos transaccionales (autorización, activación de PIN, cambio de PIN) — están fuera de este documento.

### 1.3 Resultado esperado

Al final del flujo:
- Existe un usuario en SSO Keycloak Realm CARDHOLDER asociado al `client_id` del TH.
- La tabla `preregistration` queda en estado `COMPLETADO`.
- La tabla `app_user` tiene un registro activo para el cliente.
- El TH puede autenticarse en el Portal TH con su número de documento + contraseña.
- Hay traza completa del proceso en `audit_log` y `pci_event_log`.

---

## 2. Disparadores del onboarding

El proceso se dispara en tres escenarios. En todos, el `PreRegistration_Service` decide si crea un nuevo `code` o reusa el `PENDIENTE` vigente del cliente.

| # | Disparador | Origen | Condición |
|---|---|---|---|
| 1 | **Creación de tarjeta nominada** | `Prepaid_Card_Creation_Orchestrator` | Tras `INSERT card` con `status=CREADA`, si el TH no tiene usuario en `app_user` (ADR-016) |
| 2 | **Modificación de email del TH** | `Prepaid_Non_Monetary_Novelty_Processor` | Cambio de email vía API o archivo NXXX cuando el TH no está registrado en el portal (ADR-020) |
| 3 | **Operación puntual del Portal Admin** | `Prepaid_Admin_API` | Reenvío manual del link cuando el TH reporta no haberlo recibido |

**Idempotencia:** uno por `client_id`, no por `card_id`. Si el TH recibe varias tarjetas en una misma carga o cargas sucesivas, solo se mantiene un `preregistration` con `status=PENDIENTE` vigente.

---

## 3. Flujo funcional por fases

### Fase 1 — Generación del pre-registro y envío de email

**Actor:** sistema (post-creación de tarjeta nominada).

**Pasos:**
1. `Prepaid_Card_Creation_Orchestrator` ejecuta `SELECT user FROM app_user WHERE client_id = ?`.
2. Si no existe usuario, invoca `PreRegistration_Service.createPreRegistration(client_id, email)`.
3. El service verifica si ya hay un `preregistration` con `status=PENDIENTE` y `expires_at > NOW()` para ese cliente:
   - Si existe → reusa el code y reenvía el email.
   - Si no existe → genera `code = UUID v4`, persiste en `preregistration` con `expires_at = NOW + 30 días`, `attempts = 0`, `status = PENDIENTE`.
4. Construye URL: `https://preregistro.credibanco.com/registro/{code}`.
5. Invoca `Notification_Adapter.sendPreRegistrationEmail(email, url, client_id)`.
6. `Notification_Adapter` aplica plantilla HTML, envía vía SMTP corporativo, registra en `audit_log`.

**Resultado:** TH recibe email con link único.

**Reglas:**
- Email obligatorio (campo 37 del archivo `NXXXddmmcc` o body de la API).
- Reenvío automático de correo en caso de falla (validar con Arquitectura — pendiente).
- Solo se envía a usuarios nuevos. Si el TH ya tenía usuario, se envía un correo distinto con URL de acceso directo al portal (pendiente plantilla con UX).

### Fase 2 — Verificación de identidad

**Actor:** TH (navegador) + `PreRegistration_Portal` + `PreRegistration_API`.

**Pasos:**
1. TH hace clic en el link → abre `https://preregistro.credibanco.com/registro/{code}`.
2. Portal carga, valida que el `code` esté en la URL, presenta formulario "Ingrese tipo y número de documento + correo electrónico registrado".
3. Tipos aceptados: `CC` (Cédula ciudadanía), `CE` (Cédula extranjería), `TI` (Tarjeta de identidad), `PPT` (Permiso de protección temporal).
4. TH ingresa los tres datos y presiona "Verificar".
5. Portal hace `POST /api/preregistro/verify` con `{ code, doc_type, doc_number, email }`.
6. `PreRegistration_API` delega a `PreRegistration_Service.verifyIdentity(code, doc_type, doc_number, email)`.
7. Service valida en Oracle:
   - `code` existe y `expires_at > NOW()`.
   - `status = PENDIENTE`.
   - `(doc_type, doc_number)` coinciden con el cliente asociado a la tarjeta del pre-registro.
   - `email` coincide con el `preregistration.email` o `client.email` registrado.
   - `attempts < 3`.
8. Si OK → continúa Fase 3.
9. Si falla → `attempts++`, si `attempts >= 3` → `status = BLOQUEADO`. Devuelve mensaje claro al TH.

**Resultado:** identidad verificada (factor: documento + correo) → siguiente fase.

### Fase 3 — Generación, envío y validación de OTP

**Actor:** sistema + TH.

**Pasos (3.a — emisión):**
1. Tras verificación OK, `PreRegistration_Service.requestOtp(code)` invoca a `OTP_Service.issueOtp(client_id, purpose=PRE_REGISTRO)` (ADR-024).
2. `OTP_Service` genera código de 6 dígitos numéricos aleatorios.
3. Calcula `code_hash = SHA-256(code)` y persiste en `otp` con `expires_at = NOW + 5 min`, `attempts = 0`, `max_attempts = 3`, `status = ACTIVO`.
4. Invoca `Notification_Adapter.sendOtp(email, code)` (canal: email; SMS si hay número de teléfono).
5. TH recibe OTP en su correo.

**Pasos (3.b — validación):**
6. Portal muestra campo "Ingrese el código OTP".
7. TH ingresa OTP → Portal hace `POST /api/preregistro/validate-otp` con `{ code, otp }`.
8. `PreRegistration_API.confirmOtp(code, otp)`:
   - Resuelve `otp_id` activo asociado al `client_id` del pre-registro.
   - Llama `OTP_Service.validateOtp(otp_id, otp)`.
   - `OTP_Service` calcula `SHA-256(otp)` y compara con `code_hash`.
   - Verifica `expires_at > NOW()` y `attempts < max_attempts`.
   - Si OK → marca `status = USADO`. Si falla 3 veces → `status = BLOQUEADO`.
9. Si OTP válido, el `PreRegistration_Service` emite un `session_registro` (token efímero TTL 10 min) que autoriza la fase 4.

**Resultado:** OTP validado, sesión de registro abierta.

### Fase 4 — Asignación de contraseña y creación de usuario

**Actor:** TH + `PreRegistration_Portal` + `PreRegistration_Service` + SSO Keycloak.

**Pasos:**
1. Portal muestra formulario "Cree su contraseña" con campos `password` y `password_confirm`.
2. Política de contraseña (validación lado cliente y lado servidor):
   - Mínimo 12 caracteres.
   - Al menos 1 mayúscula.
   - Al menos 1 minúscula.
   - Al menos 1 número.
   - Al menos 1 carácter especial.
3. TH ingresa y confirma → Portal hace `POST /api/preregistro/create-account` con `{ code, session_registro, password }`.
4. `PreRegistration_Service.createAccount(code, password)`:
   - Valida `session_registro` activa.
   - Valida política de contraseña en backend.
   - Llama a Keycloak: `POST /admin/realms/cardholder/users` con:
     - `username = doc_number`.
     - `email`, `firstName`, `lastName` desde `client`.
     - `attributes`: `client_id`, `entity_id`, `phone`.
     - `credentials`: password con hash `bcrypt` o `argon2` (Keycloak lo gestiona).
     - `enabled = true`.
     - `roles = [CARDHOLDER]`.
   - Si Keycloak responde 201 Created:
     - `INSERT app_user (client_id, username = doc_number, sso_subject = keycloak_user_id, status = ACTIVO)`.
     - `UPDATE preregistration SET status = COMPLETADO, completed_at = NOW`.
     - `Audit_Service.log(event=ONBOARDING_COMPLETED, ...)`.
5. `PreRegistration_Service` retorna 200 al Portal.

**Resultado:** usuario activo en SSO + `app_user` + `preregistration.status = COMPLETADO`.

### Fase 5 — Confirmación y acceso al Portal TH

**Actor:** TH + Portal.

**Pasos:**
1. Portal muestra: "✅ Cuenta creada exitosamente. Ya puede acceder al portal con su número de documento y contraseña". Botón "Ir al Portal".
2. TH hace clic → redirección a `https://portal.credibanco.com`.
3. TH ingresa `doc_number + password`.
4. Portal TH delega login a Keycloak Realm CARDHOLDER:
   - OIDC Authorization Code flow.
   - Keycloak emite JWT con `sub = sso_subject`, claims `client_id`, `entity_id`, role `CARDHOLDER`.
   - Token TTL 30 min, refresh token TTL configurable.
5. `Prepaid_Cardholder_API` valida el JWT y atiende las consultas del TH.

**Resultado:** TH operando en el Portal TH.

---

## 4. Componentes y responsabilidades

### 4.1 Componentes de Credibanco

| Componente | Tecnología | Responsabilidad | ADR |
|---|---|---|---|
| `PreRegistration_Portal` | Angular 17+ SPA | UI pública en `preregistro.credibanco.com/registro/{code}`. UX de las 5 fases. CAPTCHA + rate limiting de cliente. | ADR-023 |
| `PreRegistration_API` | Java 21 + Spring Boot 3 + WebFlux | Endpoints `/verify`, `/validate-otp`, `/create-account`. Validaciones de schema, autenticación. | ADR-023 |
| `PreRegistration_Service` | Java 21 + Spring Service | Núcleo de negocio del onboarding **y orquestador único de notificaciones**: 5 operaciones (`createPreRegistration`, `verifyIdentity`, `requestOtp`, `confirmOtp`, `createAccount`). Persistencia en `preregistration`. Llama a `OTP_Service`, `B2B_Mail_Service`, SSO. | ADR-016 / ADR-023 / ADR-024 / ADR-025 |
| `OTP_Service` | Java 21 + Spring Service (reusable) | Generación y validación de OTP. Operaciones `issueOtp` y `validateOtp`. **Retorna el código en memoria al caller; no envía emails.** Persistencia en `otp`. Reusable para login MFA y cambio de PIN. | ADR-024 / ADR-025 |
| `B2B_Mail_Service` | Servicio existente en CCO (cliente SMTP corporativo) | **Servicio único** de envío de correos transaccionales en la plataforma CCO. Compartido por onboarding TH (B2C), distribución a entidades emisoras (B2B), alertas operativas. Templates HTML por tipo de mensaje. Audit log por envío. Sin PAN ni datos sensibles. TLS 1.3 al SMTP. | ADR-022 / ADR-025 |
| SMTP Server | Infra On-Prem | Saliente. Remitente `no-reply@credibanco.com`. | — |
| SSO Keycloak (Realm CARDHOLDER) | Keycloak | Identity Provider. Almacena hash bcrypt/argon2 de la contraseña. Emite JWT. MFA opcional. | — |
| `Audit_Service` | Java 21 + Spring Boot async | Auditoría transversal y PCI 10.2. Fire-and-forget. | ADR-018 |
| Oracle Primary | Oracle 19c+ | Tablas `preregistration`, `otp`, `app_user`, `client`. | — |

### 4.2 Componentes externos / referenciados

- **`Prepaid_Card_Creation_Orchestrator`** (pestaña "Creacion de tarjetas") — disparador #1.
- **`Prepaid_Non_Monetary_Novelty_Processor`** (pestaña "Flujo Novedades No Monetarias") — disparador #2.
- **Portal Tarjetahabiente** (pestaña "Portal" / "Portal Faseado") — destino post-onboarding.

---

## 5. Contratos de API

### 5.1 `POST /api/preregistro/verify`

**Request:**
```json
{
  "code": "550e8400-e29b-41d4-a716-446655440000",
  "doc_type": "CC",
  "doc_number": "1234567890",
  "email": "juan.perez@example.com"
}
```

> **Nota — verificación dual:** la verificación de identidad valida los **tres** datos contra el cliente asociado a la tarjeta del pre-registro: `(doc_type, doc_number)` debe coincidir con `client.doc_type/client.doc_number`, y `email` debe coincidir con el `client.email` o el `preregistration.email` registrado al crear el code. Esto refuerza el factor "algo que el TH conoce" antes de emitir la OTP.

**Response 200 OK:**
```json
{
  "verified": true,
  "next_step": "OTP_REQUEST",
  "masked_email": "j***o@example.com"
}
```

**Response 400 — match falla, intentos restantes:**
```json
{
  "error_code": "PREREG_MATCH_FAIL",
  "message": "Datos no coinciden con el titular de la tarjeta",
  "remaining_attempts": 2
}
```

**Response 410 — code expirado/usado/bloqueado:**
```json
{
  "error_code": "PREREG_CODE_INVALID",
  "message": "Link expirado. Contacte a su entidad."
}
```

### 5.2 `POST /api/preregistro/validate-otp`

**Request:**
```json
{
  "code": "550e8400-e29b-41d4-a716-446655440000",
  "otp": "123456"
}
```

**Response 200 OK:**
```json
{
  "valid": true,
  "session_registro": "eyJhbGciOi...",
  "session_expires_at": "2026-05-14T15:30:00Z"
}
```

**Response 400 — OTP errada:**
```json
{
  "error_code": "OTP_INVALID",
  "message": "Código incorrecto",
  "remaining_attempts": 1
}
```

**Response 410 — OTP expirada:**
```json
{
  "error_code": "OTP_EXPIRED",
  "message": "Código expirado. Solicite uno nuevo."
}
```

### 5.3 `POST /api/preregistro/resend-otp`

**Request:**
```json
{
  "code": "550e8400-e29b-41d4-a716-446655440000"
}
```

**Response 200 OK:** confirma reenvío. Cuenta contra el límite de 3 reenvíos por sesión.

### 5.4 `POST /api/preregistro/create-account`

**Request:**
```json
{
  "code": "550e8400-e29b-41d4-a716-446655440000",
  "session_registro": "eyJhbGciOi...",
  "password": "S3cur3P@ssword!"
}
```

**Response 201 Created:**
```json
{
  "account_created": true,
  "username": "1234567890",
  "portal_url": "https://portal.credibanco.com"
}
```

**Response 400 — política de password:**
```json
{
  "error_code": "PASSWORD_POLICY_VIOLATION",
  "message": "La contraseña no cumple los requisitos",
  "details": ["min 12 caracteres", "al menos 1 mayúscula", "al menos 1 carácter especial"]
}
```

### 5.5 Reglas comunes a todos los endpoints

- TLS 1.3 obligatorio.
- `Content-Type: application/json; charset=UTF-8`.
- Rate limiting por IP (10 req/min por endpoint).
- CAPTCHA exigido en `verify` (anti-bot).
- Header `X-Correlation-Id` propagado al `Audit_Service`.
- Errores en formato Problem Details (RFC 7807).

---

## 6. Modelo de datos

### 6.1 Tabla `preregistration`

```sql
CREATE TABLE preregistration (
  preregistration_id NUMBER(10) GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  code               VARCHAR2(36)  NOT NULL UNIQUE,        -- UUID v4
  client_id          NUMBER(10)    NOT NULL,
  email              VARCHAR2(150) NOT NULL,
  status             VARCHAR2(20)  DEFAULT 'PENDIENTE' NOT NULL,
  expires_at         TIMESTAMP     NOT NULL,                -- TTL 30 días
  attempts           NUMBER(3)     DEFAULT 0 NOT NULL,
  completed_at       TIMESTAMP,
  created_at         TIMESTAMP     DEFAULT SYSTIMESTAMP NOT NULL,
  updated_at         TIMESTAMP,
  created_by         VARCHAR2(50)  NOT NULL,
  CONSTRAINT fk_pr_client       FOREIGN KEY (client_id) REFERENCES client(client_id),
  CONSTRAINT chk_pr_status      CHECK (status IN
    ('PENDIENTE','PENDING_NO_EMAIL','COMPLETADO','EXPIRADO','BLOQUEADO'))
);
CREATE INDEX idx_pr_client ON preregistration(client_id);
CREATE INDEX idx_pr_status ON preregistration(status);
```

### 6.2 Tabla `otp`

```sql
CREATE TABLE otp (
  otp_id           NUMBER(12)    GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  user_id          NUMBER(10),
  client_id        NUMBER(10),
  purpose          VARCHAR2(30)  NOT NULL,             -- PRE_REGISTRO | LOGIN_MFA | CAMBIO_PIN | OTRO
  code_hash        RAW(32)       NOT NULL,             -- SHA-256
  expires_at       TIMESTAMP     NOT NULL,             -- 5 min
  attempts         NUMBER(3)     DEFAULT 0 NOT NULL,
  max_attempts     NUMBER(3)     DEFAULT 3 NOT NULL,
  status           VARCHAR2(15)  DEFAULT 'ACTIVO' NOT NULL,
  used_at          TIMESTAMP,
  created_at       TIMESTAMP     DEFAULT SYSTIMESTAMP NOT NULL,
  CONSTRAINT fk_otp_user        FOREIGN KEY (user_id)   REFERENCES app_user(user_id),
  CONSTRAINT fk_otp_client      FOREIGN KEY (client_id) REFERENCES client(client_id),
  CONSTRAINT chk_otp_status     CHECK (status IN ('ACTIVO','USADO','EXPIRADO','BLOQUEADO'))
);
CREATE INDEX idx_otp_user      ON otp(user_id);
CREATE INDEX idx_otp_status    ON otp(status, expires_at);
```

### 6.3 Tabla `app_user`

```sql
CREATE TABLE app_user (
  user_id          NUMBER(10)    GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  client_id        NUMBER(10)    NOT NULL UNIQUE,           -- 1:1 con client
  username         VARCHAR2(100) NOT NULL UNIQUE,
  sso_subject      VARCHAR2(100),                            -- subject del JWT
  status           VARCHAR2(15)  DEFAULT 'PENDIENTE' NOT NULL,
  last_login_at    TIMESTAMP,
  created_at       TIMESTAMP     DEFAULT SYSTIMESTAMP NOT NULL,
  updated_at       TIMESTAMP,
  created_by       VARCHAR2(50)  NOT NULL,
  updated_by       VARCHAR2(50),
  CONSTRAINT fk_user_client     FOREIGN KEY (client_id) REFERENCES client(client_id),
  CONSTRAINT chk_user_status    CHECK (status IN ('PENDIENTE','ACTIVO','BLOQUEADO','INACTIVO'))
);
```

Ver `db/01_schema_prepago.sql` para el DDL completo.

---

## 7. Reglas de negocio

### 7.1 Reglas globales

| ID | Regla | Justificación |
|---|---|---|
| RG-01 | Solo aplica a tarjetas **nominadas** (`card.nominated = 'S'`) | Las innominadas se rigen por ADR-010 |
| RG-02 | Uno por cliente, no por tarjeta | Si un cliente recibe varias tarjetas, comparte usuario |
| RG-03 | Idempotente por `client_id` con `status=PENDIENTE` | Evita generar múltiples codes vivos |
| RG-04 | El email es obligatorio en creación nominada | Sin email no hay onboarding posible |
| RG-05 | Si email faltante → status `PENDING_NO_EMAIL` para gestión manual | Operaciones decide reenvío o solicitud al cliente |

### 7.2 Reglas de TTL

| Recurso | TTL | Acción al expirar |
|---|---|---|
| `preregistration.code` | 30 días | `status = EXPIRADO` (job nocturno) |
| `otp` | 5 minutos | `status = EXPIRADO` (job o lazy check) |
| `session_registro` | 10 minutos | rechazar `create-account` con 401 |
| Bloqueo OTP/login | 24 horas | desbloqueo automático tras 24h |
| JWT del Portal TH | 30 minutos | refresh token o re-login |

### 7.3 Reglas de intentos

| Acción | Máximo | Consecuencia al exceder |
|---|---|---|
| Verificación de cédula | 3 intentos | `status = BLOQUEADO`, TH debe contactar a la entidad |
| Solicitud de OTP (resend) | 3 en 24h | bloqueo de solicitudes 24h |
| Validación de OTP | 3 intentos | `otp.status = BLOQUEADO`, requiere reiniciar desde verificación de cédula |
| Login en Portal TH | 3 intentos | bloqueo de login 24h |
| Restablecer contraseña | 3 intentos | bloqueo definitivo, contactar a la entidad |
| Cambio de contraseña | 3 intentos | bloqueo definitivo, contactar a la entidad |

### 7.4 Política de contraseña (PCI 8.3)

- Mínimo **12 caracteres**.
- Al menos **1 mayúscula**.
- Al menos **1 minúscula**.
- Al menos **1 número**.
- Al menos **1 carácter especial** (`!@#$%^&*()_+-=[]{}|;:,.<>?`).
- Validación en cliente (Angular) **y** servidor (`PreRegistration_API`).
- Almacenamiento como hash en Keycloak (bcrypt o argon2 — gestionado por Keycloak).
- Histórico de contraseñas: pendiente decidir si Keycloak guarda histórico para impedir reuso de últimas N (recomendación: últimas 4).

### 7.5 Reglas de visualización

- Contraseña **ofuscada por defecto** con opción "Mostrar".
- Email enmascarado en respuestas (`j***o@example.com`).
- OTP **no se muestra ni se logea** en claro.
- Nunca capturar datos sensibles por consola del navegador.

---

## 8. Estados y máquinas de estado

### 8.1 `preregistration.status`

```
                ┌─────────────────┐
                │   PENDIENTE     │ ← creación inicial
                └────────┬────────┘
                         │
         ┌───────────────┼───────────────┐
         │               │               │
   completa fase 4   expira (30d)   3 intentos cédula
         │               │               │
         ▼               ▼               ▼
  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
  │ COMPLETADO  │ │  EXPIRADO   │ │  BLOQUEADO  │
  └─────────────┘ └─────────────┘ └─────────────┘

  PENDING_NO_EMAIL ← cuando el campo email viene nulo o inválido
                     (gestión manual por Operaciones)
```

### 8.2 `otp.status`

```
        ┌─────────────┐
        │   ACTIVO    │ ← issueOtp
        └──────┬──────┘
               │
   ┌───────────┼───────────┬─────────────┐
   │           │           │             │
 valida    no valida    expira      OTP_Service
correcto    3 veces    (5 min)      regenera
   │           │           │             │
   ▼           ▼           ▼             ▼
┌──────┐ ┌────────────┐ ┌──────────┐ ┌──────────┐
│USADO │ │ BLOQUEADO  │ │EXPIRADO  │ │ ACTIVO   │ (nuevo)
└──────┘ └────────────┘ └──────────┘ └──────────┘
```

### 8.3 `app_user.status`

```
PENDIENTE → ACTIVO → (BLOQUEADO ↔ ACTIVO)
                  ↓
                INACTIVO (cierre de cuenta)
```

---

## 9. Casos de error y manejo

| # | Caso | Código error | Mensaje al TH | Acción |
|---|---|---|---|---|
| E1 | Code no existe o malformado | `PREREG_NOT_FOUND` | "Link inválido. Contacte a su entidad." | 404, log en `audit_log` |
| E2 | Code expirado (30 días) | `PREREG_EXPIRED` | "Link expirado. Contacte a su entidad para uno nuevo." | 410, status=EXPIRADO |
| E3 | Code ya usado | `PREREG_ALREADY_USED` | "Esta cuenta ya fue creada. Ingrese al portal con su contraseña." | 409, redirect al login |
| E4 | Cédula no coincide (1° o 2° intento) | `PREREG_MATCH_FAIL` | "Datos no coinciden. Le quedan N intentos." | 400, attempts++ |
| E5 | Cédula no coincide (3° intento) | `PREREG_BLOCKED` | "Has agotado los intentos. Contacte a su entidad." | 403, status=BLOQUEADO |
| E6 | OTP incorrecta | `OTP_INVALID` | "Código incorrecto. Le quedan N intentos." | 400, attempts++ |
| E7 | OTP expirada (5 min) | `OTP_EXPIRED` | "Código expirado. Solicite uno nuevo." | 410, otp.status=EXPIRADO |
| E8 | OTP bloqueada (3 intentos) | `OTP_BLOCKED` | "OTP bloqueada. Reinicie desde la verificación." | 403 |
| E9 | Solicitudes de OTP excedidas (3 en 24h) | `OTP_RATE_LIMIT` | "Demasiadas solicitudes. Intente en 24 horas." | 429 |
| E10 | Session_registro expirada | `SESSION_EXPIRED` | "Sesión expirada. Reinicie desde el OTP." | 401 |
| E11 | Política de password no cumple | `PASSWORD_POLICY_VIOLATION` | "Su contraseña no cumple los requisitos: ..." | 400, lista de requisitos faltantes |
| E12 | Email duplicado en BD | `EMAIL_ALREADY_EXISTS` | "Este correo ya tiene cuenta. ¿Olvidó su contraseña?" | 409, ofrecer flujo de recuperación |
| E13 | Error SSO Keycloak | `SSO_ERROR` | "Error técnico. Intente nuevamente o contacte soporte." | 500, retry interno + alerta a operación |
| E14 | TLS / firma JWT inválido | `AUTH_INVALID` | "Sesión inválida." | 401 |

**Todos los errores:**
- Se logean en `audit_log` con request, response, IP y `correlation_id`.
- Eventos sensibles (E5, E8, E9, E13) van también a `pci_event_log`.
- El `Notification_Adapter` envía email al TH cuando ocurre E5, E8 o E9 (ver §12).

---

## 10. Seguridad y cumplimiento

### 10.1 PCI DSS v4.0

| Requisito | Cómo se cumple en este flujo |
|---|---|
| 4.2.1 | TLS 1.3 obligatorio en todos los endpoints, en SMTP y en la conexión a Keycloak |
| 6.4.3 | CSP + SRI en el `PreRegistration_Portal` |
| 7.2 | Acceso al `/admin/realms/cardholder/users` de Keycloak restringido al service account del `PreRegistration_Service` |
| 8.3 | Política de password: 12+ chars, mayús/minús/núm/especial |
| 8.3.6 | OTP como segundo factor antes de crear cuenta |
| 10.2 | `pci_event_log` registra: identificación de usuario, tipo de evento, fecha y hora, éxito/fallo, origen, recursos afectados, componente |
| 12.3.1 | TTLs documentados (§7.2) |

### 10.2 Otros controles

- **CAPTCHA** en formulario de cédula (anti-bot).
- **Rate limiting** por IP (10 req/min por endpoint del API).
- **Headers de seguridad:** `X-Frame-Options: DENY`, `X-Content-Type-Options: nosniff`, `Strict-Transport-Security: max-age=63072000`.
- **Cookies:** `Secure`, `HttpOnly`, `SameSite=Strict` para `session_registro`.
- **Inyección:** validación de schema con Bean Validation; consultas con bind variables (no concatenación).
- **Hash de OTP:** `SHA-256` con salt opcional. El OTP en claro nunca se almacena en BD ni en logs.
- **Hash de contraseña:** delegado a Keycloak (bcrypt/argon2).

### 10.3 Pruebas de seguridad obligatorias

- OWASP ZAP / Burp Suite contra los 4 endpoints.
- SAST (SonarQube, Snyk Code) sobre `PreRegistration_Service` y `PreRegistration_API`.
- Test de fuerza bruta sobre OTP (debe bloquear a 3 intentos).
- Test de fuerza bruta sobre verificación de cédula (debe bloquear a 3 intentos).

---

## 11. Tarjetas innominadas (exclusión)

**Este flujo NO aplica para tarjetas innominadas.** Las innominadas:

- **No** disparan pre-registro al crearse.
- **No** generan code, ni OTP, ni email automático.
- **Activación:** flujo público con Proof of Possession en `Activacion_PIN_Innominada.drawio` (ADR-010), con BIN + last4 + fecha de expiración + CVV2.
- **Onboarding al portal post-activación:** el TH puede registrarse al portal después de activar la tarjeta, ejecutando el mismo `PreRegistration_Service` pero con disparo manual desde el portal. Este caso es opcional y queda como evolutivo.

---

## 12. Notificaciones y plantillas

Todas las notificaciones del onboarding se envían a través del **`B2B_Mail_Service`** (servicio único de correos en CCO, ya existente). El **`PreRegistration_Service` orquesta** todos los envíos; el `OTP_Service` retorna el código al caller, no envía emails directamente. Plantillas HTML obligatorias. Remitente: `no-reply@credibanco.com`.

| Plantilla | Trigger | Asunto | Contenido clave |
|---|---|---|---|
| `pre_registration_email` | F1 — code generado | "Active su tarjeta prepago" | Saludo personalizado, link de pre-registro acortado, instrucciones, vigencia 30 días |
| `pre_registration_resend` | Modificación de email | "Nuevo enlace de activación" | Mismo formato, indicar que reemplaza al anterior |
| `otp_email` | F3 — OTP generada | "Su código de verificación" | Código de 6 dígitos, vigencia 5 min, advertencia "no comparta este código" |
| `otp_blocked` | E8 — 3 intentos | "Bloqueo de validación de OTP" | Razón del bloqueo, tiempo de espera, contacto |
| `otp_request_blocked` | E9 — 3 reenvíos | "Bloqueo de solicitudes de OTP" | 24h de espera, contacto entidad |
| `account_created` | F4 — cuenta creada | "Cuenta activada exitosamente" | URL del portal, doc_number como username |
| `login_notification` | F5 — login en portal | "Acceso a su cuenta" | Fecha, hora, IP/dispositivo, botón "no fui yo" |
| `existing_account` | Disparador con TH ya registrado | "Acceso a su portal" | URL acceso directo (no pre-registro) |

**Reglas:**
- Sin PAN, sin CVV2, sin password, sin OTP en cuerpo del email.
- Solo el link con UUID opaco (cuando aplique).
- Botón "no compartir este código" en plantilla de OTP.
- Reenvío automático de correo en caso de fallo SMTP (configurable, max 3 reintentos con backoff). *Validar con Arquitectura.*

> **Nota arquitectónica.** El `B2B_Mail_Service` es **un único servicio compartido** en CCO. La separación conceptual B2C / B2B (planteada antes en ADR-022) se vuelve solo una clasificación lógica de templates dentro del mismo servicio, no dos servicios distintos. El servicio decide el routing y rate limiting según el tipo de mensaje. Ver ADR-025 para la conciliación.

---

## 13. Auditoría

### 13.1 `audit_log`

Cada operación del `PreRegistration_Service` y del `OTP_Service` registra:

| Campo | Valor |
|---|---|
| `service_name` | `PreRegistration_Service` o `OTP_Service` |
| `method_name` | `createPreRegistration` / `verifyIdentity` / `requestOtp` / `confirmOtp` / `createAccount` |
| `request_payload` | JSON enmascarado (sin OTP, sin password) |
| `response_payload` | Código + mensaje |
| `status_code` | 200/400/410/etc |
| `response_time_ms` | latencia |
| `correlation_id` | propagado desde el primer request |
| `id_usuario` | `client_id` |
| `id_proceso` | `preregistration_id` |

### 13.2 `pci_event_log`

Eventos sensibles que requieren registro PCI 10.2:

- `ONBOARDING_PRE_REGISTRATION_CREATED`
- `ONBOARDING_IDENTITY_VERIFIED`
- `ONBOARDING_IDENTITY_FAILED` (incluye intentos)
- `ONBOARDING_IDENTITY_BLOCKED`
- `ONBOARDING_OTP_ISSUED`
- `ONBOARDING_OTP_VALIDATED`
- `ONBOARDING_OTP_FAILED`
- `ONBOARDING_OTP_BLOCKED`
- `ONBOARDING_ACCOUNT_CREATED`
- `ONBOARDING_PASSWORD_POLICY_VIOLATION`
- `ONBOARDING_SSO_ERROR`

**Cada evento incluye:**
- `user_identifier`: `doc_number` o `client_id`.
- `event_type`: uno de los anteriores.
- `event_date`.
- `success`: S/N.
- `origin`: IP del cliente.
- `affected_resource`: `preregistration` / `otp` / `app_user` / `keycloak`.
- `component`: `PreRegistration_Service`.
- `details`: JSON con contexto.

---

## 14. Performance y SLAs

| Métrica | Objetivo |
|---|---|
| Latencia `POST /verify` p99 | < 300 ms |
| Latencia `POST /validate-otp` p99 | < 200 ms |
| Latencia `POST /create-account` p99 | < 800 ms (incluye round-trip a Keycloak) |
| Throughput por endpoint | 100 req/s sostenidos |
| Disponibilidad mensual | 99.5% (no es hot-path transaccional) |
| Tiempo de envío email post-trigger | < 60 s p95 |

**Consideraciones:**
- El flujo es **cold-path** comparado con la autorización transaccional. No requiere los mismos límites del Authorization Engine.
- HPA del `PreRegistration_API`: 2 pods mínimo, 4 máximo.
- Cache: el `code` y el estado del `preregistration` no se cachean en Redis (consistencia fuerte requerida).

---

## 15. Pendientes

### 15.1 Pendientes funcionales del documento fuente

1. **Plantillas con UX** para correos de pre-registro y acceso directo (validar con UX).
2. **Validación con SI** de mecanismos de visualización de contraseña (botón Mostrar).
3. **Reenvío de correo en caso de fallas** (validar con Arquitectura).
4. **Histórico de contraseñas:** ¿Keycloak debe impedir reuso de las últimas N? Recomendación: 4.
5. **UX de F4 — Asignar contraseña**: criterios `xxx` del documento fuente sin refinar.
6. **UX de F5 — Confirmación**: criterios `xxx` del documento fuente sin refinar.
7. **Email de confirmación post-creación de cuenta:** está en este doc pero no en `Onboarding_Tarjetahabiente.drawio`. Decidir si formaliza o se elimina.
8. **Login con doc_number vs email:** el source tiene una inconsistencia (resolvimos a favor de doc_number). Confirmar con Producto.

### 15.2 Pendientes técnicos

1. **Proveedor SMS** para OTP cuando aplique (Twilio / AWS SNS / interno).
2. **Cuándo enviar OTP por SMS** (¿siempre que haya phone? ¿opcional al TH?). Definir UX.
3. **SMS_Adapter** dedicado vs extensión del `Notification_Adapter`.
4. **Federation** con servicios de identidad de las entidades emisoras (Fase 2+).
5. **MFA en login al Portal TH:** Keycloak soporta TOTP/WebAuthn. Decidir si se activa por defecto o solo para operaciones sensibles.

### 15.3 Pendientes operativos

1. **Catálogo de IPs permitidas** para el `PreRegistration_Portal` (si aplica geo-bloqueo).
2. **Procedimiento operativo** cuando un TH llega a `BLOQUEADO`: ¿operaciones puede desbloquear? ¿se requiere validación adicional?
3. **Métricas dashboard** para Operaciones (tasa de completitud, abandonos por fase, errores por entidad).

---

## 16. Referencias cruzadas

### 16.1 Documentos del proyecto

| Archivo | Relación |
|---|---|
| `Prepago/Requerimientos Técnicos (12).docx` | Fuente primaria — capítulos "Portal Web Prepago - Back/Front (Onboarding)" |
| `Prepago/Onboarding_Tarjetahabiente.drawio` | Diagrama de proceso por fases (swimlanes, errores, validaciones) |
| `Prepago/PrepagoUnificadoArquitectura_V_1_5.drawio` (pestaña "Onboarding Tarjetahabiente") | Vista C4 L2 con componentes y conexiones |
| `Prepago/docs/requerimientos-ciclo-vida-tarjeta-y-portal-th.md` | §6 — extracto del Portal Web Back/Front |
| `Prepago/docs/requerimientos-tecnicos-consolidados.md` | §7 — Etapa 4 |
| `Prepago/docs/decisiones-arquitectura.md` | ADR-016 / ADR-023 / ADR-024 |
| `Prepago/db/01_schema_prepago.sql` | Tablas `preregistration`, `otp`, `app_user`, `client` |
| `Prepago/MER_Prepago_V_1_0.drawio` | Modelo entidad-relación |

### 16.2 ADRs aplicables

| ADR | Tema |
|---|---|
| **ADR-001** | BFF por portal (Cardholder API separado) |
| **ADR-016** | Pre-registro disparado tras creación de tarjeta nominada (uno por cliente) |
| **ADR-018** | Auditoría transversal y PCI 10.2 (sirve como base de auditoría) |
| **ADR-020** | Disparador #2 — modificación de email del TH |
| **ADR-022** | Distinción `Notification_Adapter` (B2C) vs `B2B_Mail_Service` (B2B) |
| **ADR-023** | Pestaña "Onboarding Tarjetahabiente" en v1.5 |
| **ADR-024** | `OTP_Service` como componente reusable |

### 16.3 Diagramas relacionados

| Diagrama | Para qué sirve |
|---|---|
| `Prepago/Activacion_PIN_Innominada.drawio` | Caso excluido — onboarding alternativo para innominadas |
| `Prepago/pinpad_pci_drawio_6_mejorado.drawio` | Próximo paso post-onboarding: activación de tarjeta y asignación de PIN |
| `Prepago/Iframe_Visualizacion_DatosSensibles.drawio` | Próximo paso: TH visualiza PAN/CVV2 en su tarjeta |

---

> **Cómo usar este documento.** Es la referencia única del onboarding. Para implementar:
> 1. Verificar contratos de API (§5) contra el `PreRegistration_Portal`.
> 2. Verificar tablas en BD (§6) contra `db/01_schema_prepago.sql`.
> 3. Implementar reglas de §7 en `PreRegistration_Service` y `OTP_Service`.
> 4. Probar máquinas de estado (§8) y errores (§9) end-to-end.
> 5. Validar controles de seguridad (§10) con SI antes de pasar a UAT.
> 6. Confirmar pendientes (§15) con Producto/Arquitectura/SI antes del go-live.
