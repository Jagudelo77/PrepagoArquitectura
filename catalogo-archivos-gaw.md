# Catalogo de Archivos GoAnywhere MFT — Plataforma Prepago Credibanco

Catalogo consolidado de todos los archivos que transitan por GoAnywhere MFT (GAW) como pasarela SFTP + PGP de la plataforma prepago. Incluye archivos de entrada (Entidad/Pomelo/Franquicia hacia Credibanco) y de salida (Credibanco hacia Entidad/Realzador).

**Fuentes consolidadas:**
- `Prepago/docs/requerimientos-novedades-monetarias.md` — archivos TNM, TAV, AUSR, ABO, RAJU.
- `Prepago/docs/requerimientos-motor-cobros-diferidos.md` — archivos COMI, CSAT, CINS.
- `Prepago/docs/requerimientos-ciclo-vida-tarjeta-y-portal-th.md` — archivos TNN, TAS, NOVR, TAR, TSRX.
- `Prepago/docs/requerimientos-tecnicos-consolidados.md` — seccion 8 (archivos de salida).
- `Prepago/docs/requerimientos-flujo-transaccion.md` — seccion 12 (archivos derivados del motor txn).
- `Prepago/docs/decisiones-arquitectura.md` — ADR-028 (GMF archivo), ADR-029 (PGP en GAW).
- `Prepago/Intercambio_Archivos_SFTP_PGP.drawio` — flujo SFTP+PGP.
- `Prepago/PrepagoUnificadoArquitectura_V_1_11.drawio` — pestanas "Intercambio de Archivos", "Flujo de novedades monetarias", "Motor de Cobros Diferidos", "Realce", "Reconciliacion End-of-Day".

> Ultima actualizacion: 04/06/2026.
> Estado: **Vigente**.

---

## Tabla de contenido

1. [Rol de GoAnywhere MFT](#1-rol-de-goanywhre-mft)
2. [Archivos de entrada](#2-archivos-de-entrada)
3. [Archivos de salida](#3-archivos-de-salida)
4. [Resumen consolidado](#4-resumen-consolidado)
5. [Llaves PGP custodiadas en GAW](#5-llaves-pgp-custodiadas-en-gaw)
6. [Politicas operativas](#6-politicas-operativas)
7. [Pendientes](#7-pendientes)

---

## 1. Rol de GoAnywhere MFT

GoAnywhere interviene en dos roles simetricos:

| Direccion | Accion de GAW |
|---|---|
| **Entrada** (Entidad/Pomelo/Franquicia hacia Credibanco) | Recibe por SFTP, descifra PGP con la privada de Credibanco, valida estructura basica (longitud, fecha nombre, NIT trailer), deposita archivo plano en NFS de entrada de OpenShift. |
| **Salida** (Credibanco hacia Entidad/Realzador) | Monitorea NFS de salida, cifra PGP con la publica de cada Entidad/Realzador, despacha por SFTP al destino, borra el archivo plano de NFS tras despacho exitoso. |

**Notas tecnicas:**
- GAW NO descifra PAN. Solo opera sobre el envelope PGP del archivo completo.
- El `Output_File_Generator` escribe archivos PLANOS en NFS. GAW se encarga del cifrado PGP de salida (ADR-029).
- NFS de salida debe estar cifrado at-rest (LUKS/dm-crypt), con ACL estricta y TTL corto (ADR-029).
- GAW custodia todas las llaves PGP de Entidades y Realzadores en su keyring interno.

---

## 2. Archivos de entrada

### 2.1 Tabla de archivos de entrada

| # | Archivo | Naming | Origen | Cifrado entrante | Validaciones de GAW | Destino NFS | Consumido por |
|---|---|---|---|---|---|---|---|
| 1 | Novedades No Monetarias | `NXXXddmmcc` | Entidad emisora | PGP (pública Credibanco) | Descifra PGP, valida longitud 600 bytes/reg, fecha nombre +/-6 dias, NIT trailer | `/input/non-monetary/` | `Prepaid_Non_Monetary_Novelty_Processor` |
| 2 | Novedades Monetarias | `MXXXddmmcc` | Entidad emisora | PGP (pública Credibanco) | Descifra PGP, valida longitud 200 bytes/reg, fecha nombre +/-6 dias, NIT trailer | `/input/monetary/` | `Prepaid_Monetary_Novelty_Processor` |
| 3 | GMF Batch (Insumo Aplicacion GMF) | TBD (longitud variable) | Entidad emisora | PGP (pública Credibanco) | Descifra PGP, valida estructura (14 campos, long variable) | `/input/gmf/` | `Prepaid_Monetary_Novelty_Processor` |
| 4 | Plan Validacion Comercio (MCC/CU masivo) | TBD | Entidad emisora | PGP (pública Credibanco) | Descifra PGP, valida max 20k registros | `/input/merchant-plan/` | `Merchant_Validation_Plan_Service` |
| 5 | Archivo base de Realce | Propietario Pomelo | Pomelo | Sin PGP (VPN dedicada) | Valida existencia, no descifra | `/input/embossing/` | `Prepaid_Embossing_Processor` |
| 6 | Archivos de Clearing | TC files (formato franquicia) | Visa / Mastercard | PGP o formato propietario (segun acuerdo) | Descifra segun formato, valida integridad | `/input/clearing/` | `Clearing_File_Ingestion` |

### 2.2 Convencion de nombres de entrada

```
N XXX dd mm cc     (No Monetarias)
M XXX dd mm cc     (Monetarias)
G XXX dd mm cc     (GMF — TBD)
| |   |  |  |
| |   |  |  +-- Consecutivo (00-99)
| |   |  +----- Mes (01-12)
| |   +-------- Dia (01-31)
| +------------ AFG (3 caracteres, codigo de compensacion)
+-------------- Prefijo (N=No Monetaria, M=Monetaria, G=GMF)
```

### 2.3 Validaciones de GAW en entrada

| Validacion | Archivo | Accion si falla |
|---|---|---|
| Descifrado PGP | Todos (excepto Pomelo) | Archivo no se procesa, alerta interna |
| Longitud registro (512 para N, 128 para M) | NXXX, MXXX | Archivo no se deposita, alerta interna |
| Fecha del nombre (+/-6 dias calendario) | NXXX, MXXX, GXXX | Archivo no se deposita, alerta interna (ARCHIVO_FECHA) |
| NIT del trailer vs BD de AFG | NXXX, MXXX | Archivo no se deposita, alerta interna (TRAILER_NIT) |
| Nombre no duplicado | Todos | Archivo no se deposita, alerta (ya procesado) |
| Numero de registros vs trailer | NXXX, MXXX | Archivo no se deposita, alerta (TRAILER_NUM_REG) |

**Nota:** si GAW rechaza un archivo, genera alertamiento interno y el `Prepaid_Monetary_Novelty_Processor` (o equivalente) nunca lo ve. GAW puede configurarse para notificar via `B2B_Mail_Service` a la Entidad.

---

## 3. Archivos de salida

### 3.1 Tabla de archivos de salida

| # | Archivo | Naming | Tipo | Hora OFG | Lleva PAN | Cifrado GAW | Destino |
|---|---|---|---|---|---|---|---|
| 1 | Resumen Novedades No Monetarias | `TNNxxxMMDD` | En linea (incremental) | Por evento | No | PGP publica Entidad | Entidad emisora |
| 2 | Rechazos Novedades No Monetarias | `TASxxxMMDD` | En linea (incremental) | Por evento | No | PGP publica Entidad | Entidad emisora |
| 3 | Resumen Novedades Monetarias | `TNMxxxMMDD` | En linea (incremental) | Por evento | No | PGP publica Entidad | Entidad emisora |
| 4 | Rechazos Novedades Monetarias | `TAVxxxMMDD` | En linea (incremental) | Por evento | No | PGP publica Entidad | Entidad emisora |
| 5 | Bloqueos / Retiros | `NOVRdd.MM` | Batch | 01:00 AM | No | PGP publica Entidad | Entidad emisora |
| 6 | Solicitud Reexpedicion | `TSRXdd.MM` | Batch | 01:00 AM | No | PGP publica Entidad | Entidad emisora |
| 7 | Reversos no aplicados | `AUSRdd.MM` | Batch | 01:00 AM | No | PGP publica Entidad | Entidad emisora |
| 8 | Abonos y Debitos efectivos | `FINdd.MM` | Batch | 01:00 AM | No (Card ID) | PGP publica Entidad | Entidad emisora |
| 9 | Ajustes Pomelo (efectivos + rechazos) | `RAJU` | Batch | 01:00 AM | No | PGP publica Entidad | Entidad emisora |
| 10 | Movimientos | `AUMdd.MM` | Batch | 00:30 AM | No (Card ID — nuevo modelo) | PGP publica Entidad | Entidad emisora |
| 11 | Comisiones (cobradas + no cobradas) | `COMIdd.MM` | Batch | 01:15 AM | No (BIN+last4) | PGP publica Entidad | Entidad emisora |
| 12 | Comisiones Satisfactorias | `CSATdd.MM` | Batch | 01:15 AM | No (BIN+last4) | PGP publica Entidad | Entidad emisora |
| 13 | Comisiones Insatisfactorias | `CINSdd.MM` | Batch | 01:15 AM | No (BIN+last4) | PGP publica Entidad | Entidad emisora |
| 14 | Saldos | `SALDdd.MM` | Batch | 00:00 AM | No | PGP publica Entidad | Entidad emisora |
| 15 | Creacion de Tarjetas | `TARdd.MM` | Batch | 01:00 AM | Si | PGP publica Entidad | Entidad emisora |
| 16 | Presentaciones (txn con BIN+PAN) | TBD | Batch | TBD | Si | PGP publica Entidad | Entidad emisora |
| 17 | Transacciones (todas con BIN+PAN) | TBD | Batch | TBD | Si | PGP publica Entidad | Entidad emisora |
| 18 | Indicadores Base | `indicadores_baseddmmaaaa.xlsx` | Batch | TBD (mensual) | No | PGP publica Entidad | Entidad emisora |
| 19 | Archivo Realce por AFG | `consecutivo+AFG+fecha+realzador` | Batch | Tras IF Pomelo 09:00 COL | No | PGP publica Realzador | Realzador (embozadora) |
| 20 | Reporte de Pedido por Entidad | (adjunto email) | Batch | Post-realce | No | No aplica (email) | Entidad (via B2B_Mail_Service) |
| 21 | TEX — Indicador Exencion GMF | `TEXdd.mm` | Batch | Diaria | No | PGP publica Entidad | Entidad emisora |
| 22 | Reporte GMF (a TransUnion) | TBD | Batch | TBD | No | PGP publica Entidad | Entidad emisora |
| 23 | Procesados GMF (resumen) | TBD | Batch | Post-procesamiento GMF | No | PGP publica Entidad | Entidad emisora |
| 24 | Errados GMF (detalle errores) | TBD | Batch | Post-procesamiento GMF | No | PGP publica Entidad | Entidad emisora |
| 25 | Novedad GMF (respuesta procesamiento) | TBD | Batch | Post-procesamiento GMF | No | PGP publica Entidad | Entidad emisora |

### 3.2 Archivos que llevan PAN completo (scope CDE del OFG)

**CORRECCION (basada en xlsx "ARCHIVOS SALIDA NUEVO AUTORIZADOR"):** en el nuevo modelo, **solo TAR** (Creacion de Tarjetas) lleva PAN (Numero de Tarjeta 19 digitos). AUM, ABO/FIN, COMI y SALD usan Card ID (40 chars) en lugar de PAN. Esto reduce drasticamente el scope CDE del OFG y elimina el bottleneck del cache de DEK para archivos masivos.

| Archivo | Campos PAN | Volumen estimado diario | Cache DEK requerido |
|---|---|---|---|
| **TAR** | BIN + PAN completo (19 dig) + Card ID | 0 - 50k (solo dias de creacion) | Si (bajo volumen) |
| Presentaciones | TBD (pendiente confirmar con Pomelo) | 1.5M - 7.5M | TBD |
| Transacciones | TBD (pendiente confirmar con Pomelo) | 1.5M - 7.5M | TBD |

Los demas archivos (AUM, FIN, COMI, SALD, TNN, TAS, TNM, TAV, NOVR, AUSR, RAJU, TSRX, TEX, GMF) **NO llevan PAN** — usan Card ID.

| Archivo | Campos PAN | Volumen estimado diario |
|---|---|---|
| AUM | BIN + PAN completo + auth_code + monto + comercio | 1.5M - 7.5M registros |
| ABO | BIN + PAN completo + monto | 500k - 1M registros |
| TAR | BIN + PAN completo + fecha emision | 0 - 50k (dias de creacion) |
| Presentaciones | BIN + PAN completo + monto + auth_code | 1.5M - 7.5M registros |
| Transacciones | BIN + PAN completo + monto + all fields | 1.5M - 7.5M registros |

Los demas archivos llevan **BIN + last_four** (PAN enmascarado, PCI Req 3.3) o no llevan PAN en absoluto.

### 3.3 Archivos eliminados del Manual Tecnico Prepago

No se generan:

| Archivo | Razon |
|---|---|
| TRX (reexpedicion) | No hay reexpedicion en Fase 1 |
| CANJ (canje) | No hay canje y compensacion |
| EXT (extractos) | No se requiere |
| MXXXXXXMMDD (movimiento autorizador) | Reemplazado por AUM |
| AUMCT (consulta costo transaccion) | No se requiere |

### 3.4 Schedule escalonado del OFG (ADR-029)

```
00:00  SALD (corte de saldo — no lleva PAN)
00:30  AUM, Presentaciones, Transacciones (llevan PAN — mas lentos por cache DEK)
01:00  ABO, NOVR, AUSR, RAJU, TSRX, TAR (ABO y TAR llevan PAN; los demas no)
01:15  COMI, CSAT, CINS (no llevan PAN)
```

GAW monitorea NFS continuamente. Tan pronto el OFG termina un archivo, GAW lo cifra y despacha.

---

## 4. Resumen consolidado

| Direccion | Cantidad tipos | Con PGP en GAW | Sin PGP |
|---|---|---|---|
| **Entrada** | 6 | 5 (descifra) | 1 (Pomelo VPN) |
| **Salida** | 25 | 24 (cifra) | 1 (email, no GAW) |
| **Total** | **31** | **29** | **2** |

### Por flujo / pestana

| Pestana del unificado | Archivos entrada (GAW descifra) | Archivos salida (GAW cifra) |
|---|---|---|
| Flujo Novedades No Monetarias | NXXX | TNN, TAS, NOVR |
| Flujo de Novedades Monetarias | MXXX, GXXX (TBD) | TNM, TAV, AUSR, ABO, RAJU |
| Motor de Cobros Diferidos | — | COMI, CSAT, CINS |
| Flujo de Transacciones | — | AUM, Presentaciones, Transacciones |
| Creacion de Tarjetas | — | TAR |
| Realce | Archivo base Pomelo | Archivo Realce por AFG |
| Reconciliacion End-of-Day | Clearing (Visa/MC) | — |
| Portal TH | — | TSRX |
| Reporteria | — | SALD, Indicadores Base |

---

## 5. Llaves PGP custodiadas en GAW

| Llave | Tipo | Uso | Rotacion |
|---|---|---|---|
| Credibanco | Privada (RSA-4096) | Descifrar archivos entrantes de todas las Entidades | Anual o cuando una Entidad rota su publica |
| Entidad X (una por cada) | Publica | Cifrar archivos salientes hacia esa Entidad | Cuando la Entidad notifica rotacion |
| Realzador (embozadora) | Publica | Cifrar archivo de realce por AFG | Cuando el realzador notifica rotacion |
| Franquicia Visa | Publica / Privada (segun acuerdo) | Descifrar clearing entrante | Segun calendario de Visa |
| Franquicia Mastercard | Publica / Privada (segun acuerdo) | Descifrar clearing entrante | Segun calendario de MC |

**Custodia:**
- Todas las llaves privadas de GAW estan protegidas por passphrase + control de acceso al appliance.
- Las llaves publicas de las Entidades se importan al keyring de GAW tras verificacion de fingerprint (ceremonia coordinada por Seguridad de la Informacion).
- GAW mantiene un log de cada operacion criptografica (cifrado/descifrado) con timestamp, archivo, Entidad y resultado.

---

## 6. Politicas operativas

### 6.1 Reintentos SFTP

| Escenario | Politica |
|---|---|
| SFTP push falla (timeout, conexion rechazada) | 3 reintentos, 10 min entre cada uno |
| Si persiste tras 3 reintentos | Alerta a operaciones + alerta Dynatrace |
| Archivo queda en NFS de salida | Se acumula hasta que SFTP vuelva (no se pierde) |

### 6.2 TTL del archivo plano en NFS de salida

- GAW borra el archivo plano **inmediatamente** tras cifrar y despachar exitosamente.
- Si el despacho falla y queda pendiente, el archivo permanece en NFS hasta completar.
- Politica de limpieza diaria: si un archivo plano tiene > 24 horas sin despachar, alerta a operaciones.

### 6.3 Archivos en linea (incrementales)

Para TNN, TAS, TNM, TAV:
- El OFG escribe y cierra un archivo parcial cada N minutos (configurable, default 5 min) o al final de cada chunk procesado.
- GAW monitorea con polling (cada 1 min) o con file notification (inotify si NFS lo soporta).
- GAW cifra y despacha cada parcial como un append logico. La Entidad recibe multiples despachos del mismo dia.
- Al rolar la fecha (00:00), el OFG cierra el archivo del dia. GAW hace el push final.

### 6.4 Un archivo por Entidad

Los archivos de salida se generan **uno por Entidad** (codigo de compensacion XXX en el naming). GAW rutea al SFTP correcto segun el XXX del nombre del archivo.

### 6.5 Auditoria

GAW registra por cada archivo procesado:
- Nombre del archivo
- Direccion (entrada/salida)
- Timestamp de inicio y fin de operacion
- Resultado (OK / ERROR + causa)
- Entidad destino/origen
- Tamano del archivo (bytes)
- Hash SHA-256 del archivo plano (antes de cifrar para salida; despues de descifrar para entrada)

---

## 7. Pendientes

| # | Pendiente | Responsable |
|---|---|---|
| 1 | Nombre canonico del archivo GMF (`GXXXddmmcc` u otro) | Entidad + Producto |
| 2 | Estructura y validaciones del archivo GMF en GAW | Entidad + Producto + Desarrollo |
| 3 | Nombre de archivos Presentaciones y Transacciones | Producto + Pomelo |
| 4 | Horario exacto de Presentaciones y Transacciones | Operaciones + Pomelo |
| 5 | Estructura del archivo de Indicadores Base | Producto |
| 6 | Formato exacto del clearing Visa y Mastercard para configurar GAW | Franquicias + Operaciones |
| 7 | Confirmacion de si COMI lleva PAN completo o solo BIN+last4 | Manual Tecnico Prepago + Producto |
| 8 | Politica de retencion de logs de GAW (hot/archive) | Seguridad + Compliance |
| 9 | Confirmacion de si Pomelo envia por VPN sin PGP o requiere PGP adicional | Pomelo + Seguridad |
| 10 | Definir mecanismo de notificacion a la Entidad cuando GAW rechaza un archivo de entrada | Operaciones + Desarrollo |

---

## Referencias

- `Prepago/Intercambio_Archivos_SFTP_PGP.drawio` — diagrama de flujo SFTP+PGP (recepcion y envio).
- `Prepago/PrepagoUnificadoArquitectura_V_1_11.drawio` — pestanas "Intercambio de Archivos" y "Flujo de novedades monetarias".
- `Prepago/docs/requerimientos-novedades-monetarias.md` — seccion 6 (estructura MXXX), seccion 10 (archivos de salida), seccion 13.1 (cifrado).
- `Prepago/docs/requerimientos-motor-cobros-diferidos.md` — seccion 9 (archivos COMI/CSAT/CINS).
- `Prepago/docs/decisiones-arquitectura.md` — ADR-028 (GMF archivo), ADR-029 (PGP en GAW + NFS de salida).

---

> **Nota de versionado:** este documento se mantiene como catalogo unico de archivos que transitan por GAW. Si se agrega un nuevo archivo, registrar aqui con su naming, tipo, hora, cifrado y destino.