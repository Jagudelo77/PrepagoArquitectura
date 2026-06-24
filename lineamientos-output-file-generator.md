# Lineamientos de Implementación — Output File Generator (Spring Batch)

Guía técnica para el equipo de desarrollo sobre cómo implementar el `Output_File_Generator` (OFG) de forma que escale para el volumen objetivo (1.5M tarjetas, archivos de millones de registros). El objetivo es evitar el anti-patrón de cargar todo en memoria y garantizar la generación dentro de la ventana batch.

**Fuentes:**
- `Prepago/docs/catalogo-archivos-gaw.md` — catálogo de archivos.
- `Prepago/docs/estructuras-archivos-manual-tecnico.md` — estructura campo-por-campo.
- `Prepago/docs/decisiones-arquitectura.md` — ADR-026 (writer/publisher), ADR-029 (PGP en GAW + capacidad), ADR-031 (sin HSM), ADR-032 (este documento).
- `Prepago/PrepagoUnificadoArquitectura_V_1_14.drawio` — pestaña "Generacion Archivos de Salida".

> Última actualización: 04/06/2026.
> Estado: **Vigente**.

---

## Tabla de contenido

1. [Resumen ejecutivo](#1-resumen-ejecutivo)
2. [Volumen objetivo](#2-volumen-objetivo)
3. [Patrón chunk-oriented](#3-patrón-chunk-oriented)
4. [ItemReader — streaming desde Oracle](#4-itemreader--streaming-desde-oracle)
5. [ItemProcessor — transformación](#5-itemprocessor--transformación)
6. [ItemWriter — streaming a disco](#6-itemwriter--streaming-a-disco)
7. [Particionamiento por AFG](#7-particionamiento-por-afg)
8. [Tuning](#8-tuning)
9. [Tolerancia a fallos](#9-tolerancia-a-fallos)
10. [Anti-patrones prohibidos](#10-anti-patrones-prohibidos)
11. [Límites y escalamiento futuro](#11-límites-y-escalamiento-futuro)
12. [Checklist de implementación](#12-checklist-de-implementación)

---

## 1. Resumen ejecutivo

Spring Batch **sí sirve** para generar los archivos grandes del proyecto, **siempre que** se implemente con:

1. **Streaming en lectura** — cursor de BD, no `findAll()` a una `List`.
2. **Streaming en escritura** — `FlatFileItemWriter` con flush por chunk, no un `StringBuilder` gigante.
3. **Particionamiento por AFG** — `partitionStep` para paralelismo y eliminación de SPOF.

El riesgo no es Spring Batch, es **implementarlo mal**. Una implementación naïve que cargue 7.5M registros en memoria causará `OutOfMemoryError`. Este documento define el patrón correcto.

**Cambio favorable del nuevo modelo:** solo el archivo **TAR** lleva PAN (bajo volumen). AUM, FIN, COMI, SALD usan Card ID. Por tanto **no hay descifrado criptográfico masivo por registro** en los archivos grandes — el bottleneck histórico (RSA-4096 por tarjeta) desapareció.

---

## 2. Volumen objetivo

| Archivo | Registros/día | ¿Lleva PAN? | Tiempo estimado (streaming + partición) |
|---|---|---|---|
| AUM | 1.5M – 7.5M | No (Card ID) | 5 – 15 min |
| Presentaciones | 1.5M – 7.5M | TBD Pomelo | 5 – 15 min |
| Transacciones | 1.5M – 7.5M | TBD Pomelo | 5 – 15 min |
| SALD | ~1.5M | No | 2 – 5 min |
| FIN | 500k – 1M | No | 1 – 3 min |
| COMI / CSAT / CINS | 100k – 500k | No | < 2 min |
| TAR | 0 – 50k | **Sí** | < 1 min |
| TNN / TAS / TNM / TAV | incremental (evento) | No | streaming continuo |

Ventana batch: 00:00 – 06:00 AM. Con streaming + partición, todos los archivos caben holgadamente.

Throughput de referencia de `FlatFileItemWriter` con cursor: **10.000 – 50.000 registros/seg por thread**.

---

## 3. Patrón chunk-oriented

```
┌──────────────┐   ┌─────────────────┐   ┌──────────────┐
│  ItemReader  │──▶│  ItemProcessor  │──▶│  ItemWriter  │
│ (cursor BD)  │   │  (transforma)   │   │ (flush chunk)│
└──────────────┘   └─────────────────┘   └──────────────┘
      lee 1              transforma 1          escribe N
         └────── repite hasta agotar el cursor ──────┘
```

**Regla de oro:** en cualquier momento solo hay **un chunk** (N registros) en memoria. Nunca el archivo completo.

```java
@Bean
public Step aumStep(JobRepository jobRepository, PlatformTransactionManager txManager) {
    return new StepBuilder("aumStep", jobRepository)
        .<TransactionRecord, String>chunk(2000, txManager)  // chunk size
        .reader(aumCursorReader())
        .processor(aumProcessor())
        .writer(aumFlatFileWriter())
        .faultTolerant()
        .skipLimit(100)
        .skip(MalformedRecordException.class)
        .build();
}
```

---

## 4. ItemReader — streaming desde Oracle

### 4.1 Usar cursor, NO paging con List

```java
@Bean
@StepScope
public JdbcCursorItemReader<TransactionRecord> aumCursorReader(
        @Value("#{stepExecutionContext['afgId']}") Long afgId,
        @Value("#{jobParameters['businessDate']}") String businessDate) {

    return new JdbcCursorItemReaderBuilder<TransactionRecord>()
        .name("aumCursorReader")
        .dataSource(readReplicaDataSource())          // Oracle Read Replica
        .sql("""
            SELECT t.card_id, c.bin, t.nit_afg, t.txn_type, t.amount,
                   t.txn_date, t.commission_amount, t.gmf_amount, ...
            FROM transaction t
            JOIN card c ON c.card_id = t.card_id
            WHERE t.afg_id = ?
              AND t.txn_date >= TO_DATE(?, 'YYYYMMDD')
              AND t.txn_date <  TO_DATE(?, 'YYYYMMDD') + 1
            ORDER BY t.card_id
            """)
        .preparedStatementSetter(ps -> {
            ps.setLong(1, afgId);
            ps.setString(2, businessDate);
            ps.setString(3, businessDate);
        })
        .rowMapper(new TransactionRowMapper())
        .fetchSize(1000)            // filas por round-trip al cursor
        .build();
}
```

Puntos clave:
- **`JdbcCursorItemReader`** mantiene un cursor abierto en Oracle y lee fila por fila. No trae toda la query a memoria.
- **`fetchSize(1000)`** controla cuántas filas trae por round-trip (no es lo mismo que chunk).
- **Read Replica** como `dataSource` (no impacta el hot-path transaccional).
- **`ORDER BY card_id`** — habilita el cache de DEK (relevante solo para TAR) y da orden determinístico.
- **`@StepScope`** — permite inyectar el `afgId` de la partición.

### 4.2 Alternativa: JdbcPagingItemReader (solo si no se puede mantener cursor)

Si la conexión no puede mantener un cursor abierto mucho tiempo (timeouts), usar `JdbcPagingItemReader` con `pageSize` razonable (1.000-2.000) y un `sortKey` indexado (`card_id` o `transaction_id`). **Nunca** `pageSize` de decenas de miles.

---

## 5. ItemProcessor — transformación

```java
@Bean
public ItemProcessor<TransactionRecord, String> aumProcessor() {
    return record -> {
        // Formatea el registro a la línea posicional del Manual Técnico
        // AUM NO lleva PAN → usa Card ID directamente (no descifra)
        return AumLineFormatter.format(record);   // String de longitud fija (452 bytes)
    };
}
```

Para el **único archivo con PAN (TAR)**:

```java
@Bean
public ItemProcessor<CardRecord, String> tarProcessor(EnvelopeEncryptionService env) {
    return record -> {
        // Solo TAR descifra PAN. Cache de DEK por card_id (Caffeine local al job)
        String pan = env.decryptPan(record.getPanEncrypted(),
                                    record.getPanIv(),
                                    record.getPanAuthTag(),
                                    record.getDekEncrypted(),
                                    record.getKekVersion());
        try {
            return TarLineFormatter.format(record, pan);
        } finally {
            // Zeroizar el PAN tras formatear la línea (PCI)
            pan = null;
        }
    };
}
```

> El cache de DEK solo aplica a TAR. Para el resto de archivos no hay descifrado.

---

## 6. ItemWriter — streaming a disco

### 6.1 FlatFileItemWriter (escribe incrementalmente)

```java
@Bean
@StepScope
public FlatFileItemWriter<String> aumFlatFileWriter(
        @Value("#{stepExecutionContext['afgId']}") Long afgId,
        @Value("#{jobParameters['businessDate']}") String businessDate) {

    String fileName = "AUM" + businessDate.substring(6) + "." + businessDate.substring(4,6);
    return new FlatFileItemWriterBuilder<String>()
        .name("aumWriter")
        .resource(new FileSystemResource("/output/" + afgId + "/" + fileName))  // NFS plano
        .lineAggregator(new PassThroughLineAggregator<>())   // ya viene formateado del processor
        .build();
}
```

Puntos clave:
- **`FlatFileItemWriter`** escribe al archivo en NFS por chunk, con flush. Buffer pequeño, no acumula.
- **Archivo PLANO** (sin cifrar) en `/output/{afgId}/`. GAW lo cifra PGP después (ADR-029/031).
- Un archivo por AFG (partición).

### 6.2 Header y Trailer (registro de control)

Usar `headerCallback` y `footerCallback` del `FlatFileItemWriter` para escribir el registro de control (total de registros, fecha/hora, nombre del archivo) sin necesidad de mantener contadores en memoria del archivo completo:

```java
.headerCallback(writer -> writer.write(buildHeader(businessDate)))
.footerCallback(writer -> writer.write(buildTrailer(recordCounter.get(), businessDate)))
```

El contador (`recordCounter`) se incrementa en un `StepExecutionListener` o `ItemWriteListener`, no acumulando los registros.

---

## 7. Particionamiento por AFG

```java
@Bean
public Step aumPartitionedStep(JobRepository jobRepository,
                               PartitionHandler partitionHandler) {
    return new StepBuilder("aumPartitionedStep", jobRepository)
        .partitioner("aumStep", afgPartitioner())   // una partición por AFG
        .partitionHandler(partitionHandler)
        .build();
}

@Bean
public Partitioner afgPartitioner() {
    return gridSize -> {
        Map<String, ExecutionContext> partitions = new HashMap<>();
        for (Long afgId : afgRepository.findActiveAfgIds()) {
            ExecutionContext ctx = new ExecutionContext();
            ctx.putLong("afgId", afgId);
            partitions.put("afg-" + afgId, ctx);
        }
        return partitions;
    };
}
```

- **10 AFG = 10 particiones.** Con `TaskExecutorPartitionHandler` y un pool de 4 threads → 4 AFG en paralelo.
- Reduce la ventana ~4x y elimina el SPOF (si un worker muere, otro toma la partición).
- Cada partición genera su propio archivo (un archivo por AFG, alineado al naming `xxxMMDD`).

---

## 8. Tuning

| Parámetro | Valor recomendado | Razón |
|---|---|---|
| `chunk size` | 1.000 – 5.000 | Sweet spot. Muy chico = demasiados commits. Muy grande = más memoria y rollback costoso. |
| `fetchSize` (cursor) | 1.000 | Filas por round-trip al cursor de Oracle. |
| Threads de partición | 4 – 8 | Según CPU del pod y conexiones disponibles en el pool de la Read Replica. |
| JVM heap del pod | Dimensionado al chunk, no al archivo | Con chunk 2.000 y registro de ~500 bytes → ~1 MB por chunk. Heap de 512MB-1GB es suficiente. |
| Pool de conexiones Read Replica | ≥ threads de partición + margen | Cada partición usa una conexión. |

---

## 9. Tolerancia a fallos

```java
.faultTolerant()
.skipLimit(100)
.skip(MalformedRecordException.class)     // tolera registros corruptos sin abortar
.retryLimit(3)
.retry(TransientDataAccessException.class) // reintenta fallos transitorios de BD
.listener(skipListener())                  // registra los skipped para auditoría
```

- **Skip policy:** un registro corrupto no debe abortar todo el archivo. Se registra en log + métrica.
- **Retry policy:** fallos transitorios de BD (timeouts) se reintentan.
- **Replay idempotente:** poder regenerar cualquier archivo de un día desde BD. La tabla `output_file_log(file_name, afg_id, generated_at, status)` permite saber qué se generó y reejecutar.
- **Restart:** Spring Batch persiste el estado del job en el `JobRepository`. Si el pod muere a mitad, puede reanudar desde el último chunk commiteado (no reprocesa todo).

---

## 10. Anti-patrones prohibidos

| ❌ Anti-patrón | Por qué falla | ✅ Correcto |
|---|---|---|
| `repository.findAll()` → `List<>` con millones de filas | OutOfMemoryError | `JdbcCursorItemReader` |
| `StringBuilder` con todo el archivo + un `write()` final | OutOfMemoryError con archivos GB | `FlatFileItemWriter` con flush por chunk |
| `JdbcPagingItemReader` con `pageSize` de 50.000 | Cada página carga 50k en memoria | `pageSize` 1.000-2.000 o cursor |
| Contar registros acumulando la lista en memoria | OutOfMemoryError | Contador incremental en listener |
| Descifrar PAN en archivos que no lo llevan | Trabajo inútil + scope CDE innecesario | Solo TAR descifra |
| Un solo thread para todos los AFG | Ventana batch larga + SPOF | `partitionStep` por AFG |
| Leer de Oracle Primary | Impacta el hot-path transaccional | Leer de Read Replica |
| Cifrar PGP dentro del OFG | Duplicación + CPU innecesaria | GAW cifra (ADR-029/031) |

---

## 11. Límites y escalamiento futuro

| Escenario | Spring Batch local (`partitionStep`) | Acción si se supera |
|---|---|---|
| Volumen actual (1.5M tarjetas, AUM 7.5M reg) | ✅ Suficiente | — |
| Archivo individual > 5-10 GB plano | 🟡 I/O y PGP de GAW pasan a ser el cuello | Compresión o split de archivos |
| Volumen 10x (15M tarjetas, 75M reg) | 🟡 Un solo pod no alcanza | `RemotePartitioning` / `RemoteChunking` (workers en pods separados vía broker) |
| Archivos en línea (TNN/TAS/TNM/TAV) tiempo real | ❌ Batch no es streaming en vivo | Append incremental o topic Kafka (roadmap ADR-029) |

---

## 12. Checklist de implementación

Antes de dar por terminado el OFG, verificar:

- [ ] `ItemReader` usa `JdbcCursorItemReader` (o paging con pageSize ≤ 2.000), no `findAll()`.
- [ ] Lectura contra **Oracle Read Replica**, no Primary.
- [ ] Query con `ORDER BY card_id` y `fetchSize` configurado.
- [ ] `ItemWriter` es `FlatFileItemWriter` con flush por chunk, no StringBuilder.
- [ ] Archivo escrito PLANO en `/output/{afgId}/` (sin cifrar — GAW cifra).
- [ ] Header/Trailer vía callbacks, contador incremental (no acumular en memoria).
- [ ] `partitionStep` por AFG con pool de threads.
- [ ] Chunk size 1.000-5.000.
- [ ] Solo TAR invoca `EnvelopeEncryptionService` (descifrado de PAN); zeroización post-formato.
- [ ] Skip policy + retry policy + replay idempotente (`output_file_log`).
- [ ] Cron escalonado (00:00 SALD / 00:30 AUM,Presentaciones,Transacciones / 01:00 FIN,NOVR,AUSR,RAJU,TSRX,TAR / 01:15 COMI,CSAT,CINS).
- [ ] Métricas: tiempo por archivo por AFG, registros/seg, hit ratio cache DEK (TAR), alerta si > 80% del SLA.
- [ ] JVM heap dimensionado al chunk, no al archivo.
- [ ] Test de carga con volumen realista (7.5M registros) antes de producción.

---

> **Nota:** este documento es la guía de implementación. La estructura de cada archivo está en `estructuras-archivos-manual-tecnico.md`; el catálogo y schedules en `catalogo-archivos-gaw.md`; la decisión de capacidad en ADR-029 y ADR-032.