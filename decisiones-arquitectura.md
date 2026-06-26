# Bitácora de Decisiones Arquitectónicas — Prepago Credibanco

Documento vivo que registra las decisiones de arquitectura tomadas durante el diseño de la plataforma prepago. Cada decisión incluye contexto, opciones evaluadas y justificación.

---

## ADR-001: BFF por portal (Admin API y Cardholder API separados)

**Contexto:** Dos portales distintos (administrativo y tarjetahabiente) con necesidades muy diferentes de seguridad, datos y carga.

**Decisión:** Crear dos backends separados (Backend For Frontend pattern):
- `Admin API` → sirve al portal administrativo (CRUD de AFGs, BINs, config)
- `Cardholder API` → sirve al portal tarjetahabiente (consultas, bloqueos, extracto)

**Razón:**
- Cada frontend tiene necesidades distintas (admin necesita escritura pesada, tarjetahabiente necesita lectura rápida)
- Roles y rate limiting diferentes por portal
- Separar contaminación de lógica entre contextos
- Misma convención que usan Stripe, Adyen, Pomelo

**Alternativas descartadas:**
- API única compartida → acopla lógica de admin y tarjetahabiente
- GraphQL único → overkill para las necesidades actuales

---

## ADR-002: Redis como cache de config AFG con fallback a Oracle

**Contexto:** El Authorization Engine necesita leer la configuración completa del AFG (reglas, comisiones, GMF, límites) por cada transacción. Hacer esto contra Oracle en el hot-path no escala.

**Decisión:** La config de AFG vive en Redis. Si Redis no responde, se hace fallback a Oracle via `SP_FALLBACK_READ`.

**Razón:**
- Redis responde en <1ms vs ~10ms de Oracle
- Reduce carga en Oracle durante picos transaccionales
- Fallback garantiza disponibilidad si Redis cae
- La config cambia poco (invalidation on Admin API write)

**Estructura de keys en Redis:**
```
afg:{id} → config completa (TTL configurable)
card:{token} → card_id, afg_id, status
```

---

## ADR-003: Stored Procedures Oracle para débito atómico

**Contexto:** El débito de una transacción debe sumar monto + comisión + GMF en una sola operación atómica. Hacerlo desde Java con múltiples queries introduce riesgo de inconsistencia.

**Decisión:** El débito atómico se hace en SPs de Oracle:
- `SP_AUTHORIZE_TXN` → débito + INSERT transaction (monto + comisión + GMF)
- `SP_CHARGE_TECH_EXITOSA` → cobro por txn denegadas (51, 55, 57, 61, 75)
- `SP_FALLBACK_READ` → lee config si Redis down

**Razón:**
- Una sola round-trip a Oracle desde el Authorization Engine
- Transacción atómica garantizada por Oracle
- Menos código Java → menos superficie de bug
- Constraint `balance >= 0` se valida dentro del SP

**Alternativas descartadas:**
- Toda la lógica en Java con @Transactional → más round-trips, más complejo
- Event sourcing → overkill para un modelo financiero tradicional

---

## ADR-004: Limit Counter Service como componente separado

**Contexto:** Los contadores de límites (qty retiros, monto compras, PIN errado, utilizaciones) necesitan:
- Alta concurrencia (INCR atómicos)
- Reinicio automático por período
- Baja latencia

**Decisión:** Crear un microservicio `Limit Counter Service` (Spring Boot) que encapsula la gestión de contadores sobre Redis.

**Razón:**
- Aísla la lógica de límites del Authorization Engine
- Permite evolucionar contadores sin tocar el engine
- TTL de Redis = reinicio automático (diario/semanal/mensual)
- Contratos claros: CHECK / INCREMENT / RESET

**Estructura de keys:**
```
limit:{card_id}:retiro_qty:daily → count + TTL 24h
limit:{card_id}:retiro_amt:daily → sum + TTL 24h
limit:{card_id}:compra_qty:daily → count + TTL 24h
limit:{card_id}:compra_amt:daily → sum + TTL 24h
limit:{card_id}:pin_errado → count + TTL configurable
```

---

## ADR-005: Transaction table particionada por mes

**Contexto:** La tabla `transaction` va a tener millones de registros al año. Queries de historial deben ser rápidos.

**Decisión:** Particionar por mes usando Oracle range partitioning sobre `txn_date`.

**Razón:**
- Queries por rango de fechas solo tocan las particiones relevantes
- Archivado/purga por partición (DROP PARTITION es instantáneo)
- Reconciliación diaria solo lee la partición del día
- Mejora índices al estar particionados

---

## ADR-006: Envelope Encryption para PAN (RSA + AES, sin HSM)

**Contexto:** Se necesita almacenar el PAN cifrado en Oracle. El HSM Thales se reserva para PIN interchange. La solicitud del cliente es no usar HSM para PAN.

**Decisión:** Envelope Encryption híbrido:
- AES-256-GCM para cifrar el PAN (DEK única por registro)
- RSA-4096 (KEK) para cifrar la DEK
- KEK privada en KeyStore PKCS12 dentro de pod OpenShift

**Razón:**
- Solo RSA: lento, no escala a millones de PANs
- Solo AES: si la llave se compromete, todo se compromete
- Envelope: cada PAN tiene su propia DEK, solo se protege UNA KEK
- Rotación eficiente: re-cifrar solo DEKs, no PANs

Ver detalle en `Prepago/docs/envelope-encryption.md`

---

## ADR-007: KEK con Split Knowledge (2 custodios)

**Contexto:** PCI DSS Req 3.6.6 exige split knowledge para llaves criptográficas críticas.

**Decisión:** El password del KeyStore PKCS12 se divide en 2 mitades:
- Custodio A (Seguridad) tiene la mitad A
- Custodio B (Infraestructura) tiene la mitad B
- Cada mitad se almacena en un Secret separado de OpenShift
- El pod reconstruye el password completo en memoria al arrancar

**Razón:**
- Cumple PCI DSS Req 3.6.6 (split knowledge) y Req 3.6.7 (dual control)
- Ningún custodio solo puede acceder a la KEK
- RBAC separado por Secret (A solo ve Secret A, B solo ve Secret B)
- Nadie fuera del pod tiene el password completo

**Alternativas descartadas:**
- Password único → viola PCI DSS
- Vault (HashiCorp) → requiere infraestructura adicional que el cliente no tiene
- Shamir's Secret Sharing → se consideró pero es más complejo para la fase inicial. Queda como opción futura.

---

## ADR-008: PIN encryption E2E con SubtleCrypto + HSM Thales

**Contexto:** El PIN nunca debe verse en claro fuera del HSM o del navegador del usuario.

**Decisión:** Flujo E2E:
1. HSM Thales genera par RSA efímero (cmd EI)
2. pubKey viaja al navegador (Angular iframe)
3. SubtleCrypto cifra el PIN con RSA-OAEP
4. ciphertext viaja al backend (proxy opaco, no descifra)
5. HSM descifra con privKey bajo LMK (cmd GI)
6. HSM re-cifra con pubKey de Pomelo (cmd EO)
7. Backend envía ciphertext_POMELO a Pomelo

**Razón:**
- PIN en claro solo existe en el navegador (memoria JS, borrado inmediato) y dentro del HSM (nunca sale)
- Cumple PCI DSS Req 3.2 y Req 4.2.1
- Backend nunca es custodio del PIN (reduce scope CDE)

Ver detalle en `Prepago/pinpad_pci_drawio_6_mejorado.drawio`

---

## ADR-009: Iframe en dominio separado para visualización de datos sensibles

**Contexto:** El portal tarjetahabiente necesita mostrar PAN completo, CVV2 y fecha vencimiento. Si el portal de la entidad emisora renderiza estos datos, entra en scope PCI.

**Decisión:** La visualización se hace en un iframe hosted por Credibanco en un dominio separado (`secure-data.credibanco.com`). El portal padre (`portal.entidad.com`) nunca recibe los datos sensibles.

**Razón:**
- Same-origin policy impide que el portal padre lea el DOM del iframe
- El portal de la entidad queda fuera de scope CDE
- Mismo patrón que Stripe Elements, Adyen hosted fields
- Comunicación portal ↔ iframe solo por `postMessage` con cardId (sin datos sensibles)

Ver detalle en `Prepago/Iframe_Visualizacion_DatosSensibles.drawio`

---

## ADR-010: Activación de PIN para tarjetas innominadas vía Proof of Possession

**Contexto:** Tarjetas innominadas no tienen usuario enrolado. No se puede usar pinpad, ATM, ni portal autenticado. No se puede imprimir QR extra en el carrier.

**Decisión:** Micro-portal público de activación con Proof of Possession:
- URL genérica: `activate.credibanco.com/prepago`
- TH ingresa: BIN + last4 + fecha vencimiento + CVV2
- Max 3 intentos → bloqueo permanente de activación digital
- Si valida → carga el mismo iframe de captura de PIN (reutilización)

**Razón:**
- "Posesión de la tarjeta física" es el factor de autenticación (datos impresos en el plástico)
- BIN + last4 + exp + CVV2 dan entropía suficiente con 3 intentos
- No requiere coordinación con el realzador (no imprime nada extra)
- Reutiliza 100% el flujo de PIN ya diseñado

Ver detalle en `Prepago/Activacion_PIN_Innominada.drawio`

---

## ADR-011: Reconciliación End-of-Day contra clearing

**Contexto:** Se necesita cuadrar las transacciones autorizadas contra el clearing de las franquicias y calcular la posición neta de liquidación.

**Decisión:** Proceso batch diario (05:00 → 08:00) que:
1. Ingresa archivos de clearing vía SFTP
2. Matcher compara txn autorizadas vs clearing (card_token + amount + date + auth_code)
3. Clasifica: MATCHED, UNMATCHED_LOCAL, UNMATCHED_CLEARING, AMOUNT_MISMATCH
4. Exception Handler procesa diferencias
5. Settlement Calculator genera posición neta por entidad

**Razón:**
- Obligación regulatoria y operativa
- Detecta fraude temprano (clearing sin auth)
- Alimenta al equipo de tesorería para liquidación

---

## ADR-012: GMF en línea vs batch (config por entidad)

**Contexto:** El requerimiento permite que cada entidad elija si el GMF se cobra en línea (dentro de la txn) o en batch (proceso diario).

**Decisión:** Flag `gmf_mode` en la tabla `entity`:
- `LINEA`: se calcula y suma al débito atómico en el Authorization Engine
- `BATCH`: se omite en la txn, se calcula en proceso batch diario (GMF Batch Processor)

También existe exoneración por tarjeta (`gmf_exenta` en tabla `card`, posición 440 del archivo de creación).

**Razón:**
- Flexibilidad por entidad (algunas prefieren batch para reducir latencia transaccional)
- Exoneración por tarjeta para casos especiales (tarjetas corporativas, etc.)
- Archivo de salida separado: GMFNOVDD.MM

---

## ADR-013: Cobro de transacciones técnicamente exitosas

**Contexto:** El requerimiento exige cobrar cuando una transacción se deniega por razones "técnicamente exitosas" (la infraestructura funcionó, la negación es de negocio):
- 51 Fondos insuficientes
- 55 PIN inválido
- 57 Transacción inválida
- 61 Excede límites
- 75 Excede intentos PIN errado

**Decisión:** El Authorization Engine, tras denegar una txn con estos códigos, invoca `SP_CHARGE_TECH_EXITOSA` si hay regla activa en el AFG. El cobro es un débito adicional.

**Razón:**
- Genera ingresos por intentos de uso aunque se denieguen
- No incrementa contadores de límite (son txn fallidas)
- Parametrizable por AFG (canal, tipo, valor fijo o según tabla de comisiones)

---

## ADR-014: Recobro de comisiones hasta 3 meses

**Contexto:** Si una comisión no se puede cobrar en línea (saldo insuficiente), debe reintentarse.

**Decisión:** Batch diario `Commission Retry Scheduler`:
- Busca `commission_transaction` con `status=PENDING` y antigüedad < 3 meses
- Agrupa por tarjeta (débito consolidado)
- Si éxito → archivo CSAT (satisfactorias)
- Si > 3 meses sin cobrar → archivo CINS (insatisfactorias) + marca FAILED
- Archivo COMI contiene todas las comisiones del período

**Razón:**
- Regla de negocio: "Recobros - Cobro Recurrente por 3 meses"
- Mismo patrón aplica a cuota de manejo y técnicamente exitosas no cobradas

---

## ADR-015: Envelope Encryption visible en C4 L2 del flujo de Creación de Tarjetas

**Contexto:** La decisión de usar Envelope Encryption (ADR-006) estaba documentada y existía su diagrama de detalle (`Envelope_Encryption_PAN.drawio`), pero el C4 nivel 2 de "Creación de Tarjetas" en `PrepagoUnificadoArquitectura_V_1_3.drawio` seguía mostrando una caja **HSM** para cifrar el PAN (heredado de versiones anteriores). Esto dejaba la arquitectura principal inconsistente con la decisión vigente.

**Decisión:** Reemplazar en la página "Creacion de tarjetas" del diagrama unificado la caja HSM por dos componentes nuevos, dejando el HSM Thales únicamente para PIN interchange:

1. **`Envelope_Encryption_Service`** — componente Spring dentro del pod del orquestador (Java 21, JCE/BouncyCastle). Expone `encryptPan(pan, cardId)` y devuelve `{pan_encrypted, pan_iv, pan_auth_tag, dek_encrypted, kek_version}`.
2. **`KEK_KeyStore`** — PKCS12 montado como OpenShift Secret en `/secrets/kek-keystore.p12`, con RSA-4096 OAEP y passphrase desde env var `KEK_KEYSTORE_PASS` (split knowledge entre custodios A y B, ver ADR-007).

Además se actualizó:
- Paso 6 del orquestador (desglosado en 6a–6d: DEK, AES-256-GCM, RSA-4096 OAEP, zeroización).
- Descripción de Oracle: añadidos `pan_iv`, `pan_auth_tag`, `dek_encrypted`, `kek_version`.
- Nota PCI del diagrama: ahora refleja Req 3.4/3.5 (envelope), Req 3.6.6 (split knowledge) y Req 3.6.7 (dual control).
- Título de la página: `v1.3 — Envelope Encryption PAN + Descarte CVV2` (antes `Cifrado PAN con HSM`).

**Alineación RSA-4096:** el diagrama de detalle `Envelope_Encryption_PAN.drawio` decía RSA-2048 (inconsistente con `project-context.md`, decisión #6). Se actualizó todo el diagrama de detalle a **RSA-4096 OAEP**, incluyendo:
- Subtítulo y descripción del paso 3 del cifrado.
- `dek_encrypted: RAW(512)` (antes `RAW(256)`, por el cambio de tamaño del ciphertext RSA).
- Nota del KeyStore, nota "¿por qué envelope?" y esquema Oracle.
- Se removieron del esquema de referencia los campos `cvv2_*` (el CVV2 no se almacena en creación por PCI Req 3.3.2; la visualización temporal vía iframe es un caso separado documentado en `Iframe_Visualizacion_DatosSensibles.drawio`).

**Razón:**
- El C4 L2 debe reflejar la decisión vigente (ADR-006); tener HSM ahí confundía a lectores nuevos.
- RSA-4096 está alineado con `.kiro/steering/project-context.md` (decisión #6) y con los estándares de llaves críticas para almacenamiento multi-año.
- Mantener coherencia entre diagrama unificado y diagrama de detalle evita que una de las dos fuentes induzca implementación incorrecta.

**Impacto:**
- Diagrama unificado: boundary Credibanco extendido de 1610 a 1720, OpenShift de 1190 a 1300 para acomodar los nuevos componentes.
- Esquema Oracle (modelado): `dek_encrypted` debe ser `RAW(512)` cuando se llegue a implementar.
- Documento `envelope-encryption.md`: pendiente ajustar si usa RSA-2048 (no revisado en este cambio).

---

## ADR-016: Subproceso de Pre-Registro disparado tras creación de tarjeta nominada

**Contexto:** El flujo de pre-registro y onboarding del tarjetahabiente ya estaba diseñado en `Onboarding_Tarjetahabiente.drawio` (UUID, TTL 30 días, email con link, verify cédula, OTP, password, creación de usuario). Sin embargo, el C4 nivel 2 de "Creación de Tarjetas" en el diagrama unificado terminaba en "persist card status=CREADA" y no mostraba el gancho al pre-registro, dejando implícito el disparador entre ambos flujos.

**Decisión:** Añadir como paso 10-11 del `Prepaid_Card_Creation_Orchestrator` un subproceso de pre-registro con las siguientes reglas:

1. **Disparador:** tras persistir la tarjeta con `status=CREADA`, el orquestador verifica si el TH ya tiene usuario. Aplica a tarjetas **nominadas** (campo 37 del archivo trae email del TH). Las **innominadas** se omiten porque no hay email ni TH identificable al momento de crear la tarjeta; su activación va por el flujo público documentado en `Activacion_PIN_Innominada.drawio`.

2. **Criterio "TH registrado":** consulta a la **tabla propia `user` en Oracle** (`SELECT user WHERE client_id=?`), no al SSO. Razón: el modelo de usuarios del portal tarjetahabiente es fuente de verdad propia de Credibanco, y el SSO se sincroniza como consecuencia del registro (no como fuente). Esto evita una dependencia síncrona contra SSO durante la creación en el hot-path de batch.

3. **Granularidad: uno por cliente.** El pre-registro se crea por `client_id`, no por `card_id`. Si el cliente recibe varias tarjetas en la misma carga o en cargas sucesivas, solo existe un `preregistration` PENDING vigente por cliente (operación idempotente — si hay uno PENDING vigente lo reusa). Las tarjetas adicionales quedan asociadas al usuario cuando completa el registro.

4. **Componentes nuevos en el C4 L2:**
   - **`PreRegistration_Service`** (Spring Boot, Java 21): genera `code = UUID v4`, persiste `preregistration(code PK, client_id FK, email, status, expires_at, attempts)` con TTL 30 días y estado `PENDING`. Construye la URL `https://preregistro.credibanco.com/registro/{code}` e invoca al adaptador de notificación.
   - **`Notification_Adapter`** (Spring Boot, Java 21): cliente SMTP con template "Active su tarjeta prepago", remitente `no-reply@credibanco.com`. Audit log por envío. **El email no contiene PAN ni datos sensibles**, solo el link con UUID opaco.
   - **`SMTP Server`** (sistema externo, fuera del boundary Credibanco): servidor corporativo o proveedor a definir.

5. **Actualización Oracle:** se añadieron al cilindro del diagrama las tablas `user(client_id, username, status)` y `preregistration(code PK, client_id FK, email, status, expires_at, attempts)`.

**Razón:**
- Hace explícito en la arquitectura principal el hand-off entre creación y onboarding. Hoy era implícito y los equipos de implementación podían pasarlo por alto.
- Uno por cliente (no por tarjeta) reduce emails redundantes, simplifica la UX del TH y evita múltiples URLs vivas para el mismo usuario.
- Consultar tabla propia `user` en lugar de SSO desacopla el batch de creación de la disponibilidad de SSO y permite responder a la pregunta "¿registrado?" con una única query local.
- El detalle del flujo posterior (verify, OTP, password, SSO) no se duplica en el unificado — se referencia `Onboarding_Tarjetahabiente.drawio` como fuente de verdad.

**Alternativas descartadas:**
- **Uno por tarjeta:** rechazado — múltiples emails al mismo TH si recibe varias tarjetas en la misma carga, cada una con su URL, mala UX.
- **Consultar SSO directamente:** rechazado — acopla el batch al SSO, añade latencia y hace el batch dependiente de un sistema más (el SSO se toca solo al final del onboarding, cuando el TH completa el flujo).
- **Cola asíncrona entre orquestador y PreRegistration_Service:** considerado, diferido a futuro. Por ahora llamada síncrona dentro del mismo pod; si el volumen crece o el SMTP es lento, se introduce un broker (Kafka/RabbitMQ) sin cambiar el contrato.

**Impacto:**
- Tabla `preregistration` ya existe conceptualmente en `Onboarding_Tarjetahabiente.drawio`; aquí se formaliza en el modelo del unificado.
- Campo email del TH: en creación nominada viene del archivo/API (posición del NXXX o campo del request API). Si es nulo/inválido, el servicio registra el pre-registro en estado `PENDING_NO_EMAIL` para que operaciones gestione manualmente (pendiente decisión de manejo de excepciones en negocio).
- Boundaries del diagrama ampliados: Credibanco de 1170 a 1250 de alto, OpenShift de 940 a 1130 de alto, para acomodar los nuevos componentes y el SMTP externo.

---

## Decisiones pendientes de definición

### Multibolsillo y Tarjeta Amparada
El requerimiento menciona que se deben tener flujos y reglas para estos casos, pero no están definidos aún. Decisión: diferir hasta que negocio defina los flujos específicos.

### Fraude
El requerimiento pide que el producto esté apto para herramientas de fraude internas o externas. Decisión: preparar hooks/interfaces en la arquitectura, pero la herramienta específica queda pendiente.

### Reportería / Power BI
Se necesita extracción de data para dashboards. Decisión: usar Oracle Read Replica como fuente, pero el stack específico de reportería (Power BI, Qlik, etc.) queda por definir.

### Notificaciones (SMS/OTP/Email)
Requerimiento menciona notificación para activación y bloqueos. Decisión: pendiente definir proveedor (Twilio, SES, interno) y canales.

## ADR-017: Actualización C4 L2 "Flujo de transacciones" en v1.4 (Plan Validación Comercio + Auditoría + Reversos/Anulaciones + Catálogo de límites)

**Contexto:** El diagrama `PrepagoUnificadoArquitectura_V_1_3.drawio` en la pestaña "Flujo de transacciones" cubría el happy path de autorización (compra POS, retiro ATM, consulta saldo) con comisiones + GMF + técnicamente exitosas + SPs + Redis + Limit Counter, pero al cruzarlo contra `requerimientos-flujo-transaccion.md` (§1-§10) se detectaron cinco gaps relevantes:

1. **Reversos y anulaciones invisibles** — no se explicitaba matching contra txn original, idempotencia, ni DECREMENT del contador.
2. **Catálogo de límites incompleto en el `Limit_Counter_Service`** — faltaban consultas de saldo, trx diarias por cliente (cross-canal), VNP con 3DS, e-commerce, regla 81 (novedades monetarias) y la operación DECREMENT.
3. **Plan de Validación Comercio sin representación** — era un artefacto explícito del requerimiento (MCC/CU incl/excl, fuente SVBO en tiempo real, hasta 11 dígitos, carga API/archivo máx 20k) pero no figuraba como componente.
4. **Auditoría transversal ausente** — el requerimiento pide auditoría de servicios y PCI 10.2 por cada invocación, pero no había componente ni destino visible.
5. **Frontera de validaciones Pomelo vs Credibanco no documentada** — varias reglas (07, 08, 09, 10, 13, 69) figuran como "apagada con Pomelo" pero no había una nota que hiciera esa frontera explícita en el diagrama.

**Decisión:** Crear `PrepagoUnificadoArquitectura_V_1_4.drawio` como nueva versión y actualizar exclusivamente la pestaña "Flujo de transacciones" con:

1. **Título v1.3 → v1.4** y subtítulo ampliado a "Plan Validación Comercio + Auditoría + Reversos/Anulaciones + Catálogo completo de límites".
2. **Authorization Engine** — pipeline ampliado a 10 pasos con paso 5 ahora explícito "Reglas negocio + Plan Validación Comercio(MCC/CU incl/excl)", paso 10 añade "Audit async", y una línea nueva **"Reverso/Anulación"** con match (RRN + auth_code + amount + terminal), idempotencia (flag reversada/anulada), DECREMENT contadores, reuso de auth_code en reverso vs nuevo auth_code en anulación.
3. **`Prepaid_Limit_Counter_Service`** — catálogo expandido a 9 tipos de contadores (compras POS qty/monto, retiros ATM qty/monto, consultas de saldo, trx diarias por cliente cross-canal, compras VNP con 3DS, e-commerce nal/int, PIN errado, utilizaciones regla 11, novedades monetarias regla 81). Operaciones: **CHECK / INCREMENT / DECREMENT / RESET** (antes solo CHECK/INCREMENT/RESET).
4. **Componente nuevo `Merchant_Validation_Plan_Service`** (morado `#7C3AED`, Java 21 + Spring Boot 3) — paso 5 del pipeline. Lista MCC/CU incl/excl por AFG, CU hasta 11 dígitos con padding, carga API/archivo máx 20k, cache Redis con TTL corto, bloqueos AFG (MCC/países prohibidos, contactless sin PIN, retiros madrugada, extracash virtual, compras VP virtual).
5. **Componente externo nuevo `SVBO (SmartVista)`** (morado con borde punteado) — fuente en tiempo real de MCC/CU vigentes. Conectado al Merchant_Validation_Plan_Service con flecha punteada "Sync MCC/CU (tiempo real)".
6. **Componente nuevo `Audit_Service`** (celeste `#0EA5E9`, Java 21 + Spring Boot 3 async) — auditoría transversal + PCI 10.2. Almacena ID_AUDIT, SERVICE_NAME, MESSAGE_ID, REQUEST/RESPONSE_PAYLOAD, STATUS, MS, TECHNICAL_USER, ID_TARJETA, ID_PROCESO. Fire-and-forget desde el Auth Engine con INSERT a Oracle (tablas `audit_log` y `pci_event_log`).
7. **Nota "VALIDACIONES EN POMELO"** (azul claro, arriba a la derecha) — lista explícita de reglas delegadas al pre-autorizador: PIN (07), CVV/CVV2 (08, 09), fecha vencimiento (13), existencia y estado de tarjeta (10), BIN y AFG activos, límite PIN errado, val. PIN en F' (69). Deja claro que Credibanco no repite esas validaciones.
8. **Nota "REVERSO / ANULACIÓN"** (ámbar, arriba a la derecha) — mecánica detallada en 5 pasos (matching, idempotencia, DECREMENT, reuso/nuevo auth_code, INSERT txn).
9. **Boundaries ampliados** — OpenShift de 1000×596.5 a 1270×900, Credibanco de 1100×850 a 1370×1160, para acomodar los 3 componentes nuevos.

**Razón:**
- Pone el C4 L2 al día con los requerimientos consolidados en `requerimientos-flujo-transaccion.md` sin duplicar lógica en cada diagrama de detalle.
- Los tres componentes nuevos (Merchant Validation, Audit, Limit Counter ampliado) eran decisiones implícitas que ya existían en el texto del requerimiento, pero sin materialización arquitectónica se corre el riesgo de que el equipo de implementación los vea como "lógica dentro del Auth Engine" y se pierda la separación de responsabilidades.
- Hacer explícita la frontera Pomelo vs Credibanco evita que el Auth Engine duplique validaciones ya hechas por el pre-autorizador (waste + divergencia de reglas).
- Documentar la mecánica de reversos/anulaciones (matching + idempotencia + DECREMENT) cierra un hueco que el diagrama v1.3 dejaba abierto.

**Alternativas descartadas:**
- Modificar v1.3 en sitio sin crear v1.4 → rechazado, viola la convención del proyecto de no sobreescribir versiones de arquitectura (ver `project-context.md`, sección Convenciones).
- Separar Merchant Validation Plan en una pestaña propia → rechazado, se pierde la visión unificada del pipeline y obliga a saltar entre pestañas para entender la autorización.
- Audit_Service como sidecar por pod en vez de servicio dedicado → considerado, diferido a futuro. Servicio dedicado simplifica el routing hacia Oracle y permite evolucionar (p.ej., fan-out a SIEM) sin tocar cada pod. Queda como opción cuando se defina integración con SIEM corporativo.
- Incluir Fraud Detection como componente → rechazado, sigue como decisión pendiente (ver "Decisiones pendientes de definición" abajo).

**Impacto:**
- Archivo nuevo: `Prepago/PrepagoUnificadoArquitectura_V_1_4.drawio` (13 páginas, igual que v1.3). Solo cambia la pestaña "Flujo de transacciones".
- `project-context.md` debe apuntar a `PrepagoUnificadoArquitectura_V_1_4.drawio` como arquitectura principal vigente.
- Documento `requerimientos-flujo-transaccion.md` queda ahora 100% alineado con el C4 L2 (secciones §1-§10).
- Oracle: el Audit_Service requiere tablas `audit_log` y `pci_event_log` cuando se llegue a implementar (estructuras a definir en modelado de BD).
- Redis: el Limit_Counter_Service añade 6 familias de keys nuevas (consultas saldo, trx diarias cross-canal, VNP 3DS, e-commerce nal/int, regla 11 POS/ATM, regla 81). Estructura de keys a definir en diseño de Redis.

## ADR-018: Conciliación `docs/envelope-encryption.md` con los diagramas v1.4

**Contexto:** Al validar la página "Envelope Encryption PAN" del archivo `Envelope_Encryption_PAN.drawio` contra la pestaña "Creacion de tarjetas" del `PrepagoUnificadoArquitectura_V_1_4.drawio` (ambos coherentes entre sí), se detectó que `Prepago/docs/envelope-encryption.md` estaba desactualizado en cuatro puntos respecto a la verdad establecida en los diagramas:

1. Tipos Oracle como `VARCHAR2 + Base64` en lugar de `RAW`.
2. Ausencia de la columna `pan_auth_tag` (tag de integridad GCM).
3. Nombre de tabla `card_sensitive_data` en lugar de `card`.
4. Inclusión de columnas `encrypted_cvv2`, `encrypted_dek_cvv`, `iv_cvv` que violan PCI DSS Req 3.3.2 (CVV2 no se almacena).

**Decisión:** Actualizar `envelope-encryption.md` para alinearlo con los diagramas vigentes, sin cambiar los diagramas. Cambios aplicados:

- Introducción: añadida nota explícita "CVV2 NO se almacena. Se descarta inmediatamente post-creación (Req 3.3.2). Visualización vía iframe consulta Pomelo en tiempo real."
- Flujo de cifrado: ampliado a 8 pasos con AAD (`card_id + created_at`), separación explícita de `panCiphertext` y `authTag` (16 bytes), cálculo de `pan_hash = SHA-256(PAN)` para búsquedas, zeroización de DEK y PAN. Tipos `RAW` en el listado de columnas Oracle.
- Flujo de descifrado: 8 pasos con concatenación correcta de `ciphertext + authTag` para verificación GCM, manejo de `AEADBadTagException` cuando el tag no coincide, lectura de `pan_iv` y `pan_auth_tag` directamente como `RAW` (sin Base64), selección de KEK por `kek_version` desde un `kekRegistry` (compatibilidad multi-versión).
- Tabla Oracle: reemplazada `card_sensitive_data` (VARCHAR2/Base64, con CVV2) por la tabla `card` real con tipos `RAW(64)/RAW(12)/RAW(16)/RAW(512)/RAW(32)`, agregando `pan_hash`, `last_four`, `bin`, `exp_date`, `card_token`, `client_id` y `afg_id`. Bloque "Lo que NO se almacena" listando explícitamente `cvv2`, `pin` y datos de banda magnética.
- Sección de rotación: ajustada para referirse a `dek_encrypted` (no `encrypted_dek`) y mencionar que `pan_encrypted`, `pan_iv` y `pan_auth_tag` no se tocan durante la rotación (preserva integridad GCM).
- Tabla de cumplimiento PCI: añadida fila para Req 3.3.2 (CVV2 no almacenado).

**Razón:**
- Los diagramas (`Envelope_Encryption_PAN.drawio` y `PrepagoUnificadoArquitectura_V_1_4.drawio`) son la fuente de verdad para el modelo de datos y los algoritmos. El `.md` es documentación de soporte y debe seguirlos.
- Mantener tres fuentes desalineadas (dos drawio + un md) genera riesgo de implementación incorrecta. Era pendiente conocido (anotado en ADR-015 como "Pendiente: ajustar `envelope-encryption.md` si usa RSA-2048" — la conciliación cubre también ese punto).
- El uso de `RAW` en lugar de `VARCHAR2 + Base64` ahorra ~33% de almacenamiento, evita un round-trip de codificación/decodificación y es la práctica estándar para datos binarios en Oracle.

**Alternativas descartadas:**
- Mantener `VARCHAR2 + Base64` en el `.md`: rechazado, no aporta valor y desalinea con los diagramas.
- Mover el documento a `docs/legacy/`: rechazado, el .md es la referencia técnica detallada con código Java, manifiestos OpenShift y procedimiento de ceremonia. Sigue siendo necesario.
- Conservar las columnas de CVV2 con un comentario "no usadas": rechazado, aunque queden vacías representan un riesgo de auditoría PCI (data discovery podría flaggearlas).

**Impacto:**
- Documento `envelope-encryption.md` queda alineado con los diagramas. No requiere cambios en código (aún no implementado).
- Modelado de BD (`On-prem/MER_Prepago_Inicial.drawio`) debe revisarse para confirmar que la tabla `card` use los tipos `RAW` documentados aquí.
- Inventario de archivos: `envelope-encryption.md` sigue marcado como Vigente; el cambio es de contenido, no de estado.

## ADR-019: Modelo Entidad-Relación (MER) + script DDL Oracle como fuente de verdad del modelo de datos

**Estado:** Aceptada. *(Entrada reconstruida — ver nota al final. La decisión es de la misma época que ADR-017/018, alineada con `Requerimientos Técnicos (12).docx` y la arquitectura v1.4.)*

**Contexto:** Las decisiones de arquitectura (ADR-001 a ADR-018) describían componentes y flujos, pero el **modelo de datos** vivía implícito en los diagramas y en notas dispersas. No había una representación formal del esquema relacional ni un script DDL ejecutable. Esto generaba tres problemas: (1) el cifrado del PAN (ADR-006/018) requería tipos de columna concretos (`RAW` para `pan_encrypted`, `pan_iv`, `pan_auth_tag`, `dek_encrypted`) que no estaban materializados en ningún artefacto; (2) las reglas de negocio del AFG (límites, comisiones, GMF, plan de validación de comercio — ADR-002/012/013/014/017) necesitaban tablas concretas para parametrizarse; (3) la tabla `transaction` particionada por mes (ADR-005) no tenía su DDL de particionamiento escrito.

**Decisión:**

1. **Crear el MER** `MER_Prepago_V_1_0.drawio` (a partir del borrador `On-prem/MER_Prepago_Inicial.drawio`) como vista entidad-relación del modelo de datos de Fase 1 MVP, alineado con `Requerimientos Técnicos (12).docx` y la arquitectura unificada v1.4.

2. **Crear el script DDL** `db/01_schema_prepago.sql` como **fuente de verdad** del esquema Oracle 19c+, con las tablas del modelo agrupadas por dominio:
   - **Parametrización / catálogo:** `entity`, `bin`, `affinity_group`, `afg_business_rule` (pivote de reglas por `rule_code`), `commission`, `tech_successful_charge`, `tax_config`, `merchant_validation_plan`, `merchant_validation_item`.
   - **Tarjetahabiente y producto:** `client`, `account`, `card` (con columnas `RAW` de envelope encryption — ADR-006/018), `card_state_history`.
   - **Archivos:** `file_processing`, `file_record`, `output_file`.
   - **Transaccional:** `transaction` (particionada por mes — ADR-005), `pending_charge` (recobros hasta 3 meses — ADR-014), `gmf_accumulator`.
   - **Portal / onboarding:** `app_user`, `preregistration` (ADR-016), `otp`.
   - **Batch y liquidación:** `batch_job`, `output_file`, `settlement` (ADR-011).
   - **Auditoría:** `audit_log`, `pci_event_log` (PCI DSS 10.2).

3. **Establecer el SQL como fuente de verdad** sobre el drawio del MER: ante cualquier discrepancia, manda `db/01_schema_prepago.sql` (los tipos, constraints, índices y particionamiento se definen ahí; el MER es la vista de comunicación).

4. **Crear el documento** `docs/modelo-entidad-relacion.md` como descripción narrativa del modelo.

**Razón:**
- Un script DDL ejecutable es verificable y versionable, a diferencia de un diagrama. Tener una sola fuente de verdad evita que el modelo derive entre diagrama y código.
- Materializar los tipos `RAW` del PAN cierra el gap detectado en ADR-018 (alineación del envelope encryption con el modelo físico).
- La tabla pivote `afg_business_rule` (identificada por `rule_code`) evita columnas dispersas y soporta el catálogo de reglas variable del AFG (reglas 02, 11, 14, 19, 20, ..., 81).

**Alternativas descartadas:**
- **Mantener solo el MER en drawio:** rechazado — un diagrama no es ejecutable ni garantiza tipos/constraints; el cifrado del PAN exige precisión de tipos físicos.
- **Columnas dispersas por cada regla de AFG:** rechazado — el conjunto de reglas es variable y disperso; se optó por la tabla pivote `afg_business_rule`.

**Impacto:**
- Archivos nuevos: `MER_Prepago_V_1_0.drawio`, `db/01_schema_prepago.sql`, `docs/modelo-entidad-relacion.md`.
- Las decisiones posteriores que tocan datos (ADR-023 onboarding, ADR-026/027 cobros, ADR-034 facturación) extienden este esquema; cada una indica las tablas/vistas que añade.
- Pendiente histórico (resuelto después): `db/02_stored_procedures.sql` con los SPs del motor transaccional (ADR-003).

> **Nota de trazabilidad:** esta entrada fue **reconstruida**. El ADR-019 figuraba referenciado en `requerimientos-tecnicos-consolidados.md`, `requerimientos-ciclo-vida-tarjeta-y-portal-th.md` e `inventario-archivos-prepago.md` ("Pestaña MER + SQL + ADR", "ADR-001 a ADR-019"), pero su sección no se había escrito en esta bitácora (saltaba de ADR-018 a ADR-020). Se redactó a partir de esas referencias y del estado real de `MER_Prepago_V_1_0.drawio` y `db/01_schema_prepago.sql` para cerrar el hueco de numeración y eliminar las referencias huérfanas.

## ADR-020: Pestaña "Flujo Novedades No Monetarias" en `PrepagoUnificadoArquitectura_V_1_4.drawio`

**Contexto:** El documento `requerimientos-ciclo-vida-tarjeta-y-portal-th.md` consolidó los requerimientos de Etapa 2 — Bloqueo Definitivo (Deshabilitado), Bloqueo Temporal, Desbloqueo y Modificación de Datos (§4 y §5). La pestaña existente "Intercambio de Archivos" mostraba un único `Prepaid_Non_Monetary_Novelty_Processor` agrupando todas las novedades sin distinguir las cuatro variantes ni sus reglas específicas (estados válidos, manejo de saldo en deshabilitación, lógica de cambio de email, campos no modificables). Esto dejaba implícitas reglas críticas del requerimiento.

**Decisión:** Añadir al archivo unificado v1.4 una pestaña dedicada **"Flujo Novedades No Monetarias"**, ubicada entre "Intercambio de Archivos" y "Realce", manteniendo la misma estética C4 L2 y los mismos componentes de plataforma (SFTP, GoAnywhere, NFS, OpenShift, Pomelo). El contador de páginas pasa de `pages="14"` a `pages="15"`.

**Contenido de la pestaña:**

1. **Mismo borde de plataforma** que "Intercambio de Archivos": Credibanco, On-Prem/DMZ, OpenShift; SFTP Server, GoAnywhere MFT, File Storage NFS.
2. **Dos canales de ingreso:**
   - Batch — `Prepaid_File_Ingestion_Batch` con polling cada 15 min sobre `/input` y validaciones (estructura, fecha ≤ 6 días, no reproceso por `file_name + control_record`).
   - API — `Authorization Gateway` → `Prepaid_Ingestion_API` con endpoints `POST /api/v1/cards/{id}/block`, `POST /api/v1/cards/{id}/unblock` y `PATCH /api/v1/cards/{id}`.
3. **`Prepaid_Non_Monetary_Novelty_Processor`** como core compartido, con 4 sub-handlers diferenciados:
   - **Bloqueo Definitivo (rojo)** — procesa cualquier estado, captura el saldo final como insumo del archivo NOVR.
   - **Bloqueo Temporal (ámbar)** — solo desde Creada o Activa, deja `status=BLOQUEO_TEMPORAL`.
   - **Desbloqueo (verde)** — solo desde Bloqueo Temporal, deja `status=ACTIVA`.
   - **Modificación de Datos (morado)** — campos inmutables (`NIT`, `AFG`, `tipo_doc`, `número_doc`), modificables (`email`, `phone`, `address`, `city`, `nominated`, `gmf_exempt`).
4. **`Pomelo_API_Client`** con `PATCH /cards/{id}/status`, `PATCH /clients/{id}` y `PATCH /cards/{id}` para replicar el cambio en Pomelo.
5. **`PreRegistration_Service`** disparado solo cuando: cambio de email del TH y el TH no está registrado en el portal. Deja la trazabilidad cruzada con `Onboarding_Tarjetahabiente.drawio` (ADR-016).
6. **`Output_File_Generator`** genera tres archivos:
   - **TNN** xxxMMDD — Resumen No Monetarias en línea, incremental (creación, bloqueo definitivo, bloqueo temporal, desbloqueo, modificación).
   - **TAS** xxxMMDD — Rechazos en línea, incremental (causal del rechazo).
   - **NOVR** dd.MM — Bloqueos definitivos del día con saldo, batch 01:00 AM.
7. **`Audit_Service`** con invocación asíncrona desde el Processor, persistiendo en `audit_log` y `pci_event_log`. Auditoría dedicada para exoneración GMF (campo separado).
8. **Cuatro notas explicativas:**
   - Validaciones del Processor (pasos 1-4) con la excepción de Bloqueo Definitivo aceptando cualquier estado.
   - Reglas específicas de Modificación de Datos (campos inmutables, lógica de cambio de email).
   - Detalle del archivo NOVR (incluye saldo, solo bloqueos definitivos y retiros, no Bloqueo Temporal ni Desbloqueo).
   - Pendientes del documento fuente (validaciones GAW, multibolsillo/amparada, restricciones Pomelo, movimientos del portal admin).

**Razón:**
- Hace explícitas las cuatro variantes de novedades no monetarias y sus reglas específicas (estados válidos, manejo de saldo en deshabilitación, lógica de email).
- Reusa los componentes de "Intercambio de Archivos" sin duplicar (SFTP, GAW, NFS, Gateway, Audit, Output File Generator), garantizando coherencia.
- Permite a equipos de desarrollo y operación leer la pestaña sin necesidad de cruzar contra el .docx de requerimientos para entender las diferencias entre los cuatro tipos de novedad.
- Cumple la convención del proyecto de no sobreescribir versiones (v1.4 sigue siendo v1.4; solo se agrega una pestaña).

**Alternativas descartadas:**
- Modificar la pestaña "Intercambio de Archivos" en sitio para agregar los sub-handlers — rechazado, esa pestaña es la vista general de novedades (monetarias + no monetarias) y agregar el detalle la vuelve ilegible.
- Crear una nueva versión `V_1_5` solo por agregar una pestaña — rechazado, no hay cambios en pestañas existentes que justifiquen versión nueva. La adición es aditiva.
- Separar Modificación de Datos en pestaña propia — considerado, diferido. Si la complejidad de modificación crece (multibolsillo, tarjeta amparada), se separa entonces.

**Impacto:**
- Archivo unificado: `pages="14"` → `pages="15"`. Pestaña insertada entre "Intercambio de Archivos" y "Realce".
- No requiere cambios en otras pestañas, ni en SQL, ni en el MER.
- Inventario de archivos actualizado para reflejar el contador 15 páginas en `PrepagoUnificadoArquitectura_V_1_4.drawio`.

## ADR-021: Limpieza de la pestaña "Realce" en `PrepagoUnificadoArquitectura_V_1_4.drawio`

**Contexto:** Al verificar el componente `Prepaid_embossing_Processor` en la pestaña "Realce" del archivo unificado v1.4 se detectaron 7 inconsistencias respecto al requerimiento §2 (Realce) de `requerimientos-ciclo-vida-tarjeta-y-portal-th.md`:

1. La descripción del Processor era una copia literal del `Prepaid_Monetary_Novelty_Processor` ("recargas, débitos, reversos, anulaciones — idempotencia por novelty_id"), nada relacionado con realce.
2. El nombre rompía la convención PascalCase del proyecto (`embossing` en minúscula).
3. El título de la pestaña decía "Container Diagram — Plataforma de Intercambio de Archivos", herencia del duplicado.
4. El subtítulo enumeraba el stack genérico del proyecto, no el alcance del realce.
5. La caja de Pomelo se etiquetaba como "Sistema Externo — Entidad Emisora" cuando Pomelo es el procesador y la fuente del archivo base de realce.
6. El edge Pomelo → SFTP decía "NXXXDDMMCC / MXXXDDMMCC" (archivos de novedades), no "archivo base de realce".
7. Faltaba la flecha del Processor a Oracle Primary representando el cambio de estado a `PENDIENTE_POR_ACTIVAR`, y faltaba la representación del Reporte de Pedido por Entidad. La caja "Embozadora — SFTP" no aclaraba que era un proveedor externo (realzador).

**Decisión:** Aplicar 10 cambios quirúrgicos a la pestaña "Realce" sin tocar otras pestañas:

1. **Título**: "Container Diagram — Plataforma de Intercambio de Archivos" → "Container Diagram — Proceso de Realce de Tarjetas".
2. **Subtítulo**: stack genérico → "C4 Level 2 | Pomelo (archivo base) → Embossing Batch → archivo por AFG → SFTP a Realzadores | Generación diaria 07:00 ARG / 05:00 COL".
3. **Title del batch** (TV-20): HTML basura heredado de un copy/paste → `Prepaid_Embossing_Batch` (PascalCase).
4. **Descripción del batch** (TV-22): "ensambla multiarchivo / cada 15 min" → "Detecta archivo base de realce de Pomelo en NFS, invoca `Prepaid_Embossing_Processor`. Schedule: diario tras 05:00 COL".
5. **Title del processor** (TV-28): minúscula → `Prepaid_Embossing_Processor`.
6. **Descripción del processor** (TV-30): texto de novedades monetarias → 6 pasos del realce: lee archivo base de Pomelo → agrupa por AFG → asigna consecutivo de pedido → genera archivo por AFG (`consecutivo+AFG+fecha+realzador`) → `UPDATE card SET status=PENDIENTE_POR_ACTIVAR` → genera reporte de pedido.
7. **Tablas de Oracle Primary** (TV-41): listado genérico (`monetary_novelty`, `non_monetary_novelty`, `gmf_calculation`) → tablas relevantes al realce (`card`, `card_state_history`, `embossing_order` con consecutivo + AFG + fecha + realzador + cantidad).
8. **Caja Pomelo** (TV-47 descripción): "Entidad Emisora" → "Sistema Externo — Procesador. Dispone archivo base de realce diariamente 07:00 ARG / 05:00 COL".
9. **Edge Pomelo → SFTP** (TV-51): "NXXXDDMMCC / MXXXDDMMCC" → "Archivo base de realce (diario, 05:00 COL)".
10. **Edges del Batch al Processor** (TV-58) y del Processor a Oracle (TV-62): labels actualizados a "Procesa archivo, agrupa por AFG" y "UPDATE card SET status=PENDIENTE_POR_ACTIVAR / INSERT embossing_order".
11. **Caja Embozadora** (g-15/16): "Embozadora — SFTP / Entidad Emisora" → "Realzador (Embozadora) — SFTP / Sistema Externo — Proveedor de embozado. Recibe archivos de realce por AFG (uno por entidad, consecutivo de pedido)". Edge g-17 reetiquetado a "Recibe archivo de realce por AFG (PGP)".
12. **Nota nueva** "Reporte de Pedido por Entidad" en la esquina superior izquierda con: campos del reporte (fecha, consecutivo, AFG, cantidad, entidad, embozador) + dos warnings del requerimiento (archivo fuera de horario y errores de embozado).

**Razón:**
- El Processor estaba documentado como si fuera de novedades monetarias; cualquier desarrollador que lo leyera implementaría algo equivocado.
- Los nombres en minúscula rompían la convención del proyecto y dificultaban la búsqueda.
- El título y subtítulo confundían la pestaña con "Intercambio de Archivos", obligando al lector a leer el contenido para distinguirlas.
- La fuente del archivo (Pomelo como procesador, no entidad emisora) y la dirección del flujo (Pomelo → Credibanco → Realzador, no Entidad → Credibanco) son críticas para el equipo de operaciones.
- El cambio de estado a `PENDIENTE_POR_ACTIVAR` es un side-effect explícito del requerimiento que no estaba representado.

**Alternativas descartadas:**
- Reescribir la pestaña desde cero — rechazado, mantener boundaries y posiciones existentes minimiza el riesgo de romper otras pestañas.
- Crear una pestaña nueva y borrar la existente — rechazado, los cambios son aditivos / correctivos, no de estructura.
- Diferir la corrección del HTML basura en los `value` de TV-20 y TV-28 — rechazado, hace ilegible el diagrama al abrir en draw.io.

**Impacto:**
- Solo cambia la pestaña "Realce" del archivo unificado v1.4. Sigue siendo `pages="15"`.
- No requiere cambios en SQL ni en el MER — la tabla `embossing_order` ya estaba implícita en el modelo y se materializa cuando se implemente el batch.
- Documento `requerimientos-ciclo-vida-tarjeta-y-portal-th.md` queda 100% alineado con la pestaña "Realce" (§2 del documento).

## ADR-022: Distribución como componente en la pestaña "Realce" + integración con `B2B_Mail_Service`

**Contexto:** Tras la limpieza de la pestaña "Realce" (ADR-021), el "Reporte de Pedido por Entidad" había quedado representado solo como una nota textual en la esquina del diagrama. Sin embargo, ese reporte es la materialización del proceso de **Distribución** del requerimiento §3 de `requerimientos-ciclo-vida-tarjeta-y-portal-th.md`: tras el envío del archivo de realce al embozador, Credibanco consolida los datos del AFG (NIT, ciudad, dirección, teléfono, custodios) con los datos del realce (consecutivo, cantidad, embozador) y notifica a la entidad emisora por correo electrónico. El requerimiento exige que esto pase por un servicio B2B de correo corporativo (`B2B_Mail_Service`) y no por el `Notification_Adapter` del flujo de creación de tarjetas (que envía emails al tarjetahabiente, no a la entidad).

**Decisión:** Convertir la nota en componentes reales de C4 L2 sobre la pestaña "Realce", manteniendo la cohesión con el resto de pestañas:

1. **Componente nuevo `Distribution_Service`** (verde `#10B981`, `[Java 21 + Spring Service]`) dentro del boundary OpenShift, ubicado debajo del `Prepaid_Embossing_Processor`. Responsabilidad: construir el Reporte de Pedido consolidando datos del AFG y del realce, idem tiempos de procesamiento del realce.
2. **Componente nuevo `B2B_Mail_Service`** (celeste `#0EA5E9`, `[Cliente SMTP corporativo Credibanco]`) dentro del boundary OpenShift. Responsabilidad: servicio compartido B2B para envío de correos transaccionales a entidades emisoras (no a tarjetahabientes). TLS, plantillas HTML, audit_log, reintento con backoff.
3. **Actor externo nuevo `Entidad — Buzón Email`** fuera del boundary Credibanco. Recibe el correo HTML con el Reporte de Pedido.
4. **4 edges nuevos:**
   - `Embossing_Processor → Distribution_Service` — Trigger post-envío con consecutivo + AFG + realzador.
   - `Distribution_Service → Oracle Read Replica` — Lee AFG (NIT, ciudad, dirección, teléfono, custodios). Línea punteada por ser lectura analítica de réplica.
   - `Distribution_Service → B2B_Mail_Service` — Envía Reporte de Pedido (plantilla HTML).
   - `B2B_Mail_Service → Entidad — Buzón Email` — SMTP / TLS / HTML.
5. **Nota informativa "Reporte de Pedido (Distribución)"** con los 10 campos del reporte (fecha, consecutivo, ID/nombre AFG, cantidad, entidad, embozador, ciudad, dirección, teléfono, custodios 1 y 2) y los warnings (mismos tiempos del realce, datos desde Read Replica).

**Razón:**
- Hace explícito que **Distribución es un proceso aparte del Realce** aunque comparta tiempos y se dispare al final del realce. La nota anterior los mezclaba en un solo bloque.
- `B2B_Mail_Service` queda como componente reutilizable: sirve para Distribución hoy y mañana puede atender cualquier comunicación B2B con entidades emisoras (alertas operativas, confirmaciones de cargas, etc.) sin tener que crear un servicio nuevo para cada caso.
- Distinguir `Notification_Adapter` (para tarjetahabiente, ver pestaña "Creacion de tarjetas") de `B2B_Mail_Service` (para entidades emisoras) evita que el equipo de operaciones mezcle los dos buzones, las dos plantillas y los dos catálogos de destinatarios.
- Leer del `Oracle Read Replica` y no del Primary cumple con el lineamiento de no impactar el hot-path con consultas de reportería (ADR-005 + ADR-002).

**Alternativas descartadas:**
- Reusar el `Notification_Adapter` existente (el del flujo de creación de tarjetas) — rechazado, mezcla destinatarios B2B con destinatarios B2C (tarjetahabientes), comparte plantillas y catálogos que deberían estar separados, y dificulta el control de SLAs y rate limiting diferenciados.
- Hacer que el `Embossing_Processor` envíe el correo directamente — rechazado, viola separación de responsabilidades; el processor debe terminar al cerrar el archivo de realce y delegar la notificación.
- Mantener la nota textual sin componentes — rechazado, esconde una decisión arquitectónica (Distribución como subproceso del realce) y deja el flujo incompleto para los equipos de implementación.

**Impacto:**
- Solo cambia la pestaña "Realce" del archivo unificado v1.4. Sigue siendo `pages="15"`.
- El `B2B_Mail_Service` queda como componente nuevo a implementar; debe coordinarse con seguridad de la información para definir el dominio remitente (`b2b@credibanco.com` o equivalente) y el catálogo inicial de destinatarios por entidad emisora.
- La tabla `embossing_order` (introducida en ADR-021) es la fuente del consecutivo de pedido que dispara la Distribución.
- Este ADR no genera nuevas tablas; toda la información ya existe en `affinity_group`, `card`, `card_state_history`, `embossing_order`. La Distribución es un proceso de lectura + envío.
- Pendiente: definir el catálogo de destinatarios por entidad (¿se toma del `affinity_group.embosser_email`? ¿se requiere un nuevo campo `entity.b2b_email` o `entity.distribution_emails` para múltiples destinatarios?). Por ahora se asume que el campo existe en `affinity_group`; se materializará al implementar `B2B_Mail_Service`.

## ADR-023: v1.5 — Pestaña "Onboarding Tarjetahabiente" en el unificado e integración con `Onboarding_Tarjetahabiente.drawio`

**Contexto:** El flujo de onboarding del tarjetahabiente estaba documentado en `Prepago/Onboarding_Tarjetahabiente.drawio` (swimlanes con 5 fases: creación → email → identidad → OTP → password → SSO), pero en el unificado v1.4 solo aparecía referenciado como caja `PreRegistration_Service` dentro de la pestaña "Creacion de tarjetas" (ADR-016) y como sub-flujo de la pestaña "Flujo Novedades No Monetarias" cuando cambia el email del TH (ADR-020). Faltaba una vista C4 L2 propia que mostrara los componentes del onboarding y sus relaciones, manteniendo el detalle de pasos en el archivo aparte.

**Decisión:** Crear `PrepagoUnificadoArquitectura_V_1_5.drawio` (copia de v1.4) con una pestaña nueva **"Onboarding Tarjetahabiente"** insertada después de "Portal Faseado". El archivo `Onboarding_Tarjetahabiente.drawio` se mantiene intacto como fuente del detalle paso a paso por fases (swimlanes). El conteo del archivo unificado pasa a `pages="16"` (también se reconcilia el header que estaba en `pages="14"` y debió ser 15 desde la inserción de "Flujo Novedades No Monetarias").

**Contenido de la pestaña nueva (C4 Level 2):**

1. **Boundaries:** Credibanco · On-Prem/DMZ (SMTP Server) · OpenShift.
2. **Componentes nuevos formalizados como cajas (no notas):**
   - `PreRegistration_Portal` (Angular 17+ SPA, `preregistro.credibanco.com/registro/{code}`).
   - `PreRegistration_API` (Java 21 + Spring Boot 3 WebFlux) con endpoints `/verify`, `/validate-otp`, `/create-account`.
   - `PreRegistration_Service` (Spring Service — núcleo de negocio con 5 operaciones: `createPreRegistration`, `verifyIdentity`, `issueOtp`, `validateOtp`, `createAccount`).
   - `Notification_Adapter` (cliente SMTP B2C — diferente del `B2B_Mail_Service` de Distribución, ADR-022).
   - `SSO Keycloak` Realm CARDHOLDER.
   - `Audit_Service` (fire-and-forget).
   - `Oracle Primary` con tablas `preregistration`, `otp`, `app_user`, `client`.
3. **Referencias cruzadas** (cajas dashed apuntando a otras pestañas):
   - `Prepaid_Card_Creation_Orchestrator` (vive en "Creacion de tarjetas") — disparador post-creación de tarjeta nominada.
   - `Portal Tarjetahabiente` (vive en pestañas "Portal" / "Portal Faseado") — destino post-onboarding.
4. **Tarjetahabiente** como actor externo (recibe email link, OTP, confirmación; abre URL del portal de pre-registro).
5. **Flecha numerada del flujo** (1-12): create → email → abre URL → verify → OTP → validate → password → create account → SSO → confirmación → login portal.
6. **5 notas explicativas:**
   - Resumen de las 5 fases con remisión a `Onboarding_Tarjetahabiente.drawio` para detalle paso a paso.
   - Tarjetas innominadas — no disparan pre-registro al crearse, se rigen por `Activacion_PIN_Innominada.drawio` (ADR-010).
   - Controles de seguridad: TTLs, intentos máximos, política de password, hash bcrypt/argon2 en SSO, rate limiting, CAPTCHA, audit PCI 10.2.
   - Disparadores del pre-registro: creación nominada · cambio de email del TH no registrado (`Flujo Novedades No Monetarias`, ADR-020) · operaciones puntuales del Portal Admin.
   - Estados de `preregistration` (PENDIENTE / PENDING_NO_EMAIL / COMPLETADO / EXPIRADO / BLOQUEADO) y manejo de errores principales.

**Razón:**
- Mantener consistencia con el patrón establecido en ADR-021 y ADR-022: cada servicio del unificado debe ser una caja con sus inputs/outputs explícitos, no solo una nota textual o referencia.
- Distinguir explícitamente `Notification_Adapter` (B2C / TH) del `B2B_Mail_Service` (B2B / entidades emisoras) evita que ambos usen la misma plantilla, el mismo destinatario o el mismo SLA.
- Conservar `Onboarding_Tarjetahabiente.drawio` como fuente del detalle por fases evita inflar el unificado con swimlanes y errores específicos. La pestaña nueva es la vista de componentes; el archivo aparte es la vista de proceso.
- El número de páginas pasa a 16 — incluye también la corrección del header que estaba en 14 y debió ser 15 desde la inserción de "Flujo Novedades No Monetarias".

**Alternativas descartadas:**
- **Opción A** (copiar swimlanes al unificado): rompe la consistencia C4 L2 del resto de pestañas y duplica contenido.
- **Opción C** (solo notas de referencia cruzada): deja `PreRegistration_Service` sin definir bien y obliga a abrir dos archivos para seguir un flujo end-to-end.
- **Bajar el detalle del pre-registro a la pestaña "Creacion de tarjetas"**: rechazado, mezcla creación de tarjeta con onboarding del usuario en una sola pestaña ya cargada.
- **Crear una versión 2.0**: rechazado, una pestaña nueva no justifica un salto de versión mayor; se incrementa de v1.4 a v1.5 conforme a la convención del proyecto.

**Impacto:**
- Archivo nuevo: `PrepagoUnificadoArquitectura_V_1_5.drawio` (16 páginas).
- `PrepagoUnificadoArquitectura_V_1_4.drawio` se conserva por convención (no sobreescribir versiones).
- `Onboarding_Tarjetahabiente.drawio` queda intacto como fuente de detalle por fases.
- `project-context.md` (steering) debería actualizarse para apuntar a v1.5 como arquitectura principal vigente.
- No hay cambios en el SQL ni en el MER — las tablas `preregistration`, `otp`, `app_user` ya existían en `db/01_schema_prepago.sql`.

## ADR-024: `OTP_Service` como componente reusable + ajustes de alineación con `Onboarding_Tarjetahabiente.drawio`

**Contexto:** En la pestaña "Onboarding Tarjetahabiente" de v1.5 (ADR-023), las operaciones de OTP (`issueOtp`, `validateOtp`) figuraban como métodos dentro del `PreRegistration_Service`. El `PreRegistration_API` recibía `POST /validate-otp` y delegaba al Service. Esta concentración tiene tres problemas:

1. **OTP no es exclusivo del onboarding.** El requerimiento contempla OTP también para login MFA del portal tarjetahabiente, cambio de PIN cuando se requiera segundo factor, y eventualmente para validaciones administrativas. Si vive dentro del `PreRegistration_Service`, cada nuevo flujo termina duplicando lógica.
2. **Acoplamiento innecesario.** Cambiar el algoritmo de generación, el tamaño del código, el TTL o el medio de envío obliga a tocar el Service de pre-registro aunque el cambio sea puramente de OTP.
3. **El `PreRegistration_API` tiene tres endpoints (verify, validate-otp, create-account); el de validate-otp puede llamar al `OTP_Service` directamente** sin pasar por el Service de negocio. Eso simplifica el flujo y reduce un hop.

**Decisión:** Extraer la gestión de OTP como un nuevo componente `OTP_Service` (Java 21 + Spring Service) en la pestaña "Onboarding Tarjetahabiente":

1. **Componente nuevo `OTP_Service`** (color verde `#10B981`, mismo del SSO para indicar dominio "identidad / autenticación"). Operaciones expuestas:
   - `issueOtp(client_id, purpose)` — genera OTP de 6 dígitos, calcula SHA-256, persiste en `otp(code_hash, expires_at=NOW+5min, attempts=0, status=ACTIVO)`, delega el envío al `Notification_Adapter`.
   - `validateOtp(otp_id, code)` — busca el OTP activo, compara `SHA-256(code)` contra `code_hash`, valida no expirado, incrementa intentos, marca `USADO` si OK o `BLOQUEADO` al alcanzar 3.
   - El `purpose` permite reusar la tabla `otp` para múltiples flujos: `PRE_REGISTRO`, `LOGIN_MFA`, `CAMBIO_PIN`, `OTRO`.
2. **Cambios en el flujo de la pestaña:**
   - El `PreRegistration_Service` deja de tener `issueOtp` y `validateOtp` propias; ahora `requestOtp(code)` → `otpService.issueOtp(...)` y `confirmOtp(code, otp)` → `otpService.validateOtp(...)`.
   - El `PreRegistration_API` puede llamar al `OTP_Service` directamente para `POST /validate-otp` cuando solo se necesita validar el código y no aplicar lógica de session_registro. La lógica de session_registro (TTL 10 min) la sigue manejando el `PreRegistration_Service` tras una validación exitosa.
   - El `OTP_Service` se vuelve cliente del `Notification_Adapter` para enviar el OTP al TH (vía email/SMS), no del `PreRegistration_Service`.
   - El `OTP_Service` persiste directamente en la tabla `otp` de Oracle Primary.

3. **2 ajustes de alineación con `Onboarding_Tarjetahabiente.drawio`:**
   - Política de password: la nota de seguridad ahora indica `min 12 chars, mayúscula, minúscula, número, especial` (estaba sin `minúscula` en v1.5 inicial).
   - Email de confirmación post-creación: se quitó `9. sendConfirmation` del label del edge `e-prereg-notif`. El source no formaliza ese envío; queda fuera del scope hasta que se decida formalmente.

**Razón:**
- **Reusabilidad** — el mismo `OTP_Service` atiende pre-registro, login MFA y cambio de PIN. Una sola implementación, una sola tabla, una sola política de TTL e intentos.
- **Cohesión vs acoplamiento** — el `PreRegistration_Service` queda más enfocado en su dominio (creación de pre-registro, verificación de cédula, creación de usuario en SSO).
- **Coherencia con el patrón establecido** — Audit_Service (ADR-018), B2B_Mail_Service (ADR-022) y Notification_Adapter ya son servicios reusables. OTP_Service sigue el mismo patrón.
- **No requiere cambios de BD** — la tabla `otp` ya existe en `db/01_schema_prepago.sql` con las columnas necesarias (`purpose`, `code_hash`, `expires_at`, `attempts`, `status`, `user_id`, `client_id`). Solo se mueve la responsabilidad de escritura.

**Alternativas descartadas:**
- Dejar OTP dentro del `PreRegistration_Service` y "ya cuando aparezca otro flujo lo extraemos" — rechazado, en cuanto aparezca login MFA o cambio de PIN se va a duplicar la lógica y va a haber divergencias en TTL, intentos y formato.
- Hacer del `OTP_Service` un microservicio independiente con base de datos propia — rechazado, sobre-ingeniería para el volumen actual; basta con un Spring Service que comparta la BD.
- Que el `Notification_Adapter` también gestione el ciclo de vida del OTP (no solo lo entregue) — rechazado, mezcla responsabilidades; el adapter envía mensajes, no decide cuándo regenerar un OTP ni cómo validarlo.

**Impacto:**
- Solo cambia la pestaña "Onboarding Tarjetahabiente" del unificado v1.5. Sigue siendo `pages="16"`.
- No requiere cambios de SQL ni MER — la tabla `otp` ya está preparada.
- Cuando se implementen login MFA y cambio de PIN, deberán reusar `OTP_Service` desde su flujo respectivo. Quedan como evolutivos abiertos.
- Pendiente operacional: definir si el envío de OTP por SMS (cuando aplique) requiere un canal aparte en el `Notification_Adapter` o un nuevo `SMS_Adapter`. Por ahora se asume que `Notification_Adapter` maneja email + SMS.

## ADR-025: `B2B_Mail_Service` como servicio único de correos + verificación dual de identidad en onboarding

**Contexto:** Al revisar la pestaña "Onboarding Tarjetahabiente" contra el documento `requerimientos-onboarding-tarjetahabiente.md`, se identificaron tres aclaraciones de negocio que cambian decisiones previas:

1. **`B2B_Mail_Service` ya existe en CCO como servicio único** de envío de correos transaccionales. La separación que planteamos en ADR-022 (`Notification_Adapter` para B2C vs `B2B_Mail_Service` para B2B) no aplica: hay un único servicio compartido que gestiona todos los emails, B2C y B2B, diferenciados por tipo de plantilla y rate limiting interno.
2. **La verificación de identidad en el onboarding** valida no solo `(doc_type, doc_number)` sino también el **correo electrónico registrado** del TH. Es una verificación dual: documento + correo deben coincidir antes de emitir la OTP. Esto refuerza el factor "algo que el TH conoce" (correo registrado por la entidad emisora).
3. **El orquestador único de envíos** del onboarding es el `PreRegistration_Service`. El `OTP_Service` solo genera y valida códigos; **no** envía emails directamente. Cuando se necesita enviar la OTP, el `PreRegistration_Service` recibe el código del `OTP_Service` y orquesta el envío vía `B2B_Mail_Service`.

**Decisión:**

1. **Reemplazar `Notification_Adapter` por `B2B_Mail_Service`** como única caja de envío de correos en la pestaña "Onboarding Tarjetahabiente" del unificado v1.5. El `B2B_Mail_Service` es el servicio existente de CCO compartido por todos los componentes de la plataforma que requieran correo (onboarding TH, distribución a entidades, alertas operativas).
2. **`verifyIdentity` recibe ahora 4 parámetros:** `(code, doc_type, doc_number, email)`. El contrato del endpoint `POST /api/preregistro/verify` se actualiza para incluir `email` en el body.
3. **El `PreRegistration_Service` queda como orquestador único** de los envíos del onboarding. El `OTP_Service.issueOtp(...)` retorna el código en memoria al caller; no toca el `B2B_Mail_Service`. Cuando el `PreRegistration_Service` recibe ese código, decide si lo envía por email (vía `B2B_Mail_Service.sendOtpEmail(...)`) o por SMS (cuando el adapter SMS exista).
4. **La separación B2C / B2B se convierte en clasificación lógica de templates**, no en dos servicios distintos. El `B2B_Mail_Service` decide el routing, rate limiting y SLA por tipo de mensaje internamente.

**Cambios aplicados al diagrama (pestaña "Onboarding Tarjetahabiente" en v1.5):**
- Caja `onb-notif`: título → `B2B_Mail_Service`. Tech → `[Container: Servicio existente en CCO — cliente SMTP corporativo]`. Descripción reescrita para indicar que es servicio único compartido.
- Caja `onb-prereg-svc`: descripción reescrita con la firma `verifyIdentity(code, doc_type, doc_number, email)`. Aclara que orquesta envíos vía `B2B_Mail_Service` y que invoca `OTP_Service` solo para generar/validar códigos.
- Caja `onb-otp-d`: descripción reescrita aclarando "Retorna code en memoria; el envío lo orquesta el caller".
- Caja `onb-api-d`: cambia `(cédula)` por `(doc_type + doc_number + email)` en el endpoint `/verify`.
- Caja `onb-portal-d`: paso UX 1 ahora dice "verificar identidad (tipo+núm doc + correo)".
- Edge `e-otp-notif` (huérfano sin target): reemplazado por `e-otp-svc` desde `OTP_Service` hacia `PreRegistration_Service` con label "Retorna code (en memoria) · PreReg_Service orquesta envío".
- Edge `e-prereg-notif`: label actualizado a "sendPreRegEmail · sendOtp · sendAccountCreated (orquestado por PreRegistration_Service)".
- Nota de seguridad (`onb-note-seg`): "Verificación de identidad: tipo+número de documento + correo electrónico registrado; max 3 intentos → BLOQUEADO".

**Cambios aplicados al documento `requerimientos-onboarding-tarjetahabiente.md`:**
- §3 Fase 2 (Verificación de identidad): pasos 2-7 actualizados para incluir email.
- §4.1 Componentes: `Notification_Adapter` reemplazado por `B2B_Mail_Service`. Aclaración del rol del `OTP_Service` (no envía).
- §5.1 Contrato `POST /api/preregistro/verify`: body añade `email`. Nota explicativa sobre verificación dual.
- §12 Notificaciones: encabezado actualizado al `B2B_Mail_Service`. Nota arquitectónica al final.

**Razón:**
- **Reusar lo que ya existe en CCO** evita construir un servicio duplicado y mantiene una sola plantilla de correos para todo el grupo.
- **Verificación dual (documento + correo)** sube el bar de seguridad antes de emitir OTP. Si un atacante obtiene el link de pre-registro y conoce el documento, todavía necesita conocer el correo registrado por la entidad. Es una capa adicional barata y efectiva.
- **Orquestador único de notificaciones** mantiene el `OTP_Service` cohesivo (solo gestión de códigos) y simplifica el reuso para login MFA y cambio de PIN: en esos flujos otro caller decidirá qué adapter de notificación usar.
- **Reconciliar ADR-022** evita confusión arquitectónica futura: solo hay un servicio de correos en CCO y todos los componentes lo consumen.

**Alternativas descartadas:**
- Mantener dos servicios (`Notification_Adapter` para B2C + `B2B_Mail_Service` para B2B) — rechazado por el negocio: ya existe un solo servicio en CCO.
- Que `OTP_Service` siga llamando al `B2B_Mail_Service` directamente — rechazado, viola el principio de cohesión y dificulta su reuso para MFA/cambio de PIN donde el caller puede querer SMS u otro canal.
- Validar solo `(doc_type, doc_number)` en `verifyIdentity` y dejar el correo como verificación implícita (porque la OTP llega al correo registrado) — rechazado, no es suficiente: si el atacante intercepta el correo del TH, pasa la verificación y la OTP llega al mismo correo comprometido.

**Impacto:**
- ADR-022 queda parcialmente revertido: el `B2B_Mail_Service` sigue existiendo como caja en la pestaña "Realce" para el envío del Reporte de Pedido, pero el `Notification_Adapter` mencionado en pestañas "Creacion de tarjetas" y "Onboarding" ya no existe — todos consumen el `B2B_Mail_Service`.
- La pestaña "Creacion de tarjetas" debe revisarse para reemplazar la mención al `Notification_Adapter` (cuando dispara el pre-registro) por `B2B_Mail_Service`. Pendiente para una iteración aparte si aplica.
- El SQL no cambia (las tablas `preregistration`, `otp`, `app_user` siguen iguales).
- Las plantillas de email del onboarding pasan a estar gestionadas dentro del `B2B_Mail_Service` (configuración existente en CCO).
- La distinción B2C / B2B desaparece como decisión arquitectónica; es solo clasificación de templates.

## ADR-026: v1.7 — Cobertura total de la pestaña "Flujo de novedades monetarias"

**Contexto:** Al validar la pestaña "Flujo de novedades monetarias" del `PrepagoUnificadoArquitectura_V_1_6.drawio` contra el documento `requerimientos-novedades-monetarias.md`, se detectó que la pestaña cubría únicamente un subset (recargas/abonos por archivo + API) y dejaba fuera la mayor parte del alcance documentado:

1. **Subtítulo erróneo:** "sin dependencia de Pomelo", incompatible con el webhook `/webhooks/adjustment` de §3 y §4.3 del documento.
2. **Solo modelaba ABONO**: faltaba CARGO (débito), AJUSTE_DEBITO, AJUSTE_CREDITO, RECOBRO.
3. **Estado de tarjeta inexacto:** validaba solo `ACTIVE`. El documento exige `Creada, Activa o Bloqueo Temporal` y, para ajustes, incluso `Bloqueo Temporal` aplica.
4. **Endpoint diferente:** diagrama usaba `POST /api/v1/reloads`, doc define `POST /api/v1/monetary-novelty`.
5. **Componentes ausentes:** `Pomelo_Webhook_Adapter`, `Adjustment_Service`, `Limit_Counter_Service` (regla 81), `Recovery_Service`, `Output_File_Generator`, `Audit_Service`, `B2B_Mail_Service`.
6. **Idempotencia incorrecta:** decía `reload_request_id`; el documento define `X-Idempotency-Key` (API), `(file_name, control_record)` (archivo), `adjustment_id` (Pomelo).
7. **Archivos de salida sin nombrar:** no se mencionaban TNM, TAV, AUSR, ABO, RAJU.
8. **Validaciones de archivo ausentes:** ±6 días, longitud 128 bytes, trailer (TRAILER_NIT, ARCHIVO_FECHA, TRAILER_FECHA, TRAILER_NUM_REG, NIT_EN_NOVEDAD).
9. **PGP no documentado en GAW:** solo decía "valida estructura/formato".

**Decisión:** Crear `PrepagoUnificadoArquitectura_V_1_7.drawio` (mantener v1.6 como histórico según convención del proyecto) y **ampliar exclusivamente la pestaña "Flujo de novedades monetarias"** para alcanzar cobertura total del documento. El resto de las 15 pestañas se conservan idénticas a v1.6.

**Cambios aplicados a la pestaña:**

1. **Título y subtítulo:**
   - Título: `Flujo de Novedades Monetarias — Abonos · Cargos · Ajustes Pomelo · Recobros`.
   - Subtítulo: `C4 Level 2 | Triple canal: Archivo MXXXddmmcc · API Entidad · Webhook Pomelo · Job batch GMF mensual. Cobertura total de requerimientos-novedades-monetarias.md`.
   - Eliminada la frase "sin dependencia de Pomelo".

2. **Componentes existentes renombrados/ampliados:**
   - `Batch Novelty Processor` → **`Prepaid_Monetary_Novelty_Processor`** con descripción que menciona MXXXddmmcc, validaciones (±6 días, trailer, longitud 128 bytes), chunk 1k-5k, generación de TNM y TAV incrementales.
   - `Novelty API` → **`Prepaid_Monetary_Novelty_Service`** con endpoint `POST /api/v1/monetary-novelty` (mTLS + JWT + X-Idempotency-Key), soporte `type=ABONO|CARGO`, validación de estado tarjeta `{Creada, Activa, Bloqueo Temporal}`.
   - `Novelty Orchestrator` → **`Novelty_Orchestrator`** con 7 pasos (resolver card_id, validar estado, CHECK regla 81, manejar saldo insuficiente con `pending_charge`, recuperación de cargos pendientes en abonos, atomicidad en SP, audit async).
   - `Oracle Database` → **`Oracle Primary`** con SPs y tablas listadas (`SP_APPLY_MONETARY`, `SP_APPLY_ADJUSTMENT`, `SP_RECOVER_CHARGES`, `SP_GMF_MONTHLY_BATCH`; tablas `transaction`, `pending_charge`, `file_processing`, `file_record`, `adjustment_inbox`).
   - `Redis` → **`Redis Cluster`** con descripción de Sentinel HA, contadores Limit_Counter (regla 81), cache idempotency-key TTL 24h, fallback a Oracle.
   - `GoAnywhere MFT` → ahora dice "Descifra PGP · Valida estructura (MXXXddmmcc, ±6 días, longitud 128b) · Cifra PGP archivos respuesta".
   - `File Storage`: descripción ampliada con "MXXXddmmcc descifrado (entrada) · TNM, TAV, AUSR, ABO, RAJU (salida)".

3. **Componentes nuevos añadidos:**
   - **`Pomelo`** (sistema externo morado, borde punteado) — origen de Ajustes por Reclamaciones, 24/7, HMAC-SHA256 + mTLS.
   - **`Pomelo_Webhook_Adapter`** — endpoint `POST /webhooks/adjustment`, valida HMAC-SHA256 (X-Pomelo-Signature), idempotencia por `adjustment_id` en `adjustment_inbox`, rate limit + alerta SIEM si HMAC inválido. Marcado como CAMINO 3.
   - **`Adjustment_Service`** — aplica AJUSTE_DEBITO/AJUSTE_CREDITO via `SP_APPLY_ADJUSTMENT`, en línea 24/7, permite Bloqueo Temporal, sin `pending_charge` para débitos sin saldo, alimenta ABO + RAJU.
   - **`Limit_Counter_Service`** (rojo, Redis) — CHECK / INCREMENT / DECREMENT / RESET para regla 81 con TTL por periodicidad AFG y fallback Oracle.
   - **`Recovery_Service`** — ejecuta `SP_RECOVER_CHARGES(card_id, increment)` tras ABONO, itera `pending_charge` ASC con vigencia 3 meses, INSERT txn=RECOBRO, reporta en COMI.
   - **`Output_File_Generator`** — genera TNM/TAV (en línea, incremental), AUSR/ABO/RAJU (batch 01:00) cifrados PGP, alimenta también AUM y COMI.
   - **`Audit_Service`** — Kafka async, audit servicio + PCI 10.2 (usuario, evento, fecha, éxito/fallo, origen, datos afectados), retención hot 1 año / archive 7 años.
   - **`B2B_Mail_Service`** — notifica fallas estructurales a la Entidad (TRAILER_NIT, ARCHIVO_FECHA, TRAILER_FECHA, TRAILER_NUM_REG, NIT_EN_NOVEDAD, LONGITUD).

4. **Edges nuevos relevantes:**
   - `Pomelo → Pomelo_Webhook_Adapter` (HMAC-SHA256 + mTLS).
   - `Pomelo_Webhook_Adapter → Adjustment_Service`.
   - `Adjustment_Service → Oracle` (SP_APPLY_ADJUSTMENT + adjustment_inbox).
   - `Novelty_Orchestrator → Limit_Counter_Service` (CHECK / INCREMENT regla 81).
   - `Novelty_Orchestrator → Recovery_Service` (si type=ABONO).
   - `Recovery_Service → Oracle` (SP_RECOVER_CHARGES).
   - `Novelty_Orchestrator → Output_File_Generator` (TNM/TAV incremental).
   - `Adjustment_Service → Output_File_Generator` (ABO + RAJU batch 01:00).
   - `Output_File_Generator → Oracle` (lectura, descifra PAN en memoria).
   - `Output_File_Generator → File Storage` (TNM/TAV/AUSR/ABO/RAJU PGP).
   - `Novelty_Orchestrator → Audit_Service` async.
   - `Adjustment_Service → Audit_Service` async.
   - `Prepaid_Monetary_Novelty_Processor → B2B_Mail_Service` (falla estructural).

5. **Bloques informativos:**
   - **Leyenda** con código de colores (azul Credibanco, rojo Redis, morado externo, índigo on-prem) y los 3 caminos de entrada documentados.
   - **Referencias a pestañas complementarias**: `Comisiones Batch (Recobros)`, `GMF Batch`, `Cuotas de manejo`, `Reconciliacion End-of-Day`, `Intercambio de Archivos`.

6. **Boundary OpenShift** ampliado de 1160×610 a 1880×980 para acomodar los componentes nuevos.

7. **`pageWidth`** ampliado de 827 a 2400 y **`pageHeight`** de 1169 a 1900 para que el lienzo de la pestaña entre completo.

**Razón:**
- La pestaña ahora refleja el alcance completo del documento (4 canales de entrada: archivo, API, webhook Pomelo, GMF batch — este último referenciado en pestaña separada).
- Cualquier desarrollador que abra la pestaña obtiene una vista C4 L2 fiel a los requerimientos sin necesidad de cruzar contra el `.md`.
- La arquitectura queda alineada para que la auditoría PCI (Req 10.2) y los archivos de salida (TNM, TAV, AUSR, ABO, RAJU, COMI) sean visibles sin ambigüedad.
- Reusa componentes ya existentes en otras pestañas (`B2B_Mail_Service` de ADR-022/025, `Audit_Service` de ADR-017, `Limit_Counter_Service` de ADR-004/017) preservando coherencia transversal.

**Alternativas descartadas:**
- **Renombrar la pestaña a "Recargas / Abonos"** y dejar el resto del alcance fuera — rechazado porque el nombre histórico de la pestaña (`Flujo de novedades monetarias`) coincide con el alcance del documento; renombrar habría forzado a crear pestañas nuevas para Cargos y Ajustes y a fragmentar la lectura.
- **Crear una pestaña aparte solo para "Ajustes por Reclamaciones Pomelo"** — considerado, diferido. Si en el futuro la complejidad de los ajustes (SLAs, métricas específicas, contracargos) crece, se puede extraer entonces.
- **Modificar v1.6 en sitio sin crear v1.7** — rechazado, viola la convención del proyecto de no sobreescribir versiones de arquitectura.

**Impacto:**
- Archivo nuevo: `Prepago/PrepagoUnificadoArquitectura_V_1_7.drawio` (16 páginas, igual que v1.6). Solo cambia la pestaña "Flujo de novedades monetarias".
- `project-context.md` debe apuntar a `PrepagoUnificadoArquitectura_V_1_7.drawio` como arquitectura principal vigente.
- `requerimientos-novedades-monetarias.md` queda 100% reflejado en la pestaña; las referencias cruzadas en §17.2 del documento siguen siendo válidas.
- Oracle: el modelo entidad-relación necesita la tabla `adjustment_inbox` y la extensión de `pending_charge.charge_type` con `'CARGO_NOVEDAD'` (definidas en §9.2 del documento). El SQL `db/01_schema_prepago.sql` no incluye aún esa tabla — pendiente para una próxima iteración del DDL.
- Redis: el `Limit_Counter_Service` ahora se materializa también en esta pestaña; las claves siguen siendo las definidas en ADR-017.

### Aclaración complementaria a ADR-026 (writer / publisher)

**Contexto adicional:** durante la revisión surgió la pregunta de si el `Prepaid_Monetary_Novelty_Processor` debería encargarse también de escribir los archivos de salida (TNM/TAV) en lugar de delegar al `Output_File_Generator`.

**Decisión:** mantener la separación **writer / publisher**.

- **Writers** (procesamiento de novedades): `Prepaid_Monetary_Novelty_Processor`, `Prepaid_Monetary_Novelty_Service`, `Adjustment_Service`, `Recovery_Service`, motor transaccional, `GMF_Batch_Job`. Persisten resultados solo en BD (`transaction`, `file_record`, `pending_charge`, `adjustment_inbox`).
- **Publisher único** de archivos: `Output_File_Generator`. Lee de BD, descifra PAN en memoria, cifra PGP y escribe a NFS. Es el **único componente** que toca PAN cifrado al salir y que opera sobre el File Storage de salida.

**Razón:**
- De los archivos del catálogo (TNM, TAV, AUSR, ABO, RAJU, AUM, COMI, CSAT, CINS, SALD), solo TNM y TAV se nutren estrictamente del Processor. El resto (AUSR, ABO, RAJU, AUM, COMI, SALD) provienen de `Adjustment_Service`, motor transaccional, `Recovery_Service` o cierres nocturnos. Cargar al Processor con todos rompería su nombre y su responsabilidad; cargarlo solo con TNM/TAV mantendría el OFG igual y dejaría dos componentes escribiendo al mismo File Storage sin ahorro neto.
- Centralizar el cifrado PGP y el descifrado de PAN en un solo componente acota el blast radius PCI (Req 3.4 — PAN no legible en storage; Req 4.2.1 — TLS 1.3 / PGP en transit). Distribuir esa responsabilidad obliga a auditar PAN-handling en cada componente que escriba archivos.
- Coherencia con las pestañas "Intercambio de Archivos", "Flujo Novedades No Monetarias" y "Realce" del unificado, donde el `Output_File_Generator` ya es el publisher único.
- Independencia de fallos: si el OFG cae por problema de PGP o SFTP, los Writers siguen afectando saldos y la BD queda consistente. Cuando el OFG vuelve, regenera los archivos desde BD. Si los fusionamos, un fallo en escritura puede bloquear o duplicar la afectación de saldo.
- Schedule diferente: TNM/TAV son **incrementales por evento**, AUSR/ABO/RAJU son **batch nocturnos 01:00**, SALD es **corte 00:00**. Tres ciclos en un mismo componente complican retry, locking y cron.

**Aplicado al diagrama V_1_7:**
- Descripción del `Prepaid_Monetary_Novelty_Processor` actualizada: indica que es **writer**, persiste en `file_record` y **NO escribe archivos**.
- Descripción del `Output_File_Generator` actualizada: marcado como **publisher único** que concentra PGP y descifrado de PAN.
- Edge `Novelty_Orchestrator → Output_File_Generator` reetiquetado de "Resultado por registro (TNM/TAV incremental)" a "Notifica registro persistido (file_record) → publish TNM/TAV", que refleja el contrato real (notificación de evento, no escritura de archivo).

**Aplicado al documento `requerimientos-novedades-monetarias.md`:**
- §5 (Componentes): nota introductoria sobre el patrón writer/publisher; columna "Responsabilidad" marcada con **Writer** o **Publisher** según corresponda.
- §10 (Archivos de salida): nota inicial que reitera que el OFG es el publisher único y que los componentes de procesamiento no escriben archivos.

## ADR-027: v1.8 — Consolidación de "Cuotas de manejo" + "GMF Batch" + "Comisiones Batch (Recobros)" en pestaña única "Motor de Cobros Diferidos"

**Contexto:** En `PrepagoUnificadoArquitectura_V_1_7.drawio` coexistían tres pestañas que modelaban variaciones del mismo patrón de **carga diferida → débito → si falla saldo, pendiente → reintento al detectar abono**:

- **Cuotas de manejo** — Maintenance Fee Scheduler mensual + Fee Processor + Retry Processor (cuota, IVA, periodicidad por AFG).
- **GMF Batch** — Job mensual UVT (Ley 2277/2022) para tarjetas exentas con acumulado mensual excedido.
- **Comisiones Batch (Recobros)** — Reintento de comisiones no efectivas (drena `pending_charge` por antigüedad).

Las tres pestañas duplicaban Spring Batch + Oracle + `pending_charge` + invalidación de cache + `Output_File_Generator`. La diferencia real era cosmética (origen del cobro y schedule). Esta duplicación introducía riesgo de divergencia: cualquier mejora al ciclo "cobrar → si falla pendiente → recobro" había que aplicarla en tres lugares.

**Decisión:** Crear `PrepagoUnificadoArquitectura_V_1_8.drawio` (mantener v1.7 como histórico). En v1.8:

1. **Eliminar** las tres pestañas históricas: "Cuotas de manejo", "GMF Batch", "Comisiones Batch (Recobros)".
2. **Crear** una pestaña consolidada **"Motor de Cobros Diferidos"** con un patrón unificado de 5 capas:

   - **Fuentes de cobro (writers)** — 4 cajas independientes: `Maintenance_Fee_Scheduler` (mensual, charge_type=`CUOTA_MANEJO`), `GMF_Calculator` (mensual UVT, `GMF`), `Commission_Charge_Engine` (intra-day, `COMISION`), `Tech_Exitosa_Charge_Engine` (intra-day, `TECH_EXITOSA`).
   - **Núcleo común** — `Charge_Settlement_Service` con un único contrato `settle(card_id, charge_type, amount, concept, source)`. Internamente invoca `SP_SETTLE_CHARGE` (Oracle) que: lee saldo → si suficiente, UPDATE balance + INSERT transaction; si no, INSERT pending_charge con `expires_at = now + 90 días`. Idempotencia por (charge_type, card_id, period).
   - **Recovery** — `Recovery_Service` con dos disparadores: evento (Abono/Ajuste_Crédito desde pestaña monetaria) y batch (`Recovery_Retry_Scheduler` 03:00 diario). Ejecuta `SP_RECOVER_CHARGES` que itera `pending_charge` por antigüedad ASC, libera saldo nuevo y genera `transaction(RECOBRO)`. Vencimiento 90 días → `EXPIRADA` → archivo CINS.
   - **Persistencia y cache** — `Oracle Primary` (transaction particionada, pending_charge, account, fee_config) + `Redis Cluster` (locks distribuidos del SP, idempotency keys, cache de saldo).
   - **Salidas** — `Output_File_Generator` (publisher único, igual que en pestaña monetaria) genera COMI/CSAT/CINS/AUM. `Audit_Service` async (PCI 10.2). `Portal Admin — Cobros Diferidos` para vista operativa, cancelación manual, re-disparo del retry scheduler.

3. **Estados explícitos de `pending_charge`** documentados en una caja: `PENDIENTE → COBRADA` (por evento o batch) | `PENDIENTE → EXPIRADA` (90 días) | `PENDIENTE → CANCELADA` (operación manual). Vigencia 90 días = 3 meses.

4. **Disparadores eventuales** desde otras pestañas listados en una caja morada para que el lector entienda que la pestaña no se duplica los flujos:
   - "Flujo de novedades monetarias" → Abono → `Recovery_Service.recoverByEvent(card_id)`.
   - "Flujo de transacciones" → Denegación 51/55/57/61/75 → `Tech_Exitosa_Charge_Engine.charge(...)`. Comisión calculada → `Commission_Charge_Engine.charge(...)`.

**Razón:**
- **Reduce duplicación** estructural: tres veces los mismos componentes → una vez. Cualquier mejora al ciclo de cobros se aplica en una sola pestaña.
- **Hace explícito el patrón** writer/settlement-core/recovery que estaba implícito y disperso.
- **Soporta extensión** sin pestañas nuevas: si aparece una nueva fuente de cobro (ej. cuota de seguro, comisiones de inactividad), basta añadir un writer y enchufarlo al `Charge_Settlement_Service` con un nuevo `charge_type`.
- **Centraliza la lógica de saldo insuficiente** en un solo SP (`SP_SETTLE_CHARGE`) y un solo servicio. Antes estaba replicada en Fee Processor, GMF Batch y Commission Retry, con riesgo real de divergencia.
- **Coherencia con writer/publisher**: igual que en pestaña monetaria (ADR-026 aclaración complementaria), el `Output_File_Generator` es el único componente que escribe archivos. Los writers no tocan SFTP.
- **Mantiene independencia de schedules**: cada writer tiene su propio cron (mensual día 1, mensual día N, intra-day) sin acoplarse al settlement core.

**Alternativas descartadas:**
- **Mantener las tres pestañas separadas** (status quo): rechazado, duplica componentes y dificulta la lectura del patrón. Cualquier desarrollador nuevo tiene que mirar 3 pestañas antes de entender que es el mismo patrón.
- **Mantener las tres pestañas + crear una pestaña "vista unificada" arriba**: rechazado, multiplica las pestañas y deja al lector la duda de cuál es la fuente de verdad.
- **Fusionar también con la pestaña "Flujo de novedades monetarias"**: rechazado, esa pestaña tiene su propio dominio (canales de entrada de novedades, ajustes Pomelo, validación de archivos MXXX). Mezclarlas oculta el dominio "cobros diferidos" detrás del dominio "novedades monetarias".
- **Eliminar la pestaña "Comisiones Batch (Recobros)" sin crear pestaña nueva**, dejando la mecánica de recobro implícita en cada writer: rechazado, el ciclo de `pending_charge` con sus 4 estados es información crítica que merece visualización propia.

**Impacto:**
- Archivo nuevo: `Prepago/PrepagoUnificadoArquitectura_V_1_8.drawio` (14 páginas, antes 16). El conteo bajó porque consolidamos 3 pestañas en 1.
- `project-context.md` debe apuntar a `PrepagoUnificadoArquitectura_V_1_8.drawio` como arquitectura principal vigente.
- Modelo de datos: el SQL `db/01_schema_prepago.sql` ya tiene `pending_charge` y `transaction` (con `txn_type=RECOBRO`). Pendiente: añadir `SP_SETTLE_CHARGE` como SP unificado en `db/02_stored_procedures.sql`. Hoy existen `SP_AUTHORIZE_TXN` y `SP_CHARGE_TECH_EXITOSA`, pero el patrón unificado de settlement merece su propio SP.
- Convención: cualquier nueva fuente de cobro (cuota de seguro, etc.) debe enchufarse al `Charge_Settlement_Service` y no crear su propio flujo de débito.
- ADRs relacionados: ADR-014 (Recobro de comisiones hasta 3 meses) sigue vigente y se materializa ahora en esta pestaña; ADR-012 (GMF en línea vs batch) sigue vigente y el branch "batch" se materializa aquí; ADR-013 (Cobro técnicamente exitosas) sigue vigente, ahora con el `Tech_Exitosa_Charge_Engine` como fuente del settlement.

## ADR-028: GMF batch entra como novedad por archivo de la Entidad — eliminación de `GMF_Calculator` y `SP_GMF_MONTHLY_BATCH`

**Contexto:** Las versiones previas de la arquitectura (`requerimientos-novedades-monetarias.md` §4.5, V_1_8 pestaña "Motor de Cobros Diferidos", componentes `GMF_Calculator` / `GMF_Batch_Job` / `SP_GMF_MONTHLY_BATCH`, ADR-012 branch BATCH) describían el GMF batch como un **job interno** que:

1. Iteraba tarjetas exentas el primer día hábil del mes.
2. Sumaba transacciones gravables del mes acumuladas.
3. Si el total mensual superaba el tope `Valor UVT × Total Unidades UVT`, calculaba el 4×1000 sobre el exceso.
4. Aplicaba el cobro al saldo (o creaba `pending_charge` si el saldo no alcanzaba).

Esta interpretación era incorrecta. La realidad operativa, confirmada por negocio:

- La **Entidad emisora** calcula el GMF en su sistema (la Entidad es la que tiene la información completa del TH y aplica las reglas de UVT mensual).
- La Entidad **envía a Credibanco un archivo cifrado por SFTP** vía GoAnywhere con las novedades de GMF a aplicar por tarjeta.
- Credibanco **aplica la novedad** indicada por la Entidad. No calcula UVT, no decide el monto.

**Decisión:** Reescribir el flujo de GMF batch como una **novedad monetaria adicional** que entra por el mismo `Prepaid_Monetary_Novelty_Processor`, con `charge_type=GMF` para el `Charge_Settlement_Service`. Eliminar el componente `GMF_Calculator` (con sus variantes `GMF_Batch_Job` y `SP_GMF_MONTHLY_BATCH`) del catálogo de writers internos del Motor de Cobros Diferidos.

**Cambios aplicados:**

1. **`PrepagoUnificadoArquitectura_V_1_9.drawio`** (mantener V_1_8 como histórico):
   - Pestaña "Motor de Cobros Diferidos": la caja `GMF_Calculator` se transforma en una **caja de aviso ámbar** con título "GMF batch — entra como novedad", subtítulo "[NO es writer activo aquí — origen externo]" y descripción que apunta a la pestaña "Flujo de novedades monetarias" como origen real. Conserva el ID y la posición para no romper layout, pero cambia color y estilo a dashed ámbar para señalizar que es una referencia, no un componente activo.
   - Pestaña "Motor de Cobros Diferidos": el edge `dce-e-gmf-core` se reetiqueta de `settle(card_id, GMF, ...)` a `indirecto: vía pestaña Flujo de novedades monetarias`, con estilo dashed ámbar.
   - Pestaña "Flujo de novedades monetarias": descripción del `Prepaid_Monetary_Novelty_Processor` ampliada para mencionar que también procesa el archivo GMF de la Entidad (nombre y estructura TBD), con `type=GMF` y `charge_type=GMF`. Aclara que la Entidad calcula UVT y Credibanco aplica.
   - Pestaña "Flujo de novedades monetarias": descripción del `File Storage` añade "Archivo GMF de la Entidad — TBD" en la lista de entradas.

2. **`requerimientos-novedades-monetarias.md`**:
   - §1.2 alcance: el GMF batch ahora es novedad por archivo de la Entidad (no acumulación interna UVT).
   - §3 disparadores: el canal #4 cambia de "Job batch nocturno (Spring Batch)" a "Archivo GMF de la Entidad por GoAnywhere → SFTP/NFS".
   - §4.5 reescrito completo: nuevo flujo (Entidad calcula, envía archivo, Credibanco aplica) + lista de pendientes específicos del archivo GMF.
   - §5 componentes: `GMF_Batch_Job` marcado como **eliminado en ADR-028**.
   - §5 SPs: `SP_GMF_MONTHLY_BATCH` tachado y marcado como eliminado en ADR-028.
   - §16 pendientes: añadidos 4 ítems (nombre del archivo, estructura del registro detalle, trailer, periodicidad).

3. **`requerimientos-flujo-transaccion.md`** §11.5:
   - "GMF en línea bajo el modelo actual y en batch bajo el nuevo modelo" → "GMF en línea bajo el modelo actual y **GMF batch entra como novedad por archivo de la Entidad** bajo el nuevo modelo". Referencia explícita a §4.5 del documento de monetarias y a este ADR.

4. **ADR-012 (GMF en línea vs batch)**:
   - Sigue vigente en su intención (flag `gmf_mode` por entidad, exoneración por tarjeta posición 440).
   - El branch `BATCH` de la decisión cambia de "se omite en la txn, se calcula en proceso batch diario (GMF Batch Processor)" a "se omite en la txn, se aplica cuando llega la novedad de GMF por archivo de la Entidad".
   - Esta aclaración se refleja como anotación complementaria en este ADR-028 (no se reescribe ADR-012 íntegro para preservar trazabilidad histórica).

5. **ADR-027 (Motor de Cobros Diferidos)**:
   - El catálogo de writers activos pasa de 4 a 3: `Maintenance_Fee_Scheduler`, `Commission_Charge_Engine`, `Tech_Exitosa_Charge_Engine`. La cuarta caja sigue presente como referencia visual ámbar al GMF batch externo, pero conceptualmente no es un writer activo.
   - El `Charge_Settlement_Service` sigue recibiendo `charge_type=GMF` cuando la novedad llega por archivo y no alcanza saldo (pending_charge) — el patrón de recobro funciona idéntico para GMF que para los demás charge_types.

**Pendientes de definición (responsable: Entidad + Producto + Arquitectura):**

- **Nombre canónico del archivo.** Hipótesis: `GXXXddmmcc` (XXX=AFG, dd=día, mm=mes, cc=consecutivo) en línea con `MXXXddmmcc` y `NXXXddmmcc`. Por confirmar con la Entidad.
- **Estructura del registro detalle.** Campos esperados: tipo de identificación, número de identificación, AFG, NIT, monto GMF, periodo (mes / año), referencia de la Entidad. Longitud y separadores TBD.
- **Validaciones del trailer.** Análogas a las de `MXXXddmmcc`: NIT, fecha (±6 días), total registros, valor sumado.
- **Periodicidad de envío.** Hipótesis: mensual al cierre. Por confirmar con la Entidad.
- **Devolución del 4×1000** cuando una transacción gravada se reversa después de aplicado el GMF batch. Política a coordinar con la Entidad (¿la Entidad envía un GMF negativo? ¿lo gestiona en el siguiente envío? ¿hay flujo de ajuste manual?).

**Razón:**

- **Refleja el modelo operativo real**: la Entidad es quien tiene visibilidad completa del consumo del TH y la información tributaria; tiene sentido que sea quien calcule. Credibanco como procesador aplica.
- **Evita complejidad innecesaria** en Credibanco: no hay que mantener tabla de UVT vigente, no hay que sincronizar fechas tributarias, no hay que conciliar contra Hacienda.
- **Reusa el patrón ya existente** de novedades por archivo SFTP+PGP: GMF se vuelve un `charge_type` más, lo que reduce código duplicado y simplifica testing.
- **Mantiene el `Charge_Settlement_Service`** como punto único de aplicación atómica: si el saldo no alcanza, el GMF entra a `pending_charge` con la misma vigencia 90 días que cualquier otro cobro diferido. El recobro al detectar abono ya está cubierto por `Recovery_Service`.

**Alternativas descartadas:**

- **Mantener `GMF_Calculator` interno**: rechazado, no refleja el modelo operativo. Implicaría que Credibanco asume responsabilidad tributaria que no le corresponde y tendría que mantener catálogo de UVT.
- **Crear pestaña aparte "Flujo GMF Batch"**: rechazado. El flujo es prácticamente idéntico al de monetarias (SFTP + GAW + Processor + Settlement_Service). Crear pestaña aparte duplicaría componentes en la vista. La pestaña monetaria ya cubre el patrón.
- **Hacer del GMF un tipo de registro dentro de `MXXXddmmcc`** (en lugar de archivo aparte): rechazado por el usuario en la conversación de la decisión. Se mantiene archivo aparte (Opción A confirmada).

**Impacto:**

- Archivo nuevo: `Prepago/PrepagoUnificadoArquitectura_V_1_9.drawio` (14 páginas, igual que V_1_8). Solo cambian las pestañas "Motor de Cobros Diferidos" y "Flujo de novedades monetarias".
- `project-context.md` debe apuntar a `PrepagoUnificadoArquitectura_V_1_9.drawio` como arquitectura principal vigente.
- SQL: `pending_charge.charge_type` ya admite `GMF` (existente desde el inicio). No se agrega `SP_GMF_MONTHLY_BATCH`. El SP unificado `SP_SETTLE_CHARGE` (aún pendiente de implementar — ADR-027) cubre el caso GMF sin cambios adicionales.
- Modelo de datos: añadir `file_type='GMF'` a la tabla `file_processing` cuando el formato del archivo se confirme.
- Cuando la Entidad cierre el formato del archivo, el avance es solo: definir parser, añadir `file_type='GMF'`, ajustar las validaciones específicas. No hay rediseño arquitectónico pendiente.

## ADR-029: PGP delegado a GoAnywhere MFT y análisis de capacidad del Output_File_Generator

**Contexto:** Hasta V_1_9 los diagramas y la documentación atribuían al `Output_File_Generator` (OFG) la responsabilidad de cifrar PGP los archivos de salida y dejarlos cifrados en NFS para que GAW los despachara por SFTP. Esto era incorrecto operativamente: GoAnywhere MFT ya tiene un motor PGP nativo (lo usa para descifrar archivos de entrada de la Entidad) y por simetría está pensado para hacer ambas operaciones — descifrado a la entrada y cifrado a la salida.

Adicionalmente, al revisar la capacidad del OFG ante el volumen estimado (10 entidades × 50.000 registros/archivo de novedades + transversales como AUM con 1.5M–7.5M txn/día) se identificaron tres puntos de ajuste: descifrado de PAN sin cache de DEK, concentración de cron a las 01:00 y OFG como pod único.

**Decisión:**

### A. Delegación de PGP a GAW

1. **OFG escribe archivos PLANOS en NFS de salida** (sin cifrar PGP). Mantiene el descifrado del PAN en memoria justo antes de escribir cada línea (envelope encryption — ADR-006/018).
2. **GAW lee del NFS de salida, cifra PGP con la llave pública de cada Entidad y entrega por SFTP**. GAW ya custodia las llaves PGP de las Entidades para el flujo de entrada; reusa el mismo keyring para salida.
3. **Mitigaciones obligatorias** del NFS de salida (porque contiene archivos planos con PAN durante una ventana corta):
   - Volumen cifrado at-rest (LUKS / dm-crypt o equivalente del storage) en zona CDE.
   - ACL: solo el Service Account del OFG escribe; solo el Service Account de GAW lee y borra.
   - TTL corto: GAW elimina el archivo plano inmediatamente después de cifrar y despachar.
   - Network policy OpenShift: solo namespace OFG (escritura) y host de GAW (lectura) montan el PVC.
   - Audit log de cada operación (escritura, lectura, borrado).

### B. Refuerzos de capacidad del OFG

1. **Cache de DEK por `card_id`** durante la ejecución del job. Sin esto, AUM no termina en la ventana de 01:00 (1.5M–7.5M operaciones RSA-4096 × 3 ms = 1.25 a 6.25 horas). Con cache + ordenar por `card_id`, baja a < 25 minutos.
2. **Escalonar cron** en 4 ventanas:
   - `00:00` SALD (corte de saldo).
   - `00:30` AUM, Presentaciones, Transacciones (los que llevan PAN — más lentos).
   - `01:00` ABO, NOVR, AUSR, RAJU, TSRX, TAR (no llevan PAN o son ligeros).
   - `01:15` COMI, CSAT, CINS.
3. **Deployment con N réplicas particionadas por AFG** (Spring Batch `partitionStep`). 10 AFGs / 4 workers = 4 archivos en paralelo. Reduce ventana 4× y elimina SPOF.

### C. Roadmap diferido (activar bajo trigger de volumen)

- **Streaming PGP / Streaming write**: en GAW (cifrado) y en OFG (escritura sin construir archivo en memoria). Activar cuando el archivo individual supere 1 GB plano.
- **Topic Kafka entre Processors y OFG** para archivos en línea (TNN/TAS/TNM/TAV) con consumer group particionado por AFG, para resolver append concurrente sin file locks.
- **Replay idempotente**: `output_file_log` con `(file_name, afg_id, generated_at, status)` para regenerar cualquier archivo de un día desde BD.

**Cambios aplicados:**

1. **`PrepagoUnificadoArquitectura_V_1_10.drawio`** (V_1_9 conservado como histórico):
   - Pestaña "Flujo de novedades monetarias": descripción de `GoAnywhere MFT` ampliada — entrada (descifra PGP + valida) y salida (lee plano de NFS + cifra PGP + entrega SFTP).
   - Pestaña "Flujo de novedades monetarias": descripción del `Output_File_Generator` cambia de "concentra cifrado PGP" a "escribe archivos PLANOS en NFS. GAW cifra PGP y entrega".
   - Pestañas "Intercambio de Archivos" y "Flujo Novedades No Monetarias": labels de edges OFG → NFS y OFG → GAW corregidos a "Deposita archivos PLANOS en NFS (GAW cifra PGP)".
   - Pestaña "Motor de Cobros Diferidos": descripción del OFG actualizada para reflejar archivos planos.

2. **`requerimientos-novedades-monetarias.md`**:
   - §5: `Output_File_Generator` reescrito ("Descifra PAN en memoria y escribe archivos PLANOS en NFS. GAW cifra PGP y entrega vía SFTP").
   - §5: `GoAnywhere MFT` reescrito (cifra PGP los archivos de salida del OFG antes de despacharlos).
   - §10: tabla de archivos con columna "Cifrado a la Entidad" indicando "PGP por GAW".
   - §10 nota inicial: aclara que el OFG no cifra PGP.
   - §13.1: detalle de cifrado en tránsito + en reposo, con las mitigaciones del NFS de salida (cifrado at-rest, ACL, TTL corto, network policy).

3. **`requerimientos-flujo-transaccion.md`**: sin cambios (la pestaña de transacciones no menciona PGP directamente).

4. **`project-context.md`** (steering): apunta a `PrepagoUnificadoArquitectura_V_1_10.drawio` como arquitectura principal vigente.

**Razón:**

- **GAW ya tiene motor PGP nativo**: usar el mismo motor para entrada y salida elimina código duplicado (BouncyCastle en Java) y aprovecha tuning del producto.
- **Custodia de llaves más limpia**: las llaves públicas de las Entidades viven solo en GAW. No hay que sincronizarlas a un KeyStore en OpenShift ni rotarlas en dos lugares.
- **Failure isolation**: si GAW cae, los archivos planos se acumulan en NFS y se procesan al volver. El OFG no se entera. Antes, si GAW caía con archivos cifrados, había que regenerar.
- **CPU del pod OFG libre**: ~20% de CPU que estaba en BouncyCastle se libera para descifrado de PAN (envelope encryption) y formateo. Esto se suma al beneficio del cache de DEK.
- **Cache de DEK** es obligatorio para AUM. Sin él, la arquitectura no aguanta el volumen actual.
- **Escalonar cron** evita picos de IO/CPU/memoria en el pod a las 01:00 y reduce contention en el read replica.
- **Réplicas particionadas por AFG** elimina el único SPOF que quedaba en la cadena de generación de archivos.

**Alternativas descartadas:**

- **Mantener PGP en OFG**: rechazado por duplicación de motor PGP, custodia de llaves dispersa y CPU innecesaria en el pod.
- **OFG cifra PGP y deja `.pgp` en NFS, GAW solo despacha**: rechazado, era el modelo anterior; pierde los beneficios listados arriba.
- **Pasar todo el contenido por Kafka en lugar de NFS**: considerado, diferido. Funciona para archivos pequeños/incrementales pero un AUM de varios GB no es un buen ciudadano para Kafka. NFS (con cifrado at-rest) sigue siendo la mejor opción para archivos grandes.
- **Evitar el archivo plano en NFS** descifrando dentro de GAW: rechazado porque GAW no tiene acceso a la KEK ni a la lógica de descifrado del envelope encryption. Mover esa lógica a GAW saca a Credibanco del control directo de PCI 3.4.

**Impacto:**

- Archivo nuevo: `Prepago/PrepagoUnificadoArquitectura_V_1_10.drawio` (14 páginas, igual que V_1_9). Cambios localizados en descripciones de OFG y GAW + edges de NFS.
- `project-context.md` apunta a V_1_10 como arquitectura principal vigente.
- Configuración GAW: necesita un job/flow de salida por cada Entidad que monitoree NFS de salida, cifre PGP con la llave pública correspondiente y haga SFTP push. Pendiente operativo.
- Storage: NFS de salida debe configurarse como volumen cifrado at-rest. Pendiente con equipo de infraestructura.
- Implementación OFG: cache de DEK por `card_id` + cron escalonado + Spring Batch `partitionStep` con N réplicas. Es el cambio mayor de implementación tras este ADR.
- Métricas a instrumentar (próximo paso):
  - Tiempo por archivo por AFG.
  - Hit ratio del cache de DEK.
  - Tamaño de archivo plano y comprimido (PGP).
  - SLA: alerta si tiempo > 80% del SLA por archivo.
- ADRs relacionados: ADR-006/018 (envelope encryption del PAN — sigue vigente, el descifrado en memoria sigue en OFG), ADR-026 (cobertura del Flujo de novedades monetarias), ADR-027 (Motor de Cobros Diferidos).

### Aclaración complementaria a ADR-027 (Settlement y Recovery son inversos)

**Contexto adicional:** durante la revisión surgió la pregunta "¿en qué momento el `Charge_Settlement_Service` llama al `Recovery_Service`?". Al analizarla en detalle, la respuesta es que **no debe llamarlo nunca**. La pestaña "Motor de Cobros Diferidos" en V_1_8/V_1_9/V_1_10 tenía un edge erróneo `Charge_Settlement_Service → Recovery_Service` etiquetado "evento abono" que sugería esa relación, pero es incorrecto:

- **Charge_Settlement_Service** procesa **cobros** (débitos diferidos). Su entrada son las 4 fuentes (cuota, comisión, técnicamente exitosa, GMF / cargo de novedad). Si el saldo no alcanza, escribe en `pending_charge`. Aumenta la deuda del TH.
- **Recovery_Service** procesa **abonos** (créditos sobre pendientes). Cuando entra dinero a la cuenta, drena `pending_charge` antiguos. Reduce la deuda del TH.

Son **flujos inversos**. El Settlement no tiene visibilidad de abonos; los abonos los procesa el `Novelty_Orchestrator` (pestaña "Flujo de novedades monetarias") o el `Adjustment_Service` (ajustes Pomelo) o el motor transaccional (reversos).

**Decisión:** corregir la pestaña en `PrepagoUnificadoArquitectura_V_1_11.drawio` (V_1_10 conservado como histórico):

1. **Edge `dce-e-core-rec` eliminado** (queda como entrada invisible en el XML para no romper la numeración interna; no se renderiza en drawio porque tiene `strokeColor=none;visible=0;`).
2. **Descripción del `Recovery_Service` reescrita** para listar los 4 disparadores reales:
   - EVENTO 1 — Abono acreditado: `Novelty_Orchestrator` tras `SP_APPLY_MONETARY` con type=ABONO.
   - EVENTO 2 — Ajuste Crédito de Pomelo: `Adjustment_Service` tras `SP_APPLY_ADJUSTMENT` type=CREDITO.
   - EVENTO 3 — Reverso que devuelve saldo: motor txn (pestaña "Flujo de transacciones") tras `SP_REVERSE_TXN`.
   - BATCH — Catch-all nocturno: `Recovery_Retry_Scheduler` cron 03:00 antigüedad ASC.
   - Nota explícita en la descripción: "el `Charge_Settlement_Service` NO dispara Recovery — los flujos son inversos".
3. **Caja "Disparadores Eventuales" reescrita** dividida en dos secciones:
   - "HACIA Charge_Settlement_Service (cobros entrantes)": Tech_Exitosa_Charge_Engine, Commission_Charge_Engine, Prepaid_Monetary_Novelty_Processor (Cargo / GMF).
   - "HACIA Recovery_Service (abonos entrantes)": Novelty_Orchestrator (Abono), Adjustment_Service (Ajuste_Crédito Pomelo), Motor txn (Reverso).

**Razón:**
- El edge erróneo invitaba a confusión arquitectónica: alguien podría implementar al `Charge_Settlement_Service` como "centro de notificaciones" suscrito a abonos, lo cual es responsabilidad de los componentes que sí procesan abonos.
- Mantener la separación writer/recovery clara facilita testing: cada SP tiene un único dueño (Settlement → SP_SETTLE_CHARGE; Recovery → SP_RECOVER_CHARGES).
- Documentar el origen real de cada disparador del Recovery_Service evita el riesgo de que aparezca un quinto disparador no contemplado durante la implementación (ej. ¿desde el motor txn por reverso? — sí, ya está; ¿desde un ajuste manual del Portal Admin? — no, el portal solo cancela, no acredita).

**Impacto:**
- Archivo nuevo: `Prepago/PrepagoUnificadoArquitectura_V_1_11.drawio` (14 páginas, igual que V_1_10). Solo cambia la pestaña "Motor de Cobros Diferidos".
- `project-context.md` apunta ahora a V_1_11 como arquitectura principal vigente.
- ADR-027 sigue vigente; este bloque es aclaración complementaria.
- Implementación: el equipo debe asegurarse de que el `Novelty_Orchestrator`, `Adjustment_Service` y el motor txn sean los únicos que invoquen `Recovery_Service.recoverByEvent(card_id)`. El `Charge_Settlement_Service` no debe tener referencia a `Recovery_Service` en su código.

## ADR-030: Actualización de lineamientos por Requerimientos Técnicos (13)

**Contexto:** Se entregó una nueva versión del documento de requerimientos (`Requerimientos Técnicos (13).docx`) que reemplaza a la versión (12). El diff entre ambas versiones revela cuatro cambios sustantivos de lineamiento que impactan documentación y arquitectura. Los archivos `ARCHIVOS ENTRADA/SALIDA NUEVO AUTORIZADOR PROCESAMIENTO EMISOR.xlsx` se reentregaron pero mantienen la misma estructura ya documentada en `docs/estructuras-archivos-manual-tecnico.md` (NXXX 600 bytes, MXXX 200 bytes, archivos GMF, etc.).

**Cambios identificados en el docx (13):**

### 1. Horario del archivo de realce (IF) de Pomelo

- **Antes (v12):** "Se genera de manera diaria a las 7:00 am hora ARG → 5:00 am hora COL".
- **Ahora (v13):** "El IF se envía todos los días a las **9:00 AM hora Colombia**".
- **Corte nuevo:** todas las tarjetas creadas hasta las **05:00 AM hora COL del día anterior** entran en el IF de ese día. Las creadas después de las 05:00 AM COL entran en el IF del día siguiente.

**Impacto:** se actualizó `requerimientos-tecnicos-consolidados.md` §5.2 (Realce) y `catalogo-archivos-gaw.md` (archivo Realce por AFG: hora "Tras IF Pomelo 09:00 COL"). El `Prepaid_Embossing_Batch` ahora se dispara tras recibir el IF de las 09:00 COL, no a las 05:00.

### 2. Modelo de activación de tarjeta vía iframe de Pomelo (cambio arquitectónico)

- **Antes (v12):** "El modelo de activación de tarjeta no se realizará directamente por el portal sino por el **i-Frame que se desarrollará internamente**".
- **Ahora (v13):** "El modelo de activación de tarjeta no se realizará directamente por el portal sino **a través de la integración con el i-Frame de Pomelo**".

**Impacto (significativo):**
- Cambia el alcance de **ADR-008** (iframe E2E con HSM Thales propio para captura de PIN). Si la captura/asignación de PIN se delega al iframe de Pomelo, Credibanco ya no desarrolla ni hostea el iframe de PIN. El HSM Thales propio para interchange de PIN podría dejar de ser necesario (a confirmar — depende de si Pomelo asume todo el ciclo del PIN).
- Cambia el alcance de **ADR-010** (activación de innominadas con iframe propio reutilizado). Queda pendiente validar con Pomelo cómo se activan innominadas en su iframe.
- **NO cambia ADR-009** (iframe de visualización de datos sensibles en `secure-data.credibanco.com`): ese iframe es para mostrar PAN/CVV2, es independiente del de PIN. El docx (13) añade explícitamente "Mostrar Datos Sensibles de Tarjeta" como funcionalidad, lo que refuerza ADR-009.
- Se actualizó `requerimientos-ciclo-vida-tarjeta-y-portal-th.md` §6.5 (Activación) con el nuevo lineamiento y referencia a este ADR.

**Pendientes derivados:**
- Confirmar con Pomelo el contrato de integración del iframe de activación/PIN (¿postMessage? ¿redirect? ¿qué llaves?).
- Decidir si el HSM Thales propio sigue siendo necesario o si Pomelo asume el interchange de PIN completo.
- Revisar diagramas `pinpad_pci_drawio_6_mejorado.drawio` y `Activacion_PIN_Innominada.drawio` contra el nuevo modelo (posiblemente pasen a histórico o requieran nueva versión).
- Actualizar la pestaña de activación en el unificado si existiera.

### 3. Archivo de Indicadores Base — detalle ampliado

El docx (13) detalla el archivo `indicadores_baseddmmaaaa.xlsx`:
- Informe en formato Excel, dispuesto automáticamente **mes vencido** en la ruta definida.
- Filtrable por AFG.
- El periodo de la data se relaciona en el nombre: `indicadores_base30042025` (formato `indicadores_baseDDMMAAAA`).
- La columna "total de compras" = compras autorizadas (asociadas o no a una anulación).
- En lo posible, quitar de las operaciones originales las asociadas a un reverso.
- El informe debe tener un espacio para descripción de cada columna.

**Impacto:** se actualizó `estructuras-archivos-manual-tecnico.md` §2.18 (Indicadores Base) con estos detalles.

### 4. Renombre: "Tarjetas Inactivas" → "Tarjetas Bloqueadas"

- **Antes (v12):** la columna del Indicador Base se llamaba "TARJETAS INACTIVAS".
- **Ahora (v13):** "La columna de tarjetas inactivas se llamará **tarjetas bloqueadas**".

**Impacto:** se actualizó `estructuras-archivos-manual-tecnico.md` §2.18.

**Otros cambios menores en el docx (13):**
- Se reordenaron/consolidaron varias líneas de la sección de archivos de salida (CINS, CSAT, Presentaciones, Transacciones, EXT, CANJ, AUMCT, RAJU) — sin cambio de contenido sustantivo, solo de redacción/posición. Ya estaban reflejados en `estructuras-archivos-manual-tecnico.md`.
- Se removió la descripción larga del "Cambio de PIN" como funcionalidad separada (queda implícito en la gestión de PIN del portal). Sin impacto documental.

**Decisión:** actualizar la documentación afectada (hecho) y abrir los pendientes de validación con Pomelo sobre el iframe de activación. El cambio #2 (iframe Pomelo) es el más relevante y requiere decisión de arquitectura sobre el futuro del HSM Thales propio. **Se recomienda no descartar el HSM Thales hasta confirmar formalmente con Pomelo el alcance de su iframe de PIN.**

**Alternativas para el cambio #2:**
- **A) Delegar todo el ciclo de PIN a Pomelo** (iframe + HSM Pomelo): Credibanco no necesita HSM Thales propio. Reduce costo y complejidad pero aumenta dependencia de Pomelo y reduce control sobre el dato PIN.
- **B) Iframe de Pomelo pero con interchange hacia HSM Thales de Credibanco**: híbrido, mantiene el HSM propio. Más control, más complejidad de integración.
- **C) Mantener iframe propio (status quo ADR-008)**: contradice el docx (13). Descartada salvo que la integración con Pomelo no sea viable.

Decisión diferida hasta validación con Pomelo. Por ahora se documenta el lineamiento del docx (13) (opción A o B) y se conservan los diagramas de ADR-008/010 como referencia hasta confirmar.

**Impacto en versionado de diagramas:**
- No se crea versión nueva del unificado en este ADR (los cambios son documentales + pendientes de validación).
- Cuando se confirme el modelo de iframe Pomelo, se creará la versión del unificado correspondiente y se revisará el estado de `pinpad_pci_drawio_6_mejorado.drawio`.

## ADR-031: Eliminación del iframe interno de PIN y del HSM Thales de Credibanco — PIN delegado 100% a Pomelo

**Estado:** Aceptada. **Reemplaza (supersedes) a ADR-008.** Afecta a ADR-010.

**Contexto:** ADR-030 documentó que Requerimientos Técnicos (13) cambió el modelo de activación de tarjeta a "integración con el i-Frame de Pomelo" y dejó abierta la decisión sobre el futuro del HSM Thales propio. La decisión de arquitectura se confirmó: **Credibanco NO requiere ni el iframe interno de PIN ni el HSM Thales propio**. Todo el ciclo del PIN (captura, asignación inicial en activación, cambio, verificación) se delega a Pomelo a través de su iframe y su HSM.

**Decisión:**

1. **Eliminar el iframe interno de captura/cambio de PIN** (el que se planteaba en ADR-008 con Angular + SubtleCrypto). El PIN se captura en el **iframe de Pomelo** integrado en el portal tarjetahabiente.
2. **Eliminar el HSM Thales payShield de Credibanco.** Pomelo asume el interchange de PIN con su propio HSM. Credibanco no genera llaves RSA efímeras, no ejecuta cmd EI/GI/EO, no custodia ZPK/ZMK.
3. **El esquema de llaves de la respuesta a Pomelo (ADR-008 / `Respuesta_Pomelo_PIN_Keys.drawio`) deja de aplicar del lado Credibanco.** La ceremonia TR-34, la ZPK, el PIN block ISO 9564 — todo eso queda del lado de Pomelo. Credibanco no participa en el interchange criptográfico del PIN.
4. **El PIN nunca toca infraestructura de Credibanco.** El portal carga el iframe de Pomelo; el TH digita el PIN dentro de ese iframe (dominio Pomelo); Pomelo lo procesa con su HSM. Credibanco solo orquesta la apertura del iframe y recibe la confirmación del resultado.

**Lo que NO cambia:**

- **Envelope Encryption del PAN (ADR-006/007/018) se mantiene intacto.** El PAN se cifra con AES-256-GCM + RSA-4096 (KEK en KeyStore PKCS12), **sin HSM**. Esta decisión siempre fue independiente del HSM (el HSM era solo para PIN). El descifrado del PAN para generar archivos (TAR) lo hace el `Envelope_Encryption_Service`, no un HSM.
- **El iframe de visualización de datos sensibles (ADR-009) se mantiene.** `secure-data.credibanco.com` para mostrar PAN/CVV2 sigue siendo de Credibanco. Es independiente del iframe de PIN. Requerimientos Técnicos (13) lo refuerza al añadir "Mostrar Datos Sensibles de Tarjeta".
- **SSO, MFA, viewToken, CSP/SRI** siguen igual.

**Razón:**

- **Reduce costo y complejidad:** el HSM Thales payShield es hardware costoso de adquirir, certificar (PCI PIN Security) y operar. Eliminarlo quita una pieza crítica de infraestructura, su ceremonia de llaves, su mantenimiento y su auditoría.
- **Reduce scope PCI:** sin HSM ni iframe de PIN propio, el PIN nunca entra al CDE de Credibanco. El scope PCI PIN Security pasa a Pomelo. Credibanco mantiene scope PCI DSS solo por el PAN (envelope encryption) y los archivos.
- **Coherente con el rol de Pomelo:** Pomelo ya es el pre-autorizador y procesador. Que asuma el ciclo del PIN es consistente con su rol. Pomelo ya tiene HSM certificado.
- **Alineado con Requerimientos Técnicos (13):** el documento de negocio ya define la integración con el iframe de Pomelo.

**Consecuencias:**

- **Diagramas que pasan a HISTÓRICO:**
  - `pinpad_pci_drawio_6_mejorado.drawio` (flujo PIN E2E con HSM Thales).
  - `Flujo_Seguridad_PIN_Iframe_HSM.drawio`.
  - `Respuesta_Pomelo_PIN_Keys.drawio` (esquema de llaves ZPK/ZMK — ahora interno de Pomelo, no de Credibanco).
  - `Iframe_Credibanco_PIN_DatosSensibles.drawio` (ya era exploratorio).
- **Diagrama que ahora aplica:** `Iframe_Pomelo_PIN_DatosSensibles.drawio` (la variante donde Pomelo hostea el iframe — pasa de "descartada" a la opción vigente para el PIN).
- **Unificado:** se crea `PrepagoUnificadoArquitectura_V_1_12.drawio` que elimina las cajas HSM y corrige el descifrado de PAN (debe ser `Envelope_Encryption_Service`, no "HSM cmd M2" — corrección de un error preexistente en el diagrama).
- **ADR-010 (activación innominadas):** la activación de innominadas que reusaba el iframe propio ahora debe reusar el iframe de Pomelo. Pendiente validar con Pomelo cómo maneja innominadas (sin documento asociado) en su iframe.
- **Esquema de llaves PIN:** Credibanco ya no necesita custodiar ZPK/ZMK ni participar en ceremonia TR-34. El correo de respuesta a Pomelo (`correos/respuesta-pomelo-pin-keys.md`) queda como histórico.
- **Steering `project-context.md`:** actualizado — se elimina HSM del stack, decisión #8 reescrita.

**Corrección de error preexistente detectado:**
El diagrama unificado (pestañas de archivos) mostraba que el `Output_File_Generator` descifraba el PAN "vía HSM cmd M2". Esto era **doblemente incorrecto**: (a) el PAN nunca usó HSM (usa envelope encryption — ADR-006), y (b) ahora no hay HSM. La corrección en V_1_12: el OFG descifra el PAN (solo para el archivo TAR, único con PAN en el nuevo modelo) mediante el `Envelope_Encryption_Service` con la KEK del KeyStore.

**Pendientes derivados:**
- Confirmar con Pomelo el contrato de integración de su iframe de PIN (postMessage / redirect / parámetros / manejo de innominadas).
- Confirmar SLA y disponibilidad del iframe de Pomelo (es ahora dependencia crítica para activación).
- Definir el comportamiento del portal si el iframe de Pomelo no está disponible (degradación).
- Revisar el contrato de "confirmación de activación" que Pomelo devuelve al portal tras asignar el PIN.

**Alternativas descartadas:**
- **Mantener HSM Thales propio (status quo ADR-008):** descartado por costo, complejidad y porque Requerimientos Técnicos (13) define la integración con Pomelo.
- **Híbrido (iframe Pomelo + interchange a HSM Thales Credibanco):** descartado — si el iframe es de Pomelo, el PIN ya está en su dominio; pasar el interchange a un HSM de Credibanco añade complejidad sin beneficio.

### Ampliación de ADR-031 — el iframe de Pomelo cubre 4 procesos (PIN + datos sensibles + activación nominada e innominada)

**Contexto adicional:** tras confirmar la eliminación del iframe interno de PIN y del HSM Thales, se precisó que **el alcance del iframe de Pomelo cubre cuatro procesos del tarjetahabiente**, no solo el PIN:

1. **Activación de Tarjeta Nominada** — captura/asignación del PIN inicial.
2. **Cambio de PIN** — captura del PIN actual + nuevo.
3. **Mostrar Datos Sensibles de Tarjeta** — visualización de PAN/CVV2/expiración.
4. **Activación de Tarjeta Innominada** (Fase 2) — asignación de PIN sin documento asociado.

**Decisión ampliada:**

- **Supersede ADR-008** (iframe interno de PIN con HSM Thales) — confirmado.
- **Supersede ADR-009 para el tarjetahabiente** (iframe propio de visualización de datos sensibles en `secure-data.credibanco.com`): la visualización de PAN/CVV2 al TH ahora la hace el **iframe de Pomelo**, no un iframe propio de Credibanco. Credibanco no renderiza datos sensibles al TH.
- **Supersede / pone en revisión ADR-010** (activación de innominadas con Proof of Possession en micro-portal propio): la activación de innominadas se hará mediante el iframe de Pomelo. Pendiente definir con Pomelo el mecanismo para tarjetas sin documento asociado.

**Diagramas que pasan a HISTÓRICO (ampliación):**
- `Iframe_Visualizacion_DatosSensibles.drawio` — la visualización ahora la hace Pomelo.
- `Activacion_PIN_Innominada.drawio` — en revisión; la activación de innominadas va por iframe de Pomelo.
- (ya listados antes) `pinpad_pci_drawio_6_mejorado.drawio`, `Flujo_Seguridad_PIN_Iframe_HSM.drawio`, `Respuesta_Pomelo_PIN_Keys.drawio`, `Iframe_Credibanco_PIN_DatosSensibles.drawio`.

**Diagrama vigente para todos estos flujos:** `Iframe_Pomelo_PIN_DatosSensibles.drawio` (Pomelo hostea el iframe que cubre PIN + datos sensibles).

**Impacto en el descifrado de PAN:**
- Como la visualización del PAN al TH la hace Pomelo, Credibanco **solo descifra el PAN (envelope encryption — ADR-006) para la generación del archivo de salida TAR** (creación de tarjetas), que es el único archivo del nuevo modelo que lleva PAN (ver `estructuras-archivos-manual-tecnico.md` §4). Ya no hay descifrado de PAN para mostrar al TH.
- El `Envelope_Encryption_Service` se mantiene solo para: (a) cifrar el PAN al crear la tarjeta, (b) descifrarlo para el archivo TAR.

**Lo que se mantiene de Credibanco (no va por Pomelo):**
- Onboarding: pre-registro, login, restablecer/cambiar contraseña (SSO Keycloak Realm CARDHOLDER).
- Generación y validación de OTP (`OTP_Service`, ADR-024).
- Notificaciones por correo (`B2B_Mail_Service`, ADR-025).
- Envelope encryption del PAN para storage y archivo TAR.

**Impacto PCI:**
- El scope CDE de Credibanco se reduce aún más: ni PIN ni visualización de PAN/CVV2 al TH pasan por Credibanco.
- Credibanco mantiene scope PCI DSS por: almacenamiento del PAN cifrado (envelope encryption) y generación del archivo TAR con PAN.
- El iframe de Pomelo asume el scope de PIN Security y de visualización de datos sensibles al TH.

**Pendientes derivados (ampliación):**
- Contrato de integración del iframe de Pomelo para los 4 procesos (apertura, contexto, confirmación de resultado).
- Mecanismo de activación de innominadas en el iframe de Pomelo (sin documento asociado).
- CSP/SRI y políticas de embedding del iframe de Pomelo en el portal de Credibanco.
- Manejo de degradación si el iframe de Pomelo no está disponible.

## ADR-032: Lineamientos de implementación del Output File Generator con Spring Batch (escalabilidad para archivos grandes)

**Estado:** Aceptada.

**Contexto:** Surgió la pregunta de si la arquitectura Spring Batch del `Output_File_Generator` (OFG) sirve para generar archivos muy grandes (AUM con 1.5M-7.5M registros/día para 10 entidades). Spring Batch es la herramienta correcta (chunk-oriented, diseñado para volumen), pero el resultado depende críticamente de la implementación: una implementación naïve (cargar todo en memoria) causa `OutOfMemoryError`, mientras que el patrón correcto (streaming) escala sin problema para el volumen objetivo.

**Decisión:** Documentar lineamientos de implementación obligatorios del OFG en `docs/lineamientos-output-file-generator.md`, con tres pilares no negociables:

1. **Streaming en lectura** — `JdbcCursorItemReader` (cursor de BD) contra Oracle Read Replica, con `ORDER BY card_id` y `fetchSize` controlado. Prohibido `findAll()` a una `List`.
2. **Streaming en escritura** — `FlatFileItemWriter` con flush por chunk, escribe archivo PLANO en NFS. Prohibido construir el archivo completo en un `StringBuilder` en memoria.
3. **Particionamiento por AFG** — `partitionStep` (una partición por AFG, pool de 4-8 threads) para paralelismo y eliminación del SPOF.

**Factores favorables del nuevo modelo de archivos (ADR-028 + xlsx):**
- Solo el archivo **TAR** lleva PAN (bajo volumen: 0-50k/día). AUM, FIN, COMI, SALD usan Card ID.
- Por tanto **no hay descifrado criptográfico masivo por registro** en los archivos grandes. El bottleneck histórico (RSA-4096 por tarjeta en AUM = 6 horas) desapareció. El cache de DEK (ADR-029) solo aplica a TAR.

**Cálculo de capacidad (con streaming + partición):**
- Throughput `FlatFileItemWriter` con cursor: 10.000-50.000 registros/seg/thread.
- AUM 7.5M registros: 5-15 min con partición. Cabe en ventana 00:00-06:00.
- Chunk size 1.000-5.000; JVM heap dimensionado al chunk (~1 MB/chunk), no al archivo total.

**Anti-patrones prohibidos (documentados en el lineamiento):**
- `repository.findAll()` con millones de filas → OOM.
- `StringBuilder` con todo el archivo → OOM con archivos GB.
- `JdbcPagingItemReader` con pageSize de decenas de miles.
- Contar registros acumulando la lista en memoria.
- Leer de Oracle Primary (impacta hot-path).
- Cifrar PGP dentro del OFG (lo hace GAW — ADR-029/031).

**Límites y escalamiento futuro:**
- Volumen actual: `partitionStep` local suficiente.
- Archivo individual > 5-10 GB: el cuello pasa a I/O de NFS y PGP de GAW → compresión o split.
- Volumen 10x (75M registros): evaluar `RemotePartitioning`/`RemoteChunking` (workers en pods separados vía broker).
- Archivos en línea (TNN/TAS/TNM/TAV): batch no es streaming en vivo → append incremental o topic Kafka (roadmap ADR-029).

**Razón:**
- Spring Batch chunk-oriented mantiene solo un chunk (N registros) en memoria en cualquier momento, lo que lo hace apto para archivos de millones de registros.
- El riesgo real no es la herramienta sino la implementación. Documentar el patrón correcto + los anti-patrones + un checklist evita que el equipo caiga en el anti-patrón de cargar todo en memoria.
- Los tres pilares (cursor reader, streaming writer, partición) ya están parcial o totalmente en la arquitectura V_1_14; este ADR los formaliza como obligatorios con guía de código.

**Alternativas descartadas:**
- **No documentar (dejar a criterio del desarrollador):** rechazado — el anti-patrón de `findAll()` + StringBuilder es muy común y causa OOM en producción; el costo de documentarlo es bajo y el riesgo de no hacerlo es alto.
- **Usar una herramienta ETL dedicada (ej. Spring Cloud Data Flow, Talend):** rechazado para Fase 1 — Spring Batch es suficiente para el volumen actual, ya está en el stack y no añade infraestructura.
- **Generar archivos vía Stored Procedure de Oracle (UTL_FILE):** considerado, descartado — saca la lógica de formato del código Java al PL/SQL, dificulta testing y versionado, y el descifrado de PAN (TAR) no puede hacerse en Oracle sin exponer la KEK.

**Impacto:**
- Documento nuevo: `Prepago/docs/lineamientos-output-file-generator.md` (guía de implementación + checklist de 16 puntos).
- No cambia diagramas (los lineamientos son de implementación; la pestaña "Generacion Archivos de Salida" de V_1_14 ya refleja la arquitectura).
- El equipo de desarrollo debe seguir el checklist antes de dar por terminado el OFG, incluyendo un test de carga con 7.5M registros antes de producción.
- ADR-029 (capacidad del OFG) queda complementado con este detalle de implementación.

## ADR-033: Limpieza de pestañas borrador para entrega unificada (V_1_16)

**Estado:** Aceptada.

**Contexto:** Las versiones del unificado venían arrastrando pestañas borrador (fragmentos de trabajo sin formato C4 completo: sin título, sin boundaries, cajas sueltas). En V_1_15 quedaban 4: Página-15, Página-16, Página-18 y Página-19. Para la **entrega unificada** se requiere un archivo limpio donde todas las pestañas sean vistas formales de arquitectura. Adicionalmente, en V_1_14 se habían colado 40 páginas en blanco (Página-19 a 58) que ya se eliminaron.

**Decisión:** Crear `PrepagoUnificadoArquitectura_V_1_16.drawio` a partir de V_1_15, eliminando las 4 pestañas borrador. Resultado: **15 pestañas formales**.

**Pestañas eliminadas y justificación:**

| Pestaña | Contenido borrador | Cubierto formalmente en |
|---|---|---|
| Página-15 | Flujo PIN / iframe Pomelo (IFrame CCO, HSM Pomelo, Backend Pomelo) | Pestaña "Portal Tarjetahabiente" (V_1_13) |
| Página-16 | Procesamiento de archivos / horarios (NXXX, resumen diario, horarios batch) | Pestañas "Intercambio de Archivos" + "Generacion Archivos de Salida" |
| Página-18 | Resiliencia / caída con Dynatrace (Tx por hora, Redis, Oracle, caída 2pm) | No formalizado — es observabilidad (hueco pendiente, no se pierde info crítica) |
| Página-19 | Envelope encryption del PAN (KEK/DEK/cifrado/descifrado) | Archivo dedicado `Envelope_Encryption_PAN.drawio` + `docs/envelope-encryption.md` |

**Pestañas conservadas (15 formales):**
Portal · Creacion de tarjetas · Onboarding Tarjetahabiente · Intercambio de Archivos · Flujo Novedades No Monetarias · Realce · Flujo de transacciones · Motor de Cobros Diferidos · Flujo de novedades monetarias · Reconciliacion End-of-Day · Etapa1_V1_2 · Portal Faseado · Estados de Tarjeta · Portal Tarjetahabiente · Generacion Archivos de Salida.

**Razón:**
- La entrega unificada debe contener solo vistas formales de arquitectura, no borradores de trabajo.
- Ninguna información crítica se pierde: 3 de las 4 pestañas ya estaban cubiertas en pestañas formales o archivos dedicados; la 4ª (resiliencia/Dynatrace) es un fragmento de observabilidad que cuando se formalice tendrá su propia pestaña (hueco identificado).

**Alternativas descartadas:**
- Conservar los borradores renombrados como "Draft - X": rechazado para la entrega; los borradores confunden al lector de la entrega final. Quedan disponibles en V_1_15 (histórico) si se necesitan de base.
- Modificar V_1_15 en sitio: rechazado por la convención de no sobreescribir versiones.

**Impacto:**
- Archivo nuevo: `Prepago/PrepagoUnificadoArquitectura_V_1_16.drawio` (15 páginas). Es la versión para entrega unificada.
- `project-context.md` apunta a V_1_16 como arquitectura principal vigente.
- V_1_15 se conserva como histórico (contiene los 4 borradores por si se requieren).
- Pendiente no afectado: los huecos de cobertura siguen vigentes (Reversos/Anulaciones, GMF Batch end-to-end, observabilidad, DR, segmentación de red).

## ADR-034: Reportería y Facturación a Entidades (Req 30) — Billing_Batch + pestaña dedicada

**Estado:** Aceptada.

**Contexto:** El Req 30 "Generación de Reportes para Facturación a Entidades" tenía cobertura parcial: existía el componente `Prepaid_Reporting_API` declarado como caja en las pestañas "Portal" / "Etapa1_V1_2", la fuente de datos (Oracle Read Replica) y el canal de entrega (GAW + B2B_Mail), pero **no había un flujo de arquitectura dedicado** ni documento de requerimiento. El docx (13) detalla 4 reportes mensuales (por AFG, por BIN, Proveedor Realce, Proveedor Distribución) con campos específicos, un modelo de cobro (Tabla 1 cargos por tarjeta activa por rangos + Tabla 2 inactivas + fee de administración $23M) y una definición especial de "tarjeta activa para facturación".

**Decisión:**

1. **Crear el documento** `docs/requerimientos-facturacion-entidades.md` con los 4 reportes, sus campos literales del docx, el modelo de cobro, las definiciones, programación/reintentos y el mapeo de cada campo a su origen en el modelo de datos.

2. **Separar dos rutas** de reportería de facturación:
   - **`Billing_Batch`** (Spring Batch, CronJob mensual): genera los 4 archivos `.xlsx` el primer día del mes (datos del mes anterior), los persiste vía `Output_File_Generator` en NFS, GAW los dispone por SFTP a la carpeta del equipo de Facturación. Notifica éxito/error con `B2B_Mail_Service`, 3 reintentos cada hora.
   - **`Prepaid_Reporting_API`** (REST): consulta on-demand de indicadores desde el portal admin (`/reports/billing/entities`, `/reports/billing/providers`, `/reports/indicators`). Expone JSON, no genera archivos.

3. **Añadir tabla `billing_monthly_snapshot`** (`period, afg_id, bin, ...`, UNIQUE `(period, afg_id, bin)`) para trazabilidad e idempotencia: permite regenerar un reporte sin recalcular y deja registro histórico de lo facturado.

4. **Añadir vista `vw_billing_active_cards`** que implementa la definición de "tarjeta activa para facturación" (≥1 transacción O saldo ≠ 0 en algún momento O generó comisiones), independiente del estado en plataforma.

5. **Crear pestaña "Reporteria y Facturacion"** en `PrepagoUnificadoArquitectura_V_1_17.drawio` (16 páginas) con el flujo C4 L2: Read Replica → Billing_Batch → snapshot → OFG → NFS → GAW → equipo Facturación, más Reporting_API on-demand, B2B_Mail, Audit, y notas de modelo de cobro y definición de tarjeta activa.

**Razón:**
- El Req 30 estaba sin desarrollar como arquitectura. Documentarlo + diagramarlo cierra el hueco de cobertura #4 identificado en el análisis previo.
- Separar Billing_Batch (genera archivos) de Reporting_API (consultas on-demand) respeta el patrón writer/publisher (ADR-026) y permite que el portal admin consulte sin disparar la generación masiva.
- La tabla snapshot da idempotencia y trazabilidad regulatoria (qué se facturó cada mes).
- Leer de Read Replica cumple el lineamiento de no impactar el hot-path (ADR-002/005).
- La definición de "tarjeta activa para facturación" es distinta del estado Activa de la tarjeta; documentarla evita un error de cálculo crítico en la facturación.

**Alternativas descartadas:**
- **Solo usar `Prepaid_Reporting_API` para generar los archivos:** rechazado — la generación mensual masiva es batch (lee millones de transacciones para agregar), no encaja en un API de request/response. Mejor un Batch dedicado.
- **No persistir snapshot (calcular siempre on-the-fly):** rechazado — sin snapshot no hay trazabilidad de lo facturado ni idempotencia; un recálculo podría dar cifras distintas si los datos base cambian.
- **Reusar `Output_File_Generator` como único responsable:** parcialmente adoptado — el OFG escribe los archivos (publisher único), pero la lógica de agregación/cálculo de facturación vive en `Billing_Batch`, no en el OFG genérico.

**Impacto:**
- Documento nuevo: `requerimientos-facturacion-entidades.md`.
- Archivo nuevo: `PrepagoUnificadoArquitectura_V_1_17.drawio` (16 páginas). `project-context.md` apunta a V_1_17.
- Modelo de datos: pendiente implementar `billing_monthly_snapshot` y `vw_billing_active_cards` en `db/01_schema_prepago.sql`.
- Pendientes abiertos (10, listados en el documento): Tabla 1 y 2 de cobro (Producto/Comercial), cálculo de "saldo ≠ 0 en algún momento", origen de PINes innominadas, formato reportes proveedores, stack de BI, carpeta SFTP destino.
- Cobertura del Req 30: pasa de 🟡 parcial a ✅ documentada y diagramada (el cálculo final de la factura sigue siendo del área de Facturación, fuera del alcance de la plataforma).

## ADR-035: Pestaña de Contexto del Sistema (C4 Nivel 1) — vista de contexto del proyecto completo

**Estado:** Aceptada.

**Contexto:** El diagrama unificado contenía únicamente pestañas C4 Nivel 2 (diagramas de contenedores/componentes por flujo: Creación, Realce, Transacciones, Cobros Diferidos, Reportería, etc.), pero **no existía una vista C4 Nivel 1 (Contexto del Sistema)** que mostrara, en una sola página, la plataforma como caja negra rodeada de sus actores (personas) y sistemas externos. Sin esa vista, un lector nuevo (auditor, stakeholder, equipo entrante) no tenía un punto de entrada que resumiera con quién habla el sistema y por qué, antes de entrar al detalle de cada flujo.

**Decisión:**

1. **Clonar `V_1_17` → `V_1_18`** (sin sobrescribir, según convención de versionado) y añadir una pestaña nueva **"Contexto del Sistema (C4 L1)"**.

2. **Modelar la vista de contexto** con:
   - **Sistema central:** `Plataforma Prepago Credibanco` (caja azul C4 — software system on-premise/OpenShift, Java 21 · Spring Boot 3 · Oracle 19c · Redis 7 · Angular 17).
   - **Actores (personas):** Tarjetahabiente, Administrador Credibanco, Operaciones/Facturación, Custodios de Llaves (split knowledge / dual control).
   - **Sistemas externos:** Pomelo (procesador / iframe PIN / HSM Pomelo — relación crítica resaltada en morado), Entidades Emisoras (SFTP+PGP), Franquicias Visa/Mastercard (clearing/reconciliación), Realzador/Embozador, SmartVista (SVBO), Centralizador de Información (TransUnion, GMF), Servidor SMTP corporativo, SSO Keycloak (OIDC/MFA), Dynatrace/SIEM.
   - **Relaciones etiquetadas** con la naturaleza de cada integración (dirección y protocolo/propósito).
   - **Leyenda C4** y **mapa de las pestañas C4 L2** para navegar del contexto al detalle.

3. La pestaña se posiciona como **punto de entrada** del diagrama: contexto (L1) primero, detalle de flujos (L2) después.

**Razón:**
- Completa la jerarquía C4: ya existía abundante L2, faltaba el L1 de contexto. Un modelo C4 sin su nivel de contexto obliga al lector a inferir el panorama global desde los detalles.
- Resalta a Pomelo como integración crítica (decisión central del proyecto: pre-autorizador + delegación de PIN/HSM, ADR-031).
- Da un artefacto único para presentaciones ejecutivas/auditoría PCI sin exponer el detalle interno.

**Alternativas descartadas:**
- **Crear un archivo drawio aparte solo para el contexto:** rechazado — fragmenta la documentación; tener el L1 dentro del unificado mantiene una sola fuente de verdad navegable por pestañas.
- **Sobrescribir V_1_17:** rechazado por convención (nunca sobrescribir; V_1_17 se conserva como histórico).

**Impacto:**
- Archivo nuevo: `PrepagoUnificadoArquitectura_V_1_18.drawio` (19 páginas: 18 heredadas de V_1_17 + la nueva pestaña de contexto). `project-context.md` ahora apunta a V_1_18 como arquitectura principal vigente.
- V_1_17 se conserva como histórico.
- No cambia ningún flujo L2 existente; es una adición puramente aditiva.
- Pendiente menor: V_1_18 arrastra de versiones previas pestañas heredadas ("Portal Faseado", "Etapa1_V1_2", "Página-17", "Página-18") que en una futura entrega podrían depurarse si se confirma que son históricas.

## ADR-036: Corrección de la nota de cobro de la pestaña "Reporteria y Facturacion" — umbral del 50% para tarjetas inactivas (V_1_19)

**Estado:** Aceptada.

**Contexto:** Al verificar la cobertura de la pestaña "Reporteria y Facturacion" (creada en ADR-034) contra el documento `requerimientos-facturacion-entidades.md`, se detectó un único desajuste de contenido: la nota "Modelo de Cobro" del diagrama indicaba *"Cargo por tarjeta inactiva: Tabla 2 (si aplica) — TBD"*, una redacción vaga heredada de la versión inicial. El documento de requerimientos fue actualizado con la regla literal del `Requerimientos Técnicos (15).docx` (párrafo 1078): el cobro por tarjetas inactivas **solo aplica cuando la cantidad de inactivas supera el 50% del total de tarjetas de la entidad en el sistema**. El diagrama quedaba así desalineado con el requerimiento y con el documento.

**Decisión:**

1. **Clonar `V_1_18` → `V_1_19`** (sin sobrescribir, según convención de versionado).
2. **Actualizar la nota "Modelo de Cobro"** de la pestaña "Reporteria y Facturacion": el texto del cargo por tarjeta inactiva pasa de *"Tabla 2 (si aplica) — TBD"* a *"Tabla 2 (valor TBD), SOLO si las inactivas superan el 50% del total de tarjetas de la entidad"*.

**Razón:**
- Es una regla de negocio concreta (umbral del 50%), no un "si aplica" ambiguo. Dejarla vaga en el diagrama induce a un cálculo de facturación incorrecto.
- Alinea el diagrama con el documento de requerimientos y con la fuente (`Requerimientos Técnicos (15).docx`).

**Alternativas descartadas:**
- **Sobrescribir V_1_18:** rechazado por convención (nunca sobrescribir; V_1_18 se conserva como histórico).
- **Dejar el TBD en el diagrama:** rechazado — la condición del 50% ya está definida en el requerimiento; lo único que sigue TBD es el *valor* de la Tabla 2, no la condición de aplicación.

**Impacto:**
- Archivo nuevo: `PrepagoUnificadoArquitectura_V_1_19.drawio` (19 páginas, idéntico a V_1_18 salvo la nota corregida). `project-context.md` apunta ahora a V_1_19 como arquitectura principal vigente.
- V_1_18 se conserva como histórico.
- La pestaña "Reporteria y Facturacion" queda 100% alineada con `requerimientos-facturacion-entidades.md`. El único contenido que permanece TBD son los valores numéricos de las Tablas 1 y 2 (pendiente de Producto/Comercial), no las reglas.

## ADR-037: Lineamientos de implementación del Billing_Batch — agregación en Oracle y cálculo de tarjeta activa

**Estado:** Aceptada.

**Contexto:** Surgió la duda de si Spring Batch sería capaz de procesar y consolidar la información de facturación (millones de transacciones del mes) para generar los reportes del Req 30. El análisis mostró que el reto del `Billing_Batch` es distinto al del `Output_File_Generator`: en el OFG el cuello es el **tamaño del archivo de salida** (millones de líneas → streaming write), mientras que en facturación la salida es **diminuta** (1 línea por AFG/BIN) y el verdadero costo es la **agregación de entrada** (COUNT/SUM/AVG sobre la tabla `transaction`). Sin una guía explícita, el equipo de desarrollo podría caer en el anti-patrón de traer millones de filas a memoria para agregarlas en Java (OutOfMemoryError). Además, un cálculo del requerimiento —"tarjeta activa por saldo ≠ 0 en algún momento del periodo"— no es resoluble con el balance actual y requería diseño.

**Decisión:**

1. **Crear el documento** `docs/lineamientos-billing-batch.md` con el patrón de implementación del `Billing_Batch`.

2. **Principio rector — agregar en Oracle, no en Java:** el `ItemReader` (`JdbcCursorItemReader`) lee el resultado de un `SELECT ... GROUP BY afg_id/bin`; la JVM recibe solo el consolidado (decenas de filas), nunca las transacciones crudas.

3. **Leer de la Read Replica** (ADR-002/005) y **explotar el partition pruning** de la tabla `transaction` (ADR-005): filtro por rango `txn_date >= :start AND < :end` (sin `TRUNC`), verificado con `EXPLAIN PLAN`.

4. **Resolver "tarjeta activa por saldo ≠ 0 en algún momento"** con una tabla nueva `account_daily_balance` (cierre diario de saldo por cuenta, particionada por mes), consultada con `EXISTS` indexado, en lugar de reconstruir el saldo histórico recorriendo `transaction`. La vista `vw_billing_active_cards` consolida las tres condiciones (≥1 txn, saldo ≠ 0 en algún momento, generó comisiones).

5. **Mantener separación writer/publisher (ADR-026):** el `Billing_Batch` calcula y persiste el snapshot; el `Output_File_Generator` escribe el `.xlsx`. Formato `.xlsx` con Apache POI normal (volumen bajo); SXSSF solo si algún reporte creciera.

6. **Idempotencia y trazabilidad** vía `billing_monthly_snapshot` (UNIQUE period/afg/bin): reproceso sin recálculo.

**Razón:**
- Responde con base técnica la pregunta de capacidad: el límite no es Spring Batch, es agregar en Java o invalidar el pruning. Con agregación en BD el batch es de los menos exigentes del sistema.
- El cálculo de "saldo ≠ 0 en algún momento" es el único punto realmente costoso; el snapshot diario lo convierte en un `EXISTS` barato.
- Escalar este batch pasa por dimensionar la Read Replica e índices, no la JVM (lo contrario al OFG).

**Alternativas descartadas:**
- **Agregar en streams de Java:** rechazado — OutOfMemoryError con millones de filas.
- **Reconstruir el saldo histórico desde `transaction` por tarjeta:** rechazado — escaneo costoso; se usa `account_daily_balance`.
- **Escribir el `.xlsx` desde el propio batch:** rechazado — rompe el patrón writer/publisher (ADR-026).

**Impacto:**
- Documento nuevo: `docs/lineamientos-billing-batch.md`. `project-context.md` lo referencia.
- Cierra (a nivel de diseño) el pendiente #5 del requerimiento de facturación ("saldo ≠ 0 en algún momento").
- Modelo de datos: pendiente implementar `account_daily_balance` (nueva), `billing_monthly_snapshot` y `vw_billing_active_cards` en `db/01_schema_prepago.sql`, más un job EoD liviano que pueble el snapshot diario de saldos.
- No cambia diagramas (V_1_19 vigente); es guía de implementación.

## ADR-038: Materialización de `account_daily_balance` + `billing_monthly_snapshot` + `vw_billing_active_cards` (Opción A) y reflejo en V_1_20

**Estado:** Aceptada.

**Contexto:** El ADR-037 definió, a nivel de diseño, que el cálculo de "tarjeta activa para facturación por saldo ≠ 0 en algún momento del periodo" se resolvería con una tabla de cierre diario de saldo (`account_daily_balance`). Sin embargo, esa tabla, junto con `billing_monthly_snapshot` y la vista `vw_billing_active_cards`, **solo existían como propuesta en el lineamiento** (`lineamientos-billing-batch.md`): no estaban en el esquema `db/01_schema_prepago.sql` (fuente de verdad del modelo) ni reflejadas en ningún diagrama. Se evaluaron dos enfoques: (A) tabla de snapshot diario + job EoD dedicado; (B) derivar la condición de `transaction` + snapshot mensual de saldo de apertura. Se eligió **Opción A** por decisión del equipo (consulta de facturación más simple y barata vía `EXISTS` indexado).

**Decisión:**

1. **Materializar en `db/01_schema_prepago.sql`** (sección "Reportería y Facturación"):
   - **`account_daily_balance`** — cierre diario de saldo por cuenta, particionada por mes (partition pruning), con índices por `card_id`/`afg_id`/`(card_id, snapshot_date, eod_balance)`. PK `(snapshot_date, account_id)`.
   - **`billing_monthly_snapshot`** — consolidado mensual por `(period, afg_id, bin_code)` con `report_type` (AFG/BIN), totales de tarjetas/transacciones por canal, `closing_balance` y la bandera `applies_inactive_fee` (regla del 50%, ADR-036). UNIQUE `(period, afg_id, bin_code)`.
   - **`vw_billing_active_cards`** — vista que materializa las tres condiciones de "tarjeta activa para facturación" (≥1 txn aprobada, generó comisiones, saldo ≠ 0 vía `account_daily_balance`), con ventana del mes inmediatamente anterior.

2. **Reflejar en el diagrama (V_1_20, clon de V_1_19):** añadir a la pestaña "Reporteria y Facturacion" los nodos `EoD_Balance_Snapshot_Job` (CronJob EoD diario) y `account_daily_balance` (tabla), con los edges: el job escribe el cierre diario en la tabla, y la tabla alimenta la vista usada por el Billing_Batch / Read Replica.

3. **Introducir el job `EOD_BALANCE_SNAPSHOT`** como nuevo `job_name` en `batch_job` (proceso EoD liviano: 1 fila por cuenta/día).

**Razón:**
- El modelo de datos (SQL) es la fuente de verdad; una estructura propuesta solo en un lineamiento no es parte de la arquitectura hasta materializarse. Esto cierra la inconsistencia detectada.
- Partition pruning mensual en `account_daily_balance` mantiene barata la consulta de "saldo ≠ 0 en algún momento".
- La bandera `applies_inactive_fee` en el snapshot deja explícita la regla del 50% (ADR-036) para el área de Facturación.

**Alternativas descartadas:**
- **Opción B (derivar de `transaction` + snapshot mensual de apertura):** descartada por decisión del equipo a favor de la consulta más simple de la Opción A, asumiendo el costo de un job EoD diario y una tabla adicional.
- **Dejar las estructuras solo en el lineamiento:** rechazado — deja la arquitectura inconsistente (el diagrama y el SQL no reflejaban el diseño).
- **Sobrescribir V_1_19:** rechazado por convención de versionado.

**Impacto:**
- `db/01_schema_prepago.sql`: +2 tablas (`account_daily_balance`, `billing_monthly_snapshot`) y +1 vista (`vw_billing_active_cards`).
- Archivo nuevo: `PrepagoUnificadoArquitectura_V_1_20.drawio` (19 páginas). `project-context.md` apunta a V_1_20. V_1_19 se conserva como histórico.
- Cierra el pendiente #11 del requerimiento de facturación (DDL). Queda abierto el #12: implementar el `EoD_Balance_Snapshot_Job`.
- `account_daily_balance` introduce un proceso EoD diario nuevo a operar y monitorear (costo operativo asumido en la Opción A).

## ADR-039: Documento de requerimiento funcional de Generación de Archivos de Salida

**Estado:** Aceptada.

**Contexto:** La pestaña "Generacion Archivos de Salida" del unificado tenía abundante soporte del lado del "cómo" (`lineamientos-output-file-generator.md` — ADR-032), del catálogo (`catalogo-archivos-gaw.md`) y de la estructura campo-por-campo (`estructuras-archivos-manual-tecnico.md`), pero **carecía de un documento de requerimiento funcional dedicado** (el "qué"): qué archivos de salida debe generar la plataforma, en qué modo, para quién, con qué reglas, programación y criterios de aceptación. Esa información estaba dispersa entre el catálogo, las estructuras y el docx de requerimientos, sin un documento de entrada único — a diferencia de otros flujos (facturación, novedades, cobros) que sí tenían su `requerimientos-*.md`.

**Decisión:**

1. **Crear** `docs/requerimientos-generacion-archivos-salida.md` consolidando el requerimiento funcional: alcance, componente responsable (OFG como publisher único), catálogo funcional de los 25 archivos de salida, modos de generación (en línea / batch diario / batch mensual / por evento), reglas de negocio, manejo del PAN y scope PCI, schedule escalonado, entrega/cifrado/notificación, tolerancia a fallos, criterios de aceptación, cobertura y pendientes.

2. **Mantener la separación de responsabilidades documentales:** este documento es el "qué" (requerimiento); `lineamientos-output-file-generator.md` es el "cómo" (implementación Spring Batch); `estructuras-archivos-manual-tecnico.md` es la estructura exacta; `catalogo-archivos-gaw.md` es el naming/hora/destino. Se referencian entre sí sin duplicar contenido.

3. **Delimitar el alcance frente a facturación:** los reportes de facturación (Req 30) quedan explícitamente fuera (tienen su propio `requerimientos-facturacion-entidades.md` y componente `Billing_Batch`); el OFG es el writer común de ambos.

**Razón:**
- Cierra la inconsistencia de tener el "cómo" sin el "qué" para esta pestaña, alineándola con el resto de flujos del proyecto.
- Da un punto de entrada único para entender el requerimiento de archivos de salida, con criterios de aceptación verificables.
- Consolida reglas críticas ya dispersas: solo TAR lleva PAN, Card ID en lugar de PAN en archivos masivos, un archivo por Entidad, GAW cifra (no el OFG).

**Alternativas descartadas:**
- **Ampliar el lineamiento técnico con el requerimiento:** rechazado — mezcla el "qué" con el "cómo"; se prefiere mantenerlos en documentos separados como en el resto del proyecto.
- **Dejar el requerimiento implícito en catálogo/estructuras:** rechazado — no hay criterios de aceptación ni visión funcional consolidada.

**Impacto:**
- Documento nuevo: `docs/requerimientos-generacion-archivos-salida.md`. `project-context.md` lo referencia.
- No cambia diagramas ni esquema; es documentación de requerimiento.
- Pendientes funcionales registrados (Presentaciones/Transacciones, naming GMF, Indicadores Base, COMI/PAN, naming Realce, retención `output_file`).
