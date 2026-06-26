# Requerimientos — Generación de Archivos de Salida

Documento del requerimiento funcional del proceso de **generación de archivos de salida** de la plataforma prepago: los archivos que Credibanco produce y entrega a las Entidades emisoras, al Realzador y a terceros (TransUnion). Cubre el "qué" (qué archivos, cuándo, para quién, con qué reglas y criterios de aceptación). El "cómo" técnico está en `lineamientos-output-file-generator.md`.

**Fuentes:**
- `Prepago/docs/catalogo-archivos-gaw.md` — catálogo de archivos de salida (naming, hora, PAN, destino).
- `Prepago/docs/estructuras-archivos-manual-tecnico.md` — estructura campo-por-campo de cada archivo.
- `Prepago/docs/lineamientos-output-file-generator.md` — lineamiento técnico de implementación (Spring Batch).
- `Prepago/Requerimientos Técnicos (15).docx` — requerimientos de archivos de salida.
- `Prepago/docs/decisiones-arquitectura.md` — ADR-026 (writer/publisher), ADR-029 (PGP en GAW), ADR-032 (lineamientos OFG).
- `Prepago/PrepagoUnificadoArquitectura_V_1_20.drawio` — pestaña "Generacion Archivos de Salida".

> Fase: **Fase 1 — Línea Base (MVP)**, Etapa 5 (Procesos Batch y Archivos).
> Última actualización: 24/06/2026.
> Estado: **Vigente**.

---

## Tabla de contenido

1. [Resumen y alcance](#1-resumen-y-alcance)
2. [Componente responsable](#2-componente-responsable)
3. [Catálogo funcional de archivos de salida](#3-catálogo-funcional-de-archivos-de-salida)
4. [Modos de generación](#4-modos-de-generación)
5. [Reglas de negocio](#5-reglas-de-negocio)
6. [Manejo del PAN y scope PCI](#6-manejo-del-pan-y-scope-pci)
7. [Programación (schedule batch)](#7-programación-schedule-batch)
8. [Entrega, cifrado y notificación](#8-entrega-cifrado-y-notificación)
9. [Tolerancia a fallos y reproceso](#9-tolerancia-a-fallos-y-reproceso)
10. [Criterios de aceptación](#10-criterios-de-aceptación)
11. [Cobertura en la arquitectura](#11-cobertura-en-la-arquitectura)
12. [Pendientes](#12-pendientes)
13. [Referencias cruzadas](#13-referencias-cruzadas)

---

## 1. Resumen y alcance

### 1.1 Objetivo

Generar los archivos de salida que la plataforma debe entregar a las Entidades emisoras (y a Realzador / TransUnion), con la información de novedades procesadas, movimientos, comisiones, saldos, creación de tarjetas, GMF y realce, de forma confiable, dentro de la ventana batch y cumpliendo los requisitos PCI sobre el PAN.

### 1.2 Alcance funcional

**Incluye:**
- Generación de los archivos de salida del catálogo (§3), tanto **en línea (incrementales)** como **batch diarios** y **batch mensuales**.
- Escritura de archivos **planos posicionales** (la mayoría) y **`.xlsx`** (Indicadores Base).
- Un archivo por Entidad (código de compensación en el naming).
- Entrega de los archivos planos en NFS para que GoAnywhere (GAW) los cifre (PGP) y despache por SFTP.
- Registro de trazabilidad de cada archivo generado.

**Excluye:**
- El **cifrado PGP** y el despacho SFTP (lo realiza GAW — ADR-029).
- La **recepción / validación** de archivos de entrada (cubierto por los flujos de novedades y por el catálogo GAW).
- Los reportes de **facturación a Entidades** (Req 30), que tienen su propio documento (`requerimientos-facturacion-entidades.md`) y componente (`Billing_Batch`).

### 1.3 Resultado esperado

- Todos los archivos de salida del día quedan generados dentro de la ventana batch (00:00 – 06:00 AM) y disponibles para GAW.
- Cada archivo cumple su estructura posicional (o `.xlsx`) definida en `estructuras-archivos-manual-tecnico.md`.
- Solo TAR (y los pendientes Presentaciones/Transacciones) contiene PAN; el resto usa Card ID o BIN+last4.

---

## 2. Componente responsable

El componente responsable es el **`Output_File_Generator` (OFG)**, "publisher único": el único componente autorizado a escribir archivos en disco (patrón writer/publisher, ADR-026).

- Los servicios de negocio (novedades, motor de cobros, etc.) **persisten en BD**; no escriben archivos.
- El OFG lee de la BD (preferentemente Read Replica) y **escribe el archivo plano en NFS**.
- GAW toma el plano, lo cifra (PGP) y lo despacha por SFTP (ADR-029).

> Para los reportes de facturación, el `Billing_Batch` calcula y delega la escritura al OFG (ver `lineamientos-billing-batch.md`). El OFG es el writer común.

---

## 3. Catálogo funcional de archivos de salida

Lista funcional (estructura campo-por-campo en `estructuras-archivos-manual-tecnico.md`; naming y destino en `catalogo-archivos-gaw.md`).

| # | Archivo | Naming | Modo | Lleva PAN | Destino | Flujo origen |
|---|---|---|---|---|---|---|
| 1 | Resumen Novedades No Monetarias | `TNNxxxMMDD` | En línea | No | Entidad | Novedades No Monetarias |
| 2 | Rechazos Novedades No Monetarias | `TASxxxMMDD` | En línea | No | Entidad | Novedades No Monetarias |
| 3 | Resumen Novedades Monetarias | `TNMxxxMMDD` | En línea | No | Entidad | Novedades Monetarias |
| 4 | Rechazos Novedades Monetarias | `TAVxxxMMDD` | En línea | No | Entidad | Novedades Monetarias |
| 5 | Bloqueos / Retiros | `NOVRdd.MM` | Batch 01:00 | No | Entidad | Novedades No Monetarias |
| 6 | Solicitud Reexpedición | `TSRXdd.MM` | Batch 01:00 | No | Entidad | Portal TH |
| 7 | Reversos no aplicados | `AUSRdd.MM` | Batch 01:00 | No | Entidad | Motor txn |
| 8 | Abonos y Débitos efectivos (Financieras) | `FINdd.MM` | Batch 01:00 | No (Card ID) | Entidad | Novedades Monetarias |
| 9 | Ajustes Pomelo (rechazos) | `RAJUdd.MM` | Batch 01:00 | No | Entidad | Novedades Monetarias |
| 10 | Movimientos Autorizador | `AUMdd.MM` | Batch 00:30 | No (Card ID) | Entidad | Motor txn |
| 11 | Comisiones (cobradas + no cobradas) | `COMIdd.MM` | Batch 01:15 | No (Card ID) | Entidad | Cobros Diferidos |
| 12 | Comisiones Satisfactorias | `CSATdd.MM` | Batch 01:15 | No | Entidad | Cobros Diferidos |
| 13 | Comisiones Insatisfactorias | `CINSdd.MM` | Batch 01:15 | No | Entidad | Cobros Diferidos |
| 14 | Saldos | `SALDdd.MM` | Batch 00:00 | No | Entidad | Reportería |
| 15 | Creación de Tarjetas | `TARdd.MM` | Batch 01:00 | **Sí (PAN 19 díg.)** | Entidad | Creación de Tarjetas |
| 16 | Presentaciones | TBD | Batch TBD | **Sí** | Entidad | Motor txn |
| 17 | Transacciones | TBD | Batch TBD | **Sí** | Entidad | Motor txn |
| 18 | Indicadores Base | `indicadores_baseDDMMAAAA.xlsx` | Batch mensual | No | Entidad | Reportería |
| 19 | Archivo Realce por AFG | `consecutivo+AFG+fecha+realzador` | Batch (post IF Pomelo 09:00 COL) | No | Realzador | Realce |
| 20 | Reporte de Pedido por Entidad | adjunto email | Batch (post-realce) | No | Entidad (B2B_Mail) | Realce |
| 21 | TEX — Indicador Exención GMF | `TEXdd.mm` | Batch diario | No | Entidad | Novedades / GMF |
| 22 | Reporte GMF (a TransUnion) | TBD | Batch TBD | No | TransUnion | GMF |
| 23 | Procesados GMF (resumen) | TBD | Batch post-GMF | No | Entidad | GMF |
| 24 | Errados GMF (detalle) | TBD | Batch post-GMF | No | Entidad | GMF |
| 25 | Novedad GMF (respuesta) | TBD | Batch post-GMF | No | Entidad | GMF |

> **Archivos eliminados del modelo anterior** (no se generan): TRX (reexpedición), CANJ (canje), EXT (extractos), MXXXXXXMMDD (movimiento autorizador antiguo, reemplazado por AUM), AUMCT (consulta costo transacción).

---

## 4. Modos de generación

| Modo | Archivos | Disparador | Comportamiento |
|---|---|---|---|
| **En línea (incremental)** | TNN, TAS, TNM, TAV | A medida que se procesan los archivos de entrada del día | El OFG escribe/cierra un parcial cada N minutos (default 5) o al cerrar chunk; GAW despacha cada parcial. Al rolar la fecha (00:00) se cierra el archivo del día. |
| **Batch diario** | NOVR, TSRX, AUSR, FIN, RAJU, AUM, COMI, CSAT, CINS, SALD, TAR, TEX | CronJob en la ventana 00:00–06:00 según schedule (§7) | Lee BD (cursor), genera plano completo, escribe NFS, GAW cifra y despacha. |
| **Batch mensual** | Indicadores Base | CronJob mes vencido | Genera `.xlsx`, filtrable por AFG. |
| **Batch por evento** | Archivo Realce por AFG, Reporte de Pedido, archivos GMF | Tras evento (IF Pomelo, procesamiento GMF) | Generación disparada por la culminación del proceso correspondiente. |

---

## 5. Reglas de negocio

1. **Un archivo por Entidad.** El naming incluye el código de compensación (XXX/AFG); GAW rutea al SFTP correcto.
2. **Card ID en lugar de PAN** (nuevo modelo): AUM, FIN, COMI, SALD, NOVR, AUSR, RAJU, TSRX, TEX usan Card ID (40 chars) o BIN+last4, no PAN completo.
3. **Solo TAR lleva PAN completo** (19 dígitos) entre los archivos de volumen conocido. Presentaciones y Transacciones llevarían PAN (pendiente confirmar con Pomelo).
4. **SALD** reporta tarjetas en estados Activa (A), Embozada (E), Bloqueo Temporal (T); corte de saldo a las 00:00.
5. **NOVR** solo reporta deshabilitaciones (bloqueo definitivo / Retirada).
6. **GMF**: la Entidad envía el insumo; Credibanco aplica según lo indicado y genera los archivos de respuesta (Procesados / Errados / Novedad GMF) y el reporte a TransUnion (ADR-028). Credibanco no recalcula UVT.
7. **Registro de control / trailer**: los archivos planos llevan registro de control con total de registros, fecha/hora de disposición y nombre del archivo (estructura por archivo en el manual técnico).
8. **Indicadores Base**: formato `.xlsx`, mes vencido, filtrable por AFG; la columna "tarjetas inactivas" se renombró a "tarjetas bloqueadas" (Req. Técnicos 13).

---

## 6. Manejo del PAN y scope PCI

- **Solo TAR** (creación de tarjetas, volumen 0–50k/día) requiere **descifrado de PAN** en el OFG, vía `Envelope_Encryption_Service` (sin HSM — ADR-031). Presentaciones/Transacciones llevarían PAN (TBD Pomelo).
- El resto de archivos masivos (AUM, FIN, COMI, SALD) usan **Card ID**, por lo que **no hay descifrado criptográfico masivo por registro** — esto eliminó el bottleneck histórico del cache de DEK (ADR-029 sigue siendo buena práctica, pero ya no es crítico para la ventana).
- Archivos con PAN enmascarado usan **BIN + last4** (PCI Req 3.3).
- GAW **no descifra PAN**; solo cifra/descifra el envelope PGP del archivo completo.
- NFS de salida cifrado at-rest, ACL estricta, TTL corto; GAW borra el plano tras despacho exitoso (ADR-029).

---

## 7. Programación (schedule batch)

Schedule escalonado (ADR-029), ventana 00:00 – 06:00 AM:

```
00:00  SALD                                   (corte de saldo — sin PAN)
00:30  AUM, Presentaciones, Transacciones      (Pres./Txn con PAN — TBD Pomelo)
01:00  FIN, NOVR, AUSR, RAJU, TSRX, TAR         (TAR con PAN; resto sin PAN)
01:15  COMI, CSAT, CINS                         (sin PAN)
```

- **En línea** (TNN/TAS/TNM/TAV): durante el día, a medida que se procesan los archivos de entrada.
- **Mensual** (Indicadores Base): mes vencido.
- **Por evento**: Realce (tras IF Pomelo 09:00 COL), GMF (tras procesamiento).
- GAW monitorea NFS continuamente; tan pronto el OFG termina un archivo, GAW lo cifra y despacha.

---

## 8. Entrega, cifrado y notificación

| Aspecto | Definición |
|---|---|
| Escritura | OFG escribe plano en NFS `/output/...` (o `.xlsx` para Indicadores Base) |
| Cifrado | GAW cifra con la PGP pública de la Entidad/Realzador (ADR-029) |
| Despacho | GAW push SFTP al destino; borra el plano tras éxito |
| Reintentos SFTP | 3 reintentos, 10 min entre cada uno; si persiste, alerta a operaciones + Dynatrace |
| Notificación | Reporte de Pedido por Entidad vía `B2B_Mail_Service`; alertas de error a operaciones |

---

## 9. Tolerancia a fallos y reproceso

- **Idempotencia**: registrar cada archivo generado en la tabla `output_file` (file_name, file_type, afg/entity, file_date, record_count, status). Permite detectar duplicados y reprocesar.
- **Reproceso**: si un archivo falla, debe poder regenerarse para el periodo/fecha sin duplicar (status `REGENERADO`).
- **Skip/retry** en el batch: tolerar registros corruptos hasta un límite, reintentar fallos transitorios de BD (ver `lineamientos-output-file-generator.md` §9).
- **Ventana**: si un archivo no termina dentro del SLA, alertar (umbral > 80% de la ventana).

---

## 10. Criterios de aceptación

1. Cada archivo de salida del catálogo (§3) se genera con su naming, estructura y modo correctos.
2. Los archivos batch diarios quedan disponibles para GAW dentro de la ventana 00:00–06:00.
3. Se genera **un archivo por Entidad** (ruteo correcto por código de compensación).
4. **Solo TAR** (y Pres./Txn cuando se confirmen) contiene PAN; el resto usa Card ID/BIN+last4 — verificable por inspección.
5. Cada archivo lleva su registro de control/trailer con totales correctos.
6. El OFG escribe **planos** (no cifra); GAW realiza el PGP — verificable en el flujo.
7. Cada generación queda registrada en `output_file` con estado y conteo.
8. Indicadores Base se genera mensualmente en `.xlsx`, filtrable por AFG, con la columna "tarjetas bloqueadas".
9. Ante fallo, el archivo se puede regenerar sin duplicar entregas.
10. Los archivos GMF de respuesta se generan tras el procesamiento del insumo de la Entidad.

---

## 11. Cobertura en la arquitectura

| Elemento | Componente / artefacto | Estado |
|---|---|---|
| Componente generador | `Output_File_Generator` (Spring Batch, publisher único) | ✅ Diagramado (pestaña "Generacion Archivos de Salida") |
| Escritura plano + NFS | NFS de salida | ✅ Definido |
| Cifrado PGP + SFTP | GoAnywhere MFT | ✅ Definido (ADR-029) |
| Descifrado PAN (solo TAR) | `Envelope_Encryption_Service` (sin HSM) | ✅ Definido (ADR-031) |
| Trazabilidad | tabla `output_file` | ✅ En esquema |
| Lineamiento técnico | `lineamientos-output-file-generator.md` | ✅ Existe (ADR-032) |
| Estructura campo-por-campo | `estructuras-archivos-manual-tecnico.md` | ✅ Existe |
| Catálogo (naming/hora/destino) | `catalogo-archivos-gaw.md` | ✅ Existe |
| Requerimiento funcional (el "qué") | **este documento** | ✅ Creado |

---

## 12. Pendientes

| # | Pendiente | Responsable |
|---|---|---|
| 1 | Nombre y horario de Presentaciones y Transacciones, y si llevan PAN | Producto + Pomelo |
| 2 | Naming y estructura definitivos de los archivos GMF (Reporte/Procesados/Errados/Novedad) | Entidad + Producto |
| 3 | Estructura final del archivo de Indicadores Base | Producto |
| 4 | Confirmar si COMI lleva PAN completo o solo Card ID/BIN+last4 | Producto + Manual Técnico |
| 5 | Naming canónico del archivo de Realce por AFG | Producto + Realzador |
| 6 | Política de retención de los registros en `output_file` | Compliance |

---

## 13. Referencias cruzadas

### 13.1 Documentos

- `catalogo-archivos-gaw.md` — catálogo de archivos (naming, hora, PAN, destino).
- `estructuras-archivos-manual-tecnico.md` — estructura campo-por-campo.
- `lineamientos-output-file-generator.md` — cómo implementar el OFG (Spring Batch).
- `requerimientos-facturacion-entidades.md` — reportes de facturación (Billing_Batch, distinto de este flujo).
- `decisiones-arquitectura.md` — ADR-026, ADR-029, ADR-031, ADR-032.

### 13.2 Diagrama

- `PrepagoUnificadoArquitectura_V_1_20.drawio` — pestaña "Generacion Archivos de Salida" y "Intercambio de Archivos".

### 13.3 Datos fuente

- Tablas: `output_file`, `batch_job`, `transaction`, `card`, `account`, `commission`, `file_processing`.

---

> **Nota:** este documento cubre el **"qué"** de la generación de archivos de salida. El **"cómo"** técnico (cursor reader, streaming writer, partición por AFG, tuning, anti-patrones) está en `lineamientos-output-file-generator.md`. La estructura exacta de cada archivo está en `estructuras-archivos-manual-tecnico.md`.
