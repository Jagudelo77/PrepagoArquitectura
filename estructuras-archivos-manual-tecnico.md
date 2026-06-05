# Estructuras de Archivos — Manual Tecnico Prepago (Nuevo Autorizador)

Documento de referencia con la estructura campo-por-campo de todos los archivos de entrada y salida del nuevo autorizador de procesamiento emisor. Basado en los xlsx fuente entregados por el area de producto.

**Fuentes:**
- `Prepago/ARCHIVOS ENTRADA NUEVO AUTORIZADOR PROCESAMIENTO EMISOR.xlsx`
- `Prepago/ARCHIVOS SALIDA NUEVO AUTORIZADOR PROCESAMIENTO EMISOR.xlsx`
- `Prepago/Requerimientos Tecnicos (12).docx`

> Ultima actualizacion: 04/06/2026.
> Estado: **Vigente** — Nuevo modelo (reemplaza Manual Tecnico Prepago Visa anterior).

---

## Tabla de contenido

1. [Archivos de entrada](#1-archivos-de-entrada)
   - 1.1 Novedad No Monetaria (NXXXddmmcc)
   - 1.2 Novedad Monetaria (MXXXddmmcc)
   - 1.3 Insumo para Aplicacion GMF
2. [Archivos de salida](#2-archivos-de-salida)
   - 2.1 TNN — Resumen Novedades No Monetarias
   - 2.2 TAS — Rechazos Novedades No Monetarias
   - 2.3 TNM — Resumen Novedades Monetarias
   - 2.4 TAV — Rechazos Novedades Monetarias
   - 2.5 RAJU — Rechazos Ajustes
   - 2.6 AUSR — Cargos Parciales / Reversos No Aplicados
   - 2.7 NOVR — Bloqueos Definitivos (Retiros de Tarjeta)
   - 2.8 TAR — Creacion de Tarjetas
   - 2.9 SALD — Saldos
   - 2.10 ABO/FIN — Reporte Financieras (Abonos y Debitos)
   - 2.11 COMI — Comisiones
   - 2.12 AUM — Movimientos Autorizador
   - 2.13 TEX — Indicador Exencion GMF
   - 2.14 Reporte GMF (a TransUnion)
   - 2.15 Procesados GMF (resumen)
   - 2.16 Errados GMF (detalle errores)
   - 2.17 Novedad GMF (respuesta procesamiento)
   - 2.18 Indicadores Base
3. [Archivos eliminados](#3-archivos-eliminados)
4. [Cambios vs modelo anterior](#4-cambios-vs-modelo-anterior)

---

## 1. Archivos de entrada

### 1.1 Novedad No Monetaria — NXXXddmmcc

**Naming:** `NXXXddmmcc` (N=No Monetaria, XXX=AFG, dd=dia, mm=mes, cc=consecutivo)
**Longitud registro detalle:** 600 bytes (posicional, sin separador)
**Tipos de novedad:** C=Creacion, M=Modificacion, R=Retiro (bloqueo definitivo), T=Bloqueo Temporal, D=Desbloqueo

#### Registro de Detalle (600 bytes)

| # | Campo | Descripcion | Pos Ini | Pos Fin | Bytes | Tipo | Obligatorio |
|---|---|---|---|---|---|---|---|
| 1 | NIT AFG | NIT asociado al AFG | 1 | 15 | 15 | N | Todas |
| 2 | ID grupo de afinidad | ID del AFG | 16 | 55 | 40 | X | Todas |
| 3 | Tipo de Identificacion | 0=CC, 1=TI, 2=CE, 3=PPT | 56 | 56 | 1 | N | Creacion (OPC otros) |
| 4 | Numero de Identificacion | Documento del TH | 57 | 71 | 15 | N | Creacion (OPC otros) |
| 5 | User ID | ID del usuario | 72 | 111 | 40 | X | OPC Creacion, Oblig otros |
| 6 | Primer Apellido | | 112 | 126 | 15 | X | Creacion/Modif |
| 7 | Segundo Apellido | | 127 | 141 | 15 | X | Creacion/Modif |
| 8 | Primer Nombre | | 142 | 156 | 15 | X | Creacion/Modif |
| 9 | Segundo Nombre | | 157 | 171 | 15 | X | Creacion/Modif |
| 10 | Nombre para Realce | Nombre que aparece en tarjeta | 172 | 190 | 19 | X | Creacion/Modif |
| 11 | Direccion Residencia | | 191 | 230 | 40 | X | Creacion/Modif |
| 12 | Codigo Zona Postal | | 231 | 235 | 5 | N | Creacion/Modif |
| 13 | Codigo Ciudad Residencia | | 236 | 240 | 5 | N | Creacion/Modif |
| 14 | Telefono de Contacto | | 241 | 255 | 15 | N | Creacion/Modif |
| 15 | Card ID | ID de la tarjeta | 256 | 295 | 40 | X | Oblig para novedad diferente a Creacion |
| 16 | Fecha de Nacimiento | AAAAMMDD | 296 | 303 | 8 | N | Creacion/Modif (PREG PROD) |
| 17 | Tipo de Novedad | C/M/R/T/D | 304 | 306 | 3 | X | Todas |
| 18 | Indicador exencion GMF | S/N | 307 | 307 | 1 | X | Creacion/Modif |
| 19 | Correo Electronico | | 308 | 457 | 150 | X | Creacion/Modif |
| 20 | Tarjeta Nominada | 1=Nominada, 0=Innominada | 458 | 458 | 1 | N | Creacion/Modif |
| 21 | Reservado para uso futuro | Espacios | 459 | 600 | 142 | X | — |

#### Registro de Control (600 bytes)

| # | Campo | Descripcion | Pos Ini | Pos Fin | Bytes | Tipo |
|---|---|---|---|---|---|---|
| 1 | Numero total de registros | Incluye registro de control | 1 | 15 | 15 | N |
| 2 | Fecha y Hora Disposicion | AAAAMMDDhhmmss + espacios | 16 | 35 | 20 | N |
| 3 | Nombre del Archivo | NXXXddmmcc | 36 | 65 | 30 | X |
| 4 | Reservado para uso futuro | Espacios | 66 | 600 | 535 | X |

---

### 1.2 Novedad Monetaria — MXXXddmmcc

**Naming:** `MXXXddmmcc` (M=Monetaria, XXX=AFG, dd=dia, mm=mes, cc=consecutivo)
**Longitud registro detalle:** 200 bytes (posicional, sin separador)
**Tipos de novedad:** 0=Abono (Credito), 1=Cargo (Debito)

#### Registro de Detalle (200 bytes)

| # | Campo | Descripcion | Pos Ini | Pos Fin | Bytes | Tipo | Obligatorio |
|---|---|---|---|---|---|---|---|
| 1 | NIT AFG | NIT asociado al AFG | 1 | 15 | 15 | N | Obligatorio |
| 2 | ID grupo de afinidad | ID del AFG | 16 | 55 | 40 | X | Obligatorio |
| 3 | User ID | ID del usuario | 56 | 95 | 40 | X | Obligatorio |
| 4 | Card ID | ID de la tarjeta | 96 | 135 | 40 | X | Obligatorio |
| 5 | Tipo de Novedad | 0=Abono, 1=Cargo | 136 | 136 | 1 | X | Obligatorio |
| 6 | Monto de la Novedad | 13 enteros + 2 decimales | 137 | 149 | 13 | X | Obligatorio |
| 7 | Reservado para uso futuro | Espacios | 150 | 200 | 51 | X | — |

#### Registro de Control (200 bytes)

| # | Campo | Descripcion | Pos Ini | Pos Fin | Bytes | Tipo |
|---|---|---|---|---|---|---|
| 1 | Fecha y Hora Disposicion | AAAAMMDDhhmmss + espacios | 1 | 20 | 20 | N |
| 2 | Numero total de registros | Incluye registro de control | 21 | 35 | 15 | N |
| 3 | Nombre del Archivo | MXXXddmmcc | 36 | 65 | 30 | X |
| 4 | Reservado para uso futuro | Espacios | 66 | 200 | 135 | X |

---

### 1.3 Insumo para Aplicacion GMF

**Naming:** TBD (por confirmar con la Entidad — posiblemente archivo de longitud variable)
**Formato:** Longitud variable por campo (Long MIN / Long MAX). Separador TBD (confirmar si CSV, pipe-delimited o posicional con max).
**Origen:** Entidad emisora (calculado por la Entidad segun Ley 2277/2022).

#### Registro de Detalle (longitud variable)

| # | Campo | Long MIN | Long MAX | Dec | Tipo | Descripcion |
|---|---|---|---|---|---|---|
| 1 | Tipo de identificacion del Titular | 1 | 2 | 0 | N | 1=CC, 3=CE, 4=TI, 13=PPT |
| 2 | Numero de identificacion del Titular | 1 | 19 | 0 | N | Documento del TH |
| 3 | Digito de verificacion | vacio | 1 | 0 | N | Solo cuando aplica |
| 4 | Producto | 4 | 20 | 0 | N | Numero real del producto (tarjeta) |
| 5 | Tipo de producto | 1 | 1 | 0 | N | 3=Tarjeta prepago |
| 6 | Numero de transaccion | 1 | 100 | 0 | N | Numero de registro en la entidad |
| 7 | Tipo transaccion | 1 | 1 | 0 | N | 1=Debito |
| 8 | Indicador transaccion parcial | 1 | 1 | 0 | N | 0=no parcial, 1=parcial (exencion UVT parcial) |
| 9 | Monto aplicable GMF | 4 | 20 | 2 | N | Valor susceptible a consulta GMF (pesos + centavos) |
| 10 | Monto total transaccion | vacio | 20 | 2 | N | Solo cuando indicador parcial=1 |
| 11 | Indicador del cobro | 1 | 1 | 0 | N | 0=Cobro, 1=No cobro |
| 12 | Base GMF | 14 | 14 | 2 | N | Valor sobre el cual calcular 4x1000 |
| 13 | Valor sugerido a cobrar | 14 | 14 | 2 | N | Base GMF x 4/1000 |
| 14 | Fecha y hora de ejecucion | 14 | 14 | 0 | X | AAAAMMDDhhmmss |

**Nota:** la Entidad envia este archivo y Credibanco aplica el GMF segun lo indicado. Credibanco no recalcula UVT. Ver ADR-028.

---

## 2. Archivos de salida

### 2.1 TNN — Resumen Novedades No Monetarias

**Naming:** `TNNdd.mm` (dd=dia disposicion, mm=mes disposicion)
**Tipo:** En linea, incremental (un registro por cada archivo NXXX procesado en el dia)
**Un archivo por Entidad (codigo compensacion).**

| # | Campo | Descripcion | Bytes | Tipo |
|---|---|---|---|---|
| 1 | Nombre de Archivo | Nombre del NXXX procesado | 15 | X |
| 2 | Regs. Leidos | Cantidad registros reportados por la Entidad | 6 | N |
| 3 | Regs. Procesados | Cantidad aprobados y procesados | 6 | N |
| 4 | Regs. Errados | Cantidad rechazados y no procesados | 6 | N |
| 5 | Fecha y hora proceso | Fecha de procesamiento | 14 | X |
| 6 | Causal de Error | Solo cuando falle por validacion de data | 50 | X |
| 7 | Reservado para uso Futuro | Espacios | 103 | X |

---

### 2.2 TAS — Rechazos Novedades No Monetarias

**Naming:** `TASdd.mm`
**Tipo:** En linea, incremental (un registro por cada registro de NXXX rechazado)

| # | Campo | Descripcion | Bytes | Tipo |
|---|---|---|---|---|
| 1 | NIT AFG | | 15 | N |
| 2 | ID grupo de afinidad | | 40 | X |
| 3 | Tipo de Identificacion | | 1 | N |
| 4 | Numero de Identificacion | | 15 | N |
| 5 | User ID | | 40 | X |
| 6 | Primer Apellido | | 15 | X |
| 7 | Segundo Apellido | | 15 | X |
| 8 | Primer Nombre | | 15 | X |
| 9 | Segundo Nombre | | 15 | X |
| 10 | Nombre para Realce | | 19 | X |
| 11 | Direccion Residencia | | 40 | X |
| 12 | Codigo Zona Postal | | 5 | N |
| 13 | Codigo Ciudad Residencia | | 5 | N |
| 14 | Telefono de Contacto | | 15 | N |
| 15 | Card ID | | 40 | X |
| 16 | Fecha de Nacimiento | | 8 | N |
| 17 | Tipo de Novedad | C/M/R/T/D | 3 | X |
| 18 | Indicador exencion GMF | | 1 | X |
| 19 | Correo Electronico | | 150 | X |
| 20 | Tarjeta Nominada | | 1 | N |
| 21 | Reservado para Uso Futuro | Espacios | 109 | X |
| 22 | Campos de error | Descripcion del campo con error | 34 | X |

---

### 2.3 TNM — Resumen Novedades Monetarias

**Naming:** `TNMdd.mm`
**Tipo:** En linea, incremental (un registro por cada archivo MXXX procesado)

| # | Campo | Descripcion | Bytes | Tipo |
|---|---|---|---|---|
| 1 | Nombre de Archivo | Nombre del MXXX procesado | 15 | X |
| 2 | Regs. Leidos | Cantidad reportados por Entidad | 6 | N |
| 3 | Regs. Procesados | Aprobados | 6 | X |
| 4 | Regs. Errados | Rechazados | 6 | N |
| 5 | Monto leido | Valor total enviado por Entidad | 15 | N |
| 6 | Monto Aplicado | Valor procesado y aplicado | 15 | N |
| 7 | Monto Errado | Valor rechazado | 15 | N |
| 8 | Fecha y hora proceso | | 14 | X |
| 9 | Causal de Error | Solo si falla por validacion | 50 | X |
| 10 | Reservado para uso Futuro | Espacios | 58 | X |

---

### 2.4 TAV — Rechazos Novedades Monetarias

**Naming:** `TAVdd.mm`
**Tipo:** En linea, incremental (un registro por cada registro MXXX rechazado)

| # | Campo | Descripcion | Bytes | Tipo |
|---|---|---|---|---|
| 1 | NIT AFG | | 15 | N |
| 2 | ID grupo de afinidad | | 40 | N |
| 3 | User ID | | 40 | N |
| 4 | Card ID | | 40 | N |
| 5 | Tipo de Novedad | 0=Abono, 1=Cargo | 1 | N |
| 6 | Monto de la Novedad | Valor no aplicado | 15 | N |
| 7 | Filler | Espacios | 15 | X |
| 8 | Campo de Error | Descripcion del campo con error | 34 | X |

---

### 2.5 RAJU — Rechazos Ajustes por Reclamaciones

**Naming:** `RAJUdd.mm`
**Tipo:** Batch diario (01:00 AM)
**Un registro por cada ajuste Pomelo rechazado.**

| # | Campo | Descripcion | Bytes | Tipo |
|---|---|---|---|---|
| 1 | NIT AFG | | 15 | N |
| 2 | ID grupo de afinidad | | 40 | N |
| 3 | User ID | | 40 | N |
| 4 | Card ID | | 40 | N |
| 5 | Tipo de Ajuste | 0=Ajuste credito, 1=Ajuste debito | 1 | N |
| 6 | Monto de la Novedad | Valor no aplicado | 15 | N |
| 7 | Filler | Espacios | 15 | X |
| 8 | Campo de Error | Descripcion del campo con error | 34 | X |

---

### 2.6 AUSR — Cargos Parciales / Reversos No Aplicados

**Naming:** `AUSRdd.mm`
**Tipo:** Batch diario (01:00 AM)

| # | Campo | Descripcion | Bytes | Tipo |
|---|---|---|---|---|
| 1 | BIN | BIN asociado a la tarjeta | 9 | N |
| 2 | NIT AFG | | 15 | N |
| 3 | Valor Solicitado | Valor solicitado para el cargo | 15 | N |
| 4 | Valor Aplicado | Valor que se logro aplicar | 15 | N |
| 5 | Nuevo Saldo | Valor no aplicado (pendiente) | 15 | N |
| 6 | Card ID | | 40 | X |
| 7 | AFG ID | | 40 | N |
| 8 | Tipo de Transaccion | D=Debito, C=Credito | 1 | X |
| 9 | Fecha | Fecha de proceso AAAAMMDD | 8 | N |
| 10 | Hora | Hora de proceso hhmmss | 6 | N |
| 11 | Reservado para uso futuro | Espacios | 36 | X |

---

### 2.7 NOVR — Bloqueos Definitivos (Retiros de Tarjeta)

**Naming:** `NOVRdd.mm`
**Tipo:** Batch diario (01:00 AM)
**Solo reporta deshabilitaciones (bloqueo definitivo).**

| # | Campo | Descripcion | Bytes | Tipo |
|---|---|---|---|---|
| 1 | BIN | | 9 | N |
| 2 | Card ID | | 40 | X |
| 3 | AFG ID | | 40 | X |
| 4 | Estado de la Tarjeta | Solo R (Retirada/Deshabilitada) | 1 | X |
| 5 | Saldo Disponible | Saldo al momento de la deshabilitacion | 12 | N |
| 6 | User ID | | 40 | X |
| 7 | Fecha de Novedad | AAAAMMDD | 8 | X |
| 8 | Hora de Novedad | hhmmss | 6 | N |
| 9 | Canal | PORTAL / API / ARCHIVO | 10 | X |
| 10 | Reservado para uso futuro | Espacios | 34 | X |

---

### 2.8 TAR — Creacion de Tarjetas

**Naming:** `TARdd.mm`
**Tipo:** Batch diario (01:00 AM)
**IMPORTANTE: este archivo SI lleva PAN (Numero de Tarjeta 19 digitos). Requiere descifrado en OFG.**

| # | Campo | Descripcion | Bytes | Tipo |
|---|---|---|---|---|
| 1 | NIT AFG | | 15 | N |
| 2 | BIN | | 9 | N |
| 3 | AFG ID | | 40 | X |
| 4 | Tipo de Identificacion | 0=CC, 1=TI, 2=CE, 3=PPT | 1 | X |
| 5 | Identificacion | Numero documento TH | 15 | N |
| 6 | User ID | | 40 | X |
| 7 | Numero de la Tarjeta | **PAN completo (19 digitos)** | 19 | N |
| 8 | Card ID | | 40 | X |
| 9 | Nombre Realce | Nombre asociado a la tarjeta | 19 | X |
| 10 | Fecha Novedad | AAAAMMDD | 8 | N |
| 11 | Canal | API / ARCHIVO | 10 | X |
| 12 | Reservado para uso futuro | Espacios | 34 | X |

---

### 2.9 SALD — Saldos

**Naming:** `SALDdd.mm`
**Tipo:** Batch diario (00:00 AM — corte de saldo)
**Tarjetas en estados: Activa (A), Embozada (E), Bloqueo Temporal (T).**
**NO lleva PAN.**

| # | Campo | Descripcion | Bytes | Tipo |
|---|---|---|---|---|
| 1 | BIN | | 9 | N |
| 2 | NIT AFG | | 15 | X |
| 3 | Card ID | | 40 | X |
| 4 | Valor Disponible | Saldo al cierre del dia | 15 | N |
| 5 | Estado | A=Activa, E=Embozada, T=Bloqueo Temporal | 1 | N |
| 6 | Descripcion del Estado | TARJETA ACTIVA / EMBOZADA / BLOQUEADA TEMPORAL | 15 | X |
| 7 | User ID | | 40 | X |
| 8 | AFG ID | | 40 | X |
| 9 | Canal | PORTAL / API / ARCHIVO | 10 | X |
| 10 | Reservado para uso futuro | Espacios | 15 | X |

---

### 2.10 ABO/FIN — Reporte Financieras (Abonos y Debitos efectivos)

**Naming:** `FINdd.mm` (antes se llamaba ABO — el xlsx actualizado lo renombra como "REP FINANCIERAS")
**Tipo:** Batch diario (01:00 AM)
**NO lleva PAN — usa Card ID.**

| # | Campo | Descripcion | Bytes | Tipo |
|---|---|---|---|---|
| 1 | BIN | | 9 | N |
| 2 | NIT AFG | | 15 | N |
| 3 | Card ID | | 40 | N |
| 4 | AFG ID | | 40 | X |
| 5 | User ID | | 40 | N |
| 6 | Transaccion | ABONO / CARGO / AJ DEBITO / AJ CREDITO | 10 | X |
| 7 | Valor Transaccion | | 15 | N |
| 8 | Fecha de Novedad financiera | AAAAMMDD | 8 | N |
| 9 | Canal | AJUSTE / API / ARCHIVO | 10 | X |
| 10 | Reservado para uso futuro | Espacios | 63 | X |

---

### 2.11 COMI — Comisiones

**Naming:** `COMIdd.mm`
**Tipo:** Batch diario (01:15 AM)
**NO lleva PAN — usa Card ID.**

| # | Campo | Descripcion | Bytes | Tipo |
|---|---|---|---|---|
| 1 | NIT AFG | | 15 | N |
| 2 | AFG ID | | 40 | X |
| 3 | Medio | ATM / Cuota de Manejo / Trx 4x1000 / Tecnicamente Exitosa / Cobro Recurrente | 30 | X |
| 4 | Card ID | | 40 | X |
| 5 | Valor de transaccion | Valor de la txn original | 15 | N |
| 6 | Fecha de Transaccion | Del autorizador | 8 | X |
| 7 | Valor a cobrar | Comision/impuesto/cuota/tech exitosa/recobro descontado | 15 | N |
| 8 | Respuesta Autorizador sobre trx | Codigo respuesta (para tech exitosas) | 2 | N |
| 9 | Descripcion de Respuesta | Texto del codigo de respuesta | 30 | X |
| 10 | Respuesta cobro comision | 00=COBRO EXITOSO, 01=COMISION NO COBRADA, etc. | 30 | X |
| 11 | Fecha cobro comision | Del autorizador de comision | 8 | X |
| 12 | Reservado para uso futuro | Espacios | 67 | X |

---

### 2.12 AUM — Movimientos Autorizador

**Naming:** `AUMdd.mm`
**Tipo:** Batch diario (00:30 AM)
**IMPORTANTE: en el nuevo modelo usa Card ID (40 bytes), NO PAN directo.**

| # | Campo | Descripcion | Bytes | Tipo |
|---|---|---|---|---|
| 1 | BIN | | 9 | N |
| 2 | Card ID | **NO es PAN — es el ID opaco de 40 chars** | 40 | X |
| 3 | NIT AFG | | 15 | N |
| 4 | Dispositivo Origen | 01=ATM, 02=POS, 04=COBROS EN BATCH | 2 | X |
| 5 | Desc. Estado Cobro | COBRADA / ERROR TARJETA RETIRADA / etc. | 30 | X |
| 6 | Des. Transaccion | 0=COMPRA, 1=RETIRO, 10=COMISION, 20=CUOTA MANEJO, etc. | 30 | X |
| 7 | Valor Transaccion | | 17 | N |
| 8 | Fecha Transaccion dispositivo | AAAAMMDD | 8 | N |
| 9 | Hora Dispositivo | hhmmss | 6 | N |
| 10 | Valor Comision | | 17 | N |
| 11 | Valor Impuestos | | 17 | N |
| 12 | Indicador Reverso | S/N | 1 | X |
| 13 | Respuesta Autorizador | Codigo 2 digitos | 2 | X |
| 14 | Descrip. Respuesta | Texto | 30 | X |
| 15 | Codigo Autorizacion | Pendiente Pomelo | 6 | X |
| 16 | Tipo de transaccion | 0=Compra, 1=Retiro, 10=Comision, 20=Cuota, etc. | 2 | X |
| 17 | Fecha Autorizador | AAAAMMDD | 8 | N |
| 18 | Hora Autorizador | hhmmssSSS | 9 | N |
| 19 | Nro. Referencia | RRN (Pendiente Pomelo) | 12 | N |
| 20 | Red | (Pendiente Pomelo) | 4 | X |
| 21 | Num. Dispositivo | Terminal | 16 | X |
| 22 | Codigo establecimiento | MCC | 10 | N |
| 23 | AFG ID | | 3 | X |
| 24 | Transaction ID | (Pendiente Pomelo) | 40 | X |
| 25 | Uso futuro | Espacios | 116 | X |

---

### 2.13 TEX — Indicador Exencion GMF

**Naming:** `TEXdd.mm`
**Tipo:** Batch diario
**Tarjetas en estados: Activa, Embozada, Bloqueo Temporal.**

| # | Campo | Descripcion | Bytes | Tipo |
|---|---|---|---|---|
| 1 | User ID | | 40 | N |
| 2 | Card ID | | 40 | X |
| 3 | NIT AFG | | 40 | N |
| 4 | AFG ID | | 40 | X |
| 5 | Estado tarjeta | A=Activa, E=Embozada, T=Temporal | 1 | X |
| 6 | Identificador GMF | S=Exenta, N=No exenta | 1 | X |
| 7 | Fecha de generacion | AAAAMMDD | 8 | N |
| 8 | Uso futuro | Espacios | 30 | N |

---

### 2.14 Reporte GMF (a TransUnion)

**Naming:** TBD
**Tipo:** Batch (frecuencia TBD — posiblemente diario o per-evento)
**Estructura:** misma del archivo de entrada "Insumo para Aplicacion GMF" con campos adicionales de resultado. Formato longitud variable (16 campos).

Campos clave (ver hoja "REPORTE GMF" del xlsx para detalle completo): tipo doc, identificacion, digito verificacion, producto, tipo producto, numero transaccion, tipo transaccion, indicador parcial, monto aplicable, monto total, descripcion, fecha/hora transaccion, fecha/hora utilizacion, codigo transaccion original, fecha/hora transaccion original.

---

### 2.15 Procesados GMF (resumen)

**Naming:** TBD
**Tipo:** Batch (un registro resumen por ejecucion)

| # | Campo | Descripcion | Tipo |
|---|---|---|---|
| 1 | Registros leidos | Total en archivo insumo | N |
| 2 | Registros procesados | Aplicado GMF | N |
| 3 | Registros errados | No procesados (estructura) | N |
| 4 | Monto leido | Suma base registros validos | N |
| 5 | Monto aplicado | Suma GMF aplicado | N |
| 6 | Monto Errado | Suma base de errados | N |
| 7 | Fecha y hora procesamiento | | N |

---

### 2.16 Errados GMF (detalle errores)

**Naming:** TBD
**Tipo:** Batch (un registro por cada registro errado)
**Estructura:** misma del insumo GMF (campos 1-14) + campos adicionales:

| # | Campo adicional | Descripcion |
|---|---|---|
| 15 | Fecha y hora de procesamiento | Cuando se proceso |
| 16 | Detalle del error | Razon del rechazo |

---

### 2.17 Novedad GMF (respuesta procesamiento)

**Naming:** TBD
**Tipo:** Batch (un registro por cada novedad procesada)

| # | Campo | Descripcion | Tipo |
|---|---|---|---|
| 1 | Fecha de procesamiento | | N |
| 2 | Numero de tarjeta | Card ID o PAN (confirmar) | N |
| 3 | Codigo de Respuesta | 1=exitosa, 0=no procesada | N |
| 4 | Descripcion de Respuesta | "exitoso" o detalle del error | N |
| 5 | Valor cobrado por GMF | Monto GMF aplicado | N |
| 6 | RRN | Codigo de autorizacion del BUS | N |

---

### 2.18 Indicadores Base

**Naming:** `indicadores_baseddmmaaaa.xlsx`
**Tipo:** Batch mensual

| # | Campo | Descripcion |
|---|---|---|
| 1 | PRODUCTO | Nombre del Subtipo/AFG |
| 2 | COMPRAS AUTORIZADAS | Total compras (incluyendo reversadas posteriormente) |
| 3 | RETIROS AUTORIZADOS | Total retiros |
| 4 | CONSULTAS EXITOSAS | Total consultas |
| 5 | AJUSTE DEBITO AUTORIZADAS | Total debitos por ajuste |
| 6 | ABONOS AUTORIZADOS | Total abonos |
| 7 | REVERSOS AUTORIZADOS | Total reversos |
| 8 | ANULACIONES AUTORIZADAS | Total anulaciones (aplica a compras) |
| 9 | TARJETAS ACTIVAS | Total activas |
| 10 | TARJETAS INACTIVAS | Total inactivas |

---

## 3. Archivos eliminados

No se generan en el nuevo modelo:

| Archivo anterior | Razon de eliminacion |
|---|---|
| TRX (reexpedicion) | No se realizan reexpediciones |
| CANJ (canje) | No se realiza canje y compensacion |
| EXT (extractos) | No se requiere |
| MXXXXXXMMDD (movimiento autorizador antiguo) | Reemplazado por AUM nuevo formato |
| AUMCT (consulta costo transaccion) | Pendiente validar con Pomelo |

---

## 4. Cambios vs modelo anterior

| Aspecto | Modelo anterior (Manual Tecnico Prepago Visa) | Nuevo modelo (xlsx) |
|---|---|---|
| Longitud NXXX | 512 bytes | **600 bytes** |
| Longitud MXXX | 128 bytes | **200 bytes** |
| AUM lleva PAN | Si (19 digitos) | **No — usa Card ID (40 chars)** |
| ABO/FIN lleva PAN | Si | **No — usa Card ID** |
| TAR lleva PAN | Si | **Si (unico archivo con PAN 19 digitos)** |
| COMI lleva PAN | Asumido BIN+last4 | **No — usa Card ID** |
| SALD lleva PAN | No | **No — usa Card ID** |
| GMF | No habia archivos dedicados | **5 archivos nuevos**: Insumo GMF (entrada), Reporte GMF, Procesados GMF, Errados GMF, Novedad GMF (salida) |
| TEX (Indicador Exencion) | No existia como archivo | **Nuevo archivo de salida** |
| FIN (Reporte Financieras) | Se llamaba ABO | **Renombrado a FIN** (mismo concepto) |
| Campos User ID / Card ID / AFG ID | Variables, a veces NIT+tarjeta | **Estandarizados a 40 bytes cada uno** |
| Registro control MXXX | NIT + fecha + total + sumatorias | **Simplificado**: fecha/hora + total + nombre archivo + filler |

**Impacto critico en performance del OFG:** dado que AUM, ABO/FIN, COMI y SALD ya NO llevan PAN en el nuevo modelo, **el unico archivo que requiere descifrado de PAN es TAR** (creacion de tarjetas, volumen bajo: 0-50k/dia). Esto elimina completamente el bottleneck de cache de DEK para archivos masivos. El ADR-029 sobre cache de DEK sigue siendo buena practica pero ya no es critico para cumplir la ventana batch.