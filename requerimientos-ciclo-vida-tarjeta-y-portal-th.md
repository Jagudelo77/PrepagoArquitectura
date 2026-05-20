# Requerimientos — Ciclo de Vida de la Tarjeta y Portal Tarjetahabiente

Extracto literal y consolidado del documento `Prepago/Requerimientos Técnicos (12).docx` para los siguientes capítulos:

1. Procesamiento Novedades No Monetarias — **Creaciones** (Archivo / API)
2. **Realce**
3. **Distribución**
4. Procesamiento Novedades No Monetarias — **Bloqueo Definitivo (Deshabilitado), Bloqueo Temporal y Desbloqueo** (Archivo / API)
5. Procesamiento Novedades No Monetarias — **Modificación de Datos** (Archivo / API)
6. **Portal Web Prepago — Back** (Onboarding, Gestión de PIN, Activación)
7. **Portal Web Prepago — Front** (Onboarding, Gestión de PIN, Activación)

> Fuente: Fase 1 — Línea Base (MVP), Etapa 2 (Ciclo de Vida) y Etapa 4 (Portal Tarjetahabiente).
> Última extracción: 14/05/2026.

---

## 1. Procesamiento Novedades No Monetarias — Creaciones (Archivo / API)

**Descripción.** Se requiere realizar el proceso de Creación de tarjetas físicas y virtuales a través de archivo y/o de APIs. Aplica para tarjetas Virtuales y Físicas.

### 1.1 Criterios de Aceptación — Tarjeta Física

- Se debe garantizar el canal de solicitud en el caso de ser archivos (GAW-OpenShift, API).
- Si la solicitud es por archivo, se debe validar la estructura del archivo. Tener en cuenta que el archivo se recibe cifrado por parte de la Entidad.
- Se debe garantizar que el AFG exista, esté asociado al NIT correspondiente y a la Entidad correspondiente, y que el `User ID` tenga tarjeta en estado válido (Creada, Activa, Bloqueo Temporal).
- La solicitud de creación por API se realiza uno a uno.
- Para las creaciones por archivo, se deben generar los archivos de respuesta a las Entidades (**TNN** y **TAS**) según la estructura del "Manual Técnico Prepago". Son archivos **incrementales** — se actualizan a medida que se procesan nuevos archivos. Cifrados.
- Para las creaciones por API, la Entidad debe enviar la información según la estructura del "Manual Técnico Prepago Visa" para Novedades No Monetarias (Numeral 2.1). *Pendiente: validar si se deben incluir campos adicionales por Pomelo y Credibanco.*
- Para las creaciones por API, se debe generar respuesta de creación exitosa vía JSON con código y descripción del estado. Adicionalmente se entrega al emisor: número de la tarjeta, PAN, CVV2 y fecha de expiración, una vez creada en línea. **Cifradas.**
- Por API o archivo la Entidad debe indicar si la tarjeta es **nominada o innominada**.
- Se debe incluir de manera **obligatoria** el correo electrónico en el campo 37 (estructura del Manual Técnico Prepago Visa, Numeral 2.1).
- Validación de fecha del nombre del archivo `NXXXddmmcc` (XXX = AFG, dd = día, mm = mes, cc = consecutivo): la fecha registrada **no debe ser superior ni anterior a seis (6) días calendario**.
- No se debe procesar dos (2) veces el mismo archivo (validación por nombre del archivo + registro de control).
- Guardar nombres de archivo y registros de control para validación; depuración trimestral.
- **No se puede crear más de una tarjeta para un mismo tipo y número de identificación (User ID) en un mismo AFG.** Si el TH tiene una tarjeta en estado distinto a "Deshabilitada" o "Expirada" en el AFG, no se permite crear otra.
- Validar y guardar si la tarjeta es nominada o innominada (campo en BD y en API).
- Toda la información discrecional del TH debe almacenarse en BD.
- Almacenar fecha y detalle de procesamiento de cada movimiento administrativo y financiero.
- Marcar tarjetas exentas del cobro de impuestos GMF (posición 440) para físicas y virtuales.
- Finalizado el procesamiento del archivo no monetario, para tarjetas físicas el estado debe quedar como **"Creada"**.
- Finalizada la creación de cada tarjeta en la API de Pomelo, validar si el usuario está registrado:
  - Si está registrado: enviar correo electrónico de ingreso al portal.
  - Si no: consumir el servicio de registro (crea el usuario en el SSO) y enviar correo con la URL de pre-registro.
- Reenvío de correo en caso de fallas (validar con Arquitectura).
- Si el correo del TH ya existe en la BD, notificar que el correo ya se encuentra registrado.
- Auditoría: dejar registro de fecha de creación / modificaciones y usuario que ejecuta los cambios.
- Garantizar que se realice el proceso y validaciones que actualmente ejecuta GAW.
- Las tarjetas virtuales y físicas se deben diferenciar en cada AFG.
- La información del procesamiento se debe guardar para reportería.
- Se deben tener flujos y reglas de validación de creación para **multibolsillo y tarjeta amparada**.
- *Pendiente: complementar con detalle de la revisión de GAW.*

### 1.2 Criterios de Aceptación — Tarjeta Virtual

- **Pago con OTP**.
- La solicitud de creación de tarjetas por la Entidad se realiza por API.
- Todas las tarjetas virtuales nacen en estado **"Activo"**.
- Creación uno a uno.
- Las virtuales y físicas se diferencian en cada AFG.
- Información de procesamiento se guarda para reportería.
- Respuesta en línea con la información: PAN, CVV, fecha de expiración. *Pendiente: ¿qué otros campos se requieren? — Validación Desarrollo.*
- *Pendiente: validar si la tarjeta virtual puede generar compras físicas — Validación Producto.*
- Finalizado el procesamiento del archivo no monetario, las virtuales quedan en estado **"Activa"**.
- Mismos archivos de salida que tarjeta física.
- Los abonos se realizan con el mismo proceso de novedades monetarias que las físicas.
- Tras crear cada tarjeta en Pomelo, validar usuario registrado y disparar correo (login o pre-registro) de manera análoga a la física.
- Reenvío de correo ante fallas.
- *Pendiente: si se requiere transacción distinta a VNP, contar con el alcance — Validación Producto.*
- *Pendiente: cómo se entrega el CVV al TH (portal u otro medio a la Entidad).*
- *Pendiente: cómo se genera una tarjeta física a partir de una virtual — Refinar; definir responsables ante reclamaciones (validar proceso de primera compra).*

---

## 2. Realce

**Descripción.** Se requiere realizar la extracción diaria de la información de las tarjetas a embozar, entendiendo el archivo de realce entregado por Pomelo.

**Criterios de Aceptación:**

- La información extraída del archivo base debe generar **un archivo por AFG**.
- Crear un consecutivo que identifique el grupo de tarjetas por AFG (Pedido).
- Cada archivo se nombra con: consecutivo, identificador del AFG, fecha de procesamiento y nombre del Realzador.
- Respetar la estructura de variables expuesta por Pomelo. Credibanco solo genera los archivos por AFG.
- Los archivos se disponen en GAW para envío a cada Entidad o Realzador.
- La disposición de archivos por AFG se hace en los horarios definidos en el flujo actual.
- Actualizar el estado de la tarjeta en BD con la generación y envío del archivo de realce.
- Al final del proceso, generar reporte con:
  - Fecha de Procesamiento
  - Consecutivo del Pedido
  - ID del AFG
  - Nombre del AFG
  - Cantidad de tarjetas
  - Entidad
  - Embozador
- Si hay errores de embozado notificados por el proveedor, informar a la Entidad para bloqueo de las tarjetas creadas y solicitud de nuevas.
- Al enviar el archivo al proveedor, el estado de las tarjetas queda en **"Pendiente por Activar"**.
- Auditoría de creación / modificaciones.
- Realizar las mismas validaciones que GAW ejecuta hoy.
- Horario de disposición / generación / entrega del archivo de realce por Pomelo. **RTA:** se genera diariamente a las **07:00 hora ARG → 05:00 hora COL**.
- Si un archivo se dispone fuera de horarios, se procesa en el corte del día siguiente.
- *Pendiente: ¿el archivo de Pomelo tiene registro de control?*

---

## 3. Distribución

**Descripción.** Se requiere completar la información de distribución, teniendo en cuenta el archivo de realce y la data del AFG.

**Criterios de Aceptación:**

- Al final del proceso, generar reporte con:
  - Fecha de Procesamiento
  - Consecutivo del Pedido
  - ID del AFG
  - Nombre del AFG
  - Cantidad de tarjetas
  - Entidad
  - Embozador
  - Ciudad, Dirección, Teléfono
  - Nombre Custodio 1 tarjeta / Doc. Custodio 1 tarjeta
  - Nombre Custodio 2 tarjeta / Doc. Custodio 2 tarjeta
- Los tiempos de procesamiento del archivo de distribución son los mismos del archivo de realce.
- La información se toma de los sistemas de Credibanco.
- Auditoría de creación / modificaciones.
- Garantizar el canal de entrega de los archivos de entrada y salida.
- Realizar las mismas validaciones que GAW ejecuta hoy.

---

## 4. Procesamiento Novedades No Monetarias — Bloqueo Definitivo (Deshabilitado), Bloqueo Temporal y Desbloqueo (Archivo / API)

**Descripción.** Realizar el proceso de Bloqueo Definitivo, Temporal y Desbloqueo a través de archivo y/o de APIs. Aplica para tarjetas Virtuales y Físicas.

**Criterios de Aceptación:**

- Garantizar canal de solicitud en caso de archivos (GAW-OpenShift).
- Validar estructura si es por archivo. El archivo viene cifrado por la Entidad.
- Validar que el AFG exista, asociado al NIT y Entidad correctos, y que el `User ID` tenga tarjeta en estado válido (Creada, Activa, Bloqueo Temporal).
- Solicitud por API: uno a uno.
- Bloqueos por archivo: generar archivo de respuesta a las Entidades (**NOVR**) según el "Manual Técnico Prepago". Es **batch** y va cifrado.
- Bloqueos / desbloqueos por API: respuesta JSON con código y descripción del estado de la transacción.
- Si la Entidad envía **"Bloqueo Definitivo (Deshabilitado)"**, se debe procesar **independientemente del estado** de la tarjeta.
- Validación de fecha del nombre del archivo `NXXXddmmcc`: la fecha no debe ser superior ni anterior a 6 días calendario.
- No procesar dos (2) veces el mismo archivo (nombre + registro de control).
- Guardar nombres y registros de control; depuración trimestral.
- Almacenar fecha y detalle de procesamiento de cada movimiento administrativo.
- Auditoría de fecha de Bloqueo / Desbloqueo y modificaciones.
- En **bloqueo definitivo (Deshabilitación)**, tener en cuenta el saldo con el que queda la tarjeta — insumo del archivo de salida **NOVR**.
- Realizar el proceso y validaciones que GAW ejecuta hoy.
- *Pendiente: complementar con detalle de la revisión de GAW.*
- Considerar otros movimientos administrativos posibles (ej. portal administrativo en Postman).

---

## 5. Procesamiento Novedades No Monetarias — Modificación de Datos (Archivo / API)

**Descripción.** Realizar el proceso de modificación de datos a través de archivo y/o de APIs. Aplica para tarjetas Virtuales y Físicas.

**Criterios de Aceptación:**

- Canal por archivo (GAW-OpenShift).
- Si es por archivo, validar estructura.
- Validar que el AFG exista, asociado al NIT y Entidad correctos, y que el `User ID` tenga tarjeta en estado válido (Creada, Activa, Bloqueo Temporal).
- Modificación por API: uno a uno.
- Modificaciones por archivo: generar archivos de respuesta a las Entidades (**TNN** y **TAS**) según el "Manual Técnico Prepago". Incrementales y cifrados.
- Modificación por API: respuesta JSON con código y descripción del estado.
- Validación de fecha del nombre del archivo `NXXXddmmcc`: no superior ni anterior a 6 días calendario.
- Si en la modificación se cambia el correo electrónico del TH:
  - Validar si el usuario está registrado.
  - Si está registrado: enviar correo de ingreso al portal.
  - Si no: consumir servicio de envío de correo con URL de pre-registro.
- Si el correo del TH ya existe en BD, notificar que el correo ya se encuentra registrado.
- Los datos modificados deben actualizarse en Pomelo.
- El campo nominada / innominada **es modificable**.
- Marcar tarjetas exentas del cobro de impuestos GMF (posición 440 — Novedad No Monetaria).
- Marcar tarjetas exentas de GMF generadas por solicitud del TH a través de la Entidad.
- No procesar dos (2) veces el mismo archivo.
- **NIT, AFG, Tipo y Número de Identificación NO son modificables.** Si la Entidad solicita modificar alguno, se requiere un nuevo desarrollo.
- Guardar nombres de archivo y registros de control; depuración trimestral.
- Toda la información discrecional del TH debe ser almacenada.
- Auditoría de modificación de datos y modificaciones (usuario y fecha).
- Auditoría específica de exoneración de GMF solicitada por la Entidad.
- Realizar el proceso y validaciones que GAW ejecuta hoy.
- *Pendiente: complementar con detalle de la revisión de GAW.*

---

## 6. Portal Web Prepago — Back (Onboarding, Gestión de PIN, Activación)

**Descripción.** Definir los procesos Back-End de onboarding, gestión de PIN y activación de tarjeta.

### 6.1 Pre-Registro

**Objetivo.** El TH realiza un pre-registro antes del log-in que le permite ver y gestionar su tarjeta. El TH ingresa al link de pre-registro recibido al crear la tarjeta, selecciona tipo de documento (Cédula ciudadanía, Cédula extranjería, Permiso de protección temporal, Tarjeta de identidad), digita su número de documento y correo registrado. Se muestra un Pop Up que indica revisar el correo donde recibió la **OTP** y se habilita un campo para digitarla. Si la OTP es correcta, se habilita la asignación de contraseña con un Pop Up que indica los parámetros (mín. 12 caracteres, 1 mayúscula, 1 número, 1 carácter especial). Si la asignación es correcta, aparece un Pop Up con la URL de ingreso al portal con número de documento + contraseña.

**Criterios de Aceptación:**

- La OTP se envía únicamente al correo electrónico registrado por la Entidad.
- Construir el template HTML con la URL acortada.
- Solo se envía correo con URL de pre-registro a usuarios nuevos (primera creación de tarjeta).
- Validar existencia del usuario; si ya está registrado, enviar correo con URL de acceso directo. *Pendiente: validar plantillas con UX.*
- En la URL de pre-registro viajan: Tipo de Documento, Número de Documento y correo registrado.
- Validación de datos del TH para garantizar correctitud.
- Solo se permiten **3 intentos** de ingresos con datos incorrectos; al superarlos, se bloquea el acceso.
- Bloqueo por datos errados: **definitivo**, el usuario debe comunicarse con la Entidad.
- Reintentos de solicitud de OTP: máximo **3** en 24 horas → bloqueo de solicitudes por 24 horas (con mensaje).
- Autorizaciones de OTP: máximo **3**; al superarlas, se bloquea el usuario por 24 horas.
- En activación de usuario, bloqueo de solicitudes de OTP, autorización de OTP, enviar correo con el estado del proceso:
  - Inicio del proceso de pre-registro
  - Confirmación del registro del usuario
  - Bloqueo de solicitudes de OTP
  - Duración de **5 minutos** de la OTP
  - Bloqueo de autorización de OTP
- Toda la mensajería por correo debe estar en HTML.
- Considerar todas las respuestas que envíe el back-end.
- Botón para que el usuario solicite re-envío de OTP con duración máxima de 5 minutos. La actualización del tiempo toma como base la hora de envío de la última OTP.
- En caso de falla al recibir OTP, el TH notifica vía URL en la página; gestionar validación de la causa y habilitar la opción a las 24 horas siguientes.
- Validar que la contraseña cumpla requisitos de SI (estructura, duración); si no cumple, informar al TH con detalle del error desde el Back-End. *Validar con Seguridad de la Información.*
- Si al TH no le llega o no funciona el link de pre-registro y la Entidad modifica el correo electrónico, **reenviar el link automáticamente al nuevo correo**.
- Si en actualización de datos (correo electrónico) se identifica que la persona tiene tarjeta creada y no está registrada, reenviar correo de registro.
- Conocer y guardar la data de los usuarios registrados exitosamente.
- Conocer y guardar la data de los usuarios que no logren registrarse y la causa.
- Auditoría de fecha de modificación de datos y usuario.

### 6.2 Login

**Objetivo.** Autenticar a un TH existente en el portal con número de documento y contraseña asignada en el pre-registro. El TH ingresa su número de documento con el teclado numérico de su dispositivo. Los datos no van en claro al ingresarse.

**Criterios de Aceptación:**

- La contraseña va ofuscada, pero permite visualización mediante acción "Mostrar" cuando el TH lo requiera. *Validar con SI.*
- Si el TH no puede ingresar al portal, informar por modal el detalle del error según el Back-End.
- Si el TH digita número de identificación o contraseña errados, informar por modal el detalle desde el Back-End.
- La información de las contraseñas no se debe capturar por consola.
- Notificar por correo cada vez que el usuario ingrese a la plataforma.
- La información de tokens respecto a roles se maneja a través de **UUID** o similar.
- Duración del token: **30 minutos**; al superar, se solicita nuevo ingreso. *Validar con Producto.*
- Solo se permiten **3 intentos** de login incorrectos → bloqueo del acceso por **24 horas**.

### 6.3 Restablecer Contraseña

**Objetivo.** Mecanismo para que el TH cree nueva contraseña en caso de olvido.

**Criterios de Aceptación:**

- Contraseña ofuscada con opción "Mostrar". *Validar con SI.*
- Una vez se evidencie bloqueo de contraseña, generar comunicación con el SSO para hacer viable la opción.
- Si el TH digita identificación o contraseña errados, informar por modal con detalle del Back-End.
- En la URL de restablecimiento viajan: Tipo de Documento, Número de Documento y correo registrado.
- Validar datos del TH.
- Solo **3 intentos** de información de restablecimiento; al superar, bloqueo del acceso.
- Validar que la contraseña cumpla requisitos de SI (estructura, duración); si no cumple, informar al TH con detalle desde el Back-End.
- Bloqueo por datos errados: **definitivo**, comunicarse con la Entidad.
- No capturar contraseñas por consola.
- Cuando se muestre el mensaje "¿olvidó su contraseña?", se debe tener conexión para que el TH dirija su correo registrado.
- Una vez se verifique el correo, enviar mail con URL para que el TH siga los pasos: digitar BIN y 4 últimos dígitos de la tarjeta.

### 6.4 Cambio de Contraseña

**Objetivo.** Link dentro del portal para que el TH cambie su contraseña, ingresando primero la actual. Si existe este servicio se utiliza; si no, se utiliza el de restablecer contraseña.

**Criterios de Aceptación:**

- Link en el portal para cambiar la contraseña.
- Conexión para que al hacer click se solicite "digite su contraseña actual y digite su nueva contraseña", con confirmación, siguiendo lineamientos de SI.
- Diseñar template del proceso y notificar al correo con la respuesta.
- Solo **3 intentos** de cambio de contraseña incorrectos → bloqueo de acceso.
- Validar que la nueva contraseña cumpla requisitos de SI; si no, informar al TH con detalle del Back-End.
- Bloqueo por datos errados de la contraseña actual: **definitivo**, comunicarse con la Entidad.
- No capturar contraseñas por consola.

### 6.5 Activación de la Tarjeta (Etapa 4)

**Descripción.** Realizar el proceso de activación de tarjeta por parte del TH a través del Portal. **No se puede realizar la activación por otro medio por norma PCI**, entendiendo que requiere asignación del PIN. El modelo de activación NO se realiza directamente por el portal sino por el **i-Frame** que se desarrollará internamente.

> Pendiente: validar con Pomelo cómo es la activación de tarjetas que no tienen documento asociado (innominadas).

**Criterios de Aceptación:**

- Validar que la tarjeta pertenezca a un AFG de tarjetas físicas (no virtuales).
- Validar que el estado de la tarjeta sea **"Pendiente por Activar"**.
- La activación solo se puede realizar **una única vez** con la asignación del PIN.
- Contar con un usuario creado con toda la información requerida para la activación por el portal.
- Almacenar fecha y detalle de procesamiento de cada movimiento administrativo.
- Auditoría de fecha de modificación y usuario.

**Activación de la Tarjeta (UX):** El TH activa la tarjeta diligenciando el número del PAN; una vez se diligencie ese dato, se muestra la opción "asigna tu clave".

> Criterios de aceptación de UX detallados marcados como `xxx` en el documento — pendiente refinar.

### 6.6 Cambio de PIN (Etapa 4)

**Descripción.** Realizar el proceso de cambio de PIN por el TH.

**Criterios de Aceptación:**

- Validar que la tarjeta pertenezca a un AFG de tarjetas físicas (no virtuales).
- Validar que el estado de la tarjeta sea **"Activo"**.
- Contar con un usuario creado con toda la información para el cambio de PIN por el portal.
- En los casos en que no se tenga el PIN actual, al digitar **3 veces el PIN errado** (campo paramétrico), la tarjeta se bloquea **definitivamente**. Notificar al TH por correo electrónico o SMS.
- Almacenar fecha y detalle de procesamiento de cada movimiento administrativo.
- Auditoría de fecha de modificación y usuario.
- Cuando se bloquee la tarjeta por PIN errado, reflejar la respuesta en los archivos de salida con la causal **"PIN Errado"**.

**Cambio de PIN (UX):** Permitir al cliente actualizar de forma autónoma y segura el código personal de su tarjeta a través del portal web, cumpliendo políticas de seguridad, validación de identidad y protección de información establecidas por la Entidad.

> Criterios de aceptación de UX detallados marcados como `xxx` en el documento — pendiente refinar.

---

## 7. Portal Web Prepago — Front (Onboarding, Gestión de PIN, Activación)

**Descripción.** Definir los procesos Front-End de onboarding, gestión de PIN y activación de tarjeta.

**Criterios de Aceptación:**

- **Todos los procesos del Front deben consumir los procesos del Back-End** para garantizar su funcionamiento.

> El documento marca esta sección como pendiente de detallar más allá del consumo de Back-End. Las funcionalidades específicas del Front se enumeran en `FASE 2 REFINAMIENTOS PENDIENTES — Portal web tarjeta habiente frontend`. La UX detallada de Activación, Cambio de PIN y los flujos de Pre-Registro / Login / Restablecer / Cambiar contraseña aparecen como `xxx` en el doc fuente, pendientes de refinamiento.

---

## Pendientes del documento (compilados de las 7 secciones)

1. Detalle de la revisión de GAW para creaciones, bloqueos, modificaciones, realce y distribución.
2. Plantillas con UX para correos de pre-registro y acceso directo.
3. Validación con SI de mecanismos de visualización de contraseña.
4. Validación con Pomelo:
   - Activación de tarjetas innominadas (sin documento asociado).
   - Estructura del archivo de realce y existencia de registro de control.
   - Campos adicionales en respuesta API de creación.
5. Validación con Producto:
   - Posibilidad de que la tarjeta virtual genere compras físicas.
   - Generación de tarjeta física a partir de virtual; responsables ante reclamaciones.
   - Tipos de transacción distintos a VNP para virtual.
   - Duración de tokens (30 min vs valor por confirmar).
6. Validación con Arquitectura: reenvío de correos en caso de fallas.
7. UX de los flujos `xxx` del Portal Front (Activación de tarjeta, Cambio de PIN, Restablecer / Cambiar contraseña, Login).
8. Cómo se entrega el CVV al TH para tarjetas virtuales (portal u otro medio a la Entidad).

---

## Referencias cruzadas

| Tema | Documento / Diagrama relacionado |
|---|---|
| Activación PIN tarjetas innominadas | `Prepago/Activacion_PIN_Innominada.drawio` (ADR-010) |
| Iframe seguro para captura / cambio de PIN | `Prepago/pinpad_pci_drawio_6_mejorado.drawio` (ADR-008) |
| Iframe visualización datos sensibles | `Prepago/Iframe_Visualizacion_DatosSensibles.drawio` (ADR-009) |
| Onboarding tarjetahabiente (UUID, OTP, password, SSO) | `Prepago/Onboarding_Tarjetahabiente.drawio` (ADR-016) |
| Creación masiva de tarjetas (flujo batch) | `Prepago/creacion_masiva_tarjetas.drawio` |
| Intercambio de archivos SFTP+PGP | `Prepago/Intercambio_Archivos_SFTP_PGP.drawio` |
| Modelo de datos (entity, bin, afg, card, preregistration, app_user, otp) | `Prepago/MER_Prepago_V_1_0.drawio` y `Prepago/db/01_schema_prepago.sql` |
| Arquitectura unificada — pestaña "Creacion de tarjetas" | `Prepago/PrepagoUnificadoArquitectura_V_1_4.drawio` |
| Bitácora de decisiones | `Prepago/docs/decisiones-arquitectura.md` (ADR-001 a ADR-019) |
