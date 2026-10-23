# PHASE 2 Shared Kernel and Common Infrastructure

## Reading boundary

This file is the complete implementation spec for TASK-2.1 through TASK-2.4. Read only this file and the applicable root/backend `AGENTS.md` files while executing a Phase 2 task.

## Phase outcomes

- CCCD encryption and digest primitives comply with Nghị định 13/2023/NĐ-CP.
- All media flows use direct browser-to-Cloudflare R2 transfer.
- Transactional email is delivered only after commit with retry, idempotency and batch limits.
- Redis rate limiting enforces every SRS v2 endpoint quota atomically.

## Atomic task contract

### TASK-2.1 AesGcmCccdConverter and Sha256DigestUtil

- Package: `vn.saferent.common.crypto`.
- Required classes: `AesGcmCccdConverter`, `Sha256DigestUtil`, a key-provider port/configuration and redaction-safe exceptions.
- AES MUST be 256-bit GCM with a fresh 12-byte IV and 128-bit tag. Ciphertext MUST carry a format/key version.
- SHA-256 output MUST be deterministic lowercase 64-hex over a documented canonical input.
- NEVER log plaintext, key, IV+ciphertext envelope, or digest input.

### TASK-2.2 CloudflareR2StorageService

- Package: `vn.saferent.common.storage`.
- Required types/methods from SADD: `CloudflareR2StorageService`, `generatePresignedPutUrl(...)`, `generatePrivatePresignedGetUrl(...)`, `PresignedUploadResponse`, `R2Properties`.
- Required buckets: `saferent-public-media`, `saferent-private-vault`.
- PUT validity: 10 minutes. Private GET validity: exactly 5 minutes.
- Server MUST choose bucket/key, bind MIME/length/SHA metadata, enforce <=15 MB baseline and verify with `HeadObject` before business persistence.
- NEVER add a binary upload/download proxy controller.

### TASK-2.3 ResendEmailClient and TransactionalEmailEventListener

- Package: `vn.saferent.common.notification`.
- Required names: `ResendEmailClient`, `TransactionalEmailEventListener`, `onLeaseSigned(LeaseSignedEvent)`, `onInvoiceBatchIssued(InvoiceBatchIssuedEvent)`.
- Listener MUST use `@Async("virtualThreadExecutor")`, `@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)`, transient-only `@Retryable`, and deterministic provider idempotency keys.
- Batch endpoint is `POST /emails/batch`; maximum 100 messages per request.
- No provider call, template rendering or R2 binary download may occur inside the originating DB transaction.

### TASK-2.4 RedisSlidingWindowRateLimitFilter

- Package: `vn.saferent.common.ratelimit`.
- Required name: `RedisSlidingWindowRateLimitFilter`.
- Key: `rl:v2:{apiGroup}:{subjectId}`.
- Operation MUST be atomic using Redis Lua or an equivalent proven primitive.
- Return stable HTTP 429 with retry metadata; NEVER disclose protected-resource existence.

## Verbatim source excerpts

### SRS CCCD DDL

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    full_name VARCHAR(150) NOT NULL,
    date_of_birth DATE,
    phone_number VARCHAR(20) UNIQUE NOT NULL,
    email VARCHAR(150) UNIQUE,
    role VARCHAR(30) NOT NULL CHECK (role IN ('TENANT', 'LANDLORD_OWNER', 'PROPERTY_MANAGER', 'INSPECTOR', 'PARTNER', 'ADMIN')),
    cccd_encrypted TEXT,
    -- Mã hóa AES-256-GCM (Số CCCD + Địa chỉ thường trú phục vụ xuất CT01)
    cccd_hash VARCHAR(64) UNIQUE,
    -- SHA-256 chống trùng lặp
    ekyc_verified BOOLEAN DEFAULT FALSE,
    reputation_points INT DEFAULT 0,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
```

### SRS privacy and retention

> Mã hóa & Phân quyền Tách biệt (Data Isolation): Thông tin CCCD phục vụ xuất tờ khai tạm trú CT01 được mã hóa chuẩn AES-256-GCM (Key lưu trên Azure Key Vault / AWS KMS). Tài khoản `PROPERTY_MANAGER` (Quản lý/Bảo vệ) không được xem số CCCD gốc trừ khi được `LANDLORD_OWNER` cấp quyền xuất file CT01.

> Cơ chế Tự động Xóa Dữ liệu Sau Khi Hết Hạn (Data Retention Policy): Sau khi Hợp đồng chuyển sang trạng thái `TERMINATED` (Đã trả phòng và hoàn tất quyết toán cọc) quá 90 ngày, hệ thống tự động xóa ảnh chụp mặt CCCD gốc trên Cloudflare R2 để giảm thiểu rủi ro rò rỉ dữ liệu cá nhân.

### SADD AES-GCM code example

```java
// 1. JPA AttributeConverter: Tự động mã hóa AES-256-GCM cho trường CCCD trước khi lưu xuống PostgreSQL
@Converter
@Component
public class AesGcmCccdConverter implements AttributeConverter<String, String> {
    private static final String ALGO = "AES/GCM/NoPadding";
    private static final int IV_LENGTH = 12;
    private static final int TAG_BIT_LENGTH = 128;
    private final SecretKey masterKey; // Nạp từ biến môi trường KMS / KeyVault

    @Override
    public String convertToDatabaseColumn(String plainTextCccdJson) {
        if (plainTextCccdJson == null) return null;
        byte[] iv = new byte[IV_LENGTH];
        new SecureRandom().nextBytes(iv);
        Cipher cipher = Cipher.getInstance(ALGO);
        cipher.init(Cipher.ENCRYPT_MODE, masterKey, new GCMParameterSpec(TAG_BIT_LENGTH, iv));
        byte[] cipherText = cipher.doFinal(plainTextCccdJson.getBytes(StandardCharsets.UTF_8));
        byte[] combined = ByteBuffer.allocate(iv.length + cipherText.length).put(iv).put(cipherText).array();
        return Base64.getEncoder().encodeToString(combined);
    }
    // convertToEntityAttribute(...) giải mã tương ứng khi đọc lên phục vụ xuất Mẫu CT01
}
```

The source code example declares checked cryptographic operations without showing exception handling and annotates the converter with Spring. The implementation MUST preserve cryptographic behavior while following the backend rule that technical adapters may use framework annotations and errors must be redaction-safe.

### SADD bucket split

> 1. Bucket Công khai (saferent-public-media): Lưu ảnh tin đăng phòng trọ và ảnh trang bị PCCC. Được gắn Custom Domain qua Cloudflare CDN để phục vụ hàng chục nghìn lượt xem bản đồ với phí băng thông tải ra (Egress Fee) = 0đ.

> 2. Bucket Bảo mật Tuyệt đối (saferent-private-vault): Chặn 100% truy cập công khai. Chuyên lưu ảnh chụp CCCD, file PDF hợp đồng điện tử, ảnh/video biên bản bàn giao Check-in/Check-out, ảnh gốc công tơ điện và hóa đơn sửa chữa. Chỉ có thể Ghi (PUT) hoặc Đọc (GET) thông qua Presigned URL ngắn hạn.

### SADD direct upload request and validation

```text
POST /api/v2/storage/presigned-put
{category, targetEntityId, contentType, contentLength, sha256Hex}

- Xác thực JWT & quyền RBAC trên Entity
- Kiểm tra MIME Type & Size <= 15MB
- Ký Offline bằng AWS S3Presigner
  (Hiệu lực PUT: 10 phút, khóa cứng Content-Type & metadata sha256)

HTTP PUT <Binary Stream> trực tiếp từ trình duyệt lên Cloudflare R2
Headers: Content-Type, x-amz-meta-sha256

Gọi API Nghiệp vụ kèm {objectKey, sha256}
Gọi s3Client.headObject(objectKey) xác minh file tồn tại & khớp SHA-256
Lưu objectKey & sha256 vào PostgreSQL
```

### SADD CloudflareR2StorageService code example

```java
@Service
@RequiredArgsConstructor
public class CloudflareR2StorageService {
    private final S3Presigner s3Presigner;
    private final S3Client s3Client;
    private final R2Properties r2Props; // Chứa tên publicBucket và privateBucket

    // 1. Sinh Presigned PUT URL cho phép React upload trực tiếp lên R2 (Hiệu lực 10 phút)
    public PresignedUploadResponse generatePresignedPutUrl(
            StorageCategory category,
            UUID ownerId,
            String contentType,
            long contentLength,
            String sha256Hex) {
        validateFileConstraints(category, contentType, contentLength);
        String bucket = category.isPrivate() ? r2Props.getPrivateBucket() : r2Props.getPublicBucket();
        String objectKey = String.format("%s/%s/%s_%s", category.getPrefix(), ownerId, Instant.now().toEpochMilli(), UUID.randomUUID());

        PutObjectRequest objectRequest = PutObjectRequest.builder()
                .bucket(bucket)
                .key(objectKey)
                .contentType(contentType)
                .contentLength(contentLength)
                .metadata(Map.of("sha256", sha256Hex, "uploaded-by", ownerId.toString()))
                .build();

        PutObjectPresignRequest presignRequest = PutObjectPresignRequest.builder()
                .signatureDuration(Duration.ofMinutes(10))
                .putObjectRequest(objectRequest)
                .build();

        PresignedPutObjectRequest presignedRequest = s3Presigner.presignPutObject(presignRequest);
        return new PresignedUploadResponse(presignedRequest.url().toString(), objectKey, Instant.now().plusSeconds(600));
    }

    // 2. Sinh Presigned GET URL (Hiệu lực đúng 5 phút theo mục 6.1 SRS) để xem tài liệu nhạy cảm
    public String generatePrivatePresignedGetUrl(String objectKey, String downloadFileName) {
        GetObjectRequest getObjectRequest = GetObjectRequest.builder()
                .bucket(r2Props.getPrivateBucket())
                .key(objectKey)
                .responseContentDisposition("inline; filename=\"" + downloadFileName + "\"")
                .build();

        GetObjectPresignRequest presignRequest = GetObjectPresignRequest.builder()
                .signatureDuration(Duration.ofMinutes(5))
                .getObjectRequest(getObjectRequest)
                .build();

        return s3Presigner.presignGetObject(presignRequest).url().toString();
    }
}
```

### SADD async email code example

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class TransactionalEmailEventListener {
    private final ResendEmailClient resendClient;
    private final CloudflareR2StorageService r2StorageService;
    private final SpringTemplateEngine thymeleafEngine;

    @Async("virtualThreadExecutor")
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    @Retryable(
        retryFor = { RestClientException.class },
        maxAttempts = 4,
        backoff = @Backoff(delay = 2000, multiplier = 2.0)
    )
    public void onLeaseSigned(LeaseSignedEvent event) {
        byte[] contractPdfBytes = r2StorageService.downloadPrivateObjectBytes(event.contractPdfObjectKey());
        String htmlContent = renderTemplate("email/lease-signed", Map.of(
            "leaseCode", event.leaseId(),
            "roomNumber", event.roomNumber(),
            "sha256Hash", event.contractPdfSha256()
        ));
        ResendSendEmailRequest request = ResendSendEmailRequest.builder()
            .from("SafeRent VN <no-reply@saferent.vn>")
            .to(event.allParticipantEmails())
            .subject("[SafeRent VN] Hợp đồng thuê nhà điện tử đã ký #" + event.roomNumber())
            .html(htmlContent)
            .attachments(List.of(new ResendAttachment(
                "Hop_Dong_Thue_Nha_" + event.roomNumber() + ".pdf",
                Base64.getEncoder().encodeToString(contractPdfBytes)
            )))
            .build();
        resendClient.sendEmail(request, "lease-signed-" + event.leaseId());
    }

    @Async("virtualThreadExecutor")
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    @Retryable(retryFor = { RestClientException.class }, maxAttempts = 3, backoff = @Backoff(delay = 3000))
    public void onInvoiceBatchIssued(InvoiceBatchIssuedEvent event) {
        List<ResendSendEmailRequest> batchPayloads = event.invoices().stream()
            .map(inv -> buildInvoiceEmailPayload(inv))
            .toList();
        Lists.partition(batchPayloads, 100).forEach(chunk ->
            resendClient.sendBatchEmails(
                chunk,
                "batch-invoice-" + event.propertyId() + "-" + event.billingMonth()
            )
        );
    }
}
```

The SADD example downloads an attachment inside the async listener, not the originating transaction. Implementation MUST additionally make crash recovery reliable with an outbox if required by backend `AGENTS.md`.

### SRS and SADD rate limits

```text
GET /api/v2/properties/search: 60 req/phút/IP
POST /api/v2/contracts/ai-scan: 1 req/ngày (Guest), 3 req/ngày (Tenant)
POST /api/v2/leases/{id}/members: 10 req/phút/User
GET /api/v2/leases/{id}/export-ct01: 10 req/phút/User
POST /api/v2/leases/{id}/check-out: 10 req/phút/User
POST /api/v2/invoices/ocr-meter: 120 req/phút/User
POST /api/v2/webhooks/vietqr: Whitelist IP
```

> Key: rl:v2:{apiGroup}:{subjectId} (Redis Lua Script / Bucket4j)

## Derived implementation contract

This section is normative but not verbatim.

- Encryption envelope MUST include at least format version, key ID/version, IV and ciphertext+tag. Rotation MUST allow old ciphertext to decrypt while new writes use the active key.
- Normalize CCCD to exactly 12 ASCII digits before hashing; reject any other form. Do not salt `cccd_hash` because it serves deterministic duplicate detection, but protect lookup access and never expose it.
- R2 object keys MUST be server-generated and category-prefixed. `targetEntityId` authorization MUST happen before signing and again before registration/private GET.
- Persist an immutable upload registration/audit record sufficient to prove owner, entity, category, hash, content type and size.
- Resend idempotency key MUST include immutable event ID/version. Split batch keys by chunk index to avoid one key representing multiple changing payloads.
- Retry only timeout/network/429/retryable 5xx. Do not retry authentication/validation 4xx.
- Rate-limiter clocks and sorted-set trimming/counting MUST be atomic. Subject selection MUST follow endpoint contract: IP, IP+device, or authenticated user.

## Formula JSON and UAT inventory

- No business arithmetic formula or LLM JSON schema is owned by Phase 2.
- No standalone Gherkin UAT is assigned to Phase 2. Phase 2 provides mandatory infrastructure for UAT-02, UAT-05 and all media/email paths.

## Verification gate

- Encryption tests: randomized ciphertext, exact round-trip, null behavior, tamper failure, wrong-key/version failure, known SHA-256 vector, no secret in exception/log capture.
- R2 tests: bucket classification, signed headers/metadata, exactly 600-second PUT, exactly 300-second private GET, ownership denial, invalid MIME/size/hash, HeadObject mismatch, and proof no multipart/binary controller exists.
- Email tests: no send before commit/on rollback, async executor invocation, deterministic idempotency, transient retry/backoff, permanent failure no retry, batch sizes 100/100/remainder.
- Rate-limit tests: all quotas, exact boundary, window expiry, parallel requests, stable 429 and subject isolation.
- `mvn test` MUST pass.
