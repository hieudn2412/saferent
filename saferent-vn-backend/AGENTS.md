# SafeRent VN Backend Rules

## Scope

These rules apply to `saferent-vn-backend/**` and extend the repository-level `AGENTS.md`. They are mandatory for production code, migrations, configuration, scripts, and tests.

## Required runtime and configuration

- MUST compile and run on Java 21 LTS and Spring Boot 3.3 or newer within the 3.3 line unless an approved upgrade ADR exists.
- MUST enable Java virtual threads. I/O integrations MAY block on virtual threads; CPU-bound work MUST remain bounded.
- MUST use PostgreSQL 16, PostGIS 3.4, Redis 7 through Lettuce, Flyway, Spring Security 6, Bean Validation, Spring Retry, ShedLock, AWS SDK Java v2 S3 client/presigner, Apache POI, and Testcontainers.
- MUST set `spring.jpa.open-in-view=false`. NEVER access a lazy association from a controller/serializer.
- MUST keep the Hikari pool at 20 maximum connections for the reference deployment. NEVER increase it to mask long transactions or external I/O inside a transaction.
- MUST batch invoice/member writes with Hibernate batch size 30 and ordered inserts/updates.
- Secrets MUST come from environment/KMS/secret injection. NEVER commit R2, Redis, Resend, database, JWT, AES, PayOS, or SePay credentials.

Use this baseline configuration and preserve its semantics:

```yaml
spring:
  threads:
    virtual:
      enabled: true
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 10000
      max-lifetime: 1800000
  jpa:
    open-in-view: false
    properties:
      hibernate:
        jdbc:
          batch_size: 30
        order_inserts: true
        order_updates: true
        dialect: org.hibernate.dialect.PostgreSQLDialect
```

Production MUST validate schema through Flyway; NEVER use `ddl-auto=create`, `update`, or `create-drop`. The production profile MUST use `ddl-auto=validate` or equivalent validation.

## Package-by-module Clean Architecture

The source tree MUST follow this shape:

```text
src/main/java/vn/saferent/
├── SafeRentApplication.java
├── common/
│   ├── config/
│   ├── security/
│   ├── crypto/
│   ├── storage/
│   ├── notification/
│   ├── ratelimit/
│   └── exception/
└── modules/
    ├── identity/
    ├── discovery/
    ├── smartlease/
    ├── operations/
    └── community/
```

Every business module MUST contain exactly these architectural layers:

```text
<module>/
├── domain/          # entities, value objects, policies, domain events, ports
├── application/     # commands/queries, use cases, transaction boundary
├── infrastructure/  # JPA/Redis/R2/LLM/OCR/Resend/provider adapters
└── presentation/    # REST/SSE DTOs, controllers, validation, HTTP mapping
```

Dependency direction MUST be `presentation -> application -> domain` and `infrastructure -> domain/application ports`. Domain MUST depend on nothing outside the JDK and explicitly approved tiny utility libraries. Application MUST NOT depend on presentation. A module MUST NOT import another module's infrastructure or presentation classes.

Cross-module behavior MUST use a narrow application facade or an immutable event. Cross-module JPA entity associations are forbidden; reference another module by UUID/value object and query through a port. NEVER expose a JPA entity from a controller.

`vn.saferent.common` MUST contain only truly shared technical primitives and stable shared-kernel value types. NEVER move a business rule into `common` merely to bypass a module dependency test.

## Pure domain policy

The following classes and all equivalent calculation policies MUST be plain Java:

- `TrueCostPolicy`
- `EvnTieredTariffByCapitaPolicy`
- `MeterRolloverPolicy`
- `DepositSettlementPolicy`
- `FireSafetyBadgePolicy`
- `ReviewModerationPolicy`

They MUST NOT import `org.springframework.*`, `jakarta.persistence.*`, `org.hibernate.*`, HTTP DTOs, repositories, SDK clients, clocks tied to the system, or configuration properties. Inject time and tariff/reference data as arguments or domain ports.

Correct money style:

```java
public BigDecimal refundableDeposit(
        BigDecimal deposit,
        BigDecimal proratedRent,
        BigDecimal finalUtilities,
        BigDecimal confirmedTenantFaultDamage) {
    return deposit
            .subtract(proratedRent)
            .subtract(finalUtilities)
            .subtract(confirmedTenantFaultDamage)
            .max(BigDecimal.ZERO);
}
```

- MUST use `BigDecimal.valueOf(...)` or string constructors for decimal constants. NEVER use `new BigDecimal(double)`.
- MUST define scale and `RoundingMode` at the policy boundary. NEVER round intermediate tariff tiers unless the authoritative tariff requires it.
- Domain guards MUST reject negative money, invalid dates, impossible occupant counts, unsupported status transitions, and invoice unit prices above `leases.electricity_rate_committed`.
- State transitions MUST occur through intention-revealing methods; NEVER expose public entity setters for financial or lifecycle fields.

## Transactions and external I/O

- Put `@Transactional` on application use-case methods, not controllers and not private helpers.
- A transaction MUST perform only required database/Redis coordination. NEVER call LLM, OCR, R2 binary download, Resend, Web Push, Zalo, PayOS, SePay, document rendering, or another slow network service while holding a database transaction.
- Publish an immutable event during the transaction and process side effects only after a successful commit.
- Use `READ_COMMITTED` plus the explicit locks from the concurrency matrix. NEVER raise global isolation to `SERIALIZABLE` as a substitute for targeted correctness.
- Translate optimistic-lock and unique-constraint losers to stable HTTP 409 problem responses. Translate authorization failure to 403 without revealing protected resource details.
- All provider webhooks MUST verify signature, timestamp/replay window, provider identity/IP rule where specified, and idempotency before mutation.

## Mandatory concurrency matrix

### Double booking

- `rooms`, `leases`, `handover_records`, `utility_invoices`, and `verified_reviews` MUST have optimistic `@Version` where specified by the SADD.
- Flyway MUST create `uq_active_lease_per_room` for `ACTIVE` and `PENDING_CHECKOUT` leases.
- Lease activation MUST handle both `ObjectOptimisticLockingFailureException` and `DataIntegrityViolationException` as HTTP 409.

### VietQR partial and excess payments

Claim the event before database work:

```text
SET webhook:vietqr:<bankTxId> 1 NX EX 86400
```

Then, within one transaction, lock both invoice and lease:

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("select i from UtilityInvoiceJpaEntity i where i.id = :id")
Optional<UtilityInvoiceJpaEntity> findByIdForUpdate(UUID id);
```

- Add transfer amount to existing `paid_amount`; NEVER overwrite cumulative payment with the latest transfer.
- If cumulative `paid_amount < total_amount`, set `PARTIALLY_PAID`; the outstanding balance feeds the next invoice's `previous_debt`.
- If equal, set `PAID`.
- If greater, set `PAID` and add only the excess to `leases.credit_balance` exactly once.
- Persist an immutable provider transaction record; a Redis TTL alone MUST NOT be the permanent audit/idempotency record.

### Maintenance deduction

- Database MUST enforce `UNIQUE (lease_id, billing_month)`.
- Select only tickets with `is_deduction_approved = TRUE AND deducted_in_invoice_id IS NULL` using `FOR UPDATE`.
- Sum `tenant_advance_amount`, write `utility_invoices.maintenance_deduction`, create the invoice, and mark every ticket's `deducted_in_invoice_id` in the same transaction.
- `PROPERTY_MANAGER` MUST NOT approve a deduction. Approval requires `LANDLORD_OWNER` ownership of the property.

### Co-tenant activation

- Lock the parent lease with `PESSIMISTIC_WRITE` before checking capacity or inserting `lease_members`.
- Count active members using `is_active = TRUE` and active date semantics; compare against `rooms.max_occupants`.
- Set `leases.occupant_count` to the database count of active members after insertion/removal. NEVER perform `occupant_count = occupant_count + 1` without the parent lock.

### 48-hour jobs

- Check-in auto-lock and negative-review auto-publish MUST use `@SchedulerLock` and an entity `@Version`.
- Use a database/server clock abstraction and persisted deadline; NEVER calculate correctness from a browser clock.
- A landlord response committed before the deadline MUST prevent auto-action even if the scheduler started concurrently.

### OCR double-tap and tariff guard

- Acquire `lock:ocr:room:<roomId>` with a 5-second TTL and owner token; release only with compare-and-delete Lua semantics.
- Verify OCR `Meter_Serial` against `rooms.electric_meter_serial` before invoice creation.
- `UtilityInvoice` creation MUST guard `unitPrice.compareTo(lease.getElectricityRateCommitted()) > 0`.
- `MeterRolloverPolicy` MUST use `maxDial - oldIndex + newIndex` only when `newIndex < oldIndex` and `oldIndex >= 0.90 * maxDial`; otherwise require the explicit meter-replacement evidence flow or reject.

## Cloudflare R2 adapter

`CloudflareR2StorageService` MUST only issue URLs and metadata verification for application flows. It MUST NOT expose a backend binary upload/download endpoint.

```java
PutObjectRequest request = PutObjectRequest.builder()
        .bucket(category.isPrivate() ? privateBucket : publicBucket)
        .key(objectKey)
        .contentType(contentType)
        .contentLength(contentLength)
        .metadata(Map.of("sha256", sha256Hex, "uploaded-by", ownerId.toString()))
        .build();

PutObjectPresignRequest signed = PutObjectPresignRequest.builder()
        .signatureDuration(Duration.ofMinutes(10))
        .putObjectRequest(request)
        .build();
```

- Object keys MUST be generated server-side from an allowlisted category prefix and opaque UUID; NEVER accept a caller-supplied bucket or raw key.
- `validateFileConstraints` MUST enforce an allowlisted MIME type and the category-specific limit; no accepted object may exceed the SADD baseline of 15 MB.
- PUT signatures MUST bind content type, content length when supported, SHA-256 metadata, and the selected bucket/key.
- Private GET signatures MUST last exactly `Duration.ofMinutes(5)` and MUST be issued only after role plus resource ownership/consent checks.
- A business use case MUST verify the uploaded object through `HeadObject` before saving the reference.
- The 90-day CCCD cleanup MUST delete only eligible private object keys, tolerate missing objects, record an audit result, and be idempotent.

## Async Resend email pipeline

NEVER call Resend synchronously inside a database transaction. The required listener shape is:

```java
@Async("virtualThreadExecutor")
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
@Retryable(
    retryFor = RestClientException.class,
    maxAttempts = 4,
    backoff = @Backoff(delay = 2_000, multiplier = 2.0))
public void onLeaseSigned(LeaseSignedEvent event) {
    resendClient.sendEmail(buildLeaseEmail(event), "lease-signed-" + event.leaseId());
}
```

- Configure `virtualThreadExecutor` explicitly and test that `@Async` is applied through a Spring proxy.
- Every send MUST include a deterministic Resend `Idempotency-Key` derived from immutable business identity and event version.
- Retry only transient timeouts, connection errors, HTTP 429, and retryable 5xx responses. NEVER retry validation/authentication 4xx responses.
- Invoice issue email MUST use Resend Batch API with chunks of at most 100 emails. Each chunk MUST have its own deterministic idempotency key including property, billing month, event ID, and chunk index.
- Use an explicit bounded partition such as `Lists.partition(batchPayloads, 100)`; NEVER submit an unbounded collection to the Resend Batch API.
- Rendering or attachment-fetch failure MUST be observable and retryable without resending already accepted messages.
- If reliable delivery is required across process crashes, persist an outbox record in the originating transaction and dispatch it after commit; in-memory events alone are insufficient.

## Security implementation

- JWT access token TTL MUST be 15 minutes. Refresh token rotation state MUST live in Redis with a 7-day TTL and reuse detection.
- Endpoint authorization MUST combine `hasRole(...)` and `ResourceOwnershipEvaluator`; repository queries MUST also scope records by authorized owner/manager/member where practical.
- The Owner bank update endpoint MUST use `@RequireStepUpOtp(action = "UPDATE_VIETQR_BANK_ACCOUNT")`. Property Manager MUST receive 403.
- `AesGcmCccdConverter` MUST encrypt `users.cccd_encrypted` with AES/GCM/NoPadding, 256-bit key, fresh 12-byte IV, 128-bit tag, and an authenticated versioned envelope. `users.cccd_hash` MUST be SHA-256 normalized consistently for duplicate detection.
- CCCD plaintext MAY be decrypted only for a specifically authorized eKYC/CT01 use case and MUST remain request-scoped. NEVER put plaintext in cache, event, job payload, DTO outside that use case, or audit log.
- Validate all DTOs with explicit size/range/enum constraints. Use RFC 7807-style stable problem responses. NEVER echo untrusted provider or LLM text in error details.

## Cache and rate limiting

- Redis L2 geo key: `geo:v2:{city}:{geohash6}:{radius}:{filterHash}`, TTL 5 minutes. Cache base room parameters, then recompute True Cost for requested `N` in memory.
- Redis L2 property key: `prop:v2:detail:{propertyId}`, TTL 30 minutes. Evict after property/room change, lease activation, review publication, badge change, or an overdue AC ticket that changes AEF to 1.40.
- Caffeine L1 MUST cache current EVN tariff reference and administrative codes for 12 hours. Cache values MUST be immutable/versioned and refreshable.
- Redis sliding-window key MUST be `rl:v2:{apiGroup}:{subjectId}` and enforce the endpoint quotas in root rules.
- NEVER cache private CCCD, OTPs, bank secrets, Presigned URLs, raw contract content, or authorization decisions across principals.

## Database migration rules

- Flyway migrations MUST be forward-only, deterministic, transactional where PostgreSQL permits, and safe against production data.
- Monetary columns MUST use `DECIMAL/NUMERIC`, UUID keys MUST use UUID, and timestamps MUST use `TIMESTAMPTZ` for instants.
- Index every foreign key and query path used by ownership checks, active-member counts, invoice month lookup, review deadlines, and cleanup jobs.
- Preserve the PostGIS GIST index on `properties.location` and use `ST_DWithin` with the correct geography units.
- CHECK/UNIQUE/FOREIGN KEY/NOT NULL constraints MUST mirror domain invariants. Application validation never replaces database constraints.

## Required tests

- Domain policy tests MUST run without Spring and cover all branches, boundaries, zero/negative rejection, and exact `BigDecimal` results.
- Repository integration tests MUST use PostgreSQL 16/PostGIS and Redis Testcontainers; H2 is forbidden for spatial queries, partial indexes, locks, and production DDL behavior.
- Concurrency tests MUST use real parallel transactions and assert exactly one winner for double booking, invoice creation, ticket deduction, member count, and webhook replay.
- Security tests MUST cover every role, ownership mismatch, inactive member, manager of another property, missing/invalid step-up OTP, and UAT-05 403.
- R2 tests MUST assert bucket classification, 10-minute PUT, exactly 5-minute private GET, signed metadata, ownership checks, and absence of any backend binary endpoint.
- Email tests MUST assert AFTER_COMMIT behavior, no send on rollback, async execution, transient retry, deterministic idempotency, and batch chunks of 100 or fewer.
- ArchUnit MUST fail on Spring/JPA imports in domain policies, forbidden layer direction, cross-module infrastructure imports, and controllers exposing persistence entities.
- Integration/UAT tests MUST implement UAT-01 through UAT-06 exactly as defined in the root rules.
