# Inventario de archivos — carpeta Prepago

Inventario completo de los archivos bajo `Prepago/` con descripción, alcance y estado de vigencia. Útil para identificar qué documento es fuente de verdad, cuál es histórico y cuál es transitorio.

> Última actualización: 14/05/2026

---

## Convenciones

| Estado | Significado |
|--------|-------------|
| **Vigente** | Fuente de verdad activa. Se debe mantener actualizada. |
| **Histórico** | Versión anterior conservada por trazabilidad (no editar). |
| **Auxiliar** | Render/imagen/derivado de otro archivo principal. |
| **Plantilla** | Template para producir documentos nuevos. |
| **Temporal** | Archivo de Word/draw.io en uso o residuo (lock, backup automático). Candidato a borrar si nadie lo está editando. |
| **Legacy** | Versión inicial o exploratoria, anterior al diseño actual. |

---

## 1. Arquitectura unificada principal

| Archivo | Descripción | Alcance | Estado |
|---------|-------------|---------|--------|
| `PrepagoUnificadoArquitectura_V_1_5.drawio` | Arquitectura principal vigente (16 páginas C4 L2). v1.5 añade la pestaña "Onboarding Tarjetahabiente" como vista C4 L2 del flujo de pre-registro (ver ADR-023). | Toda la plataforma prepago — portal, ciclo de vida de tarjeta, motor transaccional, archivos de salida, batch, estados, onboarding. | **Vigente** |
| `MER_Prepago_V_1_0.drawio` | Modelo Entidad-Relación v1.0 alineado con `Requerimientos Técnicos (12).docx` y la arquitectura v1.4 (ver ADR-019). | Modelo de datos completo Fase 1 MVP. | **Vigente** |
| `db/01_schema_prepago.sql` | Script DDL Oracle 19c+: tablas, constraints, índices, vistas, particionamiento. Fuente de verdad del esquema. | Modelo de datos. | **Vigente** |
| `db/02_stored_procedures.sql` | SPs del Motor Transaccional: SP_AUTHORIZE_TXN, SP_CHARGE_TECH_EXITOSA, SP_FALLBACK_READ, SP_REVERSE_TXN, SP_CANCEL_TXN. | Motor transaccional. | **Vigente** |
| `PrepagoUnificadoArquitectura_V_1_4.drawio` | Versión anterior conservada por convención. v1.4 añadió "Flujo Novedades No Monetarias" y el flujo de Realce con `B2B_Mail_Service` y `Distribution_Service`. | Igual que v1.5 sin la pestaña de Onboarding. | Histórico |
| `PrepagoUnificadoArquitectura_V_1_4-Flujo de transacciones.png` | Render PNG de la pestaña "Flujo de transacciones" de v1.4. | Solo flujo transaccional. | Auxiliar |
| `PrepagoUnificadoArquitectura_V_1_3.drawio` | Versión anterior de la arquitectura unificada (13 páginas). Conservada por convención de versionado. | Igual que v1.4 pero sin los cambios de ADR-017. | Histórico |
| `PrepagoUnificadoArquitectura_V_1_3 - copia.drawio` | Copia accidental de v1.3. Sin valor documental. | — | Temporal |
| `PrepagoUnificadoArquitectura_V_1_3-Creacion de tarjetas.png` | Render PNG de la pestaña "Creación de tarjetas" en v1.3. | Solo creación de tarjetas. | Auxiliar |
| `PrepagoUnificadoArquitectura_V_1_3-Etapa1_V1_2.png` | Render PNG de la pestaña Etapa 1 en v1.3. | Solo Etapa 1 (Infraestructura Base). | Auxiliar |
| `PrepagoUnificadoArquitectura_V_1_3_Portal.png` | Render PNG de la pestaña Portal en v1.3. | Solo Portal admin/tarjetahabiente. | Auxiliar |
| `PrepagoUnificadoArquitectura_V_1_3_Portal_Etapa1.png` | Render PNG de la pestaña "Portal Faseado" en v1.3. | Solo Portal en faseado por etapas. | Auxiliar |
| `PrepagoUnificadoArquitectura_V_1_2.drawio` | Versión 1.2 de la arquitectura unificada. | Igual que v1.3 pero sin envelope encryption desglosado y sin pre-registro. | Histórico |
| `PrepagoUnificadoArquitectura_V_1_2-Creacion de tarjetas.png` | Render PNG de la pestaña "Creación de tarjetas" en v1.2. | Solo creación de tarjetas. | Auxiliar |

---

## 2. Diagramas C4 / componente especializados

| Archivo | Descripción | Alcance | Estado |
|---------|-------------|---------|--------|
| `Envelope_Encryption_PAN.drawio` | Diagrama de detalle del envelope encryption del PAN: AES-256-GCM (DEK) + RSA-4096 (KEK), KeyStore PKCS12 en OpenShift Secret, ceremonia de generación de KEK con split knowledge. | Persistencia segura del PAN en Oracle. PCI DSS req 3.4 / 3.5 / 3.6.6 / 3.6.7. | Vigente |
| `Envelope_Encryption_PAN.png` | Render PNG del flujo de cifrado/descifrado. | — | Auxiliar |
| `Envelope_Encryption_PAN-Ceremonia Generación KEK.png` | Render PNG de la ceremonia de generación de la KEK con dos custodios. | — | Auxiliar |
| `Iframe_Visualizacion_DatosSensibles.drawio` | Iframe en `secure-data.credibanco.com` para mostrar PAN/CVV2/expiración al TH. Portal padre nunca recibe los datos. | Visualización de datos sensibles desde portal tarjetahabiente. PCI DSS req 3.3 / 6.4.3 / 7.2 / 8.3.6. | Vigente |
| `Iframe_Visualizacion_DatosSensibles.png` | Render PNG. | — | Auxiliar |
| `Iframe_Credibanco_PIN_DatosSensibles.drawio` | Variante alterna de iframe (versión Credibanco). | Captura de PIN y datos sensibles — variante exploratoria. | Histórico |
| `Iframe_Pomelo_PIN_DatosSensibles.drawio` | Variante alterna de iframe (versión Pomelo). | Captura de PIN y datos sensibles si Pomelo hosteara el iframe — opción descartada. | Histórico |
| `Flujo_Seguridad_PIN_Iframe_HSM.drawio` | Flujo de seguridad PIN end-to-end con iframe + HSM (Thales). | Captura y cambio de PIN por parte del TH. | Vigente |
| `pinpad_pci_drawio_6_mejorado.drawio` | Flujo definitivo PIN con HSM Thales (cmd EI/GI/EO). | Captura/cambio de PIN E2E — implementación Fase 1. | **Vigente** |
| `pinpad_pci_drawio_6_mejorado.png` | Render PNG. | — | Auxiliar |
| `Respuesta_Pomelo_PIN_Keys.drawio` | Diagrama de respuesta a Pomelo sobre el esquema de llaves para intercambio del PIN: 2 capas (RSA efímero + ZPK simétrica), ceremonia TR-34, rotación TR-31, PIN block ISO 9564 Format 4, controles PCI. | Sustento técnico para reuniones con Pomelo. Soporta ADR-008. | **Vigente** |
| `pinpad_pci_drawio_V_6.png` | Otro render PNG del mismo diagrama. | — | Auxiliar |
| `pinpad_pci_drawio_5.drawio` | Versión anterior del flujo PIN. | — | Histórico |
| `pinpad_pci_drawio_5 - copia.drawio` | Copia residual de la versión 5. | — | Temporal |
| `pinpad_pci_V1_0.drawio` | Versión 1.0 del flujo PIN — primera iteración. | — | Histórico |
| `PIN_Selection_Online.drawio` | Flujo de selección de PIN online por parte del TH. | Selección/cambio de PIN desde el portal. | Vigente |
| `Activacion_PIN_Innominada.drawio` | Activación de PIN para tarjetas innominadas vía Proof of Possession (BIN + last4 + exp + CVV2). | Activación de tarjetas innominadas — micro-portal público (ADR-010). | **Vigente** |
| `Activacion_PIN_Innominada.png` | Render PNG. | — | Auxiliar |
| `PreRegistro_Tarjeta_Activacion_Portal.drawio` | Flujo de pre-registro con UUID, TTL 30 días, link por email, verify cédula, OTP, password. | Onboarding de TH para tarjetas nominadas. | Vigente |
| `Onboarding_Tarjetahabiente.drawio` | Diagrama del flujo de onboarding TH (alimenta ADR-016). | Pre-registro y enrolamiento del TH. | Vigente |
| `Onboarding_Tarjetahabiente.png` | Render PNG. | — | Auxiliar |
| `creacion_masiva_tarjetas.drawio` | Flujo batch de creación masiva de tarjetas desde archivo NXXXddmmcc. | Etapa 2 — Ciclo de vida de la tarjeta. | Vigente |
| `creacion_masiva_tarjetas - copia.drawio` | Copia residual. | — | Temporal |
| `creacion_masiva_tarjetas_onprem.drawio` | Variante on-prem del flujo de creación masiva. | Variante para despliegue on-prem. | Histórico |
| `Intercambio_Archivos_SFTP_PGP.drawio` | Intercambio de archivos cifrados con entidades vía SFTP + PGP (GoAnywhere MFT). | Recepción de NXXX/MXXX y envío de TNN/TAS/TNM/TAV/AUM/etc. | Vigente |
| `Intercambio_Archivos_SFTP_PGP-Envio Archivos SFTP+PGP.png` | Render del envío. | — | Auxiliar |
| `Intercambio_Archivos_SFTP_PGP-Recepcion Archivos SFTP+PGP.png` | Render de la recepción. | — | Auxiliar |
| `Rules_Engine_Multibolsillo.drawio` | Diagrama exploratorio de motor de reglas para multibolsillo y tarjeta amparada. | Diferido — pendiente definición funcional. | Legacy |
| `PrepagoCreacionTarjetas.png` | Render histórico de creación de tarjetas. | — | Auxiliar |
| `PrepagoIntercambio de Archivos.png` | Render histórico de intercambio de archivos. | — | Auxiliar |

---

## 3. Documentos de concepto (entregables a negocio/seguridad)

| Archivo | Descripción | Alcance | Estado |
|---------|-------------|---------|--------|
| `Concepto_Motor_Transaccional_V_1_0.docx` | Concepto formal del Motor Transaccional Fase 1 MVP Etapa 3 (basado en v1.4 del unificado). | Autorización POS/ATM, comisiones, GMF, técnicamente exitosas, reversos/anulaciones, auditoría. | **Vigente** |
| `Concepto_Ciclo_Vida_Tarjeta_V_1_0.docx` | Concepto formal del Ciclo de Vida de la Tarjeta (basado en pestañas "Creacion de tarjetas", "Realce", "Intercambio de Archivos" del unificado v1.4). | Creación, realce, distribución, bloqueo definitivo/temporal/desbloqueo, modificación de datos. Etapa 2 Fase 1 MVP. | **Vigente** |
| `ConceptoCorePrepagoFase1_V_1_2.docx` | Concepto Core Prepago Fase 1 — versión 1.2. | Visión global Fase 1 MVP. | Vigente |
| `ConceptoCorePrepagoFase1_V_1_2.pdf` | PDF del concepto Core Fase 1 v1.2. | — | Auxiliar |
| `ConceptoCorePrepagoFase1_V_1_1.docx` | Versión 1.1 del concepto Core. | — | Histórico |
| `ConceptoCorePrepagoFase1_V_1_1.pdf` | PDF de la versión 1.1. | — | Auxiliar |
| `ConceptoCorePrepagoFase1.pdf` | Versión inicial del concepto Core (PDF). | — | Histórico |
| `ConceptoEtapa3.docx` | Concepto enfocado en Etapa 3 (Motor Transaccional). | Igual scope que `Concepto_Motor_Transaccional_V_1_0.docx`. | Vigente (consolidar con el v1.0) |
| `ConceptoPrepagoFase3.docx` | Concepto Fase 3 (futuro). | Funcionalidades fuera de Fase 1 MVP. | Vigente |
| `ConceptoPrepagoFase3.pdf` | PDF Fase 3. | — | Auxiliar |
| `ConceptoActivaciónPINTarjetasPrepagoInnominadas.docx` | Concepto de la activación de PIN para tarjetas innominadas. | Soporta `Activacion_PIN_Innominada.drawio` y ADR-010. | Vigente |
| `ConceptoActivaciónPINTarjetasPrepagoInnominadas.pdf` | PDF del concepto. | — | Auxiliar |
| `Concepto_Iframe_Seguro_Captura_y_Cambio_PIN_Plataforma Prepago.docx` | Concepto del iframe seguro para captura y cambio de PIN. | Soporta `pinpad_pci_drawio_6_mejorado.drawio` y ADR-008. | Vigente |
| `Concepto_Iframe_Seguro_Captura_y_Cambio_PIN_Plataforma Prepago.pdf` | PDF. | — | Auxiliar |
| `ConceptoIframeVisualizaciónDatosSensibles.docx` | Concepto del iframe de visualización de PAN/CVV2 en dominio separado. | Soporta `Iframe_Visualizacion_DatosSensibles.drawio` y ADR-009. | Vigente |
| `ConceptoIframeVisualizaciónDatosSensibles.pdf` | PDF. | — | Auxiliar |
| `ConceptoArquitectura_PIN_Iframe.md` | Concepto en markdown del iframe de PIN. | Versión markdown del concepto del iframe PIN. | Vigente |
| `ConceptoDatosSensiblesTarjetaCreación.docx` | Concepto del manejo de datos sensibles en la creación de tarjetas (envelope encryption). | Soporta `Envelope_Encryption_PAN.drawio` y ADR-006/ADR-007. | Vigente |
| `ConceptoDatosSensiblesTarjetaCreación.pdf` | PDF. | — | Auxiliar |
| `ConceptoGeneraciónArchivosPANParaEntidades.docx` | Concepto de generación de archivos con PAN para entrega a entidades emisoras. | Salida de PAN cifrado a entidades vía SFTP+PGP. | Vigente |
| `ConceptoGeneraciónArchivosPANParaEntidades.pdf` | PDF. | — | Auxiliar |
| `ConceptoOnboardingTarjetahabientesPrepago.docx` | Concepto del onboarding del TH (pre-registro, OTP, password). | Soporta `Onboarding_Tarjetahabiente.drawio` y ADR-016. | Vigente |
| `ConceptoOnboardingTarjetahabientesPrepago.pdf` | PDF. | — | Auxiliar |

---

## 4. Documentación interna (`Prepago/docs/`)

| Archivo | Descripción | Alcance | Estado |
|---------|-------------|---------|--------|
| `docs/decisiones-arquitectura.md` | Bitácora de ADRs (ADR-001 a ADR-019). Documento vivo. | Toda la plataforma. | **Vigente** |
| `docs/envelope-encryption.md` | Detalle técnico del envelope encryption (algoritmos, esquema Oracle, ceremonia KEK, rotación). | Cifrado de PAN. Soporta ADR-006/ADR-007/ADR-018. | **Vigente** |
| `docs/requerimientos-tecnicos-consolidados.md` | **Documento maestro** que consolida `Requerimientos Técnicos (12).docx` con todos los ADR-001 a ADR-024. Es el índice de entrada para entender el proyecto. | Toda la plataforma — alcance, stack, 5 etapas, transversales, modelo, PCI, performance, observabilidad, despliegue, pendientes. | **Vigente** |
| `Cronograma_Implementacion_Prepago_V_1_0.xlsx` | Cronograma de implementación con 6 hojas: Resumen Ejecutivo, Cronograma Maestro (Gantt 14 sprints), Componentes (esfuerzo con/sin Kiro), Hitos por mes, Asignación de Equipo, Riesgos y Pendientes. | Plan de ejecución Fase 1 MVP, ~4.5 meses con Kiro. | **Vigente** |
| `docs/requerimientos-flujo-transaccion.md` | Extracto consolidado de los requerimientos del Motor Transaccional desde `Requerimientos Técnicos (12).docx`. Incluye §10 Límites consolidados. | Motor transaccional Fase 1 MVP Etapa 3. | **Vigente** |
| `docs/requerimientos-ciclo-vida-tarjeta-y-portal-th.md` | Extracto consolidado de los requerimientos de Etapa 2 (creación, realce, distribución, bloqueo, modificación) y Etapa 4 (Portal TH back/front). | Ciclo de vida de tarjeta + Portal TH Fase 1 MVP. | **Vigente** |
| `docs/requerimientos-onboarding-tarjetahabiente.md` | Requerimiento puntual del flujo de onboarding (5 fases): pre-registro, verify cédula, OTP, password, SSO. Incluye contratos API, DDL, máquinas de estado, errores, plantillas, auditoría y SLAs. | Onboarding TH (Etapa 4 Fase 1 MVP). | **Vigente** |
| `docs/modelo-entidad-relacion.md` | Explicación del MER v1.0 + mapeo requerimiento-entidad + cumplimiento PCI. | Modelo de datos Oracle. | **Vigente** |
| `docs/poc-envelope-encryption-requerimientos.md` | Requerimientos para POC del envelope encryption. | POC pre-implementación de envelope. | Vigente |
| `docs/inventario-archivos-prepago.md` | Este documento. | Inventario y trazabilidad de archivos. | **Vigente** |

---

## 5. Requerimientos y entradas de negocio

| Archivo | Descripción | Alcance | Estado |
|---------|-------------|---------|--------|
| `Requerimientos Técnicos (12).docx` | Versión 12 del documento de requerimientos técnicos por etapas (Fase 1 MVP Etapa 1-5). | Toda la plataforma — fuente primaria de requerimientos. | **Vigente** |
| `Requerimientos Técnicos.docx` | Versión inicial del documento de requerimientos técnicos. | — | Histórico |
| `Flujos Funcionales.xlsx` | Hoja con flujos funcionales del producto. | Referencia funcional (negocio). | Vigente |
| `Novedades.xlsx` | Catálogo de novedades monetarias y no monetarias. | Codificación de novedades / archivos NXXX/MXXX. | Vigente |
| `EstadoArquitecturas.xlsx` | Estado de avance de las arquitecturas. | Seguimiento. | Vigente |
| `formatoConcepto.docx` | Plantilla oficial para documentos de concepto de arquitectura empresarial. | Producir nuevos conceptos. | **Plantilla** |
| `POC_Iframe_DatosSensibles_Requerimiento.md` | Requerimiento del POC del iframe de datos sensibles. | POC pre-implementación. | Vigente |
| `CorreoSI.docx` | Correo de Seguridad de la Información — comunicación interna. | Histórico de comunicaciones SI. | Histórico |

---

## 6. Carpeta `Prepago/On-prem/` — variantes y versiones tempranas

| Archivo | Descripción | Alcance | Estado |
|---------|-------------|---------|--------|
| `On-prem/MER_Prepago_Inicial.drawio` | Modelo Entidad-Relación inicial del producto prepago. | Modelado de datos en Oracle. | Vigente |
| `On-prem/ArquitecturaUnificadaPrepago.drawio` | Iteración temprana de la arquitectura unificada (anterior a v1.1). | — | Legacy |
| `On-prem/PrepagoUnificadoArquitectura.drawio` | Otra iteración temprana sin numeración de versión. | — | Legacy |
| `On-prem/PrepagoUnificadoArquitectura_V_1_1.drawio` | Arquitectura unificada v1.1. | — | Histórico |
| `On-prem/PrepagoUnificadoArquitectura_V_1_2.drawio` | Arquitectura unificada v1.2 (copia bajo On-prem). | Igual que `Prepago/PrepagoUnificadoArquitectura_V_1_2.drawio`. | Histórico |
| `On-prem/PrepagoUnificadoArquitectura_V_1_1-Estados de Tarjeta.png` | Render de estados de tarjeta en v1.1. | — | Auxiliar |
| `On-prem/C4-autorizacion-transaccional.drawio` | Diagrama C4 separado para autorización transaccional (precursor de la pestaña "Flujo de transacciones" del unificado). | — | Legacy |
| `On-prem/C4-creacion-tarjetas.drawio` | Diagrama C4 separado para creación de tarjetas (precursor). | — | Legacy |
| `On-prem/C4-portales-admin-tarjetahabiente.drawio` | Diagrama C4 separado para portales (precursor). | — | Legacy |
| `On-prem/C4-recarga-tarjetas.drawio` | Diagrama C4 separado para recarga de tarjetas. | — | Legacy |
| `On-prem/pinpad_pci_V_6.png` | Render PNG del pinpad versión 6 — copia bajo On-prem. | — | Auxiliar |
| `On-prem/Requerimientos Técnicos (2).docx` | Versión 2 del documento de requerimientos. | — | Histórico |

---

## 7. Carpeta `Prepago/Inicial/` — exploración inicial con Pomelo

| Archivo | Descripción | Alcance | Estado |
|---------|-------------|---------|--------|
| `Inicial/Prepago_Pomelo_V2_1.drawio` | Iteración 2.1 de la integración con Pomelo (exploratoria). | Diseño inicial Credibanco-Pomelo. | Legacy |
| `Inicial/Prepago_Pomelo_V2_0.drawio` | Iteración 2.0. | — | Legacy |
| `Inicial/Prepago_Pomelo_V1_0.drawio` | Iteración 1.0 — primer diseño. | — | Legacy |

---

## 8. Archivos temporales / de bloqueo (candidatos a borrar)

Generados automáticamente por Word y draw.io. **No tienen valor documental.** Se pueden borrar de forma segura siempre que nadie tenga el archivo abierto.

| Archivo | Origen | Estado |
|---------|--------|--------|
| `~$ncepto_Motor_Transaccional_V_1_0.docx` | Lock de Word del docx en uso. | Temporal |
| `~$nceptoCorePrepagoFase1_V_1_2.docx` | Lock de Word. | Temporal |
| `~$nceptoEtapa3.docx` | Lock de Word. | Temporal |
| `~WRL0005.tmp` | Temp de Word. | Temporal |
| `Inicial/.$Prepago_Pomelo_V1_0.drawio.bkp` | Backup automático draw.io. | Temporal |
| `Inicial/.$Prepago_Pomelo_V2_0.drawio.bkp` | Backup automático draw.io. | Temporal |
| `Inicial/.$Prepago_Pomelo_V2_1.drawio.bkp` | Backup automático draw.io. | Temporal |
| `On-prem/.$MER_Prepago_Inicial.drawio.bkp` | Backup automático draw.io. | Temporal |
| `On-prem/.$PrepagoUnificadoArquitectura_V_1_1.drawio.bkp` | Backup automático draw.io. | Temporal |
| `On-prem/.$PrepagoUnificadoArquitectura_V_1_2.drawio.bkp` | Backup automático draw.io. | Temporal |
| `PrepagoUnificadoArquitectura_V_1_3 - copia.drawio` | Copia accidental. | Temporal |
| `creacion_masiva_tarjetas - copia.drawio` | Copia accidental. | Temporal |
| `pinpad_pci_drawio_5 - copia.drawio` | Copia accidental. | Temporal |

---

## Resumen ejecutivo

| Categoría | Vigentes | Históricos / Legacy | Auxiliares (PNG/PDF) | Plantillas | Temporales |
|-----------|---------:|--------------------:|---------------------:|-----------:|-----------:|
| Arquitectura unificada | 1 | 2 | 6 | 0 | 1 |
| Diagramas componente | 9 | 7 | 9 | 0 | 2 |
| Documentos de concepto | 9 | 2 | 9 | 0 | 0 |
| Documentación interna (md) | 5 | 0 | 0 | 0 | 0 |
| Requerimientos / negocio | 5 | 2 | 0 | 1 | 0 |
| Carpeta `On-prem/` | 1 | 11 | 1 | 0 | 3 |
| Carpeta `Inicial/` | 0 | 3 | 0 | 0 | 3 |
| Locks y `.tmp` | 0 | 0 | 0 | 0 | 4 |
| **Total** | **30** | **27** | **25** | **1** | **13** |

---

## Recomendaciones

1. **Borrar temporales**: los 13 archivos temporales (`~$*`, `~WRL*.tmp`, `.$*.bkp`, `* - copia.*`) se pueden eliminar cuando nadie los esté editando.
2. **Consolidar `ConceptoEtapa3.docx`** con `Concepto_Motor_Transaccional_V_1_0.docx` — son del mismo scope (Etapa 3 = Motor Transaccional).
3. **Mover a una carpeta `legacy/`** los diagramas de la carpeta `On-prem/` y `Inicial/` que están como Legacy. Reduciría el ruido al navegar `Prepago/`.
4. **Renombrar PDFs auxiliares** con prefijo `_render_` o moverlos a `Prepago/exports/` para no mezclarlos con los .docx fuente.
5. **Versionar PNGs** de la arquitectura unificada solo cuando publiques una versión hito (v1.4 ya tiene su PNG, v1.5+ debería seguir el mismo patrón).
