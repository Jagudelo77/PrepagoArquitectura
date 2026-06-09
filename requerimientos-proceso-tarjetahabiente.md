# Requerimientos — Proceso del Tarjetahabiente (Portal Web Prepago)

Documento consolidado de los requerimientos del proceso del tarjetahabiente (TH) en la plataforma prepago Credibanco: onboarding (pre-registro, login, restablecer/cambiar contraseña), activación de tarjeta, gestión de PIN, visualización de datos sensibles y autoservicio (consulta de saldo, historial, bloqueo/desbloqueo).

**Fuentes:**
- `Prepago/Requerimientos Técnicos (13).docx` — capítulos "Portal Web Prepago - Back/Front (Onboarding)" y "Etapa 4: Portal Tarjeta-Habiente".
- `Prepago/docs/requerimientos-onboarding-tarjetahabiente.md` — detalle del pre-registro (ADR-016, ADR-023, ADR-024, ADR-025).
- `Prepago/docs/requerimientos-ciclo-vida-tarjeta-y-portal-th.md` — secciones 6 y 7.
- `Prepago/docs/decisiones-arquitectura.md` — ADR-009 (iframe datos sensibles), ADR-016/023/024/025 (onboarding), ADR-030 (Req. Técnicos 13), ADR-031 (PIN delegado a Pomelo).

> Fase: **Fase 1 — Línea Base (MVP)**, Etapa 4 (Portal Tarjetahabiente).
> Última actualización: 04/06/2026.
> Estado: **Vigente** — incorpora ADR-031 (eliminación de iframe interno de PIN y HSM Thales).

---

## Tabla de contenido

1. [Resumen y alcance](#1-resumen-y-alcance)
2. [Modelo de iframe de Pomelo — alcance](#2-modelo-de-iframe-de-pomelo--alcance)
3. [Onboarding — Pre-Registro](#3-onboarding--pre-registro)
4. [Login](#4-login)
5. [Restablecer contraseña](#5-restablecer-contraseña)
6. [Cambio de contraseña](#6-cambio-de-contraseña)
7. [Activación de Tarjeta Nominada](#7-activación-de-tarjeta-nominada)
8. [Cambio de PIN](#8-cambio-de-pin)
9. [Mostrar Datos Sensibles de Tarjeta](#9-mostrar-datos-sensibles-de-tarjeta)
10. [Activación de Tarjeta Innominada (Fase 2)](#10-activación-de-tarjeta-innominada-fase-2)
11. [Autoservicio — Consulta de saldo, historial, bloqueo (Fase 2)](#11-autoservicio-fase-2)
12. [Front-End](#12-front-end)
13. [Seguridad y cumplimiento](#13-seguridad-y-cumplimiento)
14. [Pendientes](#14-pendientes)
15. [Referencias cruzadas](#15-referencias-cruzadas)

---

## 1. Resumen y alcance

### 1.1 Objetivo

Permitir que el tarjetahabiente gestione de forma autónoma y segura su tarjeta prepago a través del Portal Web: registrarse, autenticarse, activar su tarjeta, asignar y cambiar el PIN, ver datos sensibles y consultar su información.

### 1.2 Alcance funcional (Fase 1 MVP — Etapa 4)

**Incluye:**
- Pre-registro (antes del login) con verificación de identidad + OTP.
- Login con documento + contraseña.
- Restablecer contraseña (olvido).
- Cambio de contraseña (con contraseña actual).
- Activación de tarjeta nominada (vía iframe de Pomelo).
- Cambio de PIN (vía iframe de Pomelo).
- Mostrar datos sensibles de tarjeta (PAN/CVV2 vía iframe de visualización en `secure-data.credibanco.com`).

**Fase 2 (Complementos y Diferenciales):**
- Activación de tarjeta innominada.
- Consulta de saldo, historial de transacciones, bloqueo/desbloqueo, consulta de estado de tarjeta, extractos.

### 1.3 Resultado esperado

- El TH tiene un usuario en SSO Keycloak (Realm CARDHOLDER) asociado a su `client_id`.
- Puede autenticarse con documento + contraseña.
- Puede activar su tarjeta y asignar PIN a través del iframe de Pomelo.
- Puede ver sus datos sensibles sin que el portal padre los reciba.
- Hay traza completa en `audit_log` y `pci_event_log`.

---

## 2. Modelo de iframe de Pomelo — alcance

> **Decisión arquitectónica clave (ADR-031 + ampliación):** los siguientes procesos del tarjetahabiente se realizan **100% a través del iframe de Pomelo**, no por desarrollo propio de Credibanco:
> - **Activación de Tarjeta Nominada** (§7)
> - **Cambio de PIN** (§8)
> - **Mostrar Datos Sensibles de Tarjeta** (§9 — PAN/CVV2)
> - **Activación de Tarjeta Innominada** (§10)

Credibanco **NO desarrolla iframe propio** (ni para PIN ni para visualización de datos sensibles) **ni usa HSM Thales**.

Implicaciones:
- El portal tarjetahabiente integra el **iframe de Pomelo** para los 4 procesos anteriores.
- El **PIN** se digita dentro del iframe (dominio Pomelo), nunca toca infraestructura de Credibanco. Pomelo lo procesa con su HSM.
- La **visualización de PAN/CVV2** se renderiza dentro del iframe de Pomelo. El portal padre de Credibanco nunca recibe los datos sensibles; Pomelo los muestra directamente al TH.
- Credibanco solo orquesta la apertura del iframe (con el contexto de tarjeta/usuario) y recibe la **confirmación del resultado** (activación exitosa, PIN cambiado, etc.).
- El bloqueo por PIN errado (3 intentos) lo gestiona CCO/Credibanco según la respuesta de Pomelo.

**Impacto sobre ADR-009:** el iframe propio de visualización de datos sensibles (`secure-data.credibanco.com`) **queda superseded para el tarjetahabiente** — la visualización ahora la hace el iframe de Pomelo. El `Iframe_Visualizacion_DatosSensibles.drawio` pasa a histórico para este flujo. Ver ADR-031 (ampliación).

**Impacto sobre el descifrado de PAN:** como la visualización del PAN al TH la hace Pomelo, Credibanco solo necesita descifrar el PAN (envelope encryption — ADR-006) para la generación del archivo de salida **TAR** (creación de tarjetas). No para mostrarlo al TH.

---

## 3. Onboarding — Pre-Registro

**Objetivo:** que el TH realice un pre-registro antes del login que le permita ver y gestionar su tarjeta. El TH ingresa al link de pre-registro (recibido al crear la tarjeta), selecciona tipo de documento, digita su número de documento y correo registrado, recibe una OTP por correo, la valida y asigna su contraseña.

### 3.1 Criterios de aceptación

- **Tipos de documento:** Cédula de ciudadanía, Cédula de extranjería, Permiso de protección temporal (PPT), Tarjeta de identidad.
- La OTP se envía **únicamente al correo electrónico registrado por la Entidad**.
- Template del mensaje en HTML con URL acortada.
- **Solo se envía correo con URL de pre-registro a usuarios nuevos** (primera creación de tarjeta).
- Si el usuario ya está registrado → enviar correo con URL de acceso directo al portal.
- Datos que viajan en la URL de registro: **Tipo de Documento + Número de Documento + correo registrado**.
- Validar que los datos del TH estén correctos.
- **Máximo 3 intentos** de ingreso de información de pre-registro incorrectos → bloqueo de acceso.
- El bloqueo por datos errados es **definitivo**; el usuario debe comunicarse con la Entidad.
- **Reintentos de solicitud de OTP: máximo 3** → bloqueo de solicitudes por 24 horas (con mensaje).
- **Autorizaciones de OTP: máximo 3** → bloqueo del usuario por 24 horas.
- **Duración de la OTP: 5 minutos.**
- Botón de reenvío de OTP (duración máxima 5 minutos, basada en hora de envío de la última OTP).
- En activación de usuario, bloqueo de solicitudes de OTP y autorización de OTP → enviar correo con el estado del proceso:
  - Inicio del proceso de pre-registro.
  - Confirmación del registro del usuario.
  - Bloqueo de solicitudes de OTP.
  - Duración de 5 minutos de OTP.
  - Bloqueo de autorización de OTP.
- Toda la mensajería por correo construida en HTML.
- **Política de contraseña:** mínimo 12 caracteres, 1 mayúscula, 1 número, 1 carácter especial.
- Validar que la contraseña cumpla requisitos de SI (estructura, duración); si no, informar al TH con el detalle del error del Back-End.
- En caso de falla al recibir la OTP, el TH notifica por una URL; validar la causa y habilitar la opción a las 24 horas siguientes.
- **Reenvío automático del link de pre-registro** cuando la Entidad modifica el correo electrónico del TH y este no está registrado en el portal.
- Guardar la data de usuarios registrados exitosamente y de los que no logran registrarse (con su causa).
- Auditoría: fecha de modificación, modificaciones y usuario que ejecuta los cambios.

> **Detalle ampliado:** ver `requerimientos-onboarding-tarjetahabiente.md` (contratos API, máquinas de estado, tablas `preregistration`/`otp`/`app_user`). El orquestador es `PreRegistration_Service`; el `OTP_Service` solo genera/valida códigos (ADR-024, ADR-025).

---

## 4. Login

**Objetivo:** autenticar a un TH existente usando número de documento + contraseña asignada en el pre-registro.

### 4.1 Criterios de aceptación

- El TH ingresa su número de documento con teclado numérico.
- Los datos no van en claro (no visibles mientras se ingresan).
- La contraseña debe estar ofuscada, con opción **Mostrar** durante registro/login.
- Si el TH no puede ingresar → informar el detalle del error (del Back-End) por modal.
- Si digita documento o contraseña errados → informar detalle del error por modal.
- Las contraseñas **no se capturan por consola**.
- Notificar por correo cada vez que el usuario ingresa a la plataforma.
- Tokens de roles manejados con **UUID o similar**.
- **Duración del token: 30 minutos** → al vencer, solicitar nuevo ingreso.
- **Máximo 3 intentos** de login incorrectos → bloqueo de acceso por 24 horas.

---

## 5. Restablecer contraseña

**Objetivo:** permitir al TH crear una nueva contraseña en caso de olvido.

### 5.1 Criterios de aceptación

- Contraseña ofuscada con opción Mostrar.
- Ante bloqueo de contraseña, comunicarse con el SSO para habilitar la opción.
- Si digita documento o contraseña errados → informar detalle del error por modal.
- Datos que viajan en la URL de restablecimiento: **Tipo de Documento + Número de Documento + correo registrado**.
- Validar que los datos del TH estén correctos.
- **Máximo 3 intentos** de restablecimiento incorrectos → bloqueo de acceso.
- Validar que la contraseña cumpla requisitos de SI; si no, informar el detalle del error.
- El bloqueo por datos errados es **definitivo**; el usuario debe comunicarse con la Entidad.
- Las contraseñas no se capturan por consola.
- En "¿olvidó su contraseña?" → conexión para dirigir al TH a su correo registrado.
- Tras verificar correo → enviar mail con URL para restablecer, diligenciando **BIN + 4 últimos dígitos de la tarjeta**.

---

## 6. Cambio de contraseña

**Objetivo:** permitir al TH cambiar su contraseña desde el portal, con requisito de ingresar la contraseña actual.

### 6.1 Criterios de aceptación

- Link dentro del portal "¿desea cambiar la contraseña?".
- Flujo: digitar contraseña actual → digitar nueva contraseña → confirmar nueva contraseña (siguiendo lineamientos de SI).
- Diseñar template y notificar al correo con la respuesta del proceso.
- **Máximo 3 intentos** de cambio incorrectos → bloqueo de acceso.
- Validar que la contraseña cumpla requisitos de SI; si no, informar detalle del error.
- El bloqueo por datos errados de la contraseña actual es **definitivo**; el usuario debe comunicarse con la Entidad.
- Las contraseñas no se capturan por consola.

---

## 7. Activación de Tarjeta Nominada

**Descripción.** Proceso de activación de tarjeta por el TH a través del Portal. **No se puede realizar la activación por otro medio por norma PCI**, ya que requiere asignación del PIN. **El modelo de activación NO se realiza directamente por el portal sino a través de la integración con el i-Frame de Pomelo** (ADR-031).

### 7.1 Criterios de aceptación

- Validar que la tarjeta pertenezca a un **AFG de tarjetas físicas** (no virtuales).
- Validar que el estado de la tarjeta sea **"Pendiente por Activar"**.
- La activación solo se puede realizar **una única vez** con la asignación del PIN.
- Contar con un usuario creado con toda la información requerida para la activación por el portal.
- Almacenar fecha y detalle de procesamiento de cada movimiento administrativo.
- Auditoría: fecha de modificación, modificaciones y usuario.

### 7.2 UX de activación

- El TH activa la tarjeta diligenciando el **número del PAN**.
- Una vez diligenciado el PAN, se muestra la opción **"asigna tu clave"** → abre el iframe de Pomelo para capturar el PIN.
- Criterios de aceptación de UX detallados pendientes de refinar (marcados como `xxx` en el doc fuente).

---

## 8. Cambio de PIN

**Descripción.** Proceso de cambio de PIN por el TH (vía iframe de Pomelo).

### 8.1 Criterios de aceptación

- Validar que la tarjeta pertenezca a un **AFG de tarjetas físicas** (no virtuales).
- Validar que el estado de la tarjeta sea **"Activo"**.
- Contar con un usuario creado con toda la información requerida para el cambio de PIN por el portal.
- Si no se tiene el PIN actual, al digitar **3 veces el PIN errado** (campo paramétrico) → la tarjeta se bloquea **definitivamente**. Notificar al TH por correo electrónico o SMS.
- Almacenar fecha y detalle de procesamiento de cada movimiento administrativo.
- Auditoría: fecha de modificación, modificaciones y usuario.
- Cuando se bloquee la tarjeta por PIN errado → reflejar la respuesta en los archivos de salida con la causal **"PIN Errado"**.

---

## 9. Mostrar Datos Sensibles de Tarjeta

**Descripción.** Funcionalidad para que el TH visualice datos sensibles de su tarjeta (PAN, CVV2, fecha de expiración). **Se realiza a través del iframe de Pomelo** (ADR-031 ampliado). Credibanco no renderiza estos datos; el iframe de Pomelo los muestra directamente al TH.

### 9.1 Criterios de aceptación

- Validar que la tarjeta pertenezca a un **AFG de tarjetas físicas** (no virtuales).
- Validar que el estado de la tarjeta sea **"Activo"**.
- Contar con un usuario creado con toda la información requerida.
- Requiere **MFA step-up** antes de abrir el iframe (PCI Req 8.3.6).
- El portal padre de Credibanco **nunca recibe** PAN ni CVV2 — los muestra el iframe de Pomelo.
- Credibanco orquesta la apertura del iframe (contexto de tarjeta) y recibe confirmación, no los datos.
- Auditoría: fecha, usuario y evento de visualización (sin registrar los valores sensibles).

> **Cambio vs versión anterior:** se planteaba un iframe propio en `secure-data.credibanco.com` (ADR-009). Con la ampliación de ADR-031, la visualización la hace el iframe de Pomelo. El `Iframe_Visualizacion_DatosSensibles.drawio` pasa a histórico para este flujo.

---

## 10. Activación de Tarjeta Innominada (Fase 2)

**Descripción.** Activación de tarjetas que no tienen documento asociado al momento de la creación. Pertenece a **Fase 2**.

### 10.1 Consideraciones

- La activación requiere asignación de PIN, que se realiza **a través del iframe de Pomelo** (ADR-031).
- La activación de innominadas también se hará **mediante el iframe de Pomelo** (no un micro-portal propio).
- **Pendiente:** validar con Pomelo cómo se activan tarjetas sin documento asociado dentro de su iframe (mecanismo de identificación/posesión de la tarjeta).
- El mecanismo propio de Proof of Possession (ADR-010) queda **en revisión / superseded** por el iframe de Pomelo.

---

## 11. Autoservicio (Fase 2)

**Portal web Prepago - Back/Front:** Consulta de Saldo, Historial de Transacciones, Bloqueo y Desbloqueo, Consulta de Estado de Tarjeta, Extractos.

Pertenece a **Fase 2 — Complementos y Diferenciales**. Detalle pendiente de especificación.

---

## 12. Front-End

**Descripción.** Procesos Front-End de onboarding, gestión de PIN y activación de tarjeta.

### 12.1 Criterios de aceptación

- **Todos los procesos del Front deben consumir los procesos del Back-End** para garantizar su funcionamiento.
- UX detallada de Activación, Cambio de PIN y flujos de Pre-Registro/Login/Restablecer/Cambiar contraseña marcada como `xxx` en el doc fuente — **pendiente de refinamiento**.
- Stack: Angular 17+.

---

## 13. Seguridad y cumplimiento

| Requisito PCI | Aplicación |
|---|---|
| Req 3.2 | PIN no se almacena (vive solo en HSM de Pomelo) |
| Req 3.3 | PAN enmascarado por defecto (last4) en Credibanco; visualización completa solo en iframe de Pomelo |
| Req 3.3.2 | CVV2 no se almacena en Credibanco; lo muestra el iframe de Pomelo |
| Req 3.4 | PAN no legible en storage de Credibanco (envelope encryption — ADR-006); se descifra solo para archivo TAR |
| Req 4.2.1 | TLS 1.3 en canales |
| Req 6.4.3 | CSP + SRI en la integración del iframe de Pomelo |
| Req 7.2 | Acceso mínimo: MFA step-up antes de abrir iframe sensible |
| Req 8.3.6 | MFA antes de acceso a datos sensibles |
| Req 10.2 | Auditoría de cada operación del TH (sin registrar valores sensibles) |

**Controles de seguridad transversales:**
- Contraseñas: bcrypt/argon2 en SSO Keycloak.
- Tokens: UUID, TTL 30 min.
- Bloqueos: 3 intentos en pre-registro/login/restablecer/cambio.
- OTP: 5 min de vigencia, máximo 3 solicitudes y 3 autorizaciones.
- PIN: delegado a Pomelo, nunca en infra Credibanco (ADR-031).

---

## 14. Pendientes

| # | Pendiente | Origen | Responsable |
|---|---|---|---|
| 1 | UX detallada de Activación, Cambio de PIN, Pre-Registro, Login, Restablecer/Cambiar contraseña (marcados `xxx`) | docx §736-739 | Producto + UX |
| 2 | Contrato de integración del iframe de Pomelo (postMessage / redirect / parámetros) | ADR-031 | Pomelo + Arquitectura |
| 3 | Activación de tarjetas innominadas dentro del iframe de Pomelo | docx §1182 | Pomelo + Producto |
| 4 | Confirmación de la situación del iframe (cambio de PIN) — marcado "Ok Pendiente" en docx | docx §1317 | Pomelo |
| 5 | Comportamiento del portal si el iframe de Pomelo no está disponible (degradación) | Análisis técnico | Arquitectura |
| 6 | Detalle de Fase 2: consulta de saldo, historial, bloqueo/desbloqueo, extractos | docx §1180-1181 | Producto |
| 7 | Confirmar contrato de "confirmación de activación" que Pomelo devuelve al portal | ADR-031 | Pomelo |

---

## 15. Referencias cruzadas

### 15.1 Documentos

- `requerimientos-onboarding-tarjetahabiente.md` — detalle del pre-registro (contratos, máquinas de estado, errores).
- `requerimientos-ciclo-vida-tarjeta-y-portal-th.md` — secciones 6 y 7 (Back/Front).
- `decisiones-arquitectura.md` — ADR-009, ADR-016, ADR-023, ADR-024, ADR-025, ADR-030, ADR-031.

### 15.2 Diagramas

- `Onboarding_Tarjetahabiente.drawio` — flujo de onboarding por fases.
- `PrepagoUnificadoArquitectura_V_1_12.drawio` — pestaña "Onboarding Tarjetahabiente" (vista C4 L2).
- `Iframe_Visualizacion_DatosSensibles.drawio` — iframe de datos sensibles (ADR-009).
- `Iframe_Pomelo_PIN_DatosSensibles.drawio` — iframe de Pomelo para PIN (vigente tras ADR-031).

### 15.3 ADRs aplicables

- **ADR-009** — Iframe en dominio separado para datos sensibles (vigente).
- **ADR-016** — Subproceso de pre-registro tras creación de tarjeta nominada.
- **ADR-024** — `OTP_Service` como componente reusable.
- **ADR-025** — `B2B_Mail_Service` + verificación dual de identidad.
- **ADR-030** — Actualización de lineamientos por Requerimientos Técnicos (13).
- **ADR-031** — Eliminación del iframe interno de PIN y HSM Thales; PIN delegado a Pomelo.

---

> **Nota de versionado:** este documento consolida el proceso del tarjetahabiente. El detalle del pre-registro vive en `requerimientos-onboarding-tarjetahabiente.md`. Las modificaciones se registran como ADRs en `decisiones-arquitectura.md`.