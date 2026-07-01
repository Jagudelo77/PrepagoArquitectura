# CredibanCo como BIN Sponsor de Visa y Mastercard

> Documento de trabajo consolidado a partir del análisis realizado sobre los requisitos
> para convertir a CredibanCo en emisor patrocinador (BIN Sponsor) de tarjetas prepago,
> tomando como referencia el modelo del proveedor Pomelo.
>
> Fecha: 1 de julio de 2026

---

## 1. Contexto y objetivo

El objetivo es replicar el modelo de negocio de **Pomelo** en su **capa de licencia /
membresía de red (BIN Sponsor)**: convertir a CredibanCo en una entidad que, gracias a sus
propias licencias con Visa y Mastercard, permita a terceros (fintechs, retailers, billeteras)
emitir tarjetas prepago, débito o crédito bajo su patrocinio.

### ¿Qué hace Pomelo?

Pomelo opera como **BIN Sponsor**: es miembro principal de Visa/Mastercard y ofrece emisión,
procesamiento y gestión integral de programas de tarjetas usando sus propias licencias. Sus
clientes emiten tarjetas "bajo el paraguas" de esas licencias, sin necesidad de certificarse
directamente ante las redes.

Pomelo declara cumplir con estándares GDPR, PCI-DSS 4.0 e ISO 27001, además de las reglas de
Mastercard y Visa. Ofrece BINs preasignados, conciliación diaria, gestión de colateral, pagos
en moneda local, seguridad integrada (3DS, CVV dinámico, antifraude FICO Falcon) y gestión de
contracargos.

Fuentes:
- [Pomelo BIN Sponsorship](https://pomelo.la/en/bin-sponsorship/)
- [Pomelo Card Issuing](https://pomelo.la/en/card-issuing)
- [Pomelo Docs - Issuing](https://docs-stage.pomelo.la/en/docs/cards/issuing/home)

---

## 2. Las tres capas del modelo Pomelo

| Capa | Descripción | ¿Se construye con software? |
|------|-------------|------------------------------|
| **1. Licencia / membresía de red (BIN Sponsor)** | Membresía principal de Visa/Mastercard, licencia regulatoria, capital, colateral | No — es regulatorio, financiero y de gobierno corporativo |
| **2. Procesamiento (issuer processor)** | Autorización en tiempo real, saldos, clearing & settlement, ciclo de vida de tarjeta, tokenización, 3DS, antifraude, chargebacks | Sí — es el core técnico |
| **3. Producto / API** | APIs REST, dashboard, onboarding de clientes, webhooks | Sí — la cara visible |

El foco de este documento es la **Capa 1 (BIN Sponsor)**, con las conexiones necesarias a las
capas 2 y 3.

---

## 3. Concepto clave: qué significa ser BIN Sponsor

Ser BIN Sponsor significa convertirse en **Miembro Principal (Principal Member)** de Visa y
Mastercard. Requiere **dos licencias distintas**:

1. **Licencia regulatoria** (otorgada por el regulador financiero local): autoriza legalmente
   a manejar fondos de terceros y emitir medios de pago. En Colombia corresponde al marco de la
   Superintendencia Financiera y el Banco de la República.
2. **Licencia de red / membresía de scheme** (otorgada por Visa y Mastercard): otorga el ICA
   (Mastercard) y el rango de BIN (Visa), conexión directa y settlement directo con la red.

No se puede obtener la #2 sin tener antes la #1: las redes exigen que la entidad sea regulada y
supervisada antes de otorgar la membresía.

### Capacidades y obligaciones del Miembro Principal

Como miembro principal se puede emitir tarjetas, patrocinar a terceros (esto es lo que
constituye el rol de "BIN Sponsor"), pero también se asume:

- Riesgo de liquidación (**settlement risk**) — se es responsable financiero final ante la red
  si un patrocinado quiebra o comete fraude.
- Alojar el propio procesamiento.
- Reportería, cuotas continuas, titularidad de ICA y BIN.
- Cumplimiento de las reglas del scheme y registro de agentes terceros.

Fuentes:
- [Advapay - Principal Membership](https://advapay.eu/principal-membership-with-mastercard-visa-jcb-unionpay-acquiring-issuing/)
- [Mastercard - Eligibility Criteria For Membership](https://www.mastercard.com/how_to_guide/membership.html)
- [Mastercard - Access to Mastercard](https://stage.mastercard.co.uk/en-gb/issuers/access-to-mastercard.html)
- [Walletto - How to become a Visa/Mastercard issuer](https://walletto.eu/wallettos-experience-how-to-become-an-issuer-of-visa-and-mastercard/)

### Tiempos y costos de referencia

| Recurso | Realidad (entidad desde cero) |
|---------|-------------------------------|
| Tiempo (licencia regulatoria) | 12-24 meses |
| Tiempo (membresía de red, issuer) | 4-8 meses |
| Tiempo total | 2-4 años |
| Fees iniciales de membresía | ~€50.000 (Visa) y ~€100.000 (Mastercard) el primer año (referencia, no incluye colateral) |
| Capital | Capital regulatorio + colateral de settlement + fees (millones de USD) |

Fuente de referencia de costos: [Boxopay - Acquirer Licensing](https://boxopay.com/blog/acquirer-licensing-process/)

*Contenido de fuentes externas reformulado para cumplir con restricciones de licencia.*

---

## 4. Situación de partida de CredibanCo

CredibanCo NO parte de cero. Es una de las entidades de pagos más establecidas de Colombia.

### Lo que ya tiene avanzado

- **Entidad regulada y supervisada** ✅ — Empresa colombiana vigilada por la Superintendencia
  Financiera, con más de 50 años de experiencia en administración y desarrollo de sistemas de
  pago de bajo valor. (El requisito más difícil ya está cumplido.)
- **Relación y membresía con las redes** ✅ — Históricamente el adquirente de Visa en Colombia;
  ya cuenta con membresía/licencia (al menos adquirente), conexión técnica certificada,
  settlement directo e infraestructura PCI-DSS operando.
- **Producto prepago en marcha** ✅ — Ya ofrece una Tarjeta Prepago con consulta de saldo y
  movimientos.
- **Canal de atención para bancos** ✅ — Ya opera dando servicio a otras entidades financieras.

Fuentes:
- [LeadIQ - CredibanCo](https://www.leadiq.com/c/credibanco/5a1d97192300005a0085b40e/employee-directory)
- [CB Insights - Credibanco](https://www.cbinsights.com/company/credibanco/)
- [CredibanCo - sitio oficial](https://www.credibanco.com/nuestras-oficinas)

### Gaps a validar o construir

- ⚠️ **Membresía de emisión (issuing) con sponsorship**, no solo adquirencia.
- ⚠️ **Membresía Mastercard** (issuing + sponsorship), además de Visa.
- ⚠️ **Modelo de Principal con capacidad de patrocinar terceros** (registro de program managers).
- 🔨 **Plataforma tecnológica API-first multi-tenant** al estilo Pomelo (dashboard self-service,
  onboarding de patrocinados, APIs públicas).

### Resumen comparativo

| Paso | Estado en CredibanCo |
|------|----------------------|
| Entidad regulada/supervisada | ✅ Resuelto (vigilada SFC, +50 años) |
| Capital y solvencia | ✅ Muy probablemente resuelto |
| Membresía de red (adquirente) | ✅ Resuelto (Visa Colombia) |
| Conexión técnica + PCI-DSS | ✅ Operando |
| Capacidad de emisión (prepago) | ✅ Ya tienen producto prepago |
| Membresía emisor con sponsorship | ⚠️ Por validar/ampliar |
| Membresía Mastercard issuing | ⚠️ Por validar/ampliar |
| Marco de riesgo/settlement para terceros | 🔨 Por construir/formalizar |
| Plataforma API self-service tipo Pomelo | 🔨 Gap tecnológico principal |

> Nota: el detalle exacto de la membresía actual con cada red y el alcance de la licencia de
> sponsorship no es verificable desde fuentes públicas; debe confirmarlo el equipo legal / de
> alianzas de CredibanCo. La evaluación de gaps es inferida a partir de su perfil público como
> adquirente Visa.

---

## 5. Listado completo de pasos para convertirse en BIN Sponsor

Leyenda: ✅ probablemente ya resuelto · ⚠️ por validar/ampliar · 🔨 por construir

### Fase 0 — Definición estratégica y de negocio

1. Definir el alcance del programa sponsor (productos: prepago, débito, crédito; nominadas/no nominadas; locales/globales).
2. Definir el público objetivo (fintechs, retailers, billeteras, cooperativas).
3. Definir el modelo de ingresos (fees de setup, por tarjeta, por transacción, interchange compartido, márgenes de FX).
4. Decidir cobertura de redes (Visa, Mastercard o ambas).
5. Análisis de competencia y posicionamiento frente a Pomelo y bancos locales.
6. Business case y proyección financiera (volumen de tarjetas, transacciones, colateral requerido).

### Fase 1 — Habilitación regulatoria (Colombia)

7. ✅ Confirmar condición de entidad vigilada por la Superintendencia Financiera.
8. ⚠️ Validar que la licencia/objeto social cubre emisión de prepago y patrocinio a terceros; gestionar ampliación ante la SFC si aplica.
9. ⚠️ Confirmar capital regulatorio y solvencia para el nuevo rol (el settlement risk de terceros eleva requisitos de capital).
10. Validar el encuadre del producto prepago frente al marco del Banco de la República (sistema de pago de bajo valor).
11. Definir el tratamiento de los fondos de tarjetahabientes (cuentas ómnibus, fideicomiso, segregación).
12. Actualizar el marco SARLAFT (AML/CFT) para cubrir el portafolio de patrocinados.

> Los puntos 8-9 no son verificables desde fuentes públicas; los debe confirmar el equipo legal / de cumplimiento.

### Fase 2 — Membresía y licencia con las redes

13. ⚠️ Auditar la membresía actual con Visa (adquirente vs issuer; capacidad de sponsor).
14. ⚠️ Gestionar/ampliar la membresía de issuing con Visa.
15. ⚠️ Gestionar la membresía con Mastercard (issuing + sponsorship).
16. Solicitar/confirmar titularidad de rangos de BIN (Visa) e ICA (Mastercard).
17. Solicitar capacidad de registro de terceros (program managers, agentes, patrocinados).
18. Firmar los acuerdos de licencia correspondientes.
19. Constituir el colateral/garantía exigido por cada red.
20. Definir y documentar el modelo de liquidación (settlement) directo con cada red.

### Fase 3 — Riesgo, cumplimiento y gobierno del modelo sponsor

21. 🔨 Diseñar el marco de settlement risk de terceros (default de patrocinados).
22. 🔨 Definir política de colateral y garantías de los patrocinados.
23. 🔨 Crear el proceso de due diligence / onboarding KYB de patrocinados.
24. 🔨 Definir límites por programa (volúmenes, exposición, velocidad, fraude).
25. 🔨 Establecer programa de monitoreo continuo (transaccional, AML, fraude) sobre el portafolio.
26. 🔨 Diseñar el marco de gestión de contracargos (chargebacks) para los programas patrocinados.
27. Definir modelo de responsabilidad y contratos con patrocinados (responsabilidad ante la red, SLAs, indemnidades).
28. Plan de continuidad de negocio y resolución/salida de patrocinados (wind-down).

### Fase 4 — Infraestructura técnica

29. ✅ Confirmar el core de procesamiento emisor (autorización, clearing, settlement).
30. ✅ Confirmar/renovar la certificación PCI-DSS al nivel requerido (nivel 1).
31. ⚠️ Correr/actualizar la certificación técnica de issuing con cada red.
32. 🔨 Construir la plataforma multi-tenant de gestión de programas (cada patrocinado como tenant aislado).
33. 🔨 Construir las APIs públicas de emisión y gestión de tarjetas (crear tarjeta, fondear, saldo, transacciones, webhooks).
34. 🔨 Construir el dashboard self-service para patrocinados.
35. 🔨 Implementar el ledger / contabilidad por patrocinado y por tarjetahabiente.
36. 🔨 Integrar tokenización (Apple Pay, Google Pay) y 3DS.
37. 🔨 Integrar motor antifraude y CVV dinámico.
38. 🔨 Construir el motor de settlement y conciliación multi-tenant (conciliación diaria, pagos en moneda local).
39. 🔨 Construir el flujo de onboarding/KYB de patrocinados y KYC de tarjetahabientes finales.
40. 🔨 Impresión y logística de tarjetas físicas (propio o partner), si aplica.

### Fase 5 — Piloto, certificación y salida a producción

41. Seleccionar uno o dos patrocinados piloto.
42. Configurar el primer BIN/programa de prueba en ambiente de certificación de la red.
43. Correr pruebas end-to-end (emisión → autorización → clearing → settlement → chargeback).
44. Obtener aprobaciones finales de cada red para producción.
45. Go-live controlado del primer programa patrocinado.
46. Monitoreo intensivo post-lanzamiento y ajuste de límites/reglas.

### Fase 6 — Escalamiento

47. Industrializar el onboarding de nuevos patrocinados.
48. Ampliar a más marcas/países (replicar el modelo regulatorio por jurisdicción).
49. Optimizar producto (multimoneda, multiapp, lealtad).

---

## 6. Dónde está el esfuerzo real para CredibanCo

Las Fases 0-1 y buena parte de la 2 y 4 ya están parcial o totalmente resueltas gracias a la
condición de entidad vigilada y adquirente Visa con producto prepago. El trabajo se concentra en:

1. **Negociación con las redes (Fase 2):** pasar de adquirente a sponsor de emisores, y sumar Mastercard.
2. **Marco de riesgo de terceros (Fase 3):** nuevo y crítico, porque se asume el riesgo de los patrocinados.
3. **Plataforma tecnológica API-first (Fase 4):** el gap más grande frente a Pomelo.

Comparado con una fintech desde cero (2-4 años), CredibanCo podría hablar de **meses** en lugar
de años, porque el activo más difícil (entidad regulada + membresía de red + conexión certificada
+ PCI) ya existe.

---

## 7. Próximos pasos sugeridos

- [ ] Confirmar internamente el alcance exacto de la membresía actual con Visa y Mastercard.
- [ ] Iniciar conversación con Visa y Mastercard sobre el rol de sponsor de emisores terceros.
- [ ] Definir el marco de riesgo/settlement para patrocinados (Fase 3).
- [ ] Diseñar la arquitectura de la plataforma tecnológica de sponsorship (Fase 4).

---

## 8. Referencias

- Pomelo — BIN Sponsorship: https://pomelo.la/en/bin-sponsorship/
- Pomelo — Card Issuing: https://pomelo.la/en/card-issuing
- Pomelo — Docs Issuing: https://docs-stage.pomelo.la/en/docs/cards/issuing/home
- Advapay — Principal Membership: https://advapay.eu/principal-membership-with-mastercard-visa-jcb-unionpay-acquiring-issuing/
- Mastercard — Eligibility Criteria: https://www.mastercard.com/how_to_guide/membership.html
- Mastercard — Access to Mastercard: https://stage.mastercard.co.uk/en-gb/issuers/access-to-mastercard.html
- Boxopay — Acquirer Licensing Process: https://boxopay.com/blog/acquirer-licensing-process/
- Walletto — How to become a Visa/Mastercard issuer: https://walletto.eu/wallettos-experience-how-to-become-an-issuer-of-visa-and-mastercard/
- LeadIQ — CredibanCo: https://www.leadiq.com/c/credibanco/5a1d97192300005a0085b40e/employee-directory
- CB Insights — Credibanco: https://www.cbinsights.com/company/credibanco/
- CredibanCo — sitio oficial: https://www.credibanco.com/nuestras-oficinas

---

> **Aviso:** Este documento consolida análisis basado en información pública y en el modelo de
> negocio de Pomelo. No constituye asesoría legal, financiera ni regulatoria. Los requisitos
> específicos de licenciamiento, capital y membresía deben validarse con el equipo legal / de
> cumplimiento de CredibanCo y directamente con Visa, Mastercard, la Superintendencia Financiera
> de Colombia y el Banco de la República.
