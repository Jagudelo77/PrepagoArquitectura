# Requerimientos — Generación de Reportes para Facturación a Entidades (Req 30)

Documento del requerimiento de generación mensual de reportes de facturación a las Entidades emisoras y a los Proveedores (Realce/Distribución). Cubre los dos archivos de facturación por Entidad/AFG, el modelo de cobro (cargos por tarjeta activa/inactiva + fee de administración) y los reportes de proveedores.

**Fuentes:**
- `Prepago/Requerimientos Técnicos (15).docx` — "Generación de Reportes para Facturación a Entidades", "Generación de Reportes para Facturación a Proveedores", "Facturación a Entidades" (Etapa 6 — Operación y Soporte / Reportería). *(Contenido idéntico a las versiones (13) y (14); el diff (13)→(15) no introdujo cambios de contenido.)*
- `Prepago/docs/estructuras-archivos-manual-tecnico.md` — §2.18 (Indicadores Base, estructura similar).
- `Prepago/docs/decisiones-arquitectura.md` — ADR-002/005 (Read Replica para reportería), ADR-022/025 (B2B_Mail_Service), ADR-029 (PGP en GAW), ADR-034 (pestaña Reportería/Facturación).
- `Prepago/PrepagoUnificadoArquitectura_V_1_18.drawio` — pestaña "Reporteria y Facturacion" (`Billing_Batch` + `Prepaid_Reporting_API` + `billing_monthly_snapshot`).

> Fase: **Fase 1 — Línea Base (MVP)**, Etapa 6 (Operación y Soporte) / Reportería.
> Última actualización: 24/06/2026.
> Estado: **Vigente**.

---

## Tabla de contenido

1. [Resumen y alcance](#1-resumen-y-alcance)
2. [Reportes a generar](#2-reportes-a-generar)
3. [Reporte de Facturación por AFG](#3-reporte-de-facturación-por-afg)
4. [Reporte de Facturación por BIN](#4-reporte-de-facturación-por-bin)
5. [Reportes de Facturación a Proveedores (Realce y Distribución)](#5-reportes-de-facturación-a-proveedores-realce-y-distribución)
6. [Modelo de cobro a Entidades](#6-modelo-de-cobro-a-entidades)
7. [Definiciones clave](#7-definiciones-clave)
8. [Programación, entrega y reintentos](#8-programación-entrega-y-reintentos)
9. [Cobertura en la arquitectura](#9-cobertura-en-la-arquitectura)
10. [Modelo de datos y origen de cada campo](#10-modelo-de-datos-y-origen-de-cada-campo)
11. [Pendientes](#11-pendientes)
12. [Referencias cruzadas](#12-referencias-cruzadas)

---

## 1. Resumen y alcance

### 1.1 Objetivo

Generar mensualmente los reportes que el equipo de Facturación de Credibanco necesita para facturar el producto prepago a las Entidades emisoras y para gestionar la facturación de los Proveedores de Realce y Distribución.

### 1.2 Alcance funcional

**Incluye:**
- **Dos archivos de facturación a Entidades** (formato `.xlsx`):
  1. Reporte consolidado **por AFG**.
  2. Reporte consolidado **por BIN** de producto.
- **Reportes de facturación a Proveedores** (Realce y Distribución) — uno por proveedor.
- Generación mensual automática, disposición vía SFTP/GAW.
- Notificaciones por correo (éxito/error) y reintentos.

**Excluye (lo realiza el área de Facturación, no la plataforma):**
- El cálculo final de la factura y el cobro a la Entidad (proceso "Facturación a Entidades", lo ejecuta el área de Facturación a partir de estos reportes).
- La emisión de la factura tributaria.

### 1.3 Resultado esperado

- Cada primer día calendario del mes, la plataforma genera los archivos de facturación con la data del mes inmediatamente anterior.
- Los archivos quedan dispuestos en la carpeta SFTP/GAW antes de las 08:00 AM.
- El equipo de Facturación toma esos reportes para facturar a Entidades y Proveedores.

---

## 2. Reportes a generar

| # | Reporte | Granularidad | Formato | Destino |
|---|---|---|---|---|
| 1 | Facturación por AFG | 1 línea por AFG | `.xlsx` | Equipo Facturación (vía SFTP/GAW) |
| 2 | Facturación por BIN | 1 línea por BIN de entidad (si la entidad tiene 5 BINs prepago → 5 líneas) | `.xlsx` | Equipo Facturación (vía SFTP/GAW) |
| 3 | Facturación Proveedor — Distribución | 1 reporte por proveedor | (formato a definir) | Facturación (primer día hábil del mes) |
| 4 | Facturación Proveedor — Realce | 1 reporte por proveedor | (formato a definir) | Facturación (primer día hábil del mes) |

> **Nota del requerimiento:** "se deben generar dos (2) archivos, uno por Entidad y otro por AFG asociados a cada Entidad". En el detalle de campos del docx, los dos reportes a Entidades son: uno por **AFG** y otro por **BIN** de producto.

---

## 3. Reporte de Facturación por AFG

**Granularidad:** una línea por AFG. Formato `.xlsx`.

Campos (literal del docx fuente):

| # | Campo |
|---|---|
| 1 | ENTIDAD FINANCIERA |
| 2 | CÓDIGO DE COMPENSACIÓN |
| 3 | MES |
| 4 | AÑO |
| 5 | BIN |
| 6 | FRANQUICIA |
| 7 | AFG ID |
| 8 | NOMBRE AFG |
| 9 | TIPO |
| 10 | TOTAL TARJETAS ACTIVAS POR AFG |
| 11 | CANTIDAD TARJETAS EMITIDAS MES |
| 12 | CANTIDAD TARJETAS REALZADAS MES |
| 13 | CANTIDAD TARJETAS CANCELADAS MES |
| 14 | CANTIDAD DE TARJETAS INACTIVAS |
| 15 | CANTIDAD PINES EMITIDOS MES PARA TARJETAS INNOMINADAS |
| 16 | CANTIDAD DE NOVEDADES MONETARIAS RECIBIDOS MES |
| 17 | VALOR TOTAL NOVEDADES MONETARIAS MES |
| 18 | VALOR TOTAL PROMEDIO NOVEDADES MONETARIAS MES |
| 19 | SALDO TARJETAS CIERRE MES |
| 20 | PROVEEDOR DE REALCE TARJETAS |
| 21 | COURRIER TARJETAS |
| 22 | PROVEEDOR DE REALCE PIN PARA TARJETAS INNOMINADAS |
| 23 | COURRIER PIN PARA TARJETAS INNOMINADAS |
| 24 | CANTIDAD TRANSACCIONES EXITOSAS ATM |
| 25 | VALOR TOTAL TRANSACCIONES ATM |
| 26 | CANTIDAD TRANSACCIONES EXITOSAS POS |
| 27 | VALOR TOTAL TRANSACCIONES POS |
| 28 | CANTIDAD TRANSACCIONES EXITOSAS OTROS CANALES |
| 29 | VALOR TOTAL TRANSACCIONES OTROS CANALES |
| 30 | TOTAL TARJETAS REEXPEDIDAS |
| 31 | TARJETAS CON SALDO MAYOR A CERO |
| 32 | TARJETAS CON SALDO IGUAL A CERO CON TRANSACCIONES |

---

## 4. Reporte de Facturación por BIN

**Granularidad:** una línea por BIN de entidad. Si la entidad tiene 5 BINs prepago, habrá 5 líneas de detalle. Formato `.xlsx`.

Campos (literal del docx fuente):

| # | Campo |
|---|---|
| 1 | ENTIDAD FINANCIERA |
| 2 | CÓDIGO DE COMPENSACIÓN |
| 3 | MES |
| 4 | AÑO |
| 5 | BIN |
| 6 | FRANQUICIA |
| 7 | TOTAL TARJETAS ACTIVAS POR BIN |
| 8 | CANTIDAD TARJETAS EMITIDAS MES |
| 9 | CANTIDAD TARJETAS REALZADAS MES |
| 10 | CANTIDAD TARJETAS CANCELADAS MES |
| 11 | CANTIDAD DE TARJETAS INACTIVAS |
| 12 | CANTIDAD PINES EMITIDOS MES PARA TARJETAS INNOMINADAS |
| 13 | CANTIDAD DE NOVEDADES MONETARIAS RECIBIDOS MES |
| 14 | VALOR TOTAL NOVEDADES MONETARIAS MES |
| 15 | VALOR TOTAL PROMEDIO NOVEDADES MONETARIAS MES |
| 16 | SALDO TARJETAS CIERRE MES |
| 17 | CANTIDAD TRANSACCIONES EXITOSAS ATM |
| 18 | VALOR TOTAL TRANSACCIONES ATM |
| 19 | CANTIDAD TRANSACCIONES EXITOSAS POS |
| 20 | VALOR TOTAL TRANSACCIONES POS |
| 21 | CANTIDAD TRANSACCIONES EXITOSAS OTROS CANALES |
| 22 | VALOR TOTAL TRANSACCIONES OTROS CANALES |
| 23 | TOTAL TARJETAS REEXPEDIDAS |
| 24 | TARJETAS CON SALDO MAYOR A CERO |
| 25 | TARJETAS CON SALDO IGUAL A CERO CON TRANSACCIONES |

---

## 5. Reportes de Facturación a Proveedores (Realce y Distribución)

**Descripción.** Reportes mensuales de tarjetas realzadas para facturar a los Proveedores. Se toman como base los reportes diarios de Realce. Un reporte por cada proveedor.

### 5.1 Reporte de Distribución

| # | Campo |
|---|---|
| 1 | Fecha de Procesamiento |
| 2 | ID del AFG |
| 3 | Nombre del AFG |
| 4 | Cantidad de tarjetas |
| 5 | Entidad |
| 6 | Embozador |
| 7 | Ciudad, Dirección, Teléfono |
| 8 | Nombre Custodio 1 tarjeta |
| 9 | Doc. Custodio 1 tarjeta |

### 5.2 Reporte de Realce

| # | Campo |
|---|---|
| 1 | Fecha de Procesamiento |
| 2 | ID del AFG |
| 3 | Nombre del AFG |
| 4 | Cantidad de tarjetas |
| 5 | Entidad |
| 6 | Embozador |

**Programación:** ambos reportes (Distribución y Realce) se generan y envían a facturación el **primer día hábil de cada mes**.

---

## 6. Modelo de cobro a Entidades

> El cálculo de la factura lo ejecuta el **área de Facturación** a partir de los reportes anteriores. El modelo de cobro definido es:

### 6.1 Cargo por tarjeta activa

- Cobro **mensual por cada tarjeta activa**, según una **tabla de rangos por volumen** (Tabla 1: "Cargos por Tarjetas Activas — Valor — Modelo por Ubicación de Rangos"). El valor por tarjeta depende del rango de volumen del cierre del mes.

### 6.2 Cargo por tarjeta inactiva

- Cobro por tarjeta inactiva según Tabla 2 ("Cargos por Tarjetas Inactivas — Valor Mes").
- **Condición de aplicación (regla de negocio del docx):** este cobro **solo aplica cuando la cantidad de tarjetas inactivas supera el 50% sobre el total de tarjetas de la entidad en el sistema**. Si las inactivas son ≤ 50% del total de la entidad, no se cobra por inactivas.
- **Definición literal de tarjeta inactiva (docx):** *"toda tarjeta creada en el sistema que durante el periodo de liquidación NO ha realizado transacciones, ni ha generado comisiones y su saldo es igual a $0, independiente del estado de la tarjeta en la plataforma de procesamiento."*

### 6.3 Fee de Administración mensual

- **$23.000.000** de fee de administración mensual **si la entidad no alcanza el mínimo del rango** de tarjetas.

### 6.4 Definición de "Tarjeta Activa" para facturación

Una tarjeta se considera **activa** (para efectos de facturación) si durante el periodo de liquidación cumple **al menos una** de estas condiciones:
- Realizó al menos una transacción, **o**
- Mantuvo en algún momento del periodo un saldo distinto de $0, **o**
- Generó comisiones.

Esto aplica **independientemente del estado de la tarjeta** en la plataforma de procesamiento (una tarjeta puede estar bloqueada pero contar como activa para facturación si cumple alguna condición).

> **Nota importante:** la definición de "activa para facturación" es **distinta** de la de "estado Activa" en la máquina de estados de la tarjeta (pestaña "Estados de Tarjeta"). Esto debe quedar muy claro en la implementación para no confundir los dos conceptos.

---

## 7. Definiciones clave

| Término | Definición |
|---|---|
| **Tarjeta activa (facturación)** | Realizó ≥1 transacción, o tuvo saldo ≠ $0, o generó comisiones en el periodo (independiente del estado en plataforma). |
| **Tarjeta inactiva (facturación)** | Tarjeta creada que durante el periodo NO realizó transacciones, NI generó comisiones y su saldo es $0 (independiente del estado en plataforma). El cobro por inactivas solo aplica si superan el 50% del total de tarjetas de la entidad. |
| **Periodo de liquidación** | Mes calendario inmediatamente anterior. |
| **AFG** | Affinity Group — agrupación de tarjetas. Un AFG pertenece a un BIN/Entidad. |
| **BIN de producto** | Bank Identification Number. Una entidad puede tener varios BINs prepago. |
| **Fee de Administración** | Cargo fijo mensual ($23M) cuando la entidad no alcanza el rango mínimo de tarjetas. |

---

## 8. Programación, entrega y reintentos

| Aspecto | Definición |
|---|---|
| **Periodicidad** | Mensual |
| **Día de ejecución** | Primer día calendario del mes (datos del mes inmediatamente anterior) |
| **Hora de inicio** | 00:00 AM |
| **Hora límite de disposición** | Antes de las 08:00 AM en la carpeta definida |
| **Corte de datos** | 11:59:59 PM del último día del mes inmediatamente anterior |
| **Paralelismo** | Los archivos de facturación y de saldos se ejecutan en paralelo cada mes |
| **Canal de entrega** | SFTP / GoAnywhere (igual que el proceso actual) |
| **Notificaciones** | Correo de éxito y de error durante el procesamiento (vía B2B_Mail_Service) |
| **Reintentos** | 3 reintentos cada hora, tomando la información de las 11:59:59 PM del último día del mes anterior |
| **Reportes de proveedores** | Primer día hábil del mes |

---

## 9. Cobertura en la arquitectura

### 9.1 Estado actual

| Elemento | Componente / artefacto | Estado |
|---|---|---|
| Componente batch generador | `Billing_Batch` (Spring Batch, CronJob mensual) — genera los reportes `.xlsx` por AFG/BIN/Proveedores | ✅ Diagramado (V_1_17/V_1_18, ADR-034) |
| Componente consulta on-demand | `Prepaid_Reporting_API` (`/reports/billing/entities`, `/reports/billing/providers`, `/reports/indicators`) | ✅ Diagramado (V_1_17/V_1_18) |
| Fuente de datos | Oracle Read Replica (ADR-002/005) | ✅ Definida |
| Canal de entrega | SFTP / GoAnywhere + B2B_Mail_Service (notificaciones) | ✅ Existe |
| Datos disponibles | `transaction`, `card`, `commission`, `affinity_group`, `entity`, `bin`, `pending_charge` | ✅ Existen (vía vistas/queries) |
| Flujo diagramado dedicado | Pestaña "Reporteria y Facturacion" | ✅ Creada (ADR-034) |
| Tabla de snapshot | `billing_monthly_snapshot` (trazabilidad/idempotencia) | 🟡 Diagramada; pendiente implementar en `db/01_schema_prepago.sql` |
| Vista de tarjetas activas | `vw_billing_active_cards` | 🟡 Diagramada; pendiente implementar en `db/01_schema_prepago.sql` |
| Stack de BI (si aplica) | — | ❌ Diferido (Power BI vs Qlik) |

### 9.2 Flujo objetivo (a diagramar)

```
[Read Replica] ──▶ [Prepaid_Reporting_API / Billing_Batch]
                       │ (CronJob primer día del mes, 00:00, Spring Batch)
                       ├── Calcula consolidado por AFG → reporte AFG .xlsx
                       ├── Calcula consolidado por BIN → reporte BIN .xlsx
                       ├── Calcula reportes de proveedores (Realce / Distribución)
                       ▼
                  [NFS /output]
                       ▼
                  [GoAnywhere] ── SFTP ──▶ Carpeta de Facturación
                       │
                       └── B2B_Mail_Service: notifica éxito/error
```

> Debe seguir los lineamientos de Spring Batch del documento `lineamientos-output-file-generator.md` (cursor reader, streaming, partición). El volumen es 1 línea por AFG/BIN (bajo), pero las agregaciones (transacciones del mes) leen millones de filas → cursor + agregación en BD.

---

## 10. Modelo de datos y origen de cada campo

| Campo del reporte | Origen | Cálculo |
|---|---|---|
| Entidad Financiera / Código Compensación | `entity` | Static |
| MES / AÑO | Parámetro del job | Periodo |
| BIN / FRANQUICIA | `bin` | Static |
| AFG ID / NOMBRE AFG / TIPO | `affinity_group` | Static |
| TOTAL TARJETAS ACTIVAS POR AFG/BIN | `card` + `transaction` | COUNT con definición de "activa facturación" (§6.4) |
| CANTIDAD TARJETAS EMITIDAS MES | `card` | COUNT WHERE created_at en el mes |
| CANTIDAD TARJETAS REALZADAS MES | `card_state_history` / `embossing_order` | COUNT realzadas en el mes |
| CANTIDAD TARJETAS CANCELADAS MES | `card_state_history` | COUNT a estado deshabilitada en el mes |
| CANTIDAD DE TARJETAS INACTIVAS | `card` + `transaction` | COUNT que NO cumplen activa-facturación |
| CANTIDAD PINES EMITIDOS MES (innominadas) | (origen TBD — depende de gestión de PIN innominadas) | COUNT |
| CANTIDAD / VALOR NOVEDADES MONETARIAS MES | `transaction` (ABONO/CARGO) | COUNT / SUM / AVG |
| SALDO TARJETAS CIERRE MES | `account` snapshot (SALD) | SUM al cierre |
| PROVEEDOR REALCE / COURRIER | `affinity_group` config / `embossing_order` | Static |
| CANTIDAD / VALOR TRANSACCIONES ATM/POS/OTROS | `transaction` | COUNT / SUM por canal |
| TOTAL TARJETAS REEXPEDIDAS | (Fase 2 — reexpedición no aplica en Fase 1) | 0 / TBD |
| TARJETAS CON SALDO MAYOR A CERO | `account` | COUNT balance > 0 |
| TARJETAS CON SALDO IGUAL A CERO CON TRANSACCIONES | `account` + `transaction` | COUNT balance = 0 AND tuvo txn |

### 10.1 Vistas / tablas propuestas

```sql
-- Vista de tarjetas activas para facturación (definición §6.4)
CREATE OR REPLACE VIEW vw_billing_active_cards AS
SELECT c.card_id, c.afg_id, c.bin, :period AS period
FROM card c
WHERE EXISTS (SELECT 1 FROM transaction t
              WHERE t.card_id = c.card_id
                AND t.txn_date BETWEEN :period_start AND :period_end)
   OR EXISTS (SELECT 1 FROM account a
              WHERE a.card_id = c.card_id AND a.balance <> 0)   -- saldo en algún momento (requiere historial)
   OR EXISTS (SELECT 1 FROM transaction t
              WHERE t.card_id = c.card_id AND t.txn_type = 'COMISION'
                AND t.txn_date BETWEEN :period_start AND :period_end);

-- Tabla de snapshot de facturación mensual (para trazabilidad e idempotencia)
CREATE TABLE billing_monthly_snapshot (
    snapshot_id   NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    period        VARCHAR2(6),         -- YYYYMM
    afg_id        NUMBER(10),
    bin           VARCHAR2(8),
    total_active_cards   NUMBER(10),
    total_inactive_cards NUMBER(10),
    ...                                 -- resto de campos del reporte
    generated_at  TIMESTAMP DEFAULT SYSTIMESTAMP,
    CONSTRAINT uq_billing UNIQUE (period, afg_id, bin)
);
```

> **Nota sobre "saldo distinto a $0 en algún momento del periodo":** la definición §6.4 requiere saber si la tarjeta tuvo saldo ≠ $0 en **cualquier momento** del mes, no solo al cierre. Esto puede requerir consultar el historial de movimientos (`transaction`) o snapshots diarios de saldo (`SALD`), no solo el balance actual. Es un punto de implementación a precisar.

---

## 11. Pendientes

| # | Pendiente | Responsable | Estado |
|---|---|---|---|
| 1 | Tabla 1 completa: rangos de volumen y valor por tarjeta activa | Producto / Comercial | Abierto |
| 2 | Tabla 2 completa: valor por tarjeta inactiva | Producto / Comercial | Abierto |
| 3 | ~~Confirmar si "tarjetas inactivas se factura"~~ → **Resuelto:** se factura solo si las inactivas superan el 50% del total de tarjetas de la entidad (§6.2) | Producto / Comercial | ✅ Cerrado |
| 4 | Origen de "CANTIDAD PINES EMITIDOS MES PARA TARJETAS INNOMINADAS" | Producto + Pomelo (gestión PIN innominadas) | Abierto |
| 5 | Cómo calcular "saldo distinto a $0 en algún momento del periodo" (historial vs snapshot diario) | Arquitectura + Desarrollo | Abierto |
| 6 | Formato exacto de los reportes de proveedores (Realce/Distribución) | Producto | Abierto |
| 7 | Stack de reportería/BI si se requiere más allá de .xlsx | Arquitectura | Abierto |
| 8 | Confirmar carpeta SFTP/GAW destino del equipo de Facturación | Operaciones | Abierto |
| 9 | ~~Decidir si se crea `Billing_Batch` dedicado o se usa `Prepaid_Reporting_API`~~ → **Resuelto:** ambos (Billing_Batch genera archivos; Reporting_API on-demand) — ADR-034 | Arquitectura | ✅ Cerrado |
| 10 | ~~Pestaña de arquitectura "Reportería y Facturación" en el unificado~~ → **Resuelto:** creada en V_1_17/V_1_18 — ADR-034 | Arquitectura | ✅ Cerrado |
| 11 | Implementar `billing_monthly_snapshot` y `vw_billing_active_cards` en `db/01_schema_prepago.sql` | Desarrollo | Abierto |

---

## 12. Referencias cruzadas

### 12.1 Documentos

- `requerimientos-novedades-monetarias.md` §12 — reportería operativa (campos consolidados similares).
- `estructuras-archivos-manual-tecnico.md` §2.18 — Indicadores Base (estructura análoga).
- `requerimientos-ciclo-vida-tarjeta-y-portal-th.md` §3 — Distribución (insumo de proveedores).
- `lineamientos-output-file-generator.md` — patrón Spring Batch para generación.
- `decisiones-arquitectura.md` — ADR-002/005 (Read Replica), ADR-022/025 (B2B_Mail_Service).

### 12.2 Diagrama

- `PrepagoUnificadoArquitectura_V_1_18.drawio` — pestaña "Reporteria y Facturacion" (`Billing_Batch` + `Prepaid_Reporting_API` + `billing_monthly_snapshot` + OFG + GAW + B2B_Mail). Ver ADR-034.

### 12.3 Datos fuente

- Read Replica: `transaction`, `card`, `account`, `commission`, `affinity_group`, `entity`, `bin`, `card_state_history`, `embossing_order`.

---

> **Nota:** este documento cubre el **Req 30 — Generación de Reportes para Facturación a Entidades**. El cálculo final de la factura y el cobro lo ejecuta el área de Facturación (proceso "Facturación a Entidades"), fuera del alcance de la plataforma. El modelo de cobro (Tabla 1 y 2) está parcialmente definido — ver pendientes.