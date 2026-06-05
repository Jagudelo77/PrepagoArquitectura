# Envelope Encryption para PAN — Sin HSM

Documento de referencia para el almacenamiento seguro del PAN (Primary Account Number) de las tarjetas prepago usando envelope encryption (RSA + AES), sin depender del HSM Thales.

## Por qué envelope encryption

El Thales payShield se usa exclusivamente para el interchange de PIN (operaciones RSA efímeras entre navegador, backend y Pomelo). Para el almacenamiento persistente del **PAN** se usa un esquema híbrido RSA + AES implementado en el backend Java.

> **CVV2 NO se almacena.** Por PCI DSS v4.0 Req 3.3.2, el CVV2 se descarta inmediatamente después de la creación de la tarjeta. La visualización temporal del CVV2 al tarjetahabiente se hace consultando a Pomelo en tiempo real desde el iframe de datos sensibles (ver `Iframe_Visualizacion_DatosSensibles.drawio`). En este documento "envelope encryption" aplica únicamente al PAN.

**Comparación con alternativas:**

| Opción | Problema |
|---|---|
| Solo RSA | Lento, límite de ~245 bytes por operación, no escala para millones de PANs |
| Solo AES | ¿Dónde guardas la llave AES? Si es única, comprometerla compromete todo |
| HSM directo | Requiere comando Thales por cada operación de PAN (costo operativo alto) |
| **Envelope (RSA + AES)** | Cada PAN tiene su propia llave AES (DEK). Solo proteges UNA llave RSA (KEK) |

## Arquitectura del esquema

```
┌─────────────────────────────────────────────────────────┐
│  KEK (Key Encryption Key) — RSA-4096                    │
│  ├── pubKey: disponible en el backend (puede ser pública)│
│  └── privKey: en KeyStore PKCS12, dentro de pod OpenShift│
│                                                          │
│  DEK (Data Encryption Key) — AES-256                    │
│  ├── Única por cada PAN                                 │
│  ├── Generada aleatoriamente al cifrar                  │
│  ├── Se almacena cifrada con la KEK pública             │
│  └── Solo existe en claro durante operación cripto      │
│                                                          │
│  PAN cifrado: AES-256-GCM(DEK, PAN, IV) + auth_tag      │
└─────────────────────────────────────────────────────────┘
```

## Flujo de cifrado (al crear una tarjeta)

```
1. Genera DEK aleatoria (AES-256, 32 bytes)
   byte[] dek = SecureRandom.getInstanceStrong().generateSeed(32);

2. Genera IV para AES-GCM (12 bytes recomendado)
   byte[] iv = SecureRandom.getInstanceStrong().generateSeed(12);

3. Construye AAD (Additional Authenticated Data) — anti-tampering
   byte[] aad = (cardId + "|" + createdAt).getBytes(UTF_8);

4. Cifra PAN con DEK (AES-256-GCM)
   Cipher aes = Cipher.getInstance("AES/GCM/NoPadding");
   aes.init(Cipher.ENCRYPT_MODE, new SecretKeySpec(dek, "AES"),
            new GCMParameterSpec(128, iv));
   aes.updateAAD(aad);
   byte[] panCiphertextWithTag = aes.doFinal(pan.getBytes(UTF_8));
   // GCM concatena ciphertext (n bytes) + authTag (16 bytes)
   byte[] panCiphertext = Arrays.copyOfRange(panCiphertextWithTag, 0,
       panCiphertextWithTag.length - 16);
   byte[] authTag = Arrays.copyOfRange(panCiphertextWithTag,
       panCiphertextWithTag.length - 16, panCiphertextWithTag.length);

5. Cifra DEK con KEK pública (RSA-4096 OAEP con SHA-256)
   Cipher rsa = Cipher.getInstance("RSA/ECB/OAEPWithSHA-256AndMGF1Padding");
   rsa.init(Cipher.ENCRYPT_MODE, kekPublicKey);
   byte[] dekEncrypted = rsa.doFinal(dek);  // 512 bytes (RSA-4096 = 4096/8)

6. Calcula PAN hash (SHA-256) — para búsquedas exact-match sin descifrar
   byte[] panHash = MessageDigest.getInstance("SHA-256")
       .digest(pan.getBytes(UTF_8));  // 32 bytes

7. Borra DEK y PAN de memoria
   Arrays.fill(dek, (byte) 0);
   Arrays.fill(pan.getBytes(), (byte) 0);

8. Almacena en Oracle (tabla card, tipos RAW):
   - pan_encrypted   RAW(64)   — ciphertext AES sin tag
   - pan_iv          RAW(12)   — nonce GCM
   - pan_auth_tag    RAW(16)   — tag de integridad GCM
   - dek_encrypted   RAW(512)  — DEK cifrada con RSA-4096 OAEP
   - kek_version     NUMBER(3) — versión de KEK usada
   - pan_hash        RAW(32)   — SHA-256(PAN) para búsquedas
```

## Flujo de descifrado (al visualizar en iframe)

```
1. Lee de Oracle (tipos RAW, sin Base64):
   pan_encrypted, pan_iv, pan_auth_tag, dek_encrypted, kek_version

2. Selecciona la KEK correcta según kek_version (compatibilidad multi-versión)
   KekKeys keys = kekRegistry.get(kek_version);

3. Descifra DEK con KEK privada (RSA-4096 OAEP)
   Cipher rsa = Cipher.getInstance("RSA/ECB/OAEPWithSHA-256AndMGF1Padding");
   rsa.init(Cipher.DECRYPT_MODE, keys.privateKey());
   byte[] dek = rsa.doFinal(dek_encrypted);

4. Reconstruye AAD (mismo valor que en cifrado)
   byte[] aad = (cardId + "|" + createdAt).getBytes(UTF_8);

5. Descifra PAN con DEK (AES-256-GCM)
   Cipher aes = Cipher.getInstance("AES/GCM/NoPadding");
   aes.init(Cipher.DECRYPT_MODE, new SecretKeySpec(dek, "AES"),
            new GCMParameterSpec(128, pan_iv));
   aes.updateAAD(aad);
   // Concatenar ciphertext + authTag para que GCM verifique la integridad
   byte[] panCiphertextWithTag = ByteBuffer
       .allocate(pan_encrypted.length + pan_auth_tag.length)
       .put(pan_encrypted).put(pan_auth_tag).array();
   byte[] panBytes = aes.doFinal(panCiphertextWithTag);
   // Si authTag no coincide → AEADBadTagException → RECHAZA (dato tampered)

6. Borra DEK de memoria
   Arrays.fill(dek, (byte) 0);

7. Usa PAN en memoria (nunca en disco, log, ni cache)
8. Borra PAN de memoria al terminar
   Arrays.fill(panBytes, (byte) 0);
```

## Almacenamiento de la KEK — Opción elegida: Java KeyStore PKCS12

La llave privada RSA (KEK) se almacena en un archivo PKCS12 protegido con password. Ambos (archivo y password) viven en OpenShift como Secrets separados.

### Arquitectura de almacenamiento

```
OpenShift Cluster
├── Secret: kek-keystore
│   └── kek-prepago.p12 (archivo binario, montado como volumen)
│
├── Secret: kek-password-part-a
│   └── PART_A (env var, gestionado por Custodio A)
│
├── Secret: kek-password-part-b
│   └── PART_B (env var, gestionado por Custodio B)
│
└── Pod: prepaid-card-data-api
    ├── Monta /secrets/kek/kek-prepago.p12 (readOnly, 0400)
    ├── Lee PART_A y PART_B de env vars
    ├── Reconstruye password: PART_A + PART_B
    ├── Carga KeyStore en memoria del JVM
    └── privKey vive en RAM mientras el pod esté vivo
```

### Por qué Secrets separados

**Principio de separación de responsabilidades:**
- Si alguien obtiene solo el `.p12` → no tiene el password → inútil
- Si alguien obtiene solo el password → no tiene el `.p12` → inútil
- RBAC diferente por Secret: Custodio A nunca ve el Secret de Custodio B

### Manifiestos OpenShift

```yaml
# Secret con el .p12
apiVersion: v1
kind: Secret
metadata:
  name: kek-keystore
  namespace: prepago
type: Opaque
data:
  kek-prepago.p12: <base64 del archivo>

---
# Secret con la mitad A del password (solo Custodio A puede actualizar)
apiVersion: v1
kind: Secret
metadata:
  name: kek-password-part-a
type: Opaque
stringData:
  PART_A: "xJ3kP9mQ2nR7vT4wY8zA1bC5dF6gH0i"

---
# Secret con la mitad B del password (solo Custodio B puede actualizar)
apiVersion: v1
kind: Secret
metadata:
  name: kek-password-part-b
type: Opaque
stringData:
  PART_B: "L5oI4u2E8r6T3y1H9f7G5d4S2a0Q8wZ"

---
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prepaid-card-data-api
spec:
  template:
    spec:
      containers:
      - name: app
        env:
        - name: KEYSTORE_PASSWORD_PART_A
          valueFrom:
            secretKeyRef:
              name: kek-password-part-a
              key: PART_A
        - name: KEYSTORE_PASSWORD_PART_B
          valueFrom:
            secretKeyRef:
              name: kek-password-part-b
              key: PART_B
        - name: KEYSTORE_PATH
          value: "/secrets/kek/kek-prepago.p12"
        volumeMounts:
        - name: kek-volume
          mountPath: /secrets/kek
          readOnly: true
      volumes:
      - name: kek-volume
        secret:
          secretName: kek-keystore
          defaultMode: 0400
```

### Código Spring Boot

```java
@Configuration
public class KekConfig {

    @Value("${KEYSTORE_PATH}")
    private String keystorePath;

    @Value("${KEYSTORE_PASSWORD_PART_A}")
    private String partA;

    @Value("${KEYSTORE_PASSWORD_PART_B}")
    private String partB;

    @Bean
    public KekKeys kekKeys() throws Exception {
        String fullPassword = partA + partB;
        KeyStore ks = KeyStore.getInstance("PKCS12");
        try (InputStream is = new FileInputStream(keystorePath)) {
            ks.load(is, fullPassword.toCharArray());
        }
        PrivateKey priv = (PrivateKey) ks.getKey("kek-prepago",
            fullPassword.toCharArray());
        PublicKey pub = ks.getCertificate("kek-prepago").getPublicKey();
        return new KekKeys(priv, pub);
    }

    public record KekKeys(PrivateKey privateKey, PublicKey publicKey) {}
}
```

## Ceremonia de generación de la KEK

Ver documento detallado con swimlanes en `Prepago/Envelope_Encryption_PAN.drawio` (pestaña "Ceremonia Generación KEK").

### Resumen del procedimiento

1. **Ambiente:** máquina air-gapped con Linux Live USB, sin red, sin disco persistente
2. **Participantes:** Custodio A (Seguridad), Custodio B (Infraestructura), Testigo
3. **Generación de mitades:** cada custodio ejecuta `openssl rand -base64 24` y obtiene una mitad de 32 caracteres (192 bits de entropía)
4. **Unión de mitades:** script bash con `read -s` (input oculto) que concatena sin que ninguno vea la del otro
5. **Generación del KeyStore:**

```bash
# Par RSA-4096 con passphrase
openssl genrsa -aes256 -passout "pass:${FULL_PASSWORD}" \
  -out kek_private.pem 4096

# Certificado auto-firmado
openssl req -new -x509 -key kek_private.pem \
  -passin "pass:${FULL_PASSWORD}" \
  -out kek_cert.pem -days 3650 \
  -subj "/CN=KEK-Prepago-v1/O=Credibanco/C=CO"

# Empaquetar en PKCS12
openssl pkcs12 -export \
  -in kek_cert.pem -inkey kek_private.pem \
  -passin "pass:${FULL_PASSWORD}" \
  -out kek-prepago.p12 -name "kek-prepago" \
  -passout "pass:${FULL_PASSWORD}"

# Borrado seguro
shred -u kek_private.pem kek_cert.pem
unset PART_A PART_B FULL_PASSWORD
```

6. **Distribución:** cada custodio crea su Secret en OpenShift (`kek-password-part-a`, `kek-password-part-b`). El `.p12` se sube como Secret binario (`kek-keystore`)
7. **Cierre:** acta firmada, video guardado, backups cifrados en 2 ubicaciones físicas, destrucción de papeles intermedios

## Rotación de la KEK

**Frecuencia recomendada:** anual (PCI DSS Req 3.6.4)

**Proceso de rotación:**

```
1. Nueva ceremonia → genera kek-prepago-v2.p12
2. Batch re-cifra solo las DEKs (no los PANs):
   FOR each card WHERE kek_version = 1:
     DEK_plain    = RSA_DEC(privKey_v1, dek_encrypted)
     dek_encr_v2  = RSA_ENC(pubKey_v2, DEK_plain)
     UPDATE card SET dek_encrypted = dek_encr_v2,
                     kek_version   = 2
     WHERE card_id = ?;
3. Backend soporta v1 y v2 simultáneamente durante migración
4. Cuando todo esté en v2 → retira v1 (backup cifrado por compliance)
```

**Ventaja:** los `pan_encrypted`, `pan_iv` y `pan_auth_tag` NO se tocan, solo las DEKs. Esto es ~1000x más rápido que re-cifrar todos los PANs y mantiene íntegros los authTags GCM.

## Esquema Oracle

Las columnas de envelope encryption viven en la **tabla `card`** (no en una tabla `card_sensitive_data` aparte). Esto es coherente con el C4 L2 de creación de tarjetas (ver `PrepagoUnificadoArquitectura_V_1_4.drawio`, pestaña "Creacion de tarjetas") y con el diagrama de detalle (`Envelope_Encryption_PAN.drawio`).

```sql
CREATE TABLE card (
    card_id        NUMBER(10) PRIMARY KEY,
    client_id      NUMBER(10) NOT NULL,
    afg_id         NUMBER(10) NOT NULL,
    bin            VARCHAR2(8)   NOT NULL,
    last_four      CHAR(4)       NOT NULL,        -- en claro (PCI Req 3.3 permite)
    exp_date       DATE          NOT NULL,
    card_token     VARCHAR2(36)  NOT NULL UNIQUE, -- token Pomelo, no es PAN
    status         VARCHAR2(20)  NOT NULL,
    -- Envelope encryption (PAN):
    pan_encrypted  RAW(64)       NOT NULL,        -- ciphertext AES-256-GCM (sin tag)
    pan_iv         RAW(12)       NOT NULL,        -- nonce GCM
    pan_auth_tag   RAW(16)       NOT NULL,        -- tag de integridad GCM (128 bits)
    dek_encrypted  RAW(512)      NOT NULL,        -- DEK cifrada con RSA-4096 OAEP
    kek_version    NUMBER(3)     DEFAULT 1 NOT NULL,
    pan_hash       RAW(32)       NOT NULL,        -- SHA-256(PAN) para búsquedas
    -- Auditoría:
    created_at     TIMESTAMP     DEFAULT SYSTIMESTAMP,
    updated_at     TIMESTAMP
);

CREATE INDEX idx_card_pan_hash    ON card(pan_hash);     -- exact-match sin descifrar
CREATE INDEX idx_card_kek_version ON card(kek_version);  -- rotación batch por versión
CREATE INDEX idx_card_client      ON card(client_id);
```

**Lo que NO se almacena (PCI DSS v4.0 Req 3.3.2):**
- ❌ `cvv2` ni en claro ni cifrado.
- ❌ `pin` ni en claro ni cifrado (vive en HSM Thales / Pomelo).
- ❌ Datos de banda magnética.

Para visualizar el CVV2 al tarjetahabiente, el iframe consulta a Pomelo en tiempo real (ver `Iframe_Visualizacion_DatosSensibles.drawio`). El CVV2 no toca Oracle.

## Controles PCI DSS aplicables

| Req | Cómo se cumple |
|---|---|
| 3.3.2 | CVV2 no se almacena (se descarta post-creación, se consulta a Pomelo en tiempo real) |
| 3.4 | PAN cifrado AES-256-GCM con DEK única por registro |
| 3.5 / 3.5.1 | KEK custodiada en KeyStore PKCS12 protegido con passphrase split knowledge |
| 3.6 / 3.6.1 | Procedimiento de generación documentado (ceremonia) |
| 3.6.2 | Transmisión segura de mitades (sobres sellados) |
| 3.6.3 | Almacenamiento seguro (Secrets OpenShift + RBAC) |
| 3.6.4 | Rotación periódica (anual) |
| 3.6.5 | Retiro de llaves comprometidas (backup + destrucción) |
| 3.6.6 | Split knowledge (2 custodios, 2 mitades) |
| 3.6.7 | Dual control (operaciones requieren 2 personas) |

## Limitaciones conocidas y mitigaciones

| Limitación | Mitigación |
|---|---|
| privKey vive en RAM del JVM | Monitoreo de acceso al pod, detección de anomalías |
| Pod restart requiere reconstrucción del password | Ambos Secrets deben estar disponibles en startup |
| Si ambos custodios se van de la empresa | Usar Shamir's Secret Sharing (3 de 5 partes) |
| Backup del .p12 es un vector de ataque | Backup cifrado adicional con KEK de backup separada |
| Rotación requiere downtime o logic dual-version | Implementar soporte multi-versión desde el inicio |

## Alternativa futura: Shamir's Secret Sharing

En vez de 2 mitades rígidas, dividir el password en N partes con threshold M (ej: 3 de 5). Ventaja: tolerancia a fallos si un custodio se va.

```bash
apt-get install ssss

MASTER=$(openssl rand -base64 32)
echo "$MASTER" | ssss-split -t 3 -n 5
# Genera 5 partes, se necesitan 3 para reconstruir
```
