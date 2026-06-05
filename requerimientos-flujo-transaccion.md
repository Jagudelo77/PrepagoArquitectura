# Requerimientos — Flujo Transaccional (Motor de Autorización)

Extraídos del documento `Prepago/Requerimientos Técnicos (12).docx`.
Pertenecen principalmente a **Fase 1 – MVP, Etapa 3: Motor Transaccional**, complementados con los requerimientos conexos de Etapas 1 y 5 que alimentan o cierran el ciclo transaccional.

> Scope: Autorizador propio Credibanco que valida reglas de negocio y parámetros definidos para aprobar o rechazar transacciones financieras realizadas a través de POS o ATM, en integración con Pomelo como pre-autorizador.

---

## 1. Autorización Transaccional (POS-ATM) — marco general

**Descripción.** Construcción del autorizador propio Credibanco que garantice la validación de la información de reglas de negocio y parámetros definidos para la aprobación o rechazo de transacciones financieras realizadas a través de POS o ATM.

**Transacciones soportadas:**

| Canal | Transacciones |
|-------|---------------|
| POS   | Consulta de Saldo, Compra (Nacional / Internacional), Reverso de Compra, Anulación, Reverso de Anulación |
| ATM   | Consulta de Saldo, Retiro (Nacional / Internacional), Reverso de Retiro |

---

## 2. POS — Consultas de Saldo

**Concepto de transacción:** `CONSULTA DE SALDO`
**Mapeo Pomelo:** `transaction.type = BALANCE_INQUIRY`

**Criterios de aceptación:**
- Pomelo valida: PIN, fecha de vencimiento, existencia y estado de tarjeta, BIN activo, AFG activo, CVV.
- Homologar el saldo reportado con el saldo del sistema Credibanco.
- Reglas / campos parametrizables a evaluar:
  - `02` Validar Establecimiento (plan de validación)
  - `07` PIN (apagada con Pomelo)
  - `10` Validar Estado de Tarjeta (apagada con Pomelo)
  - `11` Número Utilizaciones POS (aplica compras y saldo)
  - `13` Fecha de Expiración (apagada con Pomelo)
  - `14` Trans. Permitidas x Tarjeta — VP / VNP (`extra_data.card_presence`)
  - `26` Permitir Consulta Saldo Nacional
  - `27` Permitir Consulta Saldo Internacional
  - Límite de Cantidad de Consultas de Saldo
  - `66` Validar Estado de Cuenta
  - Selección Tipo de Producto
  - Límite de cantidad de PIN Errado (apagada con Pomelo)

**Pendiente:** definir notificación cuando la tarjeta se bloquea por PIN errado y cómo se entera Credibanco de dichos bloqueos ejecutados por Pomelo.

---

## 3. POS — Compras (Nacional / Internacional)

**Concepto de transacción:** `COMPRA`

**Criterios de aceptación:**
- Pomelo valida: PIN, vencimiento, existencia y estado de tarjeta, BIN activo, AFG activo, CVV.
- Reglas / campos parametrizables:
  - `02` Validar Establecimiento (plan de validación)
  - `07` PIN, `08` CVV, `09` CVV2 (apagadas con Pomelo)
  - `10` Validar Estado de Tarjeta
  - `11` Número Utilizaciones POS
  - `13` Fecha de Expiración (apagada con Pomelo)
  - `14` Trans. Permitidas x Tarjeta (VP, VNP)
  - `19` Evaluar Cobro Impuestos
  - `20` Permitir Compras Nacionales / `21` Permitir Compra Internacional
  - `36` Límite Cantidad Compras (diario) / `37` Límite Monto Compras (diario)
  - `66` Validar Estado de Cuenta / `67` Selección Tipo de Cuenta
  - `69` Validación PIN en F' (apagada con Pomelo)
- En el plan de validación se agregan los comercios donde la operación no va a ser permitida.

---

## 4. POS — Reverso de Compras

**Concepto de transacción:** `REVERSO DE COMPRA`

**Criterios de aceptación:**
- Idempotencia: campo que indique que la compra ya fue reversada, para evitar duplicados.
- Validar que la compra a reversar exista, coincidiendo en: id tarjeta, monto, código de autorización (si viene), código de comercio, terminal, fecha, RRN, código de respuesta exitoso, tipo de movimiento, concepto y `reversada = NO / 0`.
- Se conserva el mismo código de autorización de la compra original.

---

## 5. POS — Anulaciones

**Concepto de transacción:** `ANULACION`

**Criterios de aceptación:**
- Idempotencia: campo `anulada` para evitar duplicados.
- Validar la compra a anular por los mismos campos del reverso.
- La anulación genera **un nuevo código de autorización**.
- Resta al contador de la regla `11` Número de Utilización POS.
- Reglas parametrizables:
  - `28` Permitir Anulación Nacional
  - `29` Permitir Anulación Internacional

---

## 6. POS — Reverso de Anulaciones

**Concepto de transacción:** `REVERSO DE ANULACION`

**Criterios de aceptación:**
- Idempotencia: campo `reversada` sobre la anulación.
- Validar que la anulación a reversar exista (mismos campos de matching que en compra).
- Se conserva el código de autorización de la anulación original.

---

## 7. ATM — Consultas de Saldo

**Concepto de transacción:** `CONSULTA DE SALDO`

**Criterios de aceptación:**
- Pomelo valida: PIN, vencimiento, existencia y estado de tarjeta, BIN activo, AFG activo, CVV.
- Usar `extra_data.function_code` para validar costo de la operación. *Pendiente confirmar con Pomelo si el costo puede ser dado por Credibanco.*
- Si hay costo, afecta balance y se registra como movimiento (evaluar si mostrarlo en consulta de movimientos).
- Reglas / campos parametrizables:
  - `26` Permitir Consulta Saldo Nacional
  - `27` Permitir Consulta Saldo Internacional
  - `11` Número Utilizaciones ATM
  - `15` Evaluar Cobro de Comisión
  - `19` Evaluar Cobro Impuestos
  - `66` Estado de Cuenta
  - `67` Selección Tipo de Cuenta

---

## 8. ATM — Retiros

**Concepto de transacción:** `RETIRO`

**Criterios de aceptación:**
- Pomelo valida: PIN, vencimiento, existencia y estado de tarjeta, BIN activo, AFG activo, CVV.
- Usar `extra_data.function_code` para validar costo. Si viene `ATM_FEE_INQUIRY`, devolver en el campo balance el cobro de la comisión (*pendiente Pomelo*).
- Generar código de autorización igual que en proceso de compra POS.
- Reglas / campos parametrizables:
  - `22` Permitir Retiro Nacional
  - `23` Permitir Retiro Internacional
  - `11` Número Utilizaciones ATM
  - `15` Evaluar Cobro de Comisión
  - `19` Evaluar Cobro Impuestos
  - `66` Estado de Cuenta
  - `36` Límite Cantidad de Retiros (diario)
  - `37` Límite Monto de Retiros (diario)

---

## 9. ATM — Reverso de Retiro

**Concepto de transacción:** `REVERSO DE RETIRO`

**Criterios de aceptación:**
- Idempotencia: campo que indique que el retiro ya fue reversado, para evitar duplicados.
- Validar que el retiro a reversar exista (id tarjeta, monto, código de autorización si viene, terminal / número cajero, fecha, código de respuesta exitoso, tipo de movimiento, concepto, `reversada = NO / 0`).
- Resta al contador de utilizaciones ATM.
- Se conserva el código de autorización del retiro original.

**Pendientes con Pomelo:**
- Contenido del objeto `MERCHANT` cuando la transacción viene por ATM.
- Cómo llega desde Pomelo el código de autorización.
- Cómo llega desde Pomelo el merchant ID.

---

## 10. Límites aplicables al motor transaccional

Consolidación de todos los límites que el motor transaccional debe evaluar. **Todos se implementan como contadores en Redis gestionados por el `Limit Counter Service`** (ver ADR-004 en `decisiones-arquitectura.md`). El TTL del contador = periodicidad del límite → reinicio automático.

### 10.1 Límites por transacción / operación (AFG)

Son límites "duros" que nacen de la parametría del AFG en Pomelo y también se evalúan localmente:

| Regla | Descripción | Canal | Periodicidad |
|-------|-------------|-------|--------------|
| — | Límites monto y cantidad por operación | POS / ATM | Por transacción |
| — | Compras Nacional / Internacional (monto y cantidad) | POS | Diario |
| — | e-Commerce Nacional / Internacional (monto y cantidad) | POS (VNP) | Diario |
| — | Retiro Nacional / Internacional (monto y cantidad) | ATM | Diario |
| — | Límites trx aprobadas diarias por cliente (sin importar canal) — monto y cantidad | POS + ATM | Diario |
| — | Límites cantidad y monto transacciones compras VNP con #D | POS (VNP 3DS) | Diario |

### 10.2 Límites por regla de negocio (campos parametrizables)

Son los que el documento numera explícitamente y el motor evalúa por tarjeta:

| Campo | Nombre | Tipo | Canal | Periodicidad |
|-------|--------|------|-------|--------------|
| `11`  | Número Utilizaciones POS | Numérico | POS | Según AFG |
| `11`  | Número Utilizaciones ATM | Numérico | ATM | Según AFG |
| `36`  | Límite Cantidad de Compras | Numérico | POS | Diario |
| `37`  | Límite Monto de Compras | Numérico | POS | Diario |
| `36`  | Límite Cantidad de Retiros | Numérico | ATM | Diario |
| `37`  | Límite Monto de Retiros | Numérico | ATM | Diario |
| —     | Límite Cantidad de Consultas de Saldo | Numérico | POS / ATM | Diario |
| —     | Límite de Cantidad de PIN Errado | Numérico | POS / ATM | Diario |
| `81`  | Límite Novedades Monetarias por Monto | Numérico | Novedades | Según AFG |

### 10.3 Reglas transversales sobre contadores

- Todos los límites deben contar con un **contador** que contabilice la cantidad / monto de transacciones a un determinado corte.
- El **reinicio** debe ejecutarse automáticamente de acuerdo con la periodicidad definida (diario, semanal, mensual, etc.).
- Los reversos y anulaciones **restan al contador** correspondiente (regla explícita para `11` Número Utilizaciones POS en anulaciones y para utilizaciones ATM en reverso de retiros).
- Las transacciones denegadas por exceder límites (código `61`) pueden generar cobro por transacción técnicamente exitosa (ver §11.4).
- Los intentos de PIN errado que excedan el límite (código `75`) pueden bloquear la tarjeta; el bloqueo lo notifica Pomelo — **pendiente definir cómo se entera Credibanco** (ver §14).

### 10.4 Operaciones bloqueadas a nivel AFG (no son contadores, son flags)

Aunque no son límites cuantitativos, el motor debe aplicarlos antes de evaluar contadores:

- MCC prohibidos por Pomelo
- Países prohibidos por regulación
- Comercios prohibidos por Pomelo
- Restricción de comercios por nombre
- Bloquear Extracash en Tarjeta Virtual
- Bloquear Compras Tarjeta Presente con Tarjeta Virtual
- Bloquear retiros de efectivo en la madrugada
- Bloquear transacciones contactless sin PIN

---

## 11. Requerimientos conexos al motor transaccional (post-autorización y cierres)

Pertenecen al ciclo de vida de una transacción y están acoplados al motor transaccional.

### 11.1 Aplicación de Novedades Monetarias (Abonos-Crédito y Cargos-Débito) — Archivo / API
- Canal por archivo (GAW → OpenShift, cifrado por la Entidad) o API uno a uno.
- Validar que el AFG exista, asociado al NIT y Entidad correctos, y que el User ID tenga tarjeta en estado válido (Creada, Activa, Bloqueo Temporal).
- Archivos de respuesta **TNM** y **TAV** incrementales, cifrados, según `Manual Técnico Prepago`.
- Nombre del archivo `MXXXddmmcc` (XXX = AFG, dd = día, mm = mes, cc = consecutivo):
  - Validar que la fecha no sea mayor a seis (6) días calendario ni anterior.
  - No reprocesar el mismo nombre de archivo ni el mismo registro de control.
  - Guardar nombres y registros de control; depuración trimestral.
- Afectación de saldos **en línea**.
- Si es Cargo y el saldo no alcanza: descontar lo disponible y guardar el faltante como insumo del archivo **AUSR**.
- Si hay cobro pendiente, ejecutar recobro al identificar abonos.
- Respetar regla `81` Límite Novedades Monetarias por Monto.
- Auditoría y almacenamiento de fecha / detalle de procesamiento.
- Incluir flujos y reglas para multibolsillo y tarjeta amparada.

### 11.2 Ajustes por Reclamaciones
- Reclamaciones recibidas desde Pomelo en cualquier horario; afectan Estado de Tarjeta y de Cuenta.
- Responder según documentación Pomelo (Ajustes | Pomelo API Reference).
- Según naturaleza: Débito o Crédito.
  - Débito: si no hay saldo suficiente, rechazar e informar causa.
- Ajustes efectivos → archivo **ABO**; rechazos con causa → archivo **RAJU**.
- El ajuste se aplica **en línea**.
- Generar registro de movimiento financiero con concepto (`Ajuste Débito` / `Ajuste Crédito`) y detalle, tanto en movimientos como en BD.
- Si la tarjeta está en Bloqueo Temporal, el ajuste igual se aplica.
- Causas de rechazo: Saldo Insuficiente, Tarjeta Bloqueada, Cuenta Bloqueada, Tarjeta No Existe.
- Validar estructura del archivo TAV para confirmar si tiene los campos requeridos.

### 11.3 Cobro de Comisiones
- Solo aplica para tarjetas en estado `Activa`, `Bloqueada Temporal` y `Bloqueada PIN Errado`.
- Si el cobro es no efectivo → pasa a cobro recurrente.
- Reflejar en movimientos del TH (monto, concepto, fecha y hora).
- Incluir en archivo de salida **COMI** con detalle (retiro o consulta cajero propio / otra red / internacional, etc.).
- Recurrentes cobrados → **COMI** con concepto "Proceso de Cobro Satisfactorio".
- Auditoría de fecha / detalle de procesamiento.

### 11.4 Cobro de Transacciones Técnicamente Exitosas
- Mismas tarjetas elegibles que comisiones.
- Si el cobro es no efectivo → cobro recurrente.
- Reflejar en movimientos y en **COMI** con el detalle de la transacción cobrada.
- Recurrentes cobrados → **COMI** con concepto "Proceso de Cobro Satisfactorio".

### 11.5 Cobro de Impuestos
- Si el cobro es no efectivo → cobro recurrente.
- Incluir en **COMI** con detalle de la transacción cobrada.
- GMF **en línea** bajo el modelo actual y **GMF batch** entra como **novedad por archivo de la Entidad** bajo el nuevo modelo (Ley 2277 de 2022, artículo 881-1 Estatuto Tributario). Ver `requerimientos-novedades-monetarias.md` §4.5 y ADR-028.
- Recurrentes cobrados → **COMI** con concepto "Proceso de Cobro Satisfactorio".

**Pendiente:** tratamiento de devolución del 4x1000 cuando una transacción se reversa.

### 11.6 GMF en Línea
- Validar que la tarjeta no esté exenta del cobro.
- Si no está exenta: cobrar en todas las transacciones configuradas en Impuestos.
- Si está exenta: acumular transacciones mensuales y cobrar solo cuando el TH supere el monto permitido por ley (`Valor UVT × Total Unidades UVT`).
- Movimientos GMF registrados en la tabla de movimientos.
- Información reflejada en archivos **COMI** y **AUM**.
- Validar normatividad vigente antes de aplicar la tasa.

---

## 12. Archivos de salida derivados del flujo transaccional (Etapa 5)

Archivos cuyo contenido se genera a partir de transacciones autorizadas:

| Archivo | Tipo | Hora | Contenido |
|---------|------|------|-----------|
| **AUM** (Movimientos) | Batch | 01:00 AM | Todas las transacciones financieras aprobadas del día anterior |
| **Archivo de Presentaciones** | Batch | xx:xx | Transacciones aprobadas del día anterior (incluye BIN y PAN) reportadas por Pomelo |
| **Archivo de Transacciones** | Batch | xx:xx | Todas las transacciones realizadas del día anterior (incluye BIN y PAN) |
| **COMI** (Comisiones) | Batch | 01:00 AM | Comisiones cobradas y no cobradas (las no cobradas se mantienen 3 meses) |
| **CSAT** (Comisiones Satisfactorias) | Batch | 01:00 AM | Comisiones cobradas |
| **CINS** (Comisiones Insatisfactorias) | Batch | 01:00 AM | Comisiones no cobradas (consolidadas por 3 meses) |
| **SALD** (Saldos) | Batch | 00:00 AM | Saldos de tarjetas en estado Activa, Pendiente por Activar y Bloqueo Temporal |
| **AUSR** (Reversos no aplicados) | Batch | 01:00 AM | Débitos parciales y saldos pendientes por abonos errados |

---

## 13. Insumos transversales que consume el motor transaccional

Descritos en Etapa 1, pero entrada obligada para cada autorización:

- **Parametrización de AFG** — datos de identificación (BIN, Entidad, Nombre AFG, Convenio/NIT/Ciudad), custodios, cuota de manejo, comisiones, GMF, tecnología de tarjeta (chip/contactless). Los **límites y operaciones bloqueadas del AFG se describen en §10**.
- **Reglas de Negocio** — los límites numéricos se consolidan en §10. Banderas y reglas no-límite relevantes al motor: `02` Validar Establecimiento, `05` Convenios Especiales / Pago Automático, `07` PIN, `08` CVV, `09` CVV2, `10` Estado de Tarjeta, `13` Fecha de Expiración, `14` Trans. Permitidas x Tarjeta (VP/VNP), `15` Evaluar Cobro Comisión, `19` Evaluar Cobro Impuestos, `20`/`21` Permitir Compras Nacional/Internacional, `22`/`23` Permitir Retiro Nacional/Internacional, `26`/`27` Permitir Consulta Saldo Nacional/Internacional, `28`/`29` Permitir Anulación Nacional/Internacional, `66` Estado de Cuenta, `67` Selección Tipo de Cuenta, `69` Validación PIN en F', Recobros.
- **Plan de Validación Comercio** — inclusión / exclusión de MCC o CU (hasta 11 dígitos, completando con ceros a izquierda); fuente SVBO en tiempo real; validación de la regla "Parámetro Inclusión o Exclusión MCC o CU" al autorizar.
- **Configuración de Comisiones** — tipo de red (Propia / No Propia / Internacional), canal (ATM / POS), tipo de transacción, transacciones exentas y valor de comisión.
- **Configuración de Transacciones Técnicamente Exitosas** — tipos de negación: `51` Fondos Insuficientes, `55` PIN Inválido, `57` Transacción Inválida, `61` Excede Límites, `75` Excede Intentos de PIN Errado.
- **Configuración de Impuestos (GMF)** — tipo de transacción, canal, parámetro de exoneración a nivel AFG y tarjeta.
- **Auditoría transversal de servicios** — `ID_AUDIT`, `SERVICE_NAME`, `METHOD_NAME`, `ENDPOINT`, `MESSAGE_ID`, `REQUEST_PAYLOAD`, `RESPONSE_PAYLOAD`, `STATUS_CODE`, `STATUS_MESSAGE`, `CREATED_AT`, `RESPONSE_TIME_MS`, `TECHNICAL_USER`, `SOURCE_SYSTEM`, `ID TARJETA`, `ID USUARIO`, `ID PROCESO`.
- **Auditoría PCI DSS 10.2** — Identificación de usuario, tipo de evento, fecha y hora, éxito / fallo, origen, identidad / nombre de los datos, componentes del sistema, recursos o servicios afectados.

---

## 14. Pendientes explícitos del documento (relacionados al flujo transaccional)

1. Notificación a Credibanco del bloqueo de tarjeta por PIN errado ejecutado por Pomelo (cómo nos entera Pomelo del bloqueo).
2. Costo de transacciones ATM: ¿lo fija Credibanco o Pomelo? (campos `extra_data.function_code` y `ATM_FEE_INQUIRY`).
3. Contenido del objeto `MERCHANT` y del merchant ID en el reverso de retiro ATM.
4. Cómo envía Pomelo el código de autorización en transacciones ATM.
5. Tratamiento de la devolución del 4x1000 cuando una transacción se reversa.
6. Validación del tipo de producto (ahorros / crédito / corriente) en pre-autorizaciones — ¿qué campo lo trae?, ¿Pomelo lo valida?

---

## Referencias

- `Prepago/Requerimientos Técnicos (12).docx` — fuente primaria.
- `Prepago/PrepagoUnificadoArquitectura_V_1_3.drawio` — diagramas de arquitectura (12 páginas).
- `Prepago/docs/decisiones-arquitectura.md` — decisiones arquitectónicas clave (Authorization Engine, Redis cache, SPs Oracle, Limit Counter Service).
- `Prepago/Flujos Funcionales.xlsx` — flujos funcionales.
