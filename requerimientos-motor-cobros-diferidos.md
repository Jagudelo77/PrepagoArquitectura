# Requerimientos — Motor de Cobros Diferidos

Documento de requerimientos del Motor de Cobros Diferidos de la plataforma prepago Credibanco. Cubre el patron unificado de cobros tolerantes a saldo insuficiente: cuota de manejo, comisiones no efectivas, transacciones tecnicamente exitosas, GMF batch, recobro automatico al detectar abono, y vencimiento a 90 dias.

**Fuentes consolidadas:**
- `Prepago/Requerimientos Tecnicos (12).docx` — secciones Cobro de Comisiones, Cobro de Transacciones Tecnicamente Exitosas, Cobro de Impuestos, GMF en Linea.
- `Prepago/docs/requerimientos-flujo-transaccion.md` — seccion 11.3, 11.4, 11.5, 11.6.
- `Prepago/docs/requerimientos-novedades-monetarias.md` — seccion 4.6 (Recobro).
- `Prepago/docs/decisiones-arquitectura.md` — ADR-012, ADR-013, ADR-014, ADR-027, ADR-028, ADR-029.
- `Prepago/PrepagoUnificadoArquitectura_V_1_11.drawio` — pestana "Motor de Cobros Diferidos".
- `Prepago/db/01_schema_prepago.sql` — tabla `pending_charge`, tabla `transaction`.

> Fase: **Fase 1 — Linea Base (MVP)**, Etapa 5 (Procesos Batch y Archivos de Salida).
> Ultima actualizacion: 01/06/2026.
> Estado: **Vigente**.

---

## Tabla de contenido

1. [Resumen y alcance](#1-resumen-y-alcance)
2. [Glosario](#2-glosario)
3. [Patron unificado](#3-patron-unificado)
4. [Fuentes de cobro](#4-fuentes-de-cobro)
5. [Charge Settlement Service — nucleo comun](#5-charge-settlement-service--nucleo-comun)
6. [Recovery Service — drena pending charge](#6-recovery-service--drena-pending-charge)
7. [Modelo de datos](#7-modelo-de-datos)
8. [Reglas de negocio](#8-reglas-de-negocio)
9. [Archivos de salida](#9-archivos-de-salida)
10. [Seguridad y PCI](#10-seguridad-y-pci)
11. [Performance y SLAs](#11-performance-y-slas)
12. [Pendientes](#12-pendientes)
13. [Referencias cruzadas](#13-referencias-cruzadas)

---

## 1. Resumen y alcance

### 1.1 Objetivo

Aplicar de forma confiable los cobros recurrentes y diferidos sobre tarjetas prepago, tolerando saldo insuficiente en el momento del cobro. Garantizar que todo cobro no efectivo se liquide automaticamente cuando entre dinero a la tarjeta, dentro de una ventana de 90 dias. Reportar cobros satisfactorios (CSAT), insatisfactorios (CINS) y consolidados (COMI) a las Entidades emisoras.

### 1.2 Alcance funcional

**Incluye:**
- Cobro periodico de **cuota de manejo** (mensual por AFG).
- Cobro de **comisiones** no efectivas derivadas de transacciones POS/ATM.
- Cobro de **transacciones tecnicamente exitosas** (denegaciones codigos 51, 55, 57, 61, 75).
- Cobro de **GMF batch** cuando entra como novedad de la Entidad por archivo y el saldo no alcanza.
- **Recovery automatico** al detectar Abono, Ajuste Credito Pomelo o Reverso con devolucion de saldo.
- **Recovery batch nocturno** (catch-all) para pendientes no liquidados por evento.
- **Vencimiento a 90 dias** con transicion a estado EXPIRADA y reporte en CINS.
- Generacion de archivos **COMI, CSAT, CINS** (batch 01:15 AM diario).

**Excluye:**
- GMF en linea (se cobra dentro del motor transaccional, no pasa por el Motor de Cobros Diferidos salvo que falle). Ver `requerimientos-flujo-transaccion.md` seccion 11.6.
- Novedades monetarias (Abonos/Cargos por archivo o API). Ver `requerimientos-novedades-monetarias.md`.
- Ajustes por reclamaciones Pomelo. Ver `requerimientos-novedades-monetarias.md` seccion 4.3.
- Reconciliacion End-of-Day. Pestana propia.

### 1.3 Resultado esperado

Al final de cada ciclo:
- Cada cobro efectivo genera `transaction(txn_type = CUOTA_MANEJO | COMISION | TECH_EXITOSA | GMF | RECOBRO)`.
- Cada cobro no efectivo genera `pending_charge(status=PENDIENTE, expires_at=now+90d)`.
- Al entrar dinero, `pending_charge` se drena en FIFO generando `transaction(RECOBRO)`.
- A los 90 dias sin liquidar, `pending_charge` pasa a EXPIRADA y se reporta en CINS.
- Archivos COMI/CSAT/CINS se entregan diariamente a las Entidades via GoAnywhere SFTP.

---

## 2. Glosario

| Termino | Definicion |
|---|---|
| Cobro diferido | Cualquier debito que no se pudo aplicar inline porque el saldo era insuficiente. |
| pending_charge | Registro en BD que representa un cobro no efectivo en espera de saldo. |
| Cuota de manejo | Tarifa mensual por tenencia de tarjeta, configurable por AFG. |
| Comision no efectiva | Comision por transaccion POS/ATM que no se pudo cobrar inline. |
| Tecnicamente exitosa | Cobro por transaccion denegada donde la infraestructura funciono (codigos 51/55/57/61/75). |
| Recovery | Proceso que liquida pending_charge cuando aparece saldo. |
| FIFO | First In First Out — se cobran primero los cargos mas viejos. |
| COMI | Archivo de comisiones cobradas y no cobradas del dia. |
| CSAT | Archivo subset: solo comisiones satisfactorias (cobradas). |
| CINS | Archivo subset: comisiones insatisfactorias (expiradas, consolidado 3 meses). |
| Charge_Settlement_Service | Servicio nucleo que aplica el cobro o crea pending_charge. |
| Recovery_Service | Servicio que drena pending_charge al detectar saldo nuevo. |
| SP_SETTLE_CHARGE | Stored Procedure Oracle que ejecuta el debito atomico o inserta pending_charge. |
| SP_RECOVER_CHARGES | Stored Procedure Oracle que itera pending_charge en FIFO y liquida. |

---

## 3. Patron unificado

El Motor de Cobros Diferidos implementa un unico patron para todos los tipos de cobro:

```
[Fuente de cobro]
     |
     v
Charge_Settlement_Service.settle(card_id, charge_type, amount, concept)
     |
     v
SP_SETTLE_CHARGE (Oracle, atomico):
     |
     +-- Saldo >= amount?
     |       SI --> UPDATE balance, INSERT transaction --> COBRADO
     |       NO --> INSERT pending_charge(PENDIENTE, expires_at=now+90d) --> PENDIENTE
     |
     v
Redis: invalidar cache balance
     |
     v
Audit_Service (async, PCI 10.2)
```

Cuando entra dinero a la tarjeta (Abono, Ajuste Credito, Reverso):

```
[Abono acreditado]
     |
     v
Recovery_Service.recoverByEvent(card_id)
     |
     v
SP_RECOVER_CHARGES (Oracle):
     |
     +-- CURSOR pending_charge ORDER BY created_at ASC (FIFO)
     |       Por cada registro: cobra total o parcial hasta agotar saldo
     |       INSERT transaction(RECOBRO) por cada liquidacion
     |
     v
Audit_Service
```

---

## 4. Fuentes de cobro

### 4.1 Cuota de Manejo (Maintenance_Fee_Scheduler)

| Atributo | Valor |
|---|---|
| Charge_type | CUOTA_MANEJO |
| Trigger | Cron `0 2 1 * *` (1er dia del mes, 02:00 AM) |
| Tarjetas elegibles | Activa, Bloqueo Temporal, Bloqueo PIN Errado |
| Configuracion | Por AFG: monto base + IVA + periodicidad |
| Idempotencia | (CUOTA_MANEJO, card_id, YYYYMM) — no cobra dos veces el mismo mes |

**Criterios de aceptacion:**
- Solo aplica para tarjetas en estado Activa, Bloqueada Temporal y Bloqueada PIN Errado.
- El monto se lee de `fee_config` por AFG (monto base + IVA si aplica).
- Si el cobro es no efectivo, pasa a pending_charge (cobro recurrente hasta 90 dias).
- Todos los cobros realizados se reflejan en los movimientos del TH (Monto, Concepto, Fecha y hora).
- Reflejados en archivo de salida COMI con el detalle.
- Cobros recurrentes que sean cobrados se incluyen en COMI con concepto "Proceso de Cobro Satisfactorio".
- Auditoria de fecha y detalle de procesamiento.

### 4.2 Comisiones no efectivas (Commission_Charge_Engine)

| Atributo | Valor |
|---|---|
| Charge_type | COMISION |
| Trigger | Inline en motor transaccional tras cada autorizacion |
| Tarjetas elegibles | Activa, Bloqueo Temporal, Bloqueo PIN Errado |
| Tipos | Retiro propio, retiro otra red, retiro internacional, consulta saldo propio/otra red/intl |
| Idempotencia | (COMISION, card_id, original_txn_id) |

**Criterios de aceptacion:**
- Solo aplica para tarjetas en estado Activa, Bloqueada Temporal y Bloqueada PIN Errado.
- Si el cobro es no efectivo, pasa a pending_charge.
- Todos los cobros se reflejan en movimientos del TH (Monto, Concepto, Fecha y hora).
- Se incluyen en archivo de salida COMI con el detalle de la comision cobrada (retiro o consulta saldo cajero propio, otra red, internacional).
- Cobros recurrentes que sean cobrados se incluyen en COMI con concepto "Proceso de Cobro Satisfactorio".
- Auditoria de fecha y detalle de procesamiento.

### 4.3 Transacciones Tecnicamente Exitosas (Tech_Exitosa_Charge_Engine)

| Atributo | Valor |
|---|---|
| Charge_type | TECH_EXITOSA |
| Trigger | Inline en motor transaccional tras denegacion con codigos 51, 55, 57, 61, 75 |
| Tarjetas elegibles | Activa, Bloqueo Temporal, Bloqueo PIN Errado |
| Configuracion | Por AFG: habilitado SI/NO, valor fijo o segun tabla comisiones |
| Idempotencia | (TECH_EXITOSA, card_id, original_txn_id) |

**Criterios de aceptacion:**
- Mismas tarjetas elegibles que comisiones.
- Si el cobro es no efectivo, pasa a pending_charge.
- Se refleja en movimientos y en COMI con el detalle de la transaccion cobrada.
- Cobros recurrentes que sean cobrados se incluyen en COMI con concepto "Proceso de Cobro Satisfactorio".
- Auditoria de fecha y detalle.

### 4.4 GMF por archivo de la Entidad (indirecto)

| Atributo | Valor |
|---|---|
| Charge_type | GMF |
| Trigger | Archivo de la Entidad via SFTP (nombre y estructura TBD, ADR-028) |
| Procesado por | Prepaid_Monetary_Novelty_Processor (pestana "Flujo de novedades monetarias") |
| Ruta al motor | Cuando saldo insuficiente, invoca Charge_Settlement_Service.settle(card_id, GMF, ...) |
| Idempotencia | (GMF, card_id, file_id + line_number) |

**Nota:** la Entidad calcula el GMF segun Ley 2277/2022 art. 881-1 ET y envia la novedad. Credibanco aplica; no calcula UVT. Si el saldo no alcanza, entra al mismo flujo de pending_charge con vigencia 90 dias. Ver `requerimientos-novedades-monetarias.md` seccion 4.5 y ADR-028.

---

## 5. Charge Settlement Service — nucleo comun

### 5.1 Contrato

```java
public record ChargeResult(
    ResultType result,      // COBRADO | PARCIAL | PENDIENTE | RECHAZADO | ALREADY_CHARGED
    BigDecimal balanceAfter,
    Long pendingChargeId,   // null si COBRADO
    Long transactionId      // null si PENDIENTE puro
) {}

ChargeResult settle(Long cardId, ChargeType chargeType,
                    BigDecimal amount, String concept,
                    Long originalTxnId, String periodKey);
```

### 5.2 Pipeline detallado

| Paso | Responsable | Descripcion |
|---|---|---|
| 1. Idempotencia | Java + Redis | Verifica (chargeType, cardId, periodKey) no exista ya. Si existe, retorna ALREADY_CHARGED. Cache Redis TTL 30 dias. |
| 2. Validar estado | Java + Oracle | SELECT card.status. Si NOT IN (ACTIVA, BLOQUEO_TEMPORAL, BLOQUEO_PIN_ERRADO) retorna RECHAZADO. |
| 3. Bloqueo + evaluacion | SP_SETTLE_CHARGE (Oracle) | SELECT balance FOR UPDATE. Evalua saldo vs amount. |
| 4a. Cobro total | SP_SETTLE_CHARGE | UPDATE balance = balance - amount. INSERT transaction. Retorna COBRADO. |
| 4b. Cobro parcial | SP_SETTLE_CHARGE | UPDATE balance = 0. INSERT transaction(parcial). INSERT pending_charge(restante). Retorna PARCIAL. |
| 4c. Pendiente puro | SP_SETTLE_CHARGE | INSERT pending_charge(amount completo). Retorna PENDIENTE. |
| 5. Invalidar cache | Java + Redis | Redis.delete("balance:" + cardId). |
| 6. Audit | Java + Kafka | Audit_Service async (PCI 10.2). |
| 7. Retorno | Java | ChargeResult al caller. |

### 5.3 SP_SETTLE_CHARGE (Oracle)

```sql
PROCEDURE SP_SETTLE_CHARGE (
    p_card_id        IN  NUMBER,
    p_account_id     IN  NUMBER,
    p_charge_type    IN  VARCHAR2,
    p_concept        IN  VARCHAR2,
    p_amount         IN  NUMBER,
    p_original_txn_id IN NUMBER DEFAULT NULL,
    p_period_key     IN  VARCHAR2,
    p_result         OUT VARCHAR2,
    p_balance_after  OUT NUMBER,
    p_pending_id     OUT NUMBER,
    p_txn_id         OUT NUMBER
)
```

Atomicidad: todo dentro de un bloque BEGIN...EXCEPTION...END con ROLLBACK en caso de error. El FOR UPDATE sobre account serializa cobros concurrentes sobre la misma tarjeta.

---

## 6. Recovery Service — drena pending charge

### 6.1 Disparadores

| # | Disparador | Origen | Condicion |
|---|---|---|---|
| 1 | Abono acreditado | Novelty_Orchestrator (pestana monetaria) | Tras SP_APPLY_MONETARY con type=ABONO |
| 2 | Ajuste Credito Pomelo | Adjustment_Service (pestana monetaria) | Tras SP_APPLY_ADJUSTMENT con type=CREDITO |
| 3 | Reverso con devolucion | Motor transaccional (pestana transacciones) | Tras SP_REVERSE_TXN |
| 4 | Batch nocturno catch-all | Recovery_Retry_Scheduler | Cron 0 3 * * * (03:00 AM diario) |

**Importante:** el Charge_Settlement_Service NO dispara Recovery. Son flujos inversos (settlement = debito; recovery = credito sobre pendientes).

### 6.2 SP_RECOVER_CHARGES

```sql
PROCEDURE SP_RECOVER_CHARGES (
    p_card_id              IN  NUMBER,
    p_available_balance    IN  NUMBER DEFAULT NULL,
    p_charges_recovered    OUT NUMBER,
    p_amount_recovered     OUT NUMBER,
    p_charges_remaining    OUT NUMBER
)
```

**Logica:**
1. Si p_available_balance es NULL, SELECT balance FROM account FOR UPDATE.
2. CURSOR sobre pending_charge: card_id=?, status=PENDIENTE, expires_at > SYSTIMESTAMP, ORDER BY created_at ASC.
3. Por cada registro:
   - Si saldo >= pending_amount: cobro total, UPDATE status=COBRADA, INSERT transaction(RECOBRO).
   - Si 0 < saldo < pending_amount: cobro parcial, UPDATE pending_amount, INSERT transaction(RECOBRO parcial).
   - Si saldo = 0: EXIT.
4. Al final: UPDATE pending_charge SET status=EXPIRADA WHERE expires_at <= SYSTIMESTAMP AND status=PENDIENTE.

### 6.3 Recovery_Retry_Scheduler (batch nocturno)

| Atributo | Valor |
|---|---|
| Cron | 0 3 * * * (03:00 AM diario) |
| Logica | SELECT DISTINCT card_id FROM pending_charge WHERE status=PENDIENTE AND expires_at > SYSTIMESTAMP ORDER BY created_at ASC. Por cada card con balance > 0, invoca Recovery_Service. |
| Proposito | Catch-all para abonos no capturados por evento (ej. reverso sin hook, pod caido). |
| Limpieza | Marca EXPIRADA los que pasaron 90 dias. |

---

## 7. Modelo de datos

### 7.1 Tabla pending_charge

```sql
CREATE TABLE pending_charge (
    pending_id        NUMBER(12)    GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    card_id           NUMBER(10)    NOT NULL,
    account_id        NUMBER(10)    NOT NULL,
    charge_type       VARCHAR2(20)  NOT NULL,
    concept           VARCHAR2(100) NOT NULL,
    original_amount   NUMBER(15,2)  NOT NULL,
    pending_amount    NUMBER(15,2)  NOT NULL,
    original_txn_id   NUMBER(15),
    status            VARCHAR2(15)  DEFAULT 'PENDIENTE' NOT NULL,
    attempts          NUMBER(5)     DEFAULT 0 NOT NULL,
    last_attempt_at   TIMESTAMP,
    expires_at        TIMESTAMP     NOT NULL,
    created_at        TIMESTAMP     DEFAULT SYSTIMESTAMP NOT NULL,
    updated_at        TIMESTAMP,
    CONSTRAINT chk_pc_type   CHECK (charge_type IN ('COMISION','TECH_EXITOSA','CUOTA_MANEJO','GMF','CARGO_NOVEDAD')),
    CONSTRAINT chk_pc_status CHECK (status IN ('PENDIENTE','COBRADA','EXPIRADA','CANCELADA'))
);
```

### 7.2 Diagrama de estados

```
                [Cobro no efectivo]
                       |
                       v
                +------------+
                | PENDIENTE  |
                +-----+------+
                      |
         +------------+------------+
         |            |            |
         v            v            v
   +---------+  +-----------+  +------------+
   | COBRADA |  | EXPIRADA  |  | CANCELADA  |
   |(RECOBRO)|  |(90 dias   |  |(operacion  |
   |         |  | sin cobro)|  | manual)    |
   +---------+  +-----------+  +------------+
```

Transiciones:
- PENDIENTE a COBRADA: Recovery_Service liquida total (pending_amount llega a 0).
- PENDIENTE a EXPIRADA: 90 dias sin liquidar. Recovery_Retry_Scheduler marca.
- PENDIENTE a CANCELADA: Portal Admin con dual control + justificacion + audit.

### 7.3 Transacciones generadas

Cada cobro efectivo (inline o por recovery) genera un registro en `transaction`:

| txn_type | movement_type | channel | Origen |
|---|---|---|---|
| CUOTA_MANEJO | DEBITO | BATCH | Maintenance_Fee_Scheduler |
| COMISION | DEBITO | POS/ATM | Authorization_Engine (inline) o Recovery (diferido) |
| TECH_EXITOSA | DEBITO | POS/ATM | Authorization_Engine (inline) o Recovery (diferido) |
| GMF | DEBITO | BATCH | Archivo Entidad via Novelty_Processor o Recovery (diferido) |
| RECOBRO | DEBITO | BATCH | Recovery_Service (liquidacion de pending_charge) |

---

## 8. Reglas de negocio

### 8.1 Vigencia 90 dias

Todo `pending_charge` tiene `expires_at = created_at + 90 dias`. Configurable por AFG si se requiere en el futuro. Hoy son 90 dias fijos (regla de negocio confirmada).

### 8.2 FIFO (First In First Out)

Al ejecutar SP_RECOVER_CHARGES, se itera por `created_at ASC`. Se cobra primero el cargo mas viejo. Esto minimiza expiraciones: los mas viejos tienen menos tiempo de vida restante y son prioritarios.

### 8.3 Cobro parcial permitido

Si el saldo nuevo no alcanza para cubrir todo el `pending_amount`, se cobra lo disponible y el restante sigue como PENDIENTE. El `pending_amount` se actualiza (decrece). Los `attempts` incrementan.

### 8.4 Tarjetas elegibles

Solo se aplican cobros a tarjetas en estados: `Activa`, `Bloqueo Temporal`, `Bloqueo PIN Errado`.

Tarjetas en `Deshabilitada`, `Expirada`, `Pendiente por Activar` NO son elegibles. Si una tarjeta pasa a Deshabilitada con pending_charges vigentes, quedan como PENDIENTES hasta expirar (CINS) o hasta que operacion las cancele manualmente.

### 8.5 Idempotencia

Clave: `(charge_type, card_id, period_key)`.
- Cuota de manejo: period_key = `YYYYMM` (no cobra dos veces el mismo mes).
- Comision: period_key = `original_txn_id` (no cobra la misma comision dos veces).
- Tecnicamente exitosa: period_key = `original_txn_id`.
- GMF: period_key = `file_id:line_number`.

### 8.6 No incrementa contadores de limite

Los cobros diferidos NO incrementan los contadores del Limit_Counter_Service (regla 11, regla 81). Son cobros administrativos, no transacciones del TH.

### 8.7 Concepto en movimientos del TH

Cada cobro se refleja en los movimientos del TH con:
- Monto
- Concepto descriptivo ("Cuota de manejo enero 2026", "Comision retiro ATM otra red", "Cobro transaccion denegada codigo 51", "Proceso de Cobro Satisfactorio")
- Fecha y hora

---

## 9. Archivos de salida

El Motor de Cobros Diferidos NO escribe archivos (patron writer/publisher ADR-026). Persiste en BD y el `Output_File_Generator` (publisher unico) genera:

| Archivo | Contenido del motor | Hora | Lleva PAN | Cifrado |
|---|---|---|---|---|
| COMI dd.MM | Comisiones cobradas + recobros del dia | 01:15 AM | No (BIN + last_four) | PGP por GAW |
| CSAT dd.MM | Subset cobradas (satisfactorias) | 01:15 AM | No | PGP por GAW |
| CINS dd.MM | Insatisfactorias acumuladas 3 meses (EXPIRADA) | 01:15 AM | No | PGP por GAW |
| AUM dd.MM | Movimientos del dia (incluye CUOTA_MANEJO, GMF, COMISION, TECH_EXITOSA, RECOBRO) | 00:30 AM | Si (PAN descifrado) | PGP por GAW |

**COMI incluye:**
- Comisiones cobradas inline (motor transaccional).
- Comisiones cobradas por recobro (Recovery_Service): concepto "Proceso de Cobro Satisfactorio".
- Comisiones no cobradas (pending_charge status=PENDIENTE).

**CINS incluye:**
- Cobros expirados (90 dias) consolidados.

---

## 10. Seguridad y PCI

### 10.1 PAN

El Motor de Cobros Diferidos NO descifra ni procesa PAN. Todos los componentes trabajan con `card_id` (identificador opaco). El PAN solo se descifra en el Output_File_Generator al generar AUM (ver `requerimientos-procesamiento-pan` y ADR-006/018).

### 10.2 Audit

Cada operacion de settle y recover genera registro en `audit_log` con:
- ID_AUDIT, SERVICE_NAME (Charge_Settlement_Service / Recovery_Service), METHOD_NAME (settle / recover)
- CARD_ID, CHARGE_TYPE, AMOUNT, RESULT
- CREATED_AT, RESPONSE_TIME_MS
- TECHNICAL_USER, SOURCE_SYSTEM

### 10.3 Acceso al Portal Admin (cancelacion manual)

- Roles RBAC: `PREPAID_OPS_CANCELLER` (solicita) + `PREPAID_OPS_APPROVER` (aprueba).
- Dual control: un usuario solicita, otro aprueba.
- Justificacion obligatoria escrita.
- Umbral: cancelaciones > $500.000 requieren nivel superior.
- Audit reforzado (PCI 10.2 + acta de cancelacion).

---

## 11. Performance y SLAs

| Indicador | Objetivo |
|---|---|
| Latencia settle() inline (comision, tech exitosa) | p95 < 50 ms (esta en el hot-path del motor txn) |
| Latencia settle() batch (cuota mensual) | p95 < 200 ms por tarjeta |
| Throughput Maintenance_Fee_Scheduler | 1.5M tarjetas en < 30 minutos (chunk 5k) |
| Latencia Recovery por evento | p95 < 2 segundos desde commit del abono |
| Throughput Recovery batch (03:00) | Todas las pending_charge con saldo disponible en < 20 min |
| Ventana COMI/CSAT/CINS en OFG | Generacion completa antes de 01:30 AM |
| Disponibilidad Charge_Settlement_Service | 99.9% (24/7 — invocado desde motor txn) |

---

## 12. Pendientes

| # | Pendiente | Origen | Responsable |
|---|---|---|---|
| 1 | SP_SETTLE_CHARGE: implementar en `db/02_stored_procedures.sql` | ADR-027 | Desarrollo |
| 2 | SP_RECOVER_CHARGES: implementar en `db/02_stored_procedures.sql` | ADR-027 | Desarrollo |
| 3 | Politica de cobro parcial por charge_type (cuales permiten parcial, cuales no) | Producto | Producto |
| 4 | Prorrateo de cuota de manejo para tarjetas creadas a mitad de mes | Producto | Producto |
| 5 | Archivo GMF: nombre canonico y estructura del registro | ADR-028 | Entidad + Producto |
| 6 | Hook de Recovery tras SP_REVERSE_TXN (disparador 3) | Desarrollo | Desarrollo |
| 7 | Definir umbral de cancelacion manual ($500k o configurable) | Operaciones | Producto + Ops |
| 8 | Alertamiento cuando proporcion de PENDIENTE > 20% del total | Monitoreo | Desarrollo |
| 9 | Que pasa con pending_charge cuando la tarjeta se Deshabilita | Producto | Producto |

---

## 13. Referencias cruzadas

### 13.1 Documentos

- `requerimientos-flujo-transaccion.md` — seccion 11.3 (comisiones), 11.4 (tech exitosas), 11.5 (impuestos), 11.6 (GMF linea).
- `requerimientos-novedades-monetarias.md` — seccion 4.5 (GMF batch por archivo), 4.6 (recobro de cargos no efectivos).
- `decisiones-arquitectura.md` — ADR-012 (GMF linea vs batch), ADR-013 (tech exitosas), ADR-014 (recobro 3 meses), ADR-027 (consolidacion pestanas), ADR-028 (GMF como archivo), ADR-029 (PGP en GAW + cache DEK).

### 13.2 Diagrama

- `PrepagoUnificadoArquitectura_V_1_11.drawio` — pestana "Motor de Cobros Diferidos".

### 13.3 SQL

- `db/01_schema_prepago.sql` — tabla `pending_charge`, tabla `transaction`.
- `db/02_stored_procedures.sql` — SP_SETTLE_CHARGE y SP_RECOVER_CHARGES (pendientes de implementar).

---

> **Nota de versionado:** este documento se mantiene como unica fuente para el Motor de Cobros Diferidos. Las modificaciones se registran como ADRs nuevos en `decisiones-arquitectura.md`.