# Requerimientos Técnicos Consolidados — Plataforma Prepago Credibanco

Documento maestro que consolida los requerimientos del archivo `Prepago/Requerimientos Técnicos (12).docx` con todas las decisiones técnicas y arquitectónicas tomadas durante el diseño (ADR-001 a ADR-024).

> Última actualización: 14/05/2026
> Fase: 1 MVP — Línea Base
> Versión arquitectura vigente: `PrepagoUnificadoArquitectura_V_1_5.drawio`

---

## Tabla de contenido

1. [Alcance y supuestos](#1-alcance-y-supuestos)
2. [Stack técnico](#2-stack-técnico)
3. [Arquitectura general](#3-arquitectura-general)
4. [Etapa 1 — Infraestructura base y administración](#4-etapa-1--infraestructura-base-y-administración)
5. [Etapa 2 — Ciclo de vida de la tarjeta](#5-etapa-2--ciclo-de-vida-de-la-tarjeta)
6. [Etapa 3 — Motor transaccional](#6-etapa-3--motor-transaccional)
7. [Etapa 4 — Portal tarjetahabiente y onboarding](#7-etapa-4--portal-tarjetahabiente-y-onboarding)
8. [Etapa 5 — Procesos batch y archivos de salida](#8-etapa-5--procesos-batch-y-archivos-de-salida)
9. [Requerimientos transversales](#9-requerimientos-transversales)
10. [Modelo de datos](#10-modelo-de-datos)
11. [Cumplimiento PCI DSS v4.0](#11-cumplimiento-pci-dss-v40)
12. [Performance y SLAs](#12-performance-y-slas)
13. [Observabilidad](#13-observabilidad)
14. [Despliegue e infraestructura](#14-despliegue-e-infraestructura)
15. [Decisiones diferidas y pendientes](#15-decisiones-diferidas-y-pendientes)
16. [Referencias](#16-referencias)

---

## 1. Alcance y supuestos

**Alcance Fase 1 MVP:** plataforma prepago on-premise para Credibanco, integrada con Pomelo como pre-autorizador. Soporta emisión, administración, autorización transaccional y administración del ciclo de vida de tarjetas prepago físicas y virtuales. La fase cubre las 5 etapas declaradas en el documento de requerimientos:

1. Infraestructura base y administración (parametrización de Entidad, BIN, AFG, reglas, comisiones, GMF, planes de validación de comercio).
2. Ciclo de vida de la tarjeta (creación, realce, distribución, bloqueos, modificación de datos).
3. Motor transaccional (autorización POS y ATM, comisiones, GMF, técnicamente exitosas, reversos y anulaciones).
4. Portal tarjetahabiente y onboarding (pre-registro, OTP, login, activación de tarjeta, cambio de PIN).
5. Procesos batch y archivos de salida (recobros, GMF batch, archivos para entidades, reconciliación end-of-day).

**Diferido a Fase 2:** multibolsillo, tarjeta amparada, herramientas de fraude, módulos extendidos del portal TH, activación de tarjetas innominadas desde el portal autenticado.

**Supuestos clave:**

- Pomelo es la única fuente del PAN, CVV2, fecha de expiración y código de autorización en el momento de la creación.
- Pomelo es el pre-autorizador de toda transacción financiera; valida PIN, CVV/CVV2, fecha de vencimiento, existencia y estado de tarjeta, BIN y AFG, y límite de intentos de PIN errado.
- Las entidades emisoras suben archivos cifrados con PGP vía SFTP a través de GoAnywhere MFT.
- El HSM Thales se reserva para PIN interchange. **No** se usa para cifrar el PAN ni el CVV2.
- El equipo opera sobre OpenShift con Java 21 + Spring Boot 3, Oracle 19c+ y Redis 7+ con Sentinel.
- Existe un SSO Keycloak con dos realms: `ADMIN` (operación interna) y `CARDHOLDER` (tarjetahabientes).

---

## 2. Stack técnico

| Capa | Tecnología | Justificación |
|---|---|---|
| **Backend hot-path** | Java 21 + Spring Boot 3 + WebFlux | Reactivo para el motor transaccional; non-blocking IO en Pomelo client |
| **Backend cold-path** | Java 21 + Spring Boot 3 + Spring Batch | CronJobs OpenShift para procesos diarios y de recobro |
| **Frontend** | Angular 17+ | Portal admin, portal tarjetahabiente, portal pre-registro, iframes de datos sensibles e iframe de PIN |
| **Base de datos** | Oracle 19c+ (Primary + Active Data Guard Read Replica) | Read Replica para reportería y `Output_File_Generator` (no impacta hot-path) |
| **Cache / contadores** | Redis 7+ Cluster + Sentinel (HA) | AFG config cache + `Limit Counter Service` + cache de `Merchant_Validation_Plan` |
| **Infraestructura** | OpenShift Container Platform | Despliegue, RBAC, secrets management, CronJobs |
| **Mensajería** | (TBD) — eventual broker entre orchestrators y notification | Por ahora invocaciones síncronas; introducir Kafka/RabbitMQ si el volumen lo exige |
| **HSM** | Thales payShield | Solo PIN interchange (cmd EI / GI / EO) |
| **Cifrado de PAN** | AES-256-GCM + RSA-4096 OAEP (envelope encryption) | KEK en KeyStore PKCS12 montado como OpenShift Secret. Sin HSM (ADR-006/007/018) |
| **Transferencia de archivos** | GoAnywhere MFT + SFTP + PGP | Validación de formato y depósito en NFS |
| **SSO / IdP** | Keycloak (OIDC / OAuth2 / JWT) | Realms ADMIN y CARDHOLDER, MFA |
| **Mensajería B2C** | `Notification_Adapter` (cliente SMTP corporativo) | Email transaccional al tarjetahabiente |
| **Mensajería B2B** | `B2B_Mail_Service` | Email transaccional a entidades emisoras (Distribución, alertas operativas) |
| **OTP** | `OTP_Service` (Spring Service reusable) | Pre-registro, login MFA, cambio de PIN (ADR-024) |

---

## 3. Arquitectura general

La arquitectura se documenta en `PrepagoUnificadoArquitectura_V_1_5.drawio` (16 pestañas C4 Level 2). Los componentes principales son:

### 3.1 Componentes del hot-path (autorización transaccional)

| Componente | Tecnología | Responsabilidad | ADR |
|---|---|---|---|
| `Authorization Gateway` | OpenShift Route + Spring Cloud Gateway | TLS, rate limiting, validación HMAC-SHA256, schema, JWT | — |
| `Authorization Engine` | Java 21 + Spring Boot 3 + WebFlux (3-8 pods con HPA) | Pipeline de 10 pasos para autorizar transacciones | — |
| `Prepaid_Limit_Counter_Service` | Java 21 + Spring Service | Contadores por tarjeta en Redis (CHECK / INCREMENT / DECREMENT / RESET) | ADR-004 / ADR-017 |
| `Merchant_Validation_Plan_Service` | Java 21 + Spring Service | Inclusión/exclusión MCC o CU; fuente SVBO en tiempo real | ADR-017 |
| `Audit_Service` | Java 21 + Spring Boot, async fire-and-forget | Auditoría transversal y PCI 10.2 | ADR-018 |
| **Redis** | Cluster + Sentinel | AFG config cache + Limit Counters | ADR-002 |
| **Oracle Primary** | 19c+ R/W | Datos transaccionales + tabla `transaction` particionada por mes | ADR-005 |

### 3.2 Componentes del ciclo de vida de la tarjeta

| Componente | Tecnología | Responsabilidad | ADR |
|---|---|---|---|
| `Prepaid_File_Ingestion_Batch` | Java 21 + Spring Batch (CronJob 15 min) | Polling de NFS y delegación a processors | — |
| `Prepaid_Card_Creation_API` | Java 21 + Spring Boot 3 + WebFlux | API uno-a-uno para creación | — |
| `Prepaid_Card_Creation_Orchestrator` | Spring Service | Lógica unificada batch + API (10+ pasos) | ADR-015 / ADR-016 |
| `Pomelo_API_Client` | Spring WebClient + Resilience4j | Circuit breaker, retry, timeout 5s | — |
| `Envelope_Encryption_Service` | Java 21 + JCE/BouncyCastle | AES-256-GCM + RSA-4096 OAEP | ADR-006 / ADR-018 |
| `KEK_KeyStore` | PKCS12 + OpenShift Secret | RSA-4096 con passphrase split knowledge | ADR-007 |
| `PreRegistration_Service` | Java 21 + Spring Service | Pre-registro UUID + verify cédula + `OTP_Service` + creación SSO | ADR-016 / ADR-023 / ADR-024 |
| `OTP_Service` | Java 21 + Spring Service | OTP reusable: pre-registro, login MFA, cambio de PIN | ADR-024 |
| `Notification_Adapter` | Spring Boot, cliente SMTP | Emails transaccionales al TH (B2C) | ADR-016 / ADR-023 |
| `Prepaid_Embossing_Batch` + `Prepaid_Embossing_Processor` | Java 21 + Spring Batch | Realce diario por AFG | ADR-021 |
| `Distribution_Service` | Java 21 + Spring Service | Reporte de pedido por entidad | ADR-022 |
| `B2B_Mail_Service` | Cliente SMTP corporativo | Emails B2B a entidades emisoras | ADR-022 |
| `Prepaid_Monetary_Novelty_Processor` | Spring Service | Recargas, débitos, reversos, anulaciones | — |
| `Prepaid_Non_Monetary_Novelty_Processor` | Spring Service | Bloqueos, desbloqueos, modificación de datos | ADR-020 |
| `Output_File_Generator` | Java 21 + Spring Batch | Genera TNN/TAS/NOVR/AUM/... con descifrado del PAN en memoria | — |

### 3.3 Componentes del portal y onboarding

| Componente | Tecnología | Responsabilidad |
|---|---|---|
| `Prepaid_Admin_API` | Java 21 + Spring Boot 3 | Backend del portal admin (CRUD entidades, BINes, AFG, reglas, comisiones, GMF, técnicamente exitosas, cuota de manejo, planes de validación) |
| `Prepaid_Cardholder_API` | Java 21 + Spring Boot 3 + WebFlux | Backend del portal TH (consulta saldo, historial, bloqueo/desbloqueo, modificación datos) |
| `PreRegistration_Portal` | Angular 17+ SPA | Portal público en `preregistro.credibanco.com/registro/{code}` |
| `Iframe Datos Sensibles` | Angular 17+ en dominio separado | Visualización de PAN/CVV2/exp en `secure-data.credibanco.com` |
| `Iframe PIN` | Angular 17+ + SubtleCrypto | Captura y cambio de PIN E2E con HSM Thales |

### 3.4 Componentes de proceso batch

| Componente | Frecuencia | Responsabilidad | ADR |
|---|---|---|---|
| `Commission Retry Scheduler` | Diario | Recobros pendientes hasta 3 meses (CSAT/CINS/COMI) | ADR-014 |
| `GMF Batch Processor` | Diario | GMF para entidades en modo BATCH y para tarjetas exentas | ADR-012 |
| `EOD Settlement` | Diario 05:00-08:00 | Reconciliación contra clearing de franquicias | ADR-011 |

---

## 4. Etapa 1 — Infraestructura base y administración

### 4.1 Habilitación de Entidad

**Funcional:**
- Crear entidad miembro principal en sistemas de Credibanco y Pomelo.
- Campos: nombre, código de compensación, fee_id (código de producto), pomelo_client_id (Client ID Pomelo).
- Campo `gmf_mode`: línea o batch (decisión por entidad).
- Activar/desactivar entidad.

**Técnico:**
- Tabla `entity` en Oracle.
- Endpoint `POST/PUT/DELETE /api/admin/v1/entities` en `Prepaid_Admin_API`.
- Auditoría obligatoria (created_at, updated_at, created_by, updated_by).
- Sincronización con Pomelo: pendiente cerrar contrato exacto (alta de entidad en CCO).

**ADR:** ADR-012 (GMF línea vs batch).

### 4.2 Parametrización de BIN

**Funcional:**
- Crear BINes por entidad emisora.
- Campos: bin_code (6-8 dígitos), bin_type (PREPAGO en Fase 1), canje_enabled (bool), entidad asociada.
- Activar/desactivar.
- Mantener campos del archivo IMP141-FT002.
- Procesos de siembra de llaves para garantizar funcionamiento.

**Técnico:**
- Tabla `bin` con FK a `entity`.
- CRUD en `Prepaid_Admin_API`.
- Sincronización con Pomelo (pendiente confirmar contrato CCO).

### 4.3 Parametrización de AFG

**Funcional:**
- Crear AFG en Pomelo dashboard; consultar y almacenar en BD de Credibanco.
- Pomelo es la fuente del `pomelo_afg_id` (código único de identificación).
- Campos del AFG (más de 30) — ver `db/01_schema_prepago.sql` tabla `affinity_group`.
- Campos críticos: card_type (FISICA/VIRTUAL/AMBAS), cvv_dynamic, expiration_range, embosser, custodios 1 y 2, cuota de manejo (valor, periodicidad, vigencia, gracia, fecha cobro).

**Técnico:**
- Tabla `affinity_group` con FK a `entity` y `bin`.
- CRUD en `Prepaid_Admin_API`.
- UI tipo wizard en el portal admin.

### 4.4 Reglas de negocio parametrizables

**Funcional:**
- ~30 reglas a nivel transaccional, novedades monetarias y no monetarias.
- Aplicables al ciclo de vida (emisión, administración, autorización).
- Reglas BOOLEAN: 02, 05, 07, 08, 09, 10, 13, 14, 15, 19, 20, 21, 22, 23, 26, 27, 28, 29, 66, 67, 69.
- Reglas NUMERIC: 11 (utilizaciones POS/ATM), 36 (qty diarios), 37 (monto diario), 81 (límite novedades monetarias).
- Reglas TEXT: incl/excl MCC o CU.
- Todos los límites con contador y reinicio automático según periodicidad.

**Técnico:**
- Tabla pivote `afg_business_rule` con `rule_code`, `data_type`, `value_*`, `channel`, `period`.
- Los contadores viven en Redis (`Prepaid_Limit_Counter_Service`), no en Oracle.
- CRUD en `Prepaid_Admin_API`.

**ADR:** ADR-004 (Limit Counter Service), ADR-017 (catálogo completo de límites).

### 4.5 Plan de validación de comercio

**Funcional:**
- Inclusión o exclusión de MCC o CU por AFG.
- CU hasta 11 dígitos (padding ceros izquierda si la fuente entrega menos).
- Carga API o archivo (máx 20.000 códigos por archivo).
- Archivo de respuesta consolidado (procesados + errores).
- Fuente SVBO en tiempo real para MCC/CU vigentes.

**Técnico:**
- Tablas `merchant_validation_plan` + `merchant_validation_item`.
- Componente `Merchant_Validation_Plan_Service` con cache Redis.
- Endpoint de carga masiva con streaming/chunking.
- Sincronización con SVBO (consulta de MCC/CU vigentes).

**ADR:** ADR-017.

### 4.6 Comisiones, GMF y transacciones técnicamente exitosas

**Comisiones (`commission`):**
- Por AFG: tipo de red (Propia, No Propia, Internacional), canal (POS, ATM), tipo de transacción, transacciones exentas, valor de comisión.

**GMF / impuestos (`tax_config`):**
- Solo GMF (4x1000) en Fase 1.
- Aplica por tipo de transacción (Compras, Retiros, Consulta de Saldo) y canal.
- Modo línea (sumado al débito atómico) o batch (cierre diario por entidad).
- Exención por tarjeta (`card.gmf_exempt`, posición 440 del archivo de creación).
- Acumulado mensual para tarjetas exentas (`gmf_accumulator`): se cobra solo si el TH supera UVT × N (Ley 2277/2022).

**Técnicamente exitosas (`tech_successful_charge`):**
- Cobro por respuesta 51, 55, 57, 61 o 75.
- Modo: valor fijo o tabla de comisiones.
- Activable/desactivable por AFG.

**ADR:** ADR-012 (GMF), ADR-013 (técnicamente exitosas).

### 4.7 Cuota de manejo

**Funcional:**
- Por AFG: valor, periodicidad (semanal/mensual/trimestral/semestral), forma de cobro (vencida/anticipada), vigencia, meses de gracia, fecha de cobro.

**Técnico:**
- Campos `cm_*` en `affinity_group`.
- Job `Commission Retry Scheduler` ejecuta el cobro y registra en archivo COMI.
- Si "no efectivo" → recobro recurrente hasta 3 meses (`pending_charge`).

**ADR:** ADR-014.

### 4.8 Portal admin temporal (Postman)

Hasta que se libere el portal admin Angular, las operaciones se realizan vía Postman contra el `Prepaid_Admin_API`. Operaciones mínimas: las 7 anteriores + consulta tarjetas por TH + movimientos + cambio de estado (bloqueos).

---

## 5. Etapa 2 — Ciclo de vida de la tarjeta

### 5.1 Creación de tarjetas (archivo y API)

**Funcional:**
- Canal archivo: `NXXXddmmcc` cifrado PGP por la Entidad → SFTP → GoAnywhere → NFS.
- Canal API: uno a uno.
- Validaciones: AFG existe, NIT/entidad correctos, User ID con tarjeta en estado válido, fecha del archivo no superior ni anterior a 6 días, no reproceso (UNIQUE `(file_name, control_record)`).
- Una tarjeta por (cliente, AFG): si ya hay tarjeta no Deshabilitada/Expirada → rechazar.
- Tarjeta nominada vs innominada (campo en BD).
- Email obligatorio en campo 37 del archivo.
- Marcar exentas de GMF (posición 440).
- Tarjetas físicas → `status=CREADA`. Tarjetas virtuales → `status=ACTIVA` directamente.
- Generar archivos respuesta TNN (procesadas) y TAS (rechazos) incrementales y cifrados.
- Si tarjeta nominada y TH sin usuario → invocar `PreRegistration_Service`.
- Pago con OTP para tarjetas virtuales.

**Técnico:**
- Tabla `card` con envelope encryption del PAN (`pan_encrypted`, `pan_iv`, `pan_auth_tag`, `dek_encrypted`, `kek_version`, `pan_hash`).
- CVV2 NO se almacena (PCI 3.3.2). La visualización temporal se hace consultando Pomelo desde el iframe de datos sensibles.
- `Prepaid_Card_Creation_Orchestrator` con 11 pasos (ver pestaña "Creacion de tarjetas").
- Idempotencia: UNIQUE `(client_id, afg_id)`.
- Pendiente: validación con GAW, multibolsillo, tarjeta amparada.

**ADR:** ADR-006 (envelope encryption), ADR-007 (split knowledge), ADR-015 (envelope encryption visible en C4 L2), ADR-016 (PreRegistration), ADR-018 (alineación md ↔ drawio).

### 5.2 Realce

**Funcional:**
- Pomelo dispone archivo base de realce diariamente 07:00 ARG / 05:00 COL.
- Generar un archivo por AFG con consecutivo de pedido.
- Naming: `consecutivo + AFG + fecha + nombre del Realzador`.
- Estructura de variables expuesta por Pomelo.
- Archivos cifrados PGP entregados al embozador vía SFTP.
- Estado de las tarjetas pasa a `PENDIENTE_POR_ACTIVAR`.
- Reporte de pedido por entidad (fecha, consecutivo, AFG, cantidad, entidad, embozador).
- Si Pomelo dispone fuera del horario → procesar al día siguiente.
- Errores de embozado notificados por el realzador → bloqueo y solicitud de nuevas.

**Técnico:**
- `Prepaid_Embossing_Batch` (CronJob, schedule diario tras 05:00 COL).
- `Prepaid_Embossing_Processor` agrupa por AFG, asigna consecutivo (`embossing_order`).
- `UPDATE card SET status='PENDIENTE_POR_ACTIVAR'`.

**ADR:** ADR-021 (limpieza de la pestaña Realce).

### 5.3 Distribución

**Funcional:**
- Tras envío al embozador, Credibanco notifica a la entidad emisora con el reporte de pedido.
- Datos: AFG (NIT, ciudad, dirección, teléfono, custodios) + datos del realce (consecutivo, cantidad, embozador).
- Tiempos de procesamiento idénticos al realce.

**Técnico:**
- `Distribution_Service` consume de Oracle Read Replica.
- Envío vía `B2B_Mail_Service` (cliente SMTP corporativo, distinto del `Notification_Adapter` que es B2C).
- Plantilla HTML para el correo a la entidad.
- Pendiente: definir el campo destino (¿`affinity_group.embosser_email` actual u otro nuevo?).

**ADR:** ADR-022.

### 5.4 Procesamiento de bloqueos / desbloqueos / modificación

**Funcional:**
- Bloqueo definitivo (Deshabilitación): procesa cualquier estado, captura saldo (insumo NOVR).
- Bloqueo temporal: solo desde Creada o Activa → `BLOQUEO_TEMPORAL`.
- Desbloqueo: solo desde Bloqueo Temporal → `ACTIVA`.
- Modificación de datos: NIT, AFG, doc son inmutables. Email modificable; si cambia y TH no registrado → `PreRegistration_Service`.
- Mismas validaciones de archivo (estructura, fecha, no reproceso) y respuesta TNN/TAS.

**Técnico:**
- `Prepaid_Non_Monetary_Novelty_Processor` con 4 sub-handlers diferenciados.
- Idempotencia por `novelty_id`.
- `card_state_history` registra todos los cambios.

**ADR:** ADR-020.

---

## 6. Etapa 3 — Motor transaccional

Ver detalle en `docs/requerimientos-flujo-transaccion.md`. Resumen:

### 6.1 Pipeline de autorización (10 pasos)

1. Resolver tarjeta (Oracle por `card_token`).
2. Leer AFG config desde Redis (fallback a Oracle vía `SP_FALLBACK_READ`).
3. Validar estado de tarjeta.
4. CHECK límites contra `Prepaid_Limit_Counter_Service`.
5. Reglas de negocio + Plan de Validación Comercio (`Merchant_Validation_Plan_Service`).
6. Calcular comisión.
7. Calcular GMF (si entidad=línea y tarjeta no exenta).
8. `SP_AUTHORIZE_TXN` (débito atómico monto + comisión + GMF).
9. INCREMENT contadores.
10. Response a Pomelo + `Audit_Service` async.

**Flujo alterno 1:** Si denegada con `51 / 55 / 57 / 61 / 75` → `SP_CHARGE_TECH_EXITOSA`.

**Flujo alterno 2 (Reverso/Anulación):**
1. Match transacción original (RRN + auth_code + amount + merchant + terminal + response_code).
2. Idempotencia (flag `reversada` / `anulada`).
3. DECREMENT contadores.
4. Reverso → reusa auth_code original. Anulación → genera nuevo auth_code.

### 6.2 Tipos soportados

POS: Consulta de Saldo, Compra Nacional/Internacional, Reverso de Compra, Anulación, Reverso de Anulación.
ATM: Consulta de Saldo, Retiro Nacional/Internacional, Reverso de Retiro.

### 6.3 Validaciones delegadas a Pomelo

- PIN (regla 07).
- CVV / CVV2 (reglas 08, 09).
- Fecha vencimiento (regla 13).
- Existencia y estado de tarjeta (regla 10).
- BIN y AFG activos.
- Límite intentos PIN errado.
- Validación PIN en F' (regla 69).

Credibanco **no** repite estas validaciones.

**ADRs:** ADR-002, ADR-003, ADR-004, ADR-005, ADR-013, ADR-017, ADR-018.

---

## 7. Etapa 4 — Portal tarjetahabiente y onboarding

Ver detalle en `docs/requerimientos-ciclo-vida-tarjeta-y-portal-th.md` y `Onboarding_Tarjetahabiente.drawio`.

### 7.1 Pre-registro (5 fases)

| Fase | Componente | Detalle |
|---|---|---|
| F1 | `PreRegistration_Service` | Tras crear tarjeta nominada → genera UUID v4, status=PENDIENTE, TTL 30 días, dispara email |
| F2 | `PreRegistration_Portal` + `PreRegistration_API` | TH abre URL → ingresa cédula → API verifica contra `preregistration` + tarjeta (max 3 intentos) |
| F3 | `OTP_Service` (vía Service o API directo) | OTP 6 dígitos, hash SHA-256, TTL 5 min, max 3 intentos. `session_registro` 10 min al validar |
| F4 | `PreRegistration_Service` + SSO Keycloak | TH crea password (12+ chars, mayús, minús, núm, especial). Crea usuario en Realm CARDHOLDER |
| F5 | Portal TH | TH accede con doc + password (JWT) |

**ADRs:** ADR-016 (pre-registro), ADR-023 (pestaña Onboarding en v1.5), ADR-024 (`OTP_Service` reusable).

### 7.2 Activación de tarjeta nominada

- Solo desde el portal TH (PCI: la activación requiere asignación del PIN).
- Iframe interno de PIN (`pinpad_pci_drawio_6_mejorado.drawio`).
- Validar AFG físicas (no virtuales) y `status=PENDIENTE_POR_ACTIVAR`.
- Solo una vez con la asignación del PIN.

**ADR:** ADR-008 (PIN encryption E2E).

### 7.3 Activación de tarjeta innominada

Flujo público con Proof of Possession (BIN + last4 + exp + CVV2). NO usa pre-registro.

**ADR:** ADR-010.

### 7.4 Cambio de PIN

- Desde portal TH (tarjetas físicas activas).
- Iframe de PIN.
- Si 3 PIN errados → bloqueo definitivo + notificación email/SMS.

### 7.5 Visualización de datos sensibles

- Iframe en dominio separado (`secure-data.credibanco.com`).
- Portal padre nunca ve PAN/CVV2 (PCI 6.4.3 + same-origin policy).
- CVV2 consultado a Pomelo en tiempo real.

**ADRs:** ADR-009 (iframe en dominio separado), ADR-018 (descarte CVV2).

### 7.6 Login, restablecer y cambiar contraseña

- Realm CARDHOLDER de Keycloak.
- 3 intentos máx. para login → bloqueo 24 h.
- 3 intentos máx. para restablecer → bloqueo definitivo.
- Política password: 12+ chars, mayús, minús, núm, especial.
- Token TTL 30 min.
- Notificación email cada login.

---

## 8. Etapa 5 — Procesos batch y archivos de salida

### 8.1 Cuota de manejo, comisiones e impuestos batch

- Aplican a `Activa`, `Bloqueo Temporal`, `Bloqueo PIN Errado`.
- Si "no efectivo" → recobro 3 meses (`pending_charge`).
- Cobro inmediato al detectar abono.
- Reflejados en movimientos del TH.

**ADR:** ADR-014.

### 8.2 GMF batch

Para entidades con `gmf_mode=BATCH` y para tarjetas exentas (acumulado mensual con UVT).

**ADR:** ADR-012.

### 8.3 Archivos de salida

Generados por `Output_File_Generator` desde Oracle Read Replica. Cifrados con PGP por GoAnywhere antes de entrega.

| Archivo | Tipo | Hora | Contenido |
|---|---|---|---|
| TNN xxxMMDD | En línea | Incremental | Resumen no monetarias (creación, bloqueos, modificación) |
| TAS xxxMMDD | En línea | Incremental | Rechazos no monetarias (causal) |
| TNM xxxMMDD | En línea | Incremental | Resumen novedades monetarias (abonos, débitos) |
| TAV xxxMMDD | En línea | Incremental | Rechazos monetarias |
| AUSR dd.MM | Batch | 01:00 | Reversos no aplicados (débitos parciales, abonos errados) |
| NOVR dd.MM | Batch | 01:00 | Bloqueos definitivos (con saldo) y retiros |
| TSRX dd.MM | Batch | 01:00 | Solicitudes de reexpedición desde el portal |
| COMI dd.MM | Batch | 01:00 | Comisiones cobradas y no cobradas (3 meses) |
| CINS dd.MM | Batch | 01:00 | Comisiones insatisfactorias (no cobradas) |
| CSAT dd.MM | Batch | 01:00 | Comisiones satisfactorias (cobradas) |
| SALD dd.MM | Batch | 00:00 | Saldos de tarjetas en estado válido |
| AUM dd.MM | Batch | 01:00 | Movimientos financieros del día anterior |
| Archivo Presentaciones | Batch | TBD | Trx aprobadas con BIN+PAN (Pomelo) |
| Archivo Transacciones | Batch | TBD | Todas las trx con BIN+PAN |
| ABO dd.MM | Batch | 01:00 | Abonos y débitos efectivos por novedades monetarias |
| TAR dd.MM | Batch | 01:00 | Creaciones de tarjetas |
| RAJU | Batch | TBD | Ajustes por reclamaciones |

**Eliminados del Manual Técnico Prepago:** TRX (no hay reexpedición), CANJ (no hay canje), EXT (no hay extractos), MXXXXXX (movimiento autorizador), AUMCT (consulta de costo).

### 8.4 Reconciliación End-of-Day

- Batch diario 05:00 → 08:00.
- Ingesta archivos de clearing vía SFTP.
- Matcher: card_token + amount + date + auth_code.
- Clasificación: MATCHED / UNMATCHED_LOCAL / UNMATCHED_CLEARING / AMOUNT_MISMATCH.
- `Settlement Calculator` genera posición neta por entidad.

**ADR:** ADR-011.

---

## 9. Requerimientos transversales

### 9.1 Auditoría transversal de servicios

**Obligatorio en todos los componentes.** Tabla `audit_log` particionada por mes.

Campos:
- `ID_AUDIT`, `SERVICE_NAME`, `METHOD_NAME`, `ENDPOINT`.
- `MESSAGE_ID` (id del broker), `REQUEST_PAYLOAD`, `RESPONSE_PAYLOAD`.
- `STATUS_CODE`, `STATUS_MESSAGE`, `RESPONSE_TIME_MS`.
- `TECHNICAL_USER`, `SOURCE_SYSTEM`.
- `ID_TARJETA`, `ID_USUARIO`, `ID_PROCESO`.
- `CORRELATION_ID` propagado end-to-end.
- `CREATED_AT`.

### 9.2 Auditoría PCI 10.2

Tabla `pci_event_log` particionada por mes. Campos: identificación de usuario, tipo de evento, fecha y hora, éxito/fallo, origen, recursos afectados (nombre + protocolo), componente, detalles.

### 9.3 Idempotencia

- Reverso/anulación: flag `reversada`/`anulada` en `transaction`.
- Novedades por archivo: UNIQUE `(file_name, control_record)`.
- Novedades por API: `novelty_id` en el body.
- Pre-registro: idempotente por `client_id` (uno por cliente, no por tarjeta).
- OTP: hash SHA-256 + estado `ACTIVO/USADO/EXPIRADO/BLOQUEADO`.

### 9.4 TTLs documentados (PCI 12.3.1)

| Recurso | TTL |
|---|---|
| Pre-registro UUID | 30 días |
| OTP | 5 minutos |
| Session_registro | 10 minutos |
| JWT portal TH | 30 minutos |
| Bloqueo OTP/login | 24 horas |
| Recobro pendiente | 3 meses |
| Rotación KEK | 12 meses |
| Reseguardo nombres archivo + control | 3 meses |

### 9.5 Encriptación de archivos

- **Entrada:** PGP (entidades emisoras).
- **Salida:** PGP (todos los archivos a entidades, embozadores y proveedores).

### 9.6 Notificaciones

- **B2C (al TH):** `Notification_Adapter` → SMTP corporativo. Templates HTML.
- **B2B (a entidades emisoras):** `B2B_Mail_Service` → SMTP corporativo. Plantillas HTML.

---

## 10. Modelo de datos

Ver `docs/modelo-entidad-relacion.md`, `MER_Prepago_V_1_0.drawio` y `db/01_schema_prepago.sql`. Resumen de entidades clave:

| Entidad | Propósito | ADR |
|---|---|---|
| `entity` | Entidades emisoras | — |
| `bin` | BINes por entidad | — |
| `affinity_group` | AFGs (parametría completa) | — |
| `afg_business_rule` | ~30 reglas pivote | ADR-017 |
| `commission` | Comisiones por AFG | — |
| `tax_config` | Impuestos GMF por AFG | ADR-012 |
| `tech_successful_charge` | Cobros por respuesta 51/55/57/61/75 | ADR-013 |
| `merchant_validation_plan` + `merchant_validation_item` | Plan de validación comercio | ADR-017 |
| `client` | Tarjetahabientes | — |
| `account` | Cuenta y saldo | — |
| `card` | Tarjetas + envelope encryption | ADR-006/018 |
| `card_state_history` | Historial estados | — |
| `transaction` | Movimientos (particionada por mes) | ADR-005 |
| `pending_charge` | Recobros 3 meses | ADR-014 |
| `gmf_accumulator` | Acumulado mensual GMF | ADR-012 |
| `app_user` | Usuarios portal TH | ADR-016 |
| `preregistration` | Pre-registro UUID | ADR-016 |
| `otp` | OTP (reusable: pre-registro, MFA, cambio PIN) | ADR-024 |
| `file_processing` + `file_record` | Tracking archivos | — |
| `batch_job` + `output_file` | Batch y archivos de salida | — |
| `settlement` | Reconciliación EoD | ADR-011 |
| `audit_log` + `pci_event_log` | Auditoría (particionadas por mes) | ADR-018 |
| `embossing_order` | Consecutivo de pedido por AFG | ADR-021 |

---

## 11. Cumplimiento PCI DSS v4.0

| Requisito | Cómo se cumple | ADR |
|---|---|---|
| 3.3.2 | CVV2 NO se almacena. Visualización vía iframe consultando Pomelo en tiempo real. | ADR-018 |
| 3.3 | PAN enmascarado por defecto (solo `last_four` visible). | — |
| 3.4 | PAN cifrado AES-256-GCM con DEK única por registro. | ADR-006 |
| 3.5 | KEK custodiada en KeyStore PKCS12 + OpenShift Secret. | ADR-006/007 |
| 3.6 | Rotación KEK anual; re-cifrado batch de DEKs (PANs no se tocan). `kek_version`. | ADR-007 |
| 3.6.6 | Split knowledge: passphrase dividido en 2 mitades, una por custodio. | ADR-007 |
| 3.6.7 | Dual control: toda operación sobre KEK requiere 2 personas. | ADR-007 |
| 4.2.1 | TLS 1.3 + mTLS en canales internos. | — |
| 6.4.3 | CSP + SRI en iframes; portal padre nunca lee datos sensibles. | ADR-009 |
| 7.2 | RBAC por rol (Realm ADMIN, OPERATOR, VIEWER). | — |
| 8.3.6 | MFA antes de acceso a datos sensibles. | ADR-009 |
| 10.2 | `audit_log` + `pci_event_log` particionadas por mes. | ADR-018 |
| 12.3.1 | TTLs documentados (sección 9.4). | — |

---

## 12. Performance y SLAs

### 12.1 Hot-path transaccional

- Latencia objetivo p99 del pipeline de autorización: **< 200 ms** (excluye latencia Pomelo).
- Throughput objetivo: **500 TPS** sostenidos, picos de 1500 TPS.
- HPA del `Authorization Engine`: 3 pods mínimo, 8 máximo.
- Lectura de AFG config: < 5 ms (Redis); fallback Oracle: < 50 ms.
- Débito atómico SP: < 30 ms.

### 12.2 Cold-path (batch)

- Generación archivos de salida: completar antes de las 06:00 AM (corte = 01:00 AM, 5 horas de margen).
- Reconciliación EoD: 05:00 → 08:00.
- Recobros: ejecutar cada noche.

### 12.3 Volumen estimado

- 1.500.000 tarjetas iniciales.
- Tabla `transaction` particionada por mes.

### 12.4 Disponibilidad

- Authorization Engine: 99.9% (SLA hot-path).
- Redis Cluster + Sentinel para failover automático.
- Oracle Primary + Active Data Guard.
- Fallback a Oracle si Redis no responde.

---

## 13. Observabilidad

### 13.1 Trazabilidad

- `correlation_id` propagado en headers HTTP y en mensajes de broker.
- Almacenado en `audit_log.correlation_id`.
- Se puede reconstruir un flujo end-to-end (creación → autorización → archivo de salida).

### 13.2 Logs

- Stdout en formato JSON estructurado.
- Centralización en stack ELK / OpenSearch (TBD).
- Niveles: DEBUG (dev), INFO (default), WARN, ERROR.
- **Prohibido logear:** PAN, CVV2, password, OTP en claro, dek_encrypted.

### 13.3 Métricas

- Prometheus + Grafana (TBD).
- Métricas mínimas: TPS, latencia p50/p95/p99, errores, balance de saldo, cola de pendientes.

### 13.4 Trazas distribuidas

- OpenTelemetry (TBD).
- Spans en todo el pipeline transaccional.

---

## 14. Despliegue e infraestructura

### 14.1 OpenShift

- Cada componente como Deployment con HPA.
- Secrets: `kek-keystore` (binario), `kek-password-part-a`, `kek-password-part-b`, `pomelo-credentials`, `smtp-credentials`.
- Volumes: `/secrets/kek/` (read-only), `/input`, `/output`, `/error`, `/processed` (NFS).
- Networking: TLS 1.3 internos; mTLS donde sea factible.
- Routes con TLS termination en el Authorization Gateway.

### 14.2 CI/CD

- Pipeline por componente (TBD).
- Build → tests unitarios → tests integración → análisis estático → imagen Docker → despliegue dev → smoke tests → promoción.

### 14.3 Ambientes

- Dev, QA, Staging (donde se prueba contra Pomelo sandbox), Prod.

### 14.4 Backups y DR

- Oracle: backup diario + RPO 15 min via Active Data Guard.
- Redis: persistencia AOF + replicación.
- KEK KeyStore: backup cifrado en 2 ubicaciones físicas.
- KEK antigua archivada (no destruida) hasta que no haya datos cifrados con ella.

---

## 15. Decisiones diferidas y pendientes

### 15.1 Pendientes funcionales del documento fuente

1. Detalle de validaciones que ejecuta GAW hoy (creación, bloqueo, modificación, realce, distribución).
2. Multibolsillo y tarjeta amparada (diferido a Fase 2).
3. Tipos de transacción virtuales distintos a VNP.
4. Generación de tarjeta física a partir de tarjeta virtual.
5. Mecanismo de entrega del CVV al TH para tarjetas virtuales.
6. UX detallada del Portal Front (activación, cambio PIN, restablecer/cambiar contraseña, login).
7. Devolución del 4x1000 cuando una transacción se reversa.

### 15.2 Pendientes de integración con Pomelo

1. Contrato exacto de alta de Entidad (alta CCO, Client ID).
2. Contrato de creación de BIN (siembra de llaves).
3. Notificación de bloqueos por PIN errado ejecutados por Pomelo.
4. Costo de operaciones ATM (`extra_data.function_code`, `ATM_FEE_INQUIRY`).
5. Contenido del objeto `MERCHANT` y `merchant_id` en reverso de retiro ATM.
6. Existencia de `registro_control` en archivo de realce.
7. Validación de tipo de producto en pre-autorizaciones (ahorros/crédito/corriente).
8. Restricciones a la modificación de datos vía API.

### 15.3 Pendientes técnicos

1. Stack de reportería (Power BI vs Qlik vs otro).
2. Proveedor de notificaciones email/SMS (Twilio, AWS SES, interno).
3. Herramienta de fraude (Fase 2).
4. Stack de observabilidad (ELK vs Grafana Loki, OpenTelemetry).
5. Broker de mensajería para asincronía entre orchestrators (Kafka vs RabbitMQ).
6. Definir destinatario del correo del `B2B_Mail_Service` (si reusa `affinity_group.embosser_email` o se añade nuevo campo).
7. Si el envío de OTP por SMS requiere `SMS_Adapter` aparte del `Notification_Adapter`.

### 15.4 ADRs pendientes (decisiones por tomar)

- Fraude (estructura de datos y herramientas).
- Reportería (stack y vistas adicionales).
- Notificaciones (proveedor SMS).
- Multibolsillo (modelado y reglas).
- Tarjeta amparada (modelado y reglas).

---

## 16. Referencias

### 16.1 Documentos fuente

- `Prepago/Requerimientos Técnicos (12).docx` — fuente primaria de requerimientos funcionales.
- `Prepago/docs/requerimientos-flujo-transaccion.md` — extracto consolidado del Motor Transaccional (Etapa 3).
- `Prepago/docs/requerimientos-ciclo-vida-tarjeta-y-portal-th.md` — extracto consolidado de Etapas 2 y 4.
- `Prepago/docs/decisiones-arquitectura.md` — bitácora de ADR-001 a ADR-024.

### 16.2 Diagramas de arquitectura

- `Prepago/PrepagoUnificadoArquitectura_V_1_5.drawio` — arquitectura principal vigente (16 pestañas C4 L2).
- `Prepago/Envelope_Encryption_PAN.drawio` — detalle envelope encryption + ceremonia KEK.
- `Prepago/Iframe_Visualizacion_DatosSensibles.drawio` — iframe en dominio separado.
- `Prepago/pinpad_pci_drawio_6_mejorado.drawio` — flujo PIN E2E con HSM Thales.
- `Prepago/Activacion_PIN_Innominada.drawio` — activación tarjeta innominada.
- `Prepago/Onboarding_Tarjetahabiente.drawio` — flujo de onboarding por fases (swimlanes).

### 16.3 Modelo de datos

- `Prepago/MER_Prepago_V_1_0.drawio` — diagrama Entidad-Relación.
- `Prepago/db/01_schema_prepago.sql` — DDL Oracle 19c+.
- `Prepago/db/02_stored_procedures.sql` — SPs del motor transaccional.
- `Prepago/docs/modelo-entidad-relacion.md` — explicación del MER.

### 16.4 Soporte técnico

- `Prepago/docs/envelope-encryption.md` — detalle técnico envelope encryption.
- `Prepago/docs/inventario-archivos-prepago.md` — inventario de todos los archivos del proyecto.
- `Prepago/formatoConcepto.docx` — plantilla para conceptos de arquitectura.

### 16.5 Conceptos formales

- `Prepago/Concepto_Motor_Transaccional_V_1_0.docx` — concepto del Motor Transaccional.
- `Prepago/Concepto_Ciclo_Vida_Tarjeta_V_1_0.docx` — concepto del Ciclo de Vida de la Tarjeta.

### 16.6 Bitácora de ADRs (resumen)

| ADR | Tema |
|---|---|
| ADR-001 | BFF por portal (Admin / Cardholder API separados) |
| ADR-002 | Redis cache de AFG config + fallback Oracle |
| ADR-003 | Stored Procedures Oracle para débito atómico |
| ADR-004 | Limit Counter Service como componente separado |
| ADR-005 | Tabla `transaction` particionada por mes |
| ADR-006 | Envelope Encryption (RSA + AES, sin HSM para PAN) |
| ADR-007 | KEK con Split Knowledge (2 custodios) |
| ADR-008 | PIN encryption E2E con SubtleCrypto + HSM Thales |
| ADR-009 | Iframe en dominio separado para datos sensibles |
| ADR-010 | Activación PIN innominadas vía Proof of Possession |
| ADR-011 | Reconciliación End-of-Day contra clearing |
| ADR-012 | GMF línea vs batch (config por entidad) |
| ADR-013 | Cobro de transacciones técnicamente exitosas |
| ADR-014 | Recobro de comisiones hasta 3 meses |
| ADR-015 | Envelope Encryption visible en C4 L2 (Creación) |
| ADR-016 | Pre-registro disparado tras creación nominada |
| ADR-017 | C4 L2 Flujo de transacciones v1.4 (Plan Validación + Audit + Catálogo de límites) |
| ADR-018 | Conciliación `envelope-encryption.md` con drawio (RAW, sin CVV2) |
| ADR-019 | Pestaña MER + SQL + ADR |
| ADR-020 | Pestaña "Flujo Novedades No Monetarias" en v1.4 |
| ADR-021 | Limpieza pestaña "Realce" (Embossing_Processor) |
| ADR-022 | Distribución como componente + `B2B_Mail_Service` |
| ADR-023 | v1.5 — Pestaña "Onboarding Tarjetahabiente" |
| ADR-024 | `OTP_Service` como componente reusable |

---

> **Cómo usar este documento.** Es el índice maestro. Para detalle de un tema específico, sigue las referencias a los documentos auxiliares (extractos, ADRs, drawio). No duplica el contenido de los archivos fuente; los consolida y referencia.
