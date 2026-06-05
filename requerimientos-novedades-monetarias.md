# Requerimientos — Novedades Monetarias

Documento puntual de los flujos de **Novedades Monetarias** de la plataforma prepago Credibanco. Cubre la aplicación de abonos y cargos por archivo y por API, ajustes por reclamaciones desde Pomelo, GMF en línea y en batch (Ley 2277/2022), recobro de cargos no efectivos, regla `81` (Límite de Novedades Monetarias por Monto), multibolsillo y tarjeta amparada, archivos de salida asociados y reportería.

**Fuentes consolidadas:**
- `Prepago/Requerimientos Técnicos (12).docx` — secciones "Aplicación de Novedades Monetarias (Abonos-Crédito y Cargos-Débito) (Archivo-Api)", "Ajustes por Reclamaciones", "Cobro de Comisiones", "Cobro de Transacciones Técnicamente Exitosas", "Cobro de Impuestos", "GMF en Línea", "Manejo de Información para Archivos de Salida", "Monitoreo".
- `Prepago/docs/requerimientos-flujo-transaccion.md` — §10 (Catálogo de límites incl. regla 81), §11 (post-autorización y cierres), §12 (archivos de salida).
- `Prepago/docs/requerimientos-tecnicos-consolidados.md` — secciones de componentes (`Prepaid_Monetary_Novelty_Processor`, archivos de salida).
- `Prepago/docs/requerimientos-ciclo-vida-tarjeta-y-portal-th.md` — secciones 1, 4, 5 (intercambio NXXX/MXXX y procesamiento por archivo).
- `Prepago/docs/decisiones-arquitectura.md` — ADR-005 (particionamiento), ADR-017 (límites + DECREMENT), ADR-018 (envelope encryption PAN), ADR-020 (Flujo Novedades No Monetarias — referencia simétrica), ADR-022 (`B2B_Mail_Service`).
- `Prepago/db/01_schema_prepago.sql` — tablas `file_processing`, `file_record`, `transaction`, `pending_charge`.
- `Prepago/db/02_stored_procedures.sql` — `SP_AUTHORIZE_TXN`, `SP_CHARGE_TECH_EXITOSA`.
- `Prepago/PrepagoUnificadoArquitectura_V_1_5.drawio` — pestaña "Intercambio de Archivos".

> Fase: **Fase 1 — Línea Base (MVP)**, Etapas 2 (Ciclo de Vida) y 5 (Procesos Batch y Archivos de Salida).
> Última extracción: 27/05/2026.
> Estado: **Vigente**.

---

## Tabla de contenido

1. [Resumen y alcance](#1-resumen-y-alcance)
2. [Glosario](#2-glosario)
3. [Disparadores y canales de entrada](#3-disparadores-y-canales-de-entrada)
4. [Flujo funcional por escenario](#4-flujo-funcional-por-escenario)
5. [Componentes y responsabilidades](#5-componentes-y-responsabilidades)
6. [Estructura del archivo MXXXddmmcc](#6-estructura-del-archivo-mxxxddmmcc)
7. [Contratos de API](#7-contratos-de-api)
8. [Reglas de negocio](#8-reglas-de-negocio)
9. [Modelo de datos](#9-modelo-de-datos)
10. [Archivos de salida derivados](#10-archivos-de-salida-derivados)
11. [Errores, rechazos y reintentos](#11-errores-rechazos-y-reintentos)
12. [Reportería operativa](#12-reportería-operativa)
13. [Seguridad y cumplimiento PCI](#13-seguridad-y-cumplimiento-pci)
14. [Auditoría](#14-auditoría)
15. [Performance y SLAs](#15-performance-y-slas)
16. [Pendientes](#16-pendientes)
17. [Referencias cruzadas](#17-referencias-cruzadas)

---

## 1. Resumen y alcance

### 1.1 Objetivo

Procesar de forma confiable, en línea y por lote, las **afectaciones financieras administrativas** que las Entidades Emisoras (a través de archivo o API) o Pomelo (a través de ajustes por reclamaciones) instruyen sobre las cuentas y tarjetas prepago, garantizando trazabilidad, validación de saldos, aplicación de la regla `81`, recobros automáticos de cargos no efectivos, generación de archivos de respuesta y reportería operativa.

### 1.2 Alcance funcional (Fase 1 MVP)

**Incluye:**
- Aplicación de **Abonos (Crédito)** y **Cargos (Débito)** por archivo `MXXXddmmcc` y por API uno-a-uno.
- **Ajustes por reclamaciones** recibidos desde Pomelo (Débito o Crédito), 24/7.
- **GMF en línea** (modelo actual, motor transaccional) y **GMF batch** (Ley 2277/2022 art. 881-1 ET) **recibido como novedad por archivo de la Entidad** (nombre y estructura TBD — ver §4.5 y §16). La Entidad calcula UVT; Credibanco aplica.
- **Recobro automático** de cargos no efectivos (cargos cuyo saldo no alcanzó), almacenando el faltante para cobro al detectar abonos.
- **Validación regla `81`** Límite de Novedades Monetarias por Monto (numérica, según AFG).
- **Multibolsillo y tarjeta amparada** — flujos y reglas específicas.
- Generación de archivos de salida **TNM**, **TAV**, **AUSR**, **ABO**, **RAJU**.
- Validación dura del archivo (estructura, fecha del nombre +/- 6 días, registro de control, longitud 200 bytes).
- Idempotencia: no procesar dos veces el mismo `(file_name, control_record)`.

**Excluye:**
- Autorizaciones transaccionales (POS / ATM / VNP) — están en `requerimientos-flujo-transaccion.md`.
- Novedades **No Monetarias** (creaciones, bloqueos, modificación de datos) — en `requerimientos-ciclo-vida-tarjeta-y-portal-th.md` y ADR-020.
- Comisiones por transacción (cobro de comisiones derivado de POS/ATM) — referenciado como insumo, no detallado aquí.
- Reintentos automáticos del orquestador de archivos cuando el procesamiento falla por sistema → **Desarrollo en Fase 2** (ver §11.4).

### 1.3 Resultado esperado

Al final del flujo de un archivo `MXXXddmmcc` o de un request API:

- Cada novedad procesada genera un registro en `transaction` con `txn_type` = `ABONO` | `CARGO` | `AJUSTE_CREDITO` | `AJUSTE_DEBITO` | `RECOBRO`.
- El saldo de la cuenta queda actualizado **en línea**.
- Los rechazos quedan en `file_record` con `response_code` y `response_message`.
- Se genera entrada incremental en los archivos de salida **TNM** (resumen) y **TAV** (rechazos).
- Cargos no efectivos quedan en `pending_charge` con `expires_at = now + 3 meses`.
- Hay traza completa en `audit_log` y `pci_event_log`.

---

## 2. Glosario

| Sigla / término | Definición |
|---|---|
| **Abono** | Movimiento crédito a la cuenta (incrementa saldo). Origen: Entidad Emisora. |
| **Cargo** | Movimiento débito a la cuenta (decrementa saldo). Origen: Entidad Emisora. |
| **Ajuste por Reclamación** | Movimiento (débito o crédito) originado por Pomelo tras canje y compensación de disputas/controversias. |
| **AFG** | Affinity Group — agrupación de tarjetas con misma parametrización. |
| **NIT** | Número de identificación tributaria de la Entidad Emisora. |
| **MXXXddmmcc** | Nombre canónico del archivo de novedades monetarias. `XXX`=AFG · `dd`=día · `mm`=mes · `cc`=consecutivo. |
| **Registro de Control** | Trailer del archivo con NIT, fecha, total de registros y valor total. |
| **TNM** | Archivo de salida — Resumen de Novedades Monetarias (incremental, en línea). |
| **TAV** | Archivo de salida — Rechazos de Novedades Monetarias (incremental, en línea). |
| **AUSR** | Archivo de salida — Reversos no aplicados (batch diario 01:00). |
| **ABO** | Archivo de salida — Abonos y débitos efectivos (batch diario 01:00). |
| **RAJU** | Archivo de salida — Rechazos de ajustes por reclamaciones. |
| **Recobro** | Cobro recurrente de un cargo no efectivo, aplicado al detectar abono posterior. Vigencia 3 meses. |
| **GMF** | Gravamen al Movimiento Financiero (4×1000), Ley 2277/2022. |
| **UVT** | Unidad de Valor Tributario, base para exoneración mensual de GMF. |
| **Multibolsillo** | Tarjeta con múltiples cuentas/bolsillos lógicos. |
| **Tarjeta Amparada** | Tarjeta secundaria asociada a una principal, comparte cupo o saldo. |
| **Regla 81** | Límite Novedades Monetarias por Monto (numérico, según periodicidad AFG). |

---

## 3. Disparadores y canales de entrada

| # | Disparador | Canal | Volumen | Frecuencia |
|---|---|---|---|---|
| 1 | Archivo `MXXXddmmcc` recibido por GoAnywhere → SFTP/NFS | **Archivo** | Lote (decenas a miles de registros/archivo) | Continuo durante el día, según horarios de la Entidad |
| 2 | Request API uno-a-uno desde la Entidad | **API REST** (mTLS + JWT) | 1 novedad por request | Online |
| 3 | Webhook de **Ajustes** desde Pomelo (`POST /webhooks/adjustment`) | **API REST** | 1 ajuste por request | 24/7 |
| 4 | Archivo **GMF** de la Entidad (nombre y estructura **TBD** — hipótesis: `GXXXddmmcc`) por GoAnywhere → SFTP/NFS | **Archivo** | Lote mensual (~tarjetas activas con GMF aplicable) | Mensual al cierre (por confirmar con Entidad) |

> Nota: el archivo viene **cifrado por la Entidad** (PGP). El descifrado lo hace GoAnywhere MFT antes de depositar el archivo limpio en el NFS de OpenShift.

---

## 4. Flujo funcional por escenario

### 4.1 Procesamiento de archivo `MXXXddmmcc`

```
[Entidad] → SFTP cifrado → [GoAnywhere MFT] → descifra PGP → NFS
   ↓
[File_Watcher_Batch] (Spring Batch, scheduler 5 min)
   ↓ detecta MXXX...
[Prepaid_Monetary_Novelty_Processor]
   ├── 1. Valida nombre (MXXXddmmcc, fecha ±6 días)
   ├── 2. Valida no duplicado: SELECT file_processing WHERE file_name=? AND control_record=?
   ├── 3. Valida registro de control (NIT, total registros, fecha trailer ±6 días)
   ├── 4. Valida AFG ↔ NIT ↔ Entidad
   ├── 5. Valida longitud de cada registro (200 bytes)
   └── 6. Por cada registro:
        ├── Resuelve card_id por (afg_id, doc_type, doc_number)
        ├── Valida estado tarjeta ∈ {Creada, Activa, Bloqueo Temporal}
        ├── Valida regla 81 (Limit_Counter_Service.CHECK)
        ├── Si CARGO y saldo insuficiente: descuenta lo disponible + INSERT pending_charge
        ├── Si ABONO y existe pending_charge: ejecuta SP_RECOVER_CHARGES
        ├── INSERT transaction (txn_type=ABONO/CARGO, atomic en SP_APPLY_MONETARY)
        ├── Actualiza saldo account
        ├── Limit_Counter_Service.INCREMENT (regla 81)
        └── INSERT file_record con resultado
   ↓
[Output_File_Generator] genera TNM (resumen) y TAV (rechazos), incremental, archivo PLANO en NFS
   ↓
[GoAnywhere] → Entidad
```

### 4.2 Procesamiento por API uno-a-uno

```
Entidad → POST /api/v1/monetary-novelty (mTLS + JWT)
   ↓
[Prepaid_Admin_API_Gateway]
   ├── Autentica
   ├── Valida idempotency-key (header X-Idempotency-Key)
   └── Reenvía a [Prepaid_Monetary_Novelty_Service]
        ├── Valida AFG ↔ NIT ↔ Entidad ↔ User ID ↔ tarjeta
        ├── Valida estado tarjeta
        ├── Valida regla 81
        ├── Si CARGO y saldo insuficiente → 409 + crea pending_charge
        ├── Llama SP_APPLY_MONETARY (atómico)
        └── Si ABONO → SP_RECOVER_CHARGES
   ↓
HTTP 200 + JSON {response_code, response_message, transaction_id, balance_after}
```

### 4.3 Ajuste por reclamación desde Pomelo

```
Pomelo → POST https://prepago.credibanco.com/webhooks/adjustment
   Body: { reclamation_id, card_id_pomelo, type=DEBITO|CREDITO, amount, concept, original_txn_id }
   ↓
[Pomelo_Webhook_Adapter]
   ├── Valida HMAC-SHA256 (header X-Pomelo-Signature)
   └── Reenvía a [Adjustment_Service]
        ├── Resuelve card_id local desde card_id_pomelo
        ├── Si DEBITO y saldo insuficiente:
        │     → Rechaza con {causa: "Saldo Insuficiente"}
        │     → Marca para archivo RAJU
        ├── Si DEBITO y tarjeta Bloqueada Definitiva: Rechaza
        ├── Si Bloqueo Temporal: aplica igual (regla explícita)
        ├── INSERT transaction (txn_type=AJUSTE_DEBITO|AJUSTE_CREDITO)
        ├── Actualiza saldo account
        └── Aplicación EN LÍNEA
   ↓
HTTP 200 + JSON respuesta Pomelo
   ↓
[Output_File_Generator] genera ABO (efectivos) y RAJU (rechazos) en cierre 01:00
```

### 4.4 GMF en línea (modelo actual)

Aplicado dentro del motor transaccional sobre cada transacción configurada como gravable. Se incluye en `transaction.gmf_amount` y se reporta en archivos `COMI` y `AUM`. Detallado en `requerimientos-flujo-transaccion.md` §11.6.

### 4.5 GMF en batch (Ley 2277/2022, art. 881-1 ET)

> **Corrección importante (ADR-028):** la versión inicial de este documento describía el GMF batch como un **job interno** que computaba UVT y aplicaba el cobro. Esto era una interpretación errónea. El proceso real es:
>
> - La **Entidad emisora** calcula el GMF según las reglas de UVT (Ley 2277/2022 art. 881-1 ET) en su sistema.
> - La Entidad envía a Credibanco **un archivo cifrado por SFTP** (vía GoAnywhere) con las novedades de GMF a aplicar por tarjeta.
> - Credibanco **aplica la novedad** indicada por la Entidad — somos publisher pasivo, no calculamos UVT.

Flujo:

```
Entidad → SFTP cifrado PGP → GoAnywhere descifra → NFS
   ↓ File_Watcher_Batch detecta archivo GMF
[Prepaid_Monetary_Novelty_Processor]  (mismo writer del flujo MXXX)
   ├── Valida estructura (TBD: nombre del archivo, longitud, trailer)
   ├── Valida fecha del nombre ±6 días
   ├── Valida AFG ↔ NIT ↔ Entidad
   ├── Por cada registro: type=GMF, card_id, amount, concept, period
   └── Invoca Charge_Settlement_Service.settle(card_id, GMF, amount, ...)
        ├── Si saldo OK → SP_SETTLE_CHARGE → INSERT transaction(GMF) + UPDATE balance
        └── Si saldo insuficiente → INSERT pending_charge(charge_type=GMF, expires_at = now + 90d)
   ↓
Output_File_Generator (cierre 01:00)
   • TNM (resumen — incluye GMF aplicados)
   • TAV (rechazos — NIT inválido, tarjeta no existe, etc.)
   • COMI (cobrados + recobros) | CSAT (subset cobrados) | CINS (expirados a 90 días)
```

**Pendientes de definición** (la Entidad aún no ha cerrado el formato):
- Nombre canónico del archivo. Hipótesis: `GXXXddmmcc` con prefijo dedicado, análogo a `MXXXddmmcc` para monetarias y `NXXXddmmcc` para no monetarias. Por confirmar con la Entidad.
- Estructura del registro detalle (campos esperados, longitud, separadores).
- Validaciones específicas del trailer (NIT, fecha, total, valor sumado).
- Periodicidad de envío (mensual al cierre vs. otro corte).
- Devolución del 4×1000 cuando una transacción gravada se reversa (este pendiente venía de §16 item 1; queda como decisión a coordinar con la Entidad).

**Implicación arquitectónica:**
- **No existe** un `GMF_Batch_Job` interno que calcule UVT. La caja `GMF_Calculator` que apareció en versiones previas del diagrama unificado (V_1_8) fue retirada y reemplazada en V_1_9 por una nota TBD.
- El GMF batch se materializa como una **fuente más** del `Prepaid_Monetary_Novelty_Processor`, igual que cargos y abonos.
- En el "Motor de Cobros Diferidos" el GMF aparece solo como `charge_type=GMF` en `pending_charge` cuando el cobro proveniente del archivo no alcanza saldo.

### 4.6 Recobro de cargos no efectivos

Cuando un Cargo (o cualquier cobro: comisión, técnicamente exitosa, GMF, cuota de manejo) no alcanza saldo, se descuenta lo disponible y se crea registro en `pending_charge`. El recobro se ejecuta:

1. **En línea** al recibir un Abono nuevo (Pomelo, archivo, ajuste): el `Prepaid_Monetary_Novelty_Service` invoca `SP_RECOVER_CHARGES(card_id)` que itera `pending_charge` ordenado por `created_at` y los aplica hasta agotar saldo nuevo.
2. **Vigencia:** 3 meses (`expires_at`). Si vence, queda en `EXPIRADA` y no se cobra.
3. **Reportería:** entran en archivo **COMI** con concepto "Proceso de Cobro Satisfactorio".

---

## 5. Componentes y responsabilidades

> **Patrón writer / publisher.** El procesamiento de novedades y la publicación de archivos están **separados a propósito**: los componentes de negocio (Processor, Adjustment_Service, motor transaccional, Recovery_Service) escriben **solo a la BD** (`transaction`, `file_record`, `pending_charge`). El `Output_File_Generator` es el **único componente** que descifra PAN en memoria, cifra PGP, escribe a NFS y entrega vía SFTP. Esta separación limita el blast radius PCI 3.4/4.2.1 y mantiene coherencia con las pestañas "Intercambio de Archivos", "Flujo Novedades No Monetarias" y "Realce" del unificado.

| Componente | Stack | Responsabilidad | Origen |
|---|---|---|---|
| `Prepaid_Monetary_Novelty_Processor` | Java 21 + Spring Batch | **Writer** — procesa archivos `MXXXddmmcc` chunk-oriented (1k-5k registros/chunk). Persiste resultados en `file_record`. **No escribe archivos de salida.** | `requerimientos-tecnicos-consolidados.md` |
| `Prepaid_Monetary_Novelty_Service` | Java 21 + Spring WebFlux | **Writer** — procesa requests API y orquesta `SP_APPLY_MONETARY` | Nuevo |
| `Pomelo_Webhook_Adapter` | Spring WebFlux | Recibe webhooks de Pomelo (ajustes, autorizaciones) y los enruta | Existente |
| `Adjustment_Service` | Java 21 + Spring | **Writer** — procesa ajustes por reclamaciones de Pomelo, persiste en `transaction` y `adjustment_inbox`. **No escribe archivos.** | Nuevo |
| `Limit_Counter_Service` | Java 21 + Redis 7 cluster | CHECK / INCREMENT / DECREMENT / RESET de regla `81` | ADR-017 |
| `Recovery_Service` | Java 21 + Spring | **Writer** — ejecuta `SP_RECOVER_CHARGES` al detectar abonos. Inserta `transaction` con `txn_type=RECOBRO`. | Nuevo |
| `GMF_Batch_Job` | — | **Eliminado en ADR-028.** El GMF batch ya NO es un job interno. Entra como novedad por archivo de la Entidad y se procesa con el `Prepaid_Monetary_Novelty_Processor`. El `Charge_Settlement_Service` aplica el cobro con `charge_type=GMF`. | ADR-028 |
| `Output_File_Generator` | Java 21 + Spring Batch | **Publisher único** de archivos de salida. Descifra PAN en memoria y escribe archivos **PLANOS** en NFS. **GAW cifra PGP** y entrega vía SFTP. Lee de `file_record`/`transaction`/`pending_charge` y produce TNM, TAV, AUSR, ABO, RAJU. | Existente |
| `File_Watcher_Batch` | Spring Batch + scheduler | Detecta archivos en NFS y dispara processor | Existente |
| `Audit_Service` | Java 21 + Spring | Audita servicios + PCI 10.2 | ADR-017 |
| `B2B_Mail_Service` | Cliente SMTP corporativo | Notifica fallas de procesamiento masivo a la Entidad | ADR-022, ADR-025 |
| `GoAnywhere MFT` | Producto Fortra | SFTP server + PGP encrypt/decrypt. **Cifra PGP los archivos de salida del OFG** antes de despacharlos por SFTP a la Entidad. | Existente |

### Stored Procedures (Oracle 19c)

| SP | Propósito | Parámetros clave |
|---|---|---|
| `SP_APPLY_MONETARY` | Aplica un Abono o Cargo de forma atómica: UPDATE saldo + INSERT transaction + (opcional) INSERT pending_charge | `card_id, account_id, type (ABONO/CARGO), amount, concept, file_id, line_no, out_response_code, out_balance_after` |
| `SP_RECOVER_CHARGES` | Itera `pending_charge` por card y los liquida hasta agotar saldo nuevo | `card_id, in_balance_increment, out_charges_recovered` |
| `SP_APPLY_ADJUSTMENT` | Aplica ajuste por reclamación (DEBITO o CREDITO). Permite ejecutar en `Bloqueo Temporal` | `card_id, type, amount, concept, original_txn_id` |
| ~~`SP_GMF_MONTHLY_BATCH`~~ | **Eliminado en ADR-028.** El GMF batch entra como novedad por archivo y reusa `SP_APPLY_MONETARY` / `SP_SETTLE_CHARGE`. | — |

> **Nota de implementación:** los SPs deben respetar la convención del proyecto (decisión arquitectónica clave 3): el débito atómico (monto + comisión + GMF) se hace en SP, no en código Java.

---

## 6. Estructura del archivo MXXXddmmcc

### 6.1 Convención de nombre

```
M XXX dd mm cc
│ │   │  │  └── Consecutivo (00-99) — incremental por AFG y día
│ │   │  └────── Mes (01-12)
│ │   └───────── Día (01-31)
│ └──────────── AFG (3 caracteres)
└────────────── Prefijo fijo "M" (Monetario)
```

Validaciones:
- Fecha del nombre: **±6 días calendario** respecto a la fecha actual.
- Único por par `(file_name, control_record)`.
- Persistencia: `file_processing` con `file_type='MONETARIA'`.
- **Depuración trimestral** del histórico de `file_processing` (configurable).

### 6.2 Longitud y estructura

| Sección | Longitud | Contenido |
|---|---|---|
| Registro detalle | **200 bytes** (nuevo modelo — antes era 128) | NIT AFG(15) + ID AFG(40) + User ID(40) + Card ID(40) + Tipo Novedad(1) + Monto(13) + Filler(51) |
| Registro de control (trailer) | 200 bytes | Fecha/hora disposicion(20) + Total registros(15) + Nombre archivo(30) + Filler(135) |

> **Referencia completa:** ver `docs/estructuras-archivos-manual-tecnico.md` seccion 1.2 para el detalle campo-por-campo.

### 6.3 Validaciones del trailer

| Validación | Acción si falla |
|---|---|
| `TRAILER_NIT` — el NIT del trailer no existe en BD AFG | Archivo NO PROCESADO. Se incluye causa en TAV con todos los registros. Aplica monetario y no monetario. |
| `ARCHIVO_FECHA` — fecha del nombre del archivo fuera de ±6 días | ARCHIVO_FECHA_INVALIDA. Alertamiento Interno. |
| `TRAILER_FECHA` — fecha del registro de control fuera de ±6 días | TRAILER_FECHA_INVALIDA. **Aplica solo a monetarias.** Alertamiento Interno. |
| `TRAILER_NUM_REG` — total de registros en archivo ≠ campo "Numero total de Registros Reportados" del trailer | NO PROCESADO. Alertamiento Interno. |
| `NIT_EN_NOVEDAD` — NIT del trailer ≠ NIT de cada registro de detalle | NO PROCESADO. Alertamiento Interno. |
| `LONGITUD_REGISTRO` — registro detalle ≠ 200 bytes | Registro rechazado, se reporta en TAV. |

---

## 7. Contratos de API

### 7.1 `POST /api/v1/monetary-novelty`

**Headers obligatorios:**
- `Authorization: Bearer <JWT>`
- `X-Idempotency-Key: <UUID>` — la Entidad debe enviar siempre uno
- `Content-Type: application/json`

**Request:**
```json
{
  "afg_id": "037",
  "nit": "900123456",
  "doc_type": "CC",
  "doc_number": "12345678",
  "type": "ABONO",                
  "amount": 250000.00,
  "concept": "Recarga nómina mayo",
  "entity_reference": "REF-2026-05-001234"
}
```

`type` ∈ `{ABONO, CARGO}`.

**Response 200 (procesada):**
```json
{
  "response_code": "00",
  "response_message": "Procesada",
  "transaction_id": 4503981,
  "balance_after": 750000.00,
  "applied_at": "2026-05-27T14:32:18Z"
}
```

**Response 422 (rechazo de negocio):**
```json
{
  "response_code": "61",
  "response_message": "Excede límite de novedades monetarias por monto (regla 81)",
  "rule": "81"
}
```

**Response 409 (cargo con saldo insuficiente — parcial aceptado):**
```json
{
  "response_code": "51",
  "response_message": "Saldo insuficiente, cargo parcial aplicado",
  "transaction_id": 4503982,
  "applied_amount": 50000.00,
  "pending_amount": 200000.00,
  "pending_charge_id": 12045,
  "expires_at": "2026-08-27T14:32:18Z"
}
```

### 7.2 `POST /webhooks/adjustment` (entrante desde Pomelo)

**Headers obligatorios:**
- `X-Pomelo-Signature: <HMAC-SHA256>` (clave compartida con Pomelo)
- `Content-Type: application/json`

**Request (extracto, alineado con Pomelo API Reference - Ajustes):**
```json
{
  "adjustment_id": "ADJ-9876543",
  "card_id_pomelo": "card_2K8PvX",
  "type": "CREDITO",
  "amount": 45000.00,
  "currency": "COP",
  "concept": "Disputa aprobada — devolución comercio",
  "original_txn_pomelo": "txn_9X7YzA",
  "created_at": "2026-05-27T03:15:00Z"
}
```

**Response 200:**
```json
{
  "credibanco_txn_id": 4504001,
  "status": "APPLIED",
  "balance_after": 295000.00
}
```

**Response 200 con rechazo (RAJU):**
```json
{
  "status": "REJECTED",
  "reason_code": "INSUFFICIENT_FUNDS",
  "reason": "Saldo insuficiente para débito"
}
```

Causas de rechazo: `INSUFFICIENT_FUNDS`, `CARD_BLOCKED`, `ACCOUNT_BLOCKED`, `CARD_NOT_FOUND`.

---

## 8. Reglas de negocio

### 8.1 Validaciones obligatorias por novedad

1. **AFG existe** y está asociado al NIT y a la Entidad correctos.
2. **User ID** (tipo + número de documento) tiene tarjeta en estado **Creada, Activa o Bloqueo Temporal** dentro del AFG.
3. **Estado tarjeta** ∈ estados válidos (no aplica si tarjeta `Deshabilitada`, `Expirada`, `Pendiente por Activar` para abonos directos del TH — validar Producto).
4. **Regla 81** Límite Novedades Monetarias por Monto — numérica, según periodicidad del AFG. `Limit_Counter_Service.CHECK(card_id, rule_81, amount)`.

### 8.2 Tratamiento de Cargo con saldo insuficiente

- Descontar **lo disponible** (saldo actual completo).
- Insertar registro en `pending_charge` con `pending_amount = solicitado - aplicado` y `expires_at = SYSTIMESTAMP + 90 días`.
- Reportar en archivo **AUSR** (batch 01:00 AM).
- El faltante se cobra al detectar siguiente abono (recobro).

### 8.3 Tratamiento de Abono con cobro pendiente

Al detectar un Abono nuevo:
1. Aplicar abono al saldo (incrementa).
2. Invocar `SP_RECOVER_CHARGES(card_id, increment_amount)`.
3. Iterar `pending_charge` ordenado por `created_at` ASC, `status='PENDIENTE'`, `expires_at > SYSTIMESTAMP`.
4. Liquidar uno o varios cargos pendientes hasta agotar el incremento. Cada liquidación genera `transaction` con `txn_type='RECOBRO'`.
5. Reportar recobros en **COMI** con concepto "Proceso de Cobro Satisfactorio".

### 8.4 Aplicación en línea de saldos

- La afectación de saldos se realiza **EN LÍNEA** (no diferida), tanto por archivo como por API.
- Para archivos batch, "en línea" significa por registro durante el procesamiento del chunk; el saldo final incluye toda novedad procesada hasta el commit del chunk.

### 8.5 Multibolsillo

- Cada AFG con multibolsillo define cuál bolsillo recibe el abono o cargo según el `concept` o un campo adicional `pocket_id` (validar Producto).
- La afectación de saldos es por bolsillo, no consolidada.
- La regla 81 aplica por tarjeta (no por bolsillo individual) salvo que el AFG indique lo contrario.

### 8.6 Tarjeta amparada

- Una tarjeta amparada (`card.parent_card_id IS NOT NULL`) puede compartir saldo con la principal o tener saldo independiente, según AFG.
- Si comparte saldo: la novedad afecta a la tarjeta principal pero el `transaction.card_id` queda con la amparada.
- Validar que la tarjeta principal esté en estado válido antes de aplicar.

### 8.7 Ajuste por reclamación — reglas específicas

- Se aplica **EN LÍNEA**, 24/7.
- Aplica **incluso si la tarjeta está en Bloqueo Temporal**.
- Si el ajuste es Débito y no hay saldo: **rechaza**, no genera `pending_charge` (regla diferente a Cargo administrativo).
- Causas de rechazo soportadas: `Saldo Insuficiente`, `Tarjeta Bloqueada` (Deshabilitada/Expirada), `Cuenta Bloqueada`, `Tarjeta No Existe`.
- Genera registro en `transaction` con `txn_type='AJUSTE_DEBITO'` o `'AJUSTE_CREDITO'` y `concept` = "Ajuste Débito" o "Ajuste Crédito".

### 8.8 Idempotencia

| Canal | Mecanismo |
|---|---|
| Archivo MXXX | UNIQUE `(file_name, control_record)` en `file_processing` |
| API | Header `X-Idempotency-Key` + cache Redis 24h |
| Webhook ajuste Pomelo | Campo `adjustment_id` único en tabla `adjustment_inbox` |

---

## 9. Modelo de datos

### 9.1 Tablas relevantes (existentes)

```sql
-- Tracking de archivos (alineado con db/01_schema_prepago.sql)
file_processing (
    file_id, file_name, file_type='MONETARIA', afg_id, entity_id,
    control_record, file_date, consecutive,
    total_records, processed_records, error_records,
    status, received_at, processed_at,
    UNIQUE (file_name, control_record)
)

-- Detalle por registro
file_record (
    record_id, file_id, line_number,
    record_type IN ('ABONO','CARGO','MODIFICACION'),
    card_id, response_code, response_message,
    raw_payload, processed_at
)

-- Movimientos (PARTICIONADA POR MES, ADR-005)
transaction (
    transaction_id, card_id, account_id, afg_id,
    txn_type IN ('ABONO','CARGO','AJUSTE_DEBITO','AJUSTE_CREDITO','RECOBRO','GMF',...),
    movement_type IN ('DEBITO','CREDITO'),
    concept, amount, commission_amount, gmf_amount, total_amount,
    auth_code, rrn, pomelo_txn_id, txn_date, ...
)

-- Cobros pendientes (recurrentes hasta 3 meses)
pending_charge (
    pending_id, card_id, account_id,
    charge_type IN ('COMISION','TECH_EXITOSA','CUOTA_MANEJO','GMF'),
    concept, original_amount, pending_amount,
    original_txn_id, status, attempts, expires_at
)
```

### 9.2 Extensiones propuestas (deltas)

```sql
-- Nuevo: extender pending_charge.charge_type para CARGO de novedad monetaria
ALTER TABLE pending_charge
  MODIFY CONSTRAINT chk_pc_type CHECK (
    charge_type IN ('COMISION','TECH_EXITOSA','CUOTA_MANEJO','GMF','CARGO_NOVEDAD')
  );

-- Nueva: registro de webhooks de ajuste para idempotencia
CREATE TABLE adjustment_inbox (
    inbox_id           NUMBER(12) GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    adjustment_id      VARCHAR2(50) NOT NULL UNIQUE,         -- viene de Pomelo
    received_at        TIMESTAMP DEFAULT SYSTIMESTAMP NOT NULL,
    payload_hash       VARCHAR2(64) NOT NULL,
    status             VARCHAR2(15) DEFAULT 'RECIBIDO' NOT NULL,
    transaction_id     NUMBER(15),
    response_code      VARCHAR2(3),
    response_message   VARCHAR2(200),
    CONSTRAINT chk_ai_status CHECK (status IN ('RECIBIDO','APLICADO','RECHAZADO'))
);

-- Nuevo índice sobre transaction para reportería ABO
CREATE INDEX idx_txn_abo_report
  ON transaction(afg_id, txn_date, txn_type) LOCAL;
```

### 9.3 Vistas para reportería

```sql
CREATE OR REPLACE VIEW vw_monetary_novelty_monthly AS
SELECT
    afg_id,
    TRUNC(txn_date, 'MM') AS month,
    COUNT(*) AS qty_novelties,
    SUM(amount) AS total_amount,
    AVG(amount) AS avg_amount
FROM transaction
WHERE txn_type IN ('ABONO','CARGO','AJUSTE_DEBITO','AJUSTE_CREDITO')
GROUP BY afg_id, TRUNC(txn_date, 'MM');
```

---

## 10. Archivos de salida derivados

> **Publisher único:** todos los archivos de salida los produce el `Output_File_Generator`. Los componentes de procesamiento (Processor, Service, Adjustment_Service, Recovery_Service) **no escriben archivos**, solo persisten en BD; el OFG lee de `file_record`/`transaction`/`pending_charge`, descifra PAN en memoria y deposita el archivo **plano** en NFS. **GoAnywhere MFT cifra PGP** y entrega por SFTP a la Entidad. Esto centraliza el descifrado de PAN, libera al pod del costo de PGP y aprovecha el motor PGP nativo de GAW.

| Archivo | Tipo | Periodicidad | Hora | Cifrado a la Entidad | Contenido |
|---|---|---|---|---|---|
| **TNM** xxxMMDD | En línea | Incremental | Por evento | PGP por GAW | Resumen de novedades monetarias procesadas (Abono y Débito), consolidado del día |
| **TAV** xxxMMDD | En línea | Incremental | Por evento | PGP por GAW | Rechazos de novedades monetarias con causa y campo de error |
| **AUSR** dd.MM | Batch | Diaria | 01:00 AM | PGP por GAW | Cargos parciales (valor solicitado vs aplicado vs pendiente) |
| **FIN** dd.MM | Batch | Diaria | 01:00 AM | PGP por GAW | Reporte Financieras: abonos y débitos efectivos del día (antes ABO). **NO lleva PAN — usa Card ID** |
| **RAJU** dd.MM | Batch | Diaria | 01:00 AM | PGP por GAW | Rechazos de ajustes Pomelo con campo de error |
| **AUM** dd.MM | Batch | Diaria | 00:30 AM | PGP por GAW | Movimientos autorizador. **NO lleva PAN en nuevo modelo — usa Card ID (40 chars)** |
| **COMI** dd.MM | Batch | Diaria | 01:15 AM | PGP por GAW | Comisiones: cobro exitoso, no cobrado, cuota manejo, 4x1000, tech exitosa, recobro. **NO lleva PAN — usa Card ID** |
| **CSAT** dd.MM | Batch | Diaria | 01:15 AM | PGP por GAW | Cobros satisfactorios (subset de COMI) |
| **CINS** dd.MM | Batch | Diaria | 01:15 AM | PGP por GAW | Cobros insatisfactorios (subset de COMI, 3 meses) |
| **SALD** dd.MM | Batch | Diaria | 00:00 AM | PGP por GAW | Saldos al cierre (Activa, Embozada, Bloqueo Temporal). **NO lleva PAN — usa Card ID** |
| **TEX** dd.mm | Batch | Diaria | TBD | PGP por GAW | Indicador exención GMF (S/N) por tarjeta. **Nuevo archivo** |
| **Novedad GMF** TBD | Batch | Post-procesamiento | TBD | PGP por GAW | Respuesta procesamiento GMF (codigo respuesta, valor cobrado, RRN). **Nuevo archivo** |

> **Estructura detallada:** ver `docs/estructuras-archivos-manual-tecnico.md` para campos campo-por-campo.
> **Ruta entrega:** GoAnywhere → SFTP de la Entidad.
> **Cambio critico vs modelo anterior:** AUM, FIN y COMI ya NO llevan PAN (usan Card ID). Solo TAR lleva PAN. Ver seccion 4 del documento de estructuras.
> **Eliminados del Manual:** TRX (no hay reexpedición), CANJ (no hay canje), EXT (no hay extractos), MXXXXXX (movimiento autorizador), AUMCT (consulta de costo).

---

## 11. Errores, rechazos y reintentos

### 11.1 Catálogo de códigos de respuesta (extracto)

| Código | Significado | Causa típica | Reportado en |
|---|---|---|---|
| `00` | Aprobada | Procesada exitosamente | TNM / ABO |
| `14` | Tarjeta inválida | Card no existe / no pertenece al AFG | TAV |
| `51` | Fondos insuficientes | Saldo no alcanza para cargo (parcial aplicado) | TAV / AUSR |
| `54` | Tarjeta expirada | Fecha de expiración pasada | TAV |
| `57` | Transacción inválida | Tipo de novedad no soportado | TAV |
| `61` | Excede límites | Regla 81 violada | TAV |
| `62` | Tarjeta restringida | Tarjeta Deshabilitada | TAV |
| `63` | Violación de seguridad | NIT no coincide | TAV |
| `78` | Cuenta inválida | Tarjeta encontrada pero cuenta inválida | TAV |

### 11.2 Errores estructurales del archivo

Bloquean el archivo completo. Causas:

| Código interno | Mensaje | Acción |
|---|---|---|
| `TRAILER_NIT` | NOMBRE DE ARCHIVO `[xxx]` NO PROCESADO - EL NIT DEL TRAILER NO EXISTE en el SERVICIO | Archivo NO PROCESADO. Causa incluida en TAV con todos los registros. Alertamiento Interno. |
| `ARCHIVO_FECHA` | ERROR al procesar el archivo `'xxx'`: ARCHIVO_FECHA_INVALIDA | Archivo NO PROCESADO. Alertamiento Interno. |
| `TRAILER_FECHA` | ERROR al procesar el archivo `'xxx'`: TRAILER_FECHA_INVALIDA | Aplica solo a monetarias. Alertamiento Interno. |
| `TRAILER_NUM_REG` | NOMBRE DE ARCHIVO `[xxx]` NO PROCESADO: EL NUMERO DE REGISTROS NO COINCIDE EN EL TRAILER | Archivo NO PROCESADO. Alertamiento Interno. |
| `NIT_EN_NOVEDAD` | NOMBRE DE ARCHIVO `[xxx]` NO PROCESADO: NIT EN NOVEDAD DIFERENTE AL DEL TRAILER | Archivo NO PROCESADO. Alertamiento Interno. |

### 11.3 Reintentos del archivo (Fase 2)

> **Implementación diferida a Fase 2.**

- **Falla al iniciar procesamiento:** 1 reintento inmediato, luego 3 reintentos cada 30 minutos. Aplica hasta 23:59:59 del mismo día.
- **Afectación parcial durante procesamiento:** misma política — 1 + 3 cada 30 min.
- Aplica a archivos monetarios, no monetarios y SALD.
- Si supera reintentos → **alertamiento Dynatrace** + correo a operación.

### 11.4 Notificaciones a Entidades por fallas masivas

Si un archivo falla por estructura, el `Prepaid_Monetary_Novelty_Processor` debe disparar notificación a la Entidad vía `B2B_Mail_Service` (ADR-022, ADR-025) con:
- Nombre del archivo
- Causa estructural
- Hora de detección
- Indicación de reenvío esperado

---

## 12. Reportería operativa

### 12.1 Reporte mensual consolidado por AFG / por BIN

Generación: archivo **xlsx** mensual en `Output_File_Generator`.

Campos (extracto del docx fuente, sección Reportería):

| Campo | Cálculo |
|---|---|
| ENTIDAD FINANCIERA / NOMBRE AFG | Static |
| CÓDIGO DE COMPENSACIÓN | Static |
| MES / AÑO | Periodo |
| BIN / FRANQUICIA | Static |
| TOTAL TARJETAS ACTIVAS POR BIN/AFG | `SELECT COUNT(*) FROM card WHERE status='Activa' ...` |
| **CANTIDAD DE NOVEDADES MONETARIAS RECIBIDOS MES** | `SELECT COUNT(*) FROM transaction WHERE txn_type IN ('ABONO','CARGO','AJUSTE_DEBITO','AJUSTE_CREDITO') AND TRUNC(txn_date,'MM')=:p_month` |
| **VALOR TOTAL NOVEDADES MONETARIAS MES** | `SELECT SUM(amount) ...` |
| **VALOR TOTAL PROMEDIO NOVEDADES MONETARIAS MES** | `SELECT AVG(amount) ...` |
| SALDO TARJETAS CIERRE MES | Snapshot SALD del último día |
| CANTIDAD/VALOR TRANSACCIONES POS / ATM / OTROS CANALES | Joins con transaction |

### 12.2 Indicadores en tiempo real (Grafana)

Métricas Prometheus exportadas por `Prepaid_Monetary_Novelty_Service`:

```
prepaid_monetary_novelty_total{type="ABONO|CARGO|AJUSTE",result="OK|REJECTED"}
prepaid_monetary_novelty_amount_sum{type="..."}
prepaid_monetary_novelty_processing_seconds_bucket{type="..."}
prepaid_pending_charge_total{status="PENDIENTE|COBRADA|EXPIRADA"}
prepaid_file_mxxx_processed_total{afg="xxx",status="OK|ERROR"}
```

### 12.3 Alertamientos (Dynatrace)

Heredados del requerimiento de monitoreo:

- Fallas en servicios y memoria de disco vía OpenShift Dynatrace (correo).
- **Cobro de comisiones (Cobro 4×1000)** — alerta si el lote diario falla.
- Procesamiento de archivos monetarios con errores estructurales.
- Validación de longitud (Monetario: 200 bytes, No Monetario: 600 bytes — nuevo modelo).
- Pods que interactúan con terceros (Pomelo).
- Pendiente por incluir alertamientos de GAW (Juan Diego Guchuvo).

---

## 13. Seguridad y cumplimiento PCI

### 13.1 Cifrado de archivos

- **Entrada:** Entidad envía `MXXXddmmcc` cifrado **PGP**. GoAnywhere descifra y deposita plano en NFS de entrada.
- **Salida:** `Output_File_Generator` escribe **archivo plano** en NFS de salida. **GoAnywhere lo cifra PGP** y entrega por SFTP a la Entidad.
- **En tránsito:** SFTP/TLS 1.3 (Req 4.2.1).
- **En reposo:**
  - NFS de entrada (post-descifrado) y NFS de salida (pre-cifrado) deben estar en **volumen cifrado at-rest** (LUKS / dm-crypt o equivalente del storage), en zona CDE.
  - **ACL estricta**: solo el SA del OFG escribe en NFS de salida; solo el SA de GAW lee y borra.
  - **TTL corto**: GAW elimina el archivo plano inmediatamente después de cifrar y despachar.
  - **Network policy** OpenShift: solo namespace OFG (escritura) y host de GAW (lectura) montan el PVC.
  - Tras `processed`, depuración trimestral del histórico.

### 13.2 PAN

- El archivo MXXX **no transporta PAN** (transporta NIT + AFG + tipo/número de documento del TH).
- En los archivos de salida con PAN (ABO, AUM, Presentaciones, Transacciones), `Output_File_Generator` descifra el PAN en memoria justo antes de escribir; no se almacena en claro (envelope encryption — ADR-018).

### 13.3 Aplicables PCI DSS v4.0

| Requisito | Aplicación en este flujo |
|---|---|
| Req 3.4 | PAN no legible en BD (envelope encryption) |
| Req 4.2.1 | SFTP/TLS 1.3 + mTLS Pomelo |
| Req 7.2 | Acceso mínimo: roles `MONETARY_PROCESSOR`, `RECONCILER` |
| Req 8.3 | MFA en accesos administrativos |
| Req 10.2 | Auditoría completa de cada novedad procesada |
| Req 10.7 | Logs no repudiables, retención 1 año hot, 7 años archive |

### 13.4 Idempotencia y prevención de duplicados

- UNIQUE `(file_name, control_record)` en `file_processing`.
- `X-Idempotency-Key` en API.
- `adjustment_id` UNIQUE en `adjustment_inbox`.
- En reverso/anulación de novedad (Fase 2): `parent_txn_id` + flag `reversed`.

### 13.5 Validación HMAC en webhook Pomelo

- Header `X-Pomelo-Signature: <HMAC-SHA256(body, shared_secret)>`.
- Si falla → 401 + alerta SIEM.
- Shared secret en OpenShift Secret, rotación trimestral.

---

## 14. Auditoría

### 14.1 Auditoría de servicios (campos)

Cada invocación del `Prepaid_Monetary_Novelty_Service` y `Adjustment_Service` debe registrar en `audit_log`:

- `ID_AUDIT`, `SERVICE_NAME`, `METHOD_NAME`, `ENDPOINT`, `MESSAGE_ID`
- `REQUEST_PAYLOAD` (con PAN enmascarado si aplicara)
- `RESPONSE_PAYLOAD`, `STATUS_CODE`, `STATUS_MESSAGE`
- `CREATED_AT`, `RESPONSE_TIME_MS`
- `TECHNICAL_USER`, `SOURCE_SYSTEM`
- `ID_TARJETA`, `ID_USUARIO`, `ID_PROCESO`

### 14.2 PCI 10.2

Por cada movimiento aplicado:

- Identificación de usuario (técnico de la Entidad o Pomelo)
- Tipo de evento (ABONO, CARGO, AJUSTE_DEBITO, AJUSTE_CREDITO, RECOBRO)
- Fecha y hora
- Éxito / fallo
- Origen (IP, archivo, webhook)
- Identidad de los datos afectados (`card_id`, `account_id`)
- Componentes del sistema (servicio, SP, BD)

### 14.3 Eventos especiales que requieren acta

- Aplicación de ajuste por reclamación con monto > umbral configurado (ej. COP $5.000.000).
- Recobros que liquiden múltiples cargos vencidos (> 3 cargos en una sola operación).
- Modificación de la regla 81 a nivel AFG.

---

## 15. Performance y SLAs

| Indicador | Objetivo |
|---|---|
| Latencia API `POST /api/v1/monetary-novelty` | p95 < 500 ms |
| Latencia webhook `/webhooks/adjustment` | p95 < 800 ms |
| Throughput batch processor | 1.000 registros/segundo (chunk-oriented) |
| Tiempo procesamiento archivo MXXX típico (5.000 reg.) | < 6 minutos |
| Disponibilidad endpoint webhook Pomelo | 99,9 % (24/7) |
| Retención `file_processing` | Trimestral en hot, 7 años archive |
| Reintentos automáticos archivo (Fase 2) | 1 inmediato + 3 cada 30 min, hasta 23:59 |
| Recobro tras abono | < 2 segundos desde commit del abono |

---

## 16. Pendientes

| # | Pendiente | Origen | Responsable |
|---|---|---|---|
| 1 | Tratamiento de devolución del 4×1000 cuando una transacción se reversa | docx §11.5 | Desarrollo / Equipo Técnico / Entidad |
| 2 | Cómo identificar transacciones tipo 22 y 26 en archivo ABO | docx §archivos-salida | Desarrollo / Equipo Técnico |
| 3 | Validar estructura del archivo TAV — confirmar campos requeridos | docx Ajustes | Producto + Desarrollo |
| 4 | Reglas multibolsillo para regla 81 — ¿por bolsillo o por tarjeta? | docx §novedades monetarias | Producto |
| 5 | Reglas tarjeta amparada — comportamiento de saldo compartido vs independiente | docx §novedades monetarias | Producto |
| 6 | Detalle de la revisión de GAW (validaciones que ejecuta hoy) | docx (varias secciones) | GAW (Juan Diego Guchuvo) |
| 7 | Confirmar campos obligatorios en respuesta JSON de la API | docx §646-647 | Producto + Pomelo |
| 8 | Estructura del archivo de Indicadores Base (`indicadores_baseddmmaaaa.xlsx`) | docx §823 | Producto |
| 9 | Definir `pocket_id` en API y archivo MXXX para multibolsillo | Análisis técnico | Producto + Arquitectura |
| 10 | Alertamientos GAW a incluir en monitoreo | docx §1100 | GAW |
| 11 | **Nombre canónico del archivo GMF** (¿GXXXddmmcc?) | ADR-028 | Producto + Entidad |
| 12 | **Estructura del registro detalle del archivo GMF** (campos, longitud, separadores) | ADR-028 | Producto + Entidad + Desarrollo |
| 13 | **Trailer del archivo GMF** (NIT, fecha, total registros, valor sumado) | ADR-028 | Producto + Entidad |
| 14 | **Periodicidad de envío** del archivo GMF (mensual vs otro corte) | ADR-028 | Producto + Entidad |

---

## 17. Referencias cruzadas

### 17.1 Documentos

- `Prepago/Requerimientos Técnicos (12).docx` — fuente original.
- `Prepago/docs/requerimientos-flujo-transaccion.md` — §10 (regla 81) y §11 (post-autorización).
- `Prepago/docs/requerimientos-tecnicos-consolidados.md` — componentes y archivos de salida.
- `Prepago/docs/requerimientos-ciclo-vida-tarjeta-y-portal-th.md` — §1, §4, §5 (intercambio de archivos).
- `Prepago/docs/decisiones-arquitectura.md` — ADR-005, ADR-017, ADR-018, ADR-020, ADR-022.
- `Prepago/docs/modelo-entidad-relacion.md` — modelo de datos.
- `Prepago/db/01_schema_prepago.sql` — DDL.
- `Prepago/db/02_stored_procedures.sql` — SPs.

### 17.2 Diagramas

- `Prepago/PrepagoUnificadoArquitectura_V_1_5.drawio` — pestaña "Intercambio de Archivos" (vista C4 L2 de archivos NXXX/MXXX) y pestaña "Flujo de transacciones" (regla 81).
- `Prepago/Intercambio_Archivos_SFTP_PGP.drawio` — flujo de SFTP+PGP.
- `Prepago/MER_Prepago_V_1_0.drawio` — modelo entidad-relación.

### 17.3 ADRs aplicables

- **ADR-005** — Particionamiento mensual de `transaction` (relevante para reportería mensual).
- **ADR-017** — `Limit_Counter_Service` con CHECK / INCREMENT / DECREMENT / RESET (regla 81).
- **ADR-018** — Envelope encryption del PAN (cifrado de archivos de salida con PAN).
- **ADR-020** — Pestaña "Flujo Novedades No Monetarias" (referencia simétrica de procesamiento de archivos).
- **ADR-022** — `B2B_Mail_Service` para notificación a entidades.
- **ADR-025** — Verificación dual identidad + correo en onboarding (no aplica directamente a este flujo, mencionado por trazabilidad de procesos similares).

---

> **Nota de versionado:** este documento se mantiene como única fuente para Novedades Monetarias. Las modificaciones se registran como ADRs nuevos en `decisiones-arquitectura.md`, no como nuevas versiones del documento. Para cambios de requerimiento que rompan la línea base aquí descrita, crear un ADR explícito.
