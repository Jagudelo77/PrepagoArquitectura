# POC — Envelope Encryption PAN (RSA + AES sin HSM)

Artefacto para arrancar la POC en un workspace nuevo. Pegar este documento como contexto inicial en la nueva sesión de Kiro.

## Contexto

Credibanco necesita almacenar el PAN (Primary Account Number) de tarjetas prepago de forma segura en Oracle, sin usar HSM. La solución elegida es Envelope Encryption híbrido: AES-256-GCM para el PAN + RSA-4096 para proteger la llave AES.

Esta POC valida el esquema criptográfico, el flujo de tampering detection y la rotación de llaves, con un cliente visual para pruebas manuales.

Diagrama de referencia (debe copiarse al workspace nuevo):
- `Envelope_Encryption_PAN.drawio`

Documentación técnica:
- `envelope-encryption.md` (explicación completa del esquema)

---

## Arquitectura de la POC

```
┌──────────────────────────────────────────────────────────────┐
│ Cliente Web (Angular 17+ SPA)         Puerto 4200            │
│ ├── Form: crear tarjeta (ingresa PAN + CVV2)                 │
│ ├── Tabla: listar tarjetas (PAN enmascarado ****1234)        │
│ ├── Botón "Ver PAN completo" (descifra y muestra 30s)        │
│ ├── Botón "Simular tampering" (muestra excepción)            │
│ ├── Botón "Rotar KEK" (ejecuta rotación y muestra resultado) │
│ └── Log visual: ciphertext, IV, authTag, DEK cifrada         │
└────────────────────┬─────────────────────────────────────────┘
                     │ HTTP REST
                     ▼
┌──────────────────────────────────────────────────────────────┐
│ Backend (Spring Boot 3 + Java 21)     Puerto 8080            │
│                                                               │
│ REST Endpoints:                                               │
│ • POST /api/cards        → cifra PAN y lo guarda             │
│ • GET  /api/cards        → lista tarjetas (PAN enmascarado)  │
│ • GET  /api/cards/{id}/pan  → descifra y retorna PAN full    │
│ • POST /api/cards/{id}/tamper → simula tampering (demo)      │
│ • POST /api/admin/rotate-kek  → rota KEK y re-cifra DEKs     │
│ • GET  /swagger-ui.html  → Swagger UI                        │
│                                                               │
│ Servicios internos:                                           │
│ • PanEncryptionService  → cifra/descifra con envelope        │
│ • KekService            → carga KEK del KeyStore             │
│ • KekRotationService    → rota KEK y re-cifra DEKs           │
│ • AuditService          → log de cada acceso a PAN           │
│                                                               │
│ H2 in-memory → simula Oracle                                 │
│ KeyStore PKCS12 → autogenerado al primer arranque            │
└──────────────────────────────────────────────────────────────┘
```

---

## Stack técnico

| Componente | Tecnología |
|---|---|
| Backend | Java 21 + Spring Boot 3.2+ + Spring Web |
| Persistencia | Spring Data JPA + H2 in-memory |
| Criptografía | javax.crypto (JCE built-in) |
| Validación | jakarta.validation |
| OpenAPI | springdoc-openapi-starter-webmvc-ui |
| Frontend | Angular 17+ standalone components |
| UI Framework | Angular Material o Tailwind CSS |
| HTTP Client | HttpClient nativo de Angular |
| Build | Maven (backend), npm (frontend) |

---

## User Stories

### US-01: Crear tarjeta con PAN cifrado
**Como** usuario de la POC,
**quiero** ingresar un PAN en el cliente web y que se cifre antes de almacenarse,
**para** validar que el envelope encryption funciona.

**Criterios:**
- Form con campos: PAN (16 dígitos), CVV2 (3 dígitos), fecha vencimiento
- POST `/api/cards` con JSON `{ pan, cvv2, expDate }`
- Backend:
  - Genera DEK aleatoria AES-256 (32 bytes)
  - Genera IV aleatorio (12 bytes)
  - Construye AAD = `cardId + "|" + createdAt`
  - Cifra PAN con AES-256-GCM(DEK, PAN, IV, AAD)
  - Cifra DEK con RSA-OAEP(KEK_public)
  - Almacena en H2: `encrypted_pan`, `encrypted_dek`, `iv`, `kek_version`
  - Borra DEK de memoria con `Arrays.fill(dek, (byte) 0)`
- Respuesta: `{ cardId, lastFour, status }` (nunca el PAN completo)

### US-02: Listar tarjetas con PAN enmascarado
**Como** usuario de la POC,
**quiero** ver una tabla con todas las tarjetas creadas,
**para** validar que el PAN nunca se muestra completo por defecto.

**Criterios:**
- GET `/api/cards` retorna lista con: `{ cardId, lastFour, expDate, createdAt, kekVersion }`
- Cliente renderiza: `**** **** **** 1234`
- Ningún campo del JSON contiene el PAN completo ni la DEK ni el ciphertext

### US-03: Ver PAN completo (descifrar)
**Como** usuario de la POC,
**quiero** presionar "Ver PAN completo" y ver el PAN descifrado,
**para** validar el flujo de descifrado.

**Criterios:**
- GET `/api/cards/{id}/pan` retorna `{ pan, decryptionTimeMs }`
- Backend:
  - Lee `encrypted_pan`, `encrypted_dek`, `iv`, `kek_version` de H2
  - Descifra DEK con privKey RSA del KeyStore
  - Reconstruye AAD = `cardId + "|" + createdAt`
  - Descifra PAN con AES-256-GCM
  - Borra DEK de memoria
  - Registra auditoría (card_id, timestamp, duration)
- Cliente:
  - Muestra PAN por 30 segundos
  - Countdown visual (barra de progreso)
  - Al expirar: sobreescribe DOM y muestra enmascarado de nuevo
- Headers en la respuesta: `Cache-Control: no-store`

### US-04: Simular tampering — detectar modificación
**Como** usuario de la POC,
**quiero** modificar un byte del ciphertext y que el descifrado falle,
**para** validar que AES-GCM detecta tampering.

**Criterios:**
- POST `/api/cards/{id}/tamper?target=ciphertext` modifica 1 byte del `encrypted_pan` en H2
- POST `/api/cards/{id}/tamper?target=aad` cambia el `card_id` al descifrar (simulación)
- POST `/api/cards/{id}/tamper?target=iv` modifica 1 byte del IV
- Al intentar GET `/api/cards/{id}/pan` después → retorna HTTP 422 con:
  ```json
  {
    "error": "TAMPER_DETECTED",
    "message": "AEADBadTagException: authentication tag mismatch",
    "cardId": 12345
  }
  ```
- Cliente muestra alerta en rojo: "⚠️ Tampering detectado — datos comprometidos"
- Log visual muestra el stack trace (educativo para la demo)
- Botón "Restaurar" para volver al estado correcto

### US-05: Rotación de KEK
**Como** usuario de la POC,
**quiero** rotar la KEK y que las DEKs se re-cifren sin tocar los PANs,
**para** validar el proceso de rotación.

**Criterios:**
- POST `/api/admin/rotate-kek` inicia la rotación:
  - Genera nuevo par RSA-4096 → `kek-v{N+1}.p12`
  - Para cada tarjeta: descifra DEK con privKey_v1 → re-cifra con pubKey_v2 → UPDATE `kek_version`
  - Los PANs NO se tocan
  - Respuesta: `{ cardsRotated, oldVersion, newVersion, durationMs }`
- Después de rotar, GET `/api/cards/{id}/pan` sigue funcionando (backend soporta ambas versiones)
- Cliente muestra:
  - Versión actual de KEK
  - Tarjetas por versión
  - Botón "Rotar KEK" que ejecuta la rotación
  - Log de duración (demuestra que es rápido: solo DEKs)

### US-06: Panel de inspección criptográfica
**Como** usuario técnico de la POC,
**quiero** ver los valores criptográficos intermedios,
**para** entender visualmente el flujo de envelope encryption.

**Criterios:**
- Botón "Inspeccionar tarjeta" en cada fila
- GET `/api/cards/{id}/internals` retorna (sin PAN):
  ```json
  {
    "cardId": 12345,
    "encryptedPan": "base64...",
    "encryptedPanBytes": 32,
    "iv": "base64...",
    "ivBytes": 12,
    "authTag": "base64... (últimos 16 bytes del ciphertext)",
    "encryptedDek": "base64...",
    "encryptedDekBytes": 512,
    "aad": "12345|2024-01-15T10:30:00Z",
    "kekVersion": 1,
    "algorithm": "AES-256-GCM / RSA-OAEP-SHA256"
  }
  ```
- Cliente renderiza con formato legible:
  - Ciphertext en hex colorizado
  - IV destacado
  - AuthTag destacado (últimos 16 bytes)
  - DEK cifrada (muestra que RSA-4096 produce ~512 bytes)

### US-07: Auditoría
**Como** usuario de la POC,
**quiero** ver un log de accesos al PAN,
**para** validar el control PCI.

**Criterios:**
- Tabla `audit_log`: `{ id, cardId, action, timestamp, ipAddress, result }`
- Acciones: `CREATE`, `VIEW_PAN`, `ROTATE_KEK`, `TAMPER_DETECTED`
- GET `/api/audit` retorna los últimos 100 eventos
- Cliente muestra tabla cronológica con filtros por acción

---

## Endpoints REST

```
POST   /api/cards                    → Crear tarjeta (cifra PAN)
GET    /api/cards                    → Listar (PAN enmascarado)
GET    /api/cards/{id}/pan           → Descifrar PAN (con auditoría)
GET    /api/cards/{id}/internals     → Ver valores criptográficos
POST   /api/cards/{id}/tamper        → Simular tampering
POST   /api/cards/{id}/restore       → Restaurar después de tamper

POST   /api/admin/rotate-kek         → Rotar KEK
GET    /api/admin/kek-status         → Ver versión actual + distribución

GET    /api/audit                    → Listar eventos de auditoría

GET    /swagger-ui.html              → Documentación interactiva
```

---

## Modelo de datos (H2)

```sql
CREATE TABLE card (
    card_id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    last_four        VARCHAR(4) NOT NULL,
    exp_date         VARCHAR(5) NOT NULL,
    encrypted_pan    VARCHAR(500) NOT NULL,   -- Base64 (ciphertext + authTag)
    encrypted_dek    VARCHAR(1000) NOT NULL,  -- Base64 (DEK cifrada RSA-4096)
    iv               VARCHAR(32) NOT NULL,    -- Base64 (12 bytes)
    encrypted_cvv2   VARCHAR(200),
    encrypted_dek_cvv VARCHAR(1000),
    iv_cvv           VARCHAR(32),
    kek_version      INT NOT NULL DEFAULT 1,
    created_at       TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE audit_log (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    card_id     BIGINT,
    action      VARCHAR(50) NOT NULL,
    timestamp   TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    ip_address  VARCHAR(45),
    result      VARCHAR(20) NOT NULL,
    details     TEXT
);

CREATE TABLE kek_metadata (
    version     INT PRIMARY KEY,
    created_at  TIMESTAMP NOT NULL,
    active      BOOLEAN NOT NULL DEFAULT TRUE,
    keystore_path VARCHAR(500) NOT NULL
);
```

---

## Estructura de proyecto

```
poc-envelope-encryption/
├── backend/                         # Spring Boot
│   ├── pom.xml
│   ├── src/main/java/com/credibanco/poc/
│   │   ├── PocApplication.java
│   │   ├── controller/
│   │   │   ├── CardController.java
│   │   │   ├── AdminController.java
│   │   │   └── AuditController.java
│   │   ├── service/
│   │   │   ├── PanEncryptionService.java      # Envelope encryption
│   │   │   ├── KekService.java                # Carga KeyStore
│   │   │   ├── KekRotationService.java        # Rotación
│   │   │   └── AuditService.java
│   │   ├── model/
│   │   │   ├── Card.java                      # JPA entity
│   │   │   └── AuditLog.java
│   │   ├── repository/
│   │   │   ├── CardRepository.java
│   │   │   └── AuditRepository.java
│   │   ├── dto/
│   │   │   ├── CreateCardRequest.java
│   │   │   ├── CardResponse.java
│   │   │   └── CardInternalsResponse.java
│   │   ├── config/
│   │   │   ├── KekConfig.java
│   │   │   ├── SecurityConfig.java
│   │   │   └── OpenApiConfig.java
│   │   └── exception/
│   │       ├── TamperDetectedException.java
│   │       └── GlobalExceptionHandler.java
│   ├── src/main/resources/
│   │   ├── application.yml
│   │   └── schema.sql
│   └── src/test/java/com/credibanco/poc/
│       ├── PanEncryptionServiceTest.java      # Tests críticos
│       ├── TamperDetectionTest.java
│       ├── KekRotationTest.java
│       └── integration/
│           └── CardControllerIT.java
│
└── frontend/                        # Angular
    ├── package.json
    ├── angular.json
    └── src/app/
        ├── app.component.ts
        ├── pages/
        │   ├── cards/               # Lista + crear
        │   ├── internals/           # Panel cripto
        │   ├── tamper-demo/         # Simular tampering
        │   └── rotation/            # Rotar KEK
        ├── services/
        │   └── card-api.service.ts
        └── shared/
            ├── pan-mask.pipe.ts
            └── crypto-log.component.ts
```

---

## Tests unitarios críticos

### PanEncryptionServiceTest

```java
@Test
void roundtrip_encryptAndDecrypt_returnsOriginalPan() {
    String originalPan = "4532012345671234";
    EncryptedPan encrypted = service.encrypt(originalPan, cardId, createdAt);
    String decrypted = service.decrypt(encrypted, cardId, createdAt);
    assertEquals(originalPan, decrypted);
}

@Test
void tamper_modifyCiphertext_throwsAEADBadTagException() {
    EncryptedPan encrypted = service.encrypt("4532012345671234", 1L, now);
    byte[] tampered = encrypted.ciphertext().clone();
    tampered[0] ^= 0x01; // Flip un bit

    assertThrows(TamperDetectedException.class, () -> {
        service.decrypt(new EncryptedPan(tampered, encrypted.dek(), encrypted.iv()), 1L, now);
    });
}

@Test
void tamper_changeCardIdInAad_throwsAEADBadTagException() {
    EncryptedPan encrypted = service.encrypt("4532012345671234", 1L, now);
    // Intenta descifrar con card_id diferente (AAD no coincide)
    assertThrows(TamperDetectedException.class, () -> {
        service.decrypt(encrypted, 999L, now);
    });
}

@Test
void eachEncryption_producesDifferentCiphertext() {
    String pan = "4532012345671234";
    EncryptedPan enc1 = service.encrypt(pan, 1L, now);
    EncryptedPan enc2 = service.encrypt(pan, 1L, now);
    // Mismo PAN, pero IV aleatorio → ciphertext diferente
    assertNotEquals(enc1.ciphertext(), enc2.ciphertext());
    assertNotEquals(enc1.iv(), enc2.iv());
}
```

### KekRotationTest

```java
@Test
void rotation_rekeysAllDeks_withoutTouchingPans() {
    // Crea 10 tarjetas con KEK v1
    List<Long> cardIds = createTestCards(10);

    // Rota a v2
    RotationResult result = rotationService.rotateKek();

    // Valida que todas quedaron en v2
    assertEquals(10, result.cardsRotated());
    cardIds.forEach(id -> {
        Card card = repo.findById(id).orElseThrow();
        assertEquals(2, card.getKekVersion());
    });

    // Valida que los PANs siguen descifrables
    cardIds.forEach(id -> {
        String pan = service.decryptCardPan(id);
        assertNotNull(pan);
    });
}

@Test
void backendSupportsBothKekVersions_duringMigration() {
    Long v1CardId = createCard("4532011111111111"); // KEK v1
    rotationService.rotateKek();
    Long v2CardId = createCard("4532022222222222"); // KEK v2

    // Ambos deben descifrar correctamente
    assertEquals("4532011111111111", service.decryptCardPan(v1CardId));
    assertEquals("4532022222222222", service.decryptCardPan(v2CardId));
}
```

---

## Configuración clave

### application.yml

```yaml
server:
  port: 8080

spring:
  datasource:
    url: jdbc:h2:mem:pocdb;DB_CLOSE_DELAY=-1
    driver-class-name: org.h2.Driver
    username: sa
    password:
  h2:
    console:
      enabled: true
  jpa:
    hibernate:
      ddl-auto: create-drop
    show-sql: false

poc:
  kek:
    keystore-dir: ./keys
    keystore-name: kek-prepago
    keystore-password: poc-password-change-in-prod
    key-alias: kek-prepago
    auto-generate: true

springdoc:
  swagger-ui:
    path: /swagger-ui.html
  api-docs:
    path: /v3/api-docs

cors:
  allowed-origins: http://localhost:4200
```

### Ejemplo: PanEncryptionService

```java
@Service
public class PanEncryptionService {

    private final PublicKey kekPublicKey;
    private final PrivateKey kekPrivateKey;
    private final SecureRandom secureRandom;

    public EncryptedPan encrypt(String pan, Long cardId, Instant createdAt) {
        try {
            byte[] dek = new byte[32];
            secureRandom.nextBytes(dek);

            byte[] iv = new byte[12];
            secureRandom.nextBytes(iv);

            String aad = cardId + "|" + createdAt.toString();

            Cipher aes = Cipher.getInstance("AES/GCM/NoPadding");
            aes.init(Cipher.ENCRYPT_MODE,
                new SecretKeySpec(dek, "AES"),
                new GCMParameterSpec(128, iv));
            aes.updateAAD(aad.getBytes(StandardCharsets.UTF_8));
            byte[] ciphertext = aes.doFinal(pan.getBytes(StandardCharsets.UTF_8));

            Cipher rsa = Cipher.getInstance("RSA/ECB/OAEPWithSHA-256AndMGF1Padding");
            rsa.init(Cipher.ENCRYPT_MODE, kekPublicKey);
            byte[] encryptedDek = rsa.doFinal(dek);

            Arrays.fill(dek, (byte) 0);

            return new EncryptedPan(ciphertext, encryptedDek, iv);
        } catch (GeneralSecurityException e) {
            throw new CryptoException("Failed to encrypt PAN", e);
        }
    }

    public String decrypt(EncryptedPan encrypted, Long cardId, Instant createdAt) {
        byte[] dek = null;
        try {
            Cipher rsa = Cipher.getInstance("RSA/ECB/OAEPWithSHA-256AndMGF1Padding");
            rsa.init(Cipher.DECRYPT_MODE, kekPrivateKey);
            dek = rsa.doFinal(encrypted.dek());

            String aad = cardId + "|" + createdAt.toString();

            Cipher aes = Cipher.getInstance("AES/GCM/NoPadding");
            aes.init(Cipher.DECRYPT_MODE,
                new SecretKeySpec(dek, "AES"),
                new GCMParameterSpec(128, encrypted.iv()));
            aes.updateAAD(aad.getBytes(StandardCharsets.UTF_8));
            byte[] plaintext = aes.doFinal(encrypted.ciphertext());

            return new String(plaintext, StandardCharsets.UTF_8);
        } catch (AEADBadTagException e) {
            throw new TamperDetectedException(
                "Tampering detected for card " + cardId, e);
        } catch (GeneralSecurityException e) {
            throw new CryptoException("Failed to decrypt PAN", e);
        } finally {
            if (dek != null) Arrays.fill(dek, (byte) 0);
        }
    }
}
```

### Ejemplo: KekService (auto-genera KeyStore en primera ejecución)

```java
@Service
public class KekService {

    @Value("${poc.kek.keystore-dir}")
    private String keystoreDir;

    @Value("${poc.kek.keystore-name}")
    private String keystoreName;

    @Value("${poc.kek.keystore-password}")
    private String password;

    @Value("${poc.kek.key-alias}")
    private String alias;

    @PostConstruct
    public void ensureKeystoreExists() throws Exception {
        Path dir = Paths.get(keystoreDir);
        Files.createDirectories(dir);
        Path keystorePath = dir.resolve(keystoreName + ".p12");

        if (!Files.exists(keystorePath)) {
            generateNewKeystore(keystorePath, 1);
            log.info("Generated new KEK KeyStore at {}", keystorePath);
        }
    }

    private void generateNewKeystore(Path path, int version) throws Exception {
        KeyPairGenerator kpg = KeyPairGenerator.getInstance("RSA");
        kpg.initialize(4096);
        KeyPair keyPair = kpg.generateKeyPair();

        // Generar certificado auto-firmado con BouncyCastle o keytool
        X509Certificate cert = SelfSignedCertGenerator.generate(
            keyPair, "CN=KEK-Prepago-v" + version + ",O=Credibanco,C=CO");

        KeyStore ks = KeyStore.getInstance("PKCS12");
        ks.load(null, password.toCharArray());
        ks.setKeyEntry(alias, keyPair.getPrivate(),
            password.toCharArray(), new Certificate[]{cert});

        try (OutputStream os = Files.newOutputStream(path)) {
            ks.store(os, password.toCharArray());
        }
    }
}
```

---

## Criterios de éxito de la POC

1. Crear 10+ tarjetas sin errores
2. Cada tarjeta tiene ciphertext, IV y DEK cifrada únicos
3. Descifrar PAN retorna el valor original
4. Modificar 1 byte del ciphertext → excepción `TamperDetectedException`
5. Cambiar `card_id` al descifrar → excepción (AAD mismatch)
6. Rotar KEK → todas las DEKs se re-cifran, PANs intactos, siguen descifrables
7. Cliente web muestra todo el flujo de forma visual
8. Swagger UI permite probar endpoints directamente
9. Auditoría registra cada acceso al PAN
10. Tests unitarios en verde (roundtrip, tamper, rotación)

---

## Qué NO incluye la POC

- Ceremonia real con 2 custodios (password se autogenera en `application.yml`)
- Secrets de OpenShift (KeyStore local)
- Split knowledge del password (documentado, no implementado)
- Oracle real (H2 in-memory)
- TLS/mTLS (HTTP en localhost)
- MFA / autenticación (sin login en la POC)
- Rate limiting
- Iframe separado para visualización (PAN se muestra en el mismo dominio)

---

## Secuencia sugerida de desarrollo

1. Generar proyecto backend con Spring Initializr (Java 21, Spring Web, JPA, H2, Validation)
2. Implementar `KekService` con auto-generación de KeyStore
3. Implementar `PanEncryptionService` con tests unitarios (TDD)
4. Implementar entidades JPA y repositorios
5. Implementar `CardController` y DTOs
6. Implementar `KekRotationService` con tests
7. Implementar `AuditService`
8. Configurar Swagger UI + CORS para frontend
9. Generar proyecto Angular con `ng new frontend --standalone`
10. Implementar componentes: lista de tarjetas, crear, inspeccionar, tamper, rotación
11. Conectar frontend con backend vía HttpClient
12. Probar flujo completo manualmente
13. Validar todos los criterios de éxito

---

## Al arrancar la nueva sesión en Kiro

Di algo como:

> "Lee el archivo `poc-envelope-encryption-requerimientos.md` que copié en este workspace y construye la POC completa siguiendo el plan. Usa TDD para los servicios criptográficos."

Kiro cargará todo el contexto y podrá arrancar inmediatamente.
