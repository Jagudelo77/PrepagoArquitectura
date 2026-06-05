# Modelo Entidad-Relación — Plataforma Prepago Credibanco v1.0

Modelo de datos Oracle 19c+ que cubre los requerimientos de `Requerimientos Técnicos (12).docx` (Fase 1 MVP, Etapas 1-5) y se alinea con la arquitectura `PrepagoUnificadoArquitectura_V_1_4.drawio`.

## Artefactos

| Archivo | Propósito |
|---|---|
| `MER_Prepago_V_1_0.drawio` | Diagrama MER visual con todas las entidades y relaciones, agrupadas por etapa. |
| `db/01_schema_prepago.sql` | Script DDL completo (tablas, constraints, índices, vistas). Fuente de verdad. |
| `db/02_stored_procedures.sql` | SPs del Motor Transaccional (SP_AUTHORIZE_TXN, SP_CHARGE_TECH_EXITOSA, SP_FALLBACK_READ, SP_REVERSE_TXN, SP_CANCEL_TXN). |
| `docs/modelo-entidad-relacion.md` | Este documento — explicación del modelo. |

## Decisiones de diseño

Las decisiones del modelo no se inventan aquí; son consecuencia de los ADRs ya documentados:

| ADR | Impacto en el modelo |
|---|---|
| **ADR-002** | Redis cachea AFG; el modelo Oracle es la fuente de verdad y el fallback (`SP_FALLBACK_READ`). |
| **ADR-003** | Débito atómico vive en SPs (`SP_AUTHORIZE_TXN`, `SP_CHARGE_TECH_EXITOSA`). El esquema expone una tabla `transaction` única. |
| **ADR-004** | Los contadores de límites NO viven en Oracle, viven en Redis. La tabla `transaction` los reconstruye si hace falta. |
| **ADR-005** | `transaction`, `audit_log` y `pci_event_log` particionadas por mes con `INTERVAL`. |
| **ADR-006/007/018** | Tabla `card` tiene columnas `RAW` para envelope encryption del PAN. Sin CVV2. |
| **ADR-012** | Flag `entity.gmf_mode` (LINEA/BATCH) y `card.gmf_exempt`. Tabla `gmf_accumulator` para tarjetas exentas. |
| **ADR-013** | Tabla `tech_successful_charge` parametriza códigos 51, 55, 57, 61, 75. |
| **ADR-014** | Tabla `pending_charge` con `expires_at = ADD_MONTHS(SYSTIMESTAMP, 3)`. |
| **ADR-016** | Tablas `app_user` (1:1 con client) y `preregistration` (UUID + TTL 30 días). |
| **ADR-017** | Tablas `merchant_validation_plan` + `merchant_validation_item` (Plan Validación Comercio). |

## Mapeo Requerimiento → Entidad

### Etapa 1 — Administración

| Requerimiento | Entidad / columna |
|---|---|
| Habilitar Entidad en Credibanco y Pomelo | `entity` |
| Activar / desactivar entidad | `entity.status` |
| Definir `gmf_mode` por entidad | `entity.gmf_mode` |
| Parametrizar BIN | `bin` |
| Tipo de BIN (PREPAGO en Fase 1) | `bin.bin_type` |
| Activar / desactivar BIN | `bin.status` |
| Parametrizar AFG (con campos Pomelo + Credibanco) | `affinity_group` |
| Custodios 1 y 2 de la tarjeta | `affinity_group.custodian1_*`, `custodian2_*` |
| Cuota de manejo (valor, periodicidad, vigencia, gracia, fecha) | `affinity_group.cm_*` |
| Pomelo es la fuente del afg_id | `affinity_group.pomelo_afg_id` |
| Reglas de negocio parametrizables (02, 11, 14, 19, 20-29, 36, 37, 66, 67, 69, 81…) | `afg_business_rule` (pivot por `rule_code`) |
| Comisiones por red / canal / tipo / exentas | `commission` |
| Cobro Transacciones Técnicamente Exitosas | `tech_successful_charge` |
| Impuesto GMF (4x1000) | `tax_config` |
| Plan Validación Comercio (MCC/CU, inclusión/exclusión) | `merchant_validation_plan` + `merchant_validation_item` |
| Auditoría transversal (ID_AUDIT, SERVICE_NAME, REQUEST/RESPONSE, MS, etc.) | `audit_log` |
| Auditoría PCI 10.2 (eventos, success/fail, recursos) | `pci_event_log` |

### Etapa 2 — Ciclo de Vida de la Tarjeta

| Requerimiento | Entidad / columna |
|---|---|
| Tarjetahabiente | `client` |
| Cuenta y saldo | `account` |
| Tarjeta (física o virtual, nominada o innominada) | `card` |
| Estados: CREADA / PENDIENTE_ACTIVAR / ACTIVA / BLOQUEO_TEMPORAL / BLOQUEO_PIN_ERRADO / DESHABILITADA / EXPIRADA | `card.status` (CHECK constraint) |
| Idempotencia: una tarjeta por (cliente, AFG) | UNIQUE `(client_id, afg_id)` en `card` |
| PAN cifrado (envelope encryption) | `card.pan_encrypted/iv/auth_tag/dek_encrypted/kek_version/pan_hash` |
| CVV2 NO se almacena (PCI 3.3.2) | Sin columna en `card` |
| Exención GMF por tarjeta (pos. 440) | `card.gmf_exempt` |
| Auditoría de cambios de estado | `card_state_history` |
| Tracking de archivos NXXXddmmcc / MXXXddmmcc | `file_processing` |
| Validación: no procesar 2 veces el mismo nombre + control | UNIQUE `(file_name, control_record)` |
| Detalle de cada registro procesado | `file_record` |

### Etapa 3 — Motor Transaccional

| Requerimiento | Entidad / columna |
|---|---|
| Movimientos POS y ATM | `transaction` (particionada por mes) |
| Tipos: COMPRA, RETIRO, CONSULTA_SALDO, REVERSO_*, ANULACION, REVERSO_ANULACION, TECH_EXITOSA, AJUSTE_*, ABONO, CARGO, COMISION, GMF, CUOTA_MANEJO, RECOBRO | `transaction.txn_type` (CHECK) |
| Movimiento DÉBITO/CRÉDITO | `transaction.movement_type` |
| Idempotencia reverso/anulación | `transaction.reversed`, `transaction.cancelled` |
| Reuso de auth_code (reverso) / nuevo (anulación) | Lógica en `SP_REVERSE_TXN` / `SP_CANCEL_TXN` |
| Matching contra txn original | `transaction.parent_txn_id` |
| Comisión + GMF sumados al débito | columnas `commission_amount`, `gmf_amount`, `total_amount` |
| Recobros recurrentes hasta 3 meses | `pending_charge.expires_at` |
| Acumulado GMF mensual para tarjetas exentas | `gmf_accumulator` |
| Contadores de límites en Redis | NO en Oracle (ADR-004), nota en el MER |

### Etapa 4 — Portal Tarjetahabiente

| Requerimiento | Entidad / columna |
|---|---|
| Usuario portal TH (1:1 con cliente) | `app_user` |
| Pre-registro UUID + TTL 30 días | `preregistration` |
| OTP (5 minutos, máx 3 intentos) | `otp` |
| OTP almacena hash, no el código | `otp.code_hash` (RAW SHA-256) |

### Etapa 5 — Procesos Batch y Archivos de Salida

| Requerimiento | Entidad / columna |
|---|---|
| Tracking de jobs (commission retry, GMF batch, EOD, realce…) | `batch_job` |
| Archivos de salida TNN, TAS, TNM, TAV, AUSR, NOVR, TSRX, COMI, CINS, CSAT, SALD, AUM, PRESENT, TXNS, REALCE, ABO, RAJU | `output_file` con `file_type` |
| Archivos cifrados PGP | `output_file.encryption_status = 'PGP'` |
| Reconciliación End-of-Day vs clearing | `settlement` |
| Posición neta de liquidación por entidad | `settlement.net_position` |

## Particiones e índices

- **`transaction`** — `PARTITION BY RANGE (txn_date) INTERVAL (NUMTOYMINTERVAL(1,'MONTH'))`. Crea particiones automáticas mes a mes. Índices LOCAL en `card_id`, `account_id`, `auth_code`, `rrn`, `pomelo_txn_id`, `response_code`. Beneficios: queries por rango de fechas solo tocan particiones relevantes; archivado/purga por `DROP PARTITION`; reconciliación EoD lee solo la partición del día.
- **`audit_log`** y **`pci_event_log`** — mismo esquema de particionamiento. Mantiene los logs accesibles por mes y permite retention policy declarativa.
- **`card`** — índices en `pan_hash` (búsqueda exact-match sin descifrar), `kek_version` (rotación batch), `client_id`, `afg_id`, `status`, `card_token`.

## Cumplimiento PCI DSS v4.0

| Req | Cumplimiento en el modelo |
|---|---|
| 3.3.2 | Sin columna CVV2. El CVV2 se consulta a Pomelo en tiempo real. |
| 3.4 | `card.pan_encrypted` AES-256-GCM con DEK por registro. |
| 3.5 | KEK en PKCS12 + OpenShift Secret. `card.dek_encrypted` (RSA-4096 OAEP). |
| 3.6 | `card.kek_version` para rotación. Re-cifrado batch de DEKs (no PANs). |
| 3.6.6 / 3.6.7 | Split knowledge + dual control en passphrase del KeyStore (operacional, no en BD). |
| 4.2.1 | TLS 1.3 / mTLS — capa de transporte, no en el esquema. |
| 7.2 | RBAC sobre el usuario `prepago` (asignar GRANTs por rol). |
| 8.3.6 | MFA en el portal antes de visualizar PAN — operacional. |
| 10.2 | Tabla `pci_event_log` con todos los campos requeridos. |
| 12.3.1 | TTLs documentados: OTP 5 min, preregistration 30 días, pending_charge 3 meses, KEK rotación 12 meses. |

## Ejecución del DDL

Orden recomendado:

```bash
# 1. Crear usuario y tablespaces (DBA)
sqlplus / as sysdba @00_setup.sql   # opcional, comentado en el header del script

# 2. Crear esquema
sqlplus prepago/<password> @db/01_schema_prepago.sql

# 3. Crear stored procedures
sqlplus prepago/<password> @db/02_stored_procedures.sql

# 4. (Pendiente) Crear datos semilla y catálogos
# sqlplus prepago/<password> @db/03_seed_data.sql
```

## Pendientes y decisiones diferidas

1. **Multibolsillo y tarjeta amparada** — modelo actual asume cuenta 1:1 con cliente. Cuando se defina, se añadirá una entidad `pocket` o `wallet` que rompa la relación (1 cliente → N cuentas).
2. **Fraude** — la arquitectura deja hooks pero el modelo no incluye tablas específicas. Cuando se elija herramienta, se añadirá `fraud_event`, `fraud_rule`, etc.
3. **Fee_id concreto de Pomelo** — el campo `entity.fee_id` está como `VARCHAR2(20)` opcional; ajustar tipo cuando Pomelo lo confirme.
4. **Reportería / Power BI** — usa el Read Replica vía vistas (`v_card_active`, `v_recent_transactions`); más vistas se irán añadiendo según necesidad.

## Diferencias contra `Prepago/On-prem/MER_Prepago_Inicial.drawio`

El MER inicial fue una primera aproximación. Las diferencias principales del v1.0:

- **Envelope encryption explícito** en `card` (RAW) en lugar de columnas genéricas.
- **`afg_business_rule` como pivot** en lugar de columnas dispersas en `affinity_group`.
- **`merchant_validation_plan` + `_item`** separadas (antes no aparecía).
- **`pending_charge` y `gmf_accumulator`** como tablas dedicadas (antes implícitas).
- **Particionamiento de `transaction`** declarado (antes solo mencionado).
- **Auditoría transversal y PCI 10.2** como tablas particionadas (antes ausentes).
- **Pre-registro y OTP** como entidades de Etapa 4 (antes no estaban).
- **`output_file` y `settlement`** como entidades de Etapa 5 (antes no estaban).

Si el equipo prefiere mantener el `MER_Prepago_Inicial.drawio` actualizado, el siguiente paso sería sustituir su contenido por el v1.0. Por convención de versionado, el v1.0 se publica como archivo separado.
