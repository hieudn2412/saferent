# SafeRent VN v2.0 Global Project Rules

## Scope and sources of truth

These instructions apply to the entire repository. A deeper `AGENTS.md` MAY add stricter module rules but MUST NEVER weaken this file.

Treat these documents as normative:

1. `docs/SafeRent_VN_SRS_v2.0_Professional.docx` defines product behavior, legal constraints, database vocabulary, API behavior, RBAC, and UAT-01 through UAT-06.
2. `docs/SafeRent_VN_Architecture_Design_Document_v2.docx` defines the implementation architecture, concurrency controls, direct Cloudflare R2 transfer, async Resend pipeline, security, and caching.

When the documents differ, preserve the SRS business/legal outcome and implement it with the SADD technical design. NEVER silently invent a third behavior. Record unresolved conflicts in the change description and obtain an explicit architecture decision before merging.

Use the exact ubiquitous-language names in the documents: `LANDLORD_OWNER`, `PROPERTY_MANAGER`, `PRIMARY`, `CO_TENANT`, `STATE_TIERED_BY_CAPITA`, `FIXED_RATE`, `PENDING_CHECKOUT`, `PARTIALLY_PAID`, and `PENDING_LANDLORD_RESPONSE`. NEVER introduce synonyms that split the domain model.

## Product and unit economics first

- MUST optimize for a landlord SaaS price of VND 150,000–500,000 per building per month, 5,000 MAU, and a peak of 500 requests/minute during days 28 through 05.
- MUST keep the reference MVP infrastructure below approximately USD 40/month where the SADD assumptions hold: normally 1–2 Spring Boot instances at about 2 vCPU/4 GB plus PostgreSQL and Redis. Treat higher recurring cost as an architectural decision requiring a measured justification.
- MUST keep the initial deployment a Modular Monolith. NEVER introduce microservices, Kubernetes, a service mesh, or a distributed saga unless an approved ADR proves that the product has outgrown the SADD target.
- MUST keep ordinary API p95 below 300 ms; the architecture target for the main application path is p95 below 150 ms. Move AI, OCR, PDF/CT01 generation, bulk notifications, and other slow I/O out of the request transaction.
- ALWAYS prefer Web Push PWA and Resend email, then a one-tap personal Zalo deep link. Zalo ZNS MUST be quota-controlled and reserved for paid plans; NEVER send unbounded ZNS/SMS reminders.
- MUST rate-limit paid APIs and deduplicate paid work. A repeated click, retry, or webhook MUST NOT create a second OCR charge, LLM charge, email, invoice, deduction, or payment posting.

## Architectural guardrails

- Backend MUST use Java 21, Spring Boot 3.3+, PostgreSQL 16 with PostGIS 3.4, Redis 7, and the package-by-module Clean/Hexagonal structure under `vn.saferent`.
- Frontend MUST use React 18 with TypeScript and Vite PWA, TanStack Query v5, Zustand, Tailwind CSS with Shadcn UI, Dexie.js with Workbox, and Mapbox GL or MapLibre GL.
- The only backend business modules are `identity`, `discovery`, `smartlease`, `operations`, and `community`. Shared technical code belongs in `vn.saferent.common`; shared business entities MUST NOT become a dumping ground in `common`.
- A module MUST expose behavior through its application boundary or published domain events. NEVER import another module's `infrastructure` or `presentation` package. ArchUnit/Spring Modulith tests MUST enforce the boundaries.
- Domain calculation policies MUST be plain Java with zero Spring, JPA, Hibernate, web, storage, or vendor imports. Money and measured numeric values MUST use `BigDecimal`; NEVER use `float` or `double` for rent, tariffs, debt, credit, deductions, deposits, or meter arithmetic.

## Zero-bandwidth media rule

Binary media MUST NEVER pass through the Spring Boot application. This prohibition covers meter images, fire-safety images, both sides of CCCD, lease PDFs, CT01 output, repair receipts, and check-in/check-out images or video.

The mandatory flow is:

1. The browser MUST crop/compress/hash the file locally.
2. The browser calls `POST /api/v2/storage/presigned-put` with `{category, targetEntityId, contentType, contentLength, sha256Hex}`.
3. The backend MUST authenticate the caller, verify resource ownership, allowlisted MIME type and category limit (never more than 15 MB under the SADD baseline), then return a 10-minute Presigned PUT URL and an opaque `objectKey`.
4. The browser MUST PUT the binary directly to R2 with the signed `Content-Type` and `x-amz-meta-sha256`.
5. The business API receives only `{objectKey, sha256}`. Before persisting, the backend MUST call `HeadObject` and verify existence, expected bucket/key prefix, content length, content type, and SHA-256 metadata.

Bucket rules are absolute:

- `saferent-public-media`: listing and fire-safety evidence only; delivery MAY use a Cloudflare CDN custom domain.
- `saferent-private-vault`: CCCD, signed lease PDF, CT01, original meter evidence, repair receipts, and handover media. Public access MUST remain disabled.
- A private GET MUST require JWT, RBAC plus ownership/consent checks and MUST use a Presigned GET URL with an expiry of exactly 5 minutes.
- NEVER persist a Presigned URL. Persist only `objectKey`, SHA-256, content metadata, owner/entity linkage, and audit data.
- NEVER add a multipart binary controller, `MultipartFile` media proxy, base64 media JSON, or backend download relay.

## Legal, privacy, and integrity requirements

- MUST comply with Nghị định 13/2023/NĐ-CP. Store CCCD payload in `users.cccd_encrypted` using randomized AES-256-GCM through a JPA `AttributeConverter`; store only the deterministic SHA-256 duplicate-detection digest in `users.cccd_hash` (`VARCHAR(64) UNIQUE`).
- AES-GCM MUST use a fresh 12-byte IV and a 128-bit authentication tag per encryption. The key MUST come from KMS/Key Vault or an injected secret; NEVER commit, log, cache, or return the key or plaintext CCCD.
- Logs, traces, exceptions, analytics, prompts, and domain events MUST NEVER contain plaintext CCCD, full phone numbers, signed URLs, JWTs, OTPs, bank account secrets, or raw contract text containing PII.
- PII MUST be masked before any contract text is sent to an LLM. Mask at minimum CCCD, phone number, and full name.
- The CCCD retention job MUST delete original CCCD face images from `saferent-private-vault` exactly when the related lease has been `TERMINATED` for more than 90 days. It MUST be idempotent, audited, and safe to retry. Retention cleanup MUST NOT delete `cccd_hash` or legally required audit metadata without an approved policy.
- Electronic consent evidence MUST retain eKYC result, verified phone OTP, signing IP, server timestamp, and `contract_pdf_sha256`. Check-in/out evidence MUST retain GPS, server timestamp in UTC+7 display semantics, lease identifier, and SHA-256.
- SafeRent MUST call the fire badge “Xác thực Hiện trạng Trang bị An toàn / Safety Equipment Verified”. NEVER claim to issue an official fire-police certificate.
- SafeRent MUST NOT hold tenant deposits in its own account during MVP. Use the signed lease, immutable handover evidence, settlement statement, and tenant VietQR for owner-to-tenant refund.

## RBAC and resource ownership

Authorization MUST combine role checks with resource ownership/assignment checks on every protected operation. A role alone is NEVER sufficient.

| Actor | Stored identity | Mandatory authorization rules |
|---|---|---|
| Guest | Anonymous | MAY search public properties/reviews. AI scan is limited to 1/day by IP plus device fingerprint. Mask the last four landlord phone digits. |
| Primary Tenant | `TENANT` + active `lease_members.member_role = PRIMARY` | MAY sign, invite co-tenants, submit check-in, issue termination notice, pay, create tickets, review, and confirm checkout settlement for its own lease. |
| Co-Tenant | `TENANT` + active `lease_members.member_role = CO_TENANT` | MAY view lease/invoices, pay, create tickets, and write a verified review for its own lease. MUST NOT terminate the lease or change the Primary Tenant refund account. |
| Property Manager | `PROPERTY_MANAGER` + `properties.manager_id = userId` | MAY capture meters, inspect handover, process technical ticket work, and download CT01 only with Owner grant. MUST receive 403 when changing VietQR bank data, approving rent deductions, or viewing aggregate revenue. |
| Landlord Owner | `LANDLORD_OWNER` + `properties.owner_id = userId` | MAY manage property, assign manager, approve lease, deduction, invoice, refund, and review response. Updating VietQR bank data MUST pass `@RequireStepUpOtp`. |
| Inspector | `INSPECTOR` | MAY perform assigned level-2 fire-safety field verification and upload evidence only for assigned inspections. |
| Partner | `PARTNER` | MAY receive and quote explicitly referred maintenance tickets; MUST NOT see unrelated tenant, financial, or CCCD data. |
| Admin | `ADMIN` | MAY moderate and adjudicate with audit logging. MUST NOT delete a substantiated negative review merely because it is negative. |

The UAT-05 guard is non-negotiable:

```java
@PreAuthorize("hasRole('LANDLORD_OWNER') and @resourceGuard.isPropertyOwner(#propertyId, authentication)")
@RequireStepUpOtp(action = "UPDATE_VIETQR_BANK_ACCOUNT")
@PutMapping("/api/v2/properties/{propertyId}/bank-account")
ResponseEntity<BankConfigResponse> updateBankAccount(/* ... */) { /* ... */ }
```

The UI MUST hide unauthorized controls, but the API MUST independently return `403 Forbidden`. NEVER rely on client-side hiding as authorization.

## Canonical data and database constraints

Use PostgreSQL migrations as the final persistence contract. At minimum, preserve these SRS/SADD names and constraints:

- `users`: `id`, `full_name`, `date_of_birth`, `phone_number UNIQUE NOT NULL`, `email UNIQUE`, `role`, `cccd_encrypted`, `cccd_hash UNIQUE`, `ekyc_verified`, `reputation_points`, `created_at`.
- `properties`: `owner_id`, `manager_id`, address codes, `location GEOGRAPHY(Point,4326)` with GIST index, `fire_safety_level IN (0,1,2)`, `fire_badge_expires_at`, safety flags, `electricity_pricing_type IN ('STATE_TIERED_BY_CAPITA','FIXED_RATE')`, utility prices, `water_pricing_type IN ('PER_M3','PER_PERSON')`, `service_fee_per_person`, and `flood_risk_score BETWEEN 0 AND 5`.
- `rooms`: property, room/floor/area, `max_occupants`, `base_rent_price`, `ac_condition_type`, `electric_meter_serial`, `electric_meter_max_dial DEFAULT 100000`, status, and `version`.
- `leases`: room, `primary_tenant_id`, `landlord_id`, dates, `occupant_count`, `notice_period_days DEFAULT 30`, `termination_requested_at`, rent/deposit, `electricity_rate_committed`, `credit_balance`, `contract_pdf_sha256`, status, and `version`.
- `lease_members`: `lease_id`, `user_id`, `member_role IN ('PRIMARY','CO_TENANT')`, `joined_date`, `left_date`, `residence_reg_status IN ('PENDING_CT01','CT01_EXPORTED','VNEID_CONFIRMED')`, and `is_active`.
- `handover_records`: immutable media/hash evidence, `settlement_summary JSONB`, lifecycle state, deadline timestamps, and `version`.
- `maintenance_tickets`: lease/room/creator, category, `fault_attribution IN ('NATURAL_WEAR','TENANT_FAULT')`, `evidence_urls JSONB`, `tenant_advance_amount`, `is_deduction_approved`, status, and `deducted_in_invoice_id`.
- `utility_invoices`: lease/month, old/new electricity and water indices, `is_meter_rollover`, meter evidence object key, `previous_debt`, `maintenance_deduction`, `total_amount`, `paid_amount`, `vietqr_payload`, payment status, `paid_at`, and `version`.
- `verified_reviews`: MUST support verified membership, evidence, `PENDING_LANDLORD_RESPONSE`, the 48-hour deadline, landlord response, publication state, and `version`.

These database guards MUST exist even when application validation also exists:

```sql
CREATE UNIQUE INDEX uq_active_lease_per_room
ON leases (room_id)
WHERE status IN ('ACTIVE', 'PENDING_CHECKOUT');

ALTER TABLE utility_invoices
ADD CONSTRAINT uq_invoice_lease_month UNIQUE (lease_id, billing_month);

ALTER TABLE maintenance_tickets
ADD COLUMN deducted_in_invoice_id UUID REFERENCES utility_invoices(id);
```

NEVER rename `maintenance_deduction` to the earlier narrative alias `deduction_amount`; the final DDL/UAT field name is canonical.

## Concurrency and idempotency invariants

- Double booking MUST use `@Version` plus `uq_active_lease_per_room`; map losing writes to HTTP 409.
- VietQR webhook MUST first claim `webhook:vietqr:<bankTxId>` with Redis `SET NX EX 86400`, verify the provider signature/IP policy, then lock `utility_invoices` and `leases` with `PESSIMISTIC_WRITE` before updating `paid_amount`, `payment_status`, `previous_debt`, or `credit_balance`.
- Monthly invoice creation MUST rely on `UNIQUE (lease_id, billing_month)`, lock approved tickets where `deducted_in_invoice_id IS NULL`, total `tenant_advance_amount`, create the invoice, and set `deducted_in_invoice_id` in one ACID transaction.
- Co-tenant activation MUST lock the parent `leases` row, enforce `rooms.max_occupants`, insert the active member, and recompute `occupant_count` from active `lease_members`; NEVER use an unlocked increment.
- The 48-hour check-in/review jobs MUST use ShedLock `@SchedulerLock` and entity `@Version`. A valid owner response committed before the deadline MUST win over the job.
- OCR submission MUST acquire `lock:ocr:room:<roomId>` for 5 seconds. Domain code MUST reject `unitPrice > lease.electricityRateCommitted` and MUST apply the rollover rule when appropriate.

## API and rate-limit contract

Preserve these v2 endpoints unless an approved versioned API change is made:

- `GET /api/v2/properties/search`: 60/min/IP; supports `occupants=N` and returns `estimated_true_monthly_cost`.
- `POST /api/v2/contracts/ai-scan`: SSE; 1/day Guest and 3/day authenticated Tenant; PII masking precedes LLM access.
- `POST /api/v2/leases/{id}/members`: 10/min/User.
- `GET /api/v2/leases/{id}/export-ct01`: 10/min/User.
- `POST /api/v2/leases/{id}/check-out`: 10/min/User.
- `POST /api/v2/invoices/ocr-meter`: 120/min/User.
- `POST /api/v2/webhooks/vietqr`: verified provider callback only; idempotent.

Use Redis sliding-window keys `rl:v2:{apiGroup}:{subjectId}`. Rate-limit responses MUST be stable, observable, and MUST NOT leak whether a protected resource exists.

## Verification and definition of done

Every feature change MUST include tests at the cheapest correct level: pure domain unit tests, application tests, repository/integration tests with PostgreSQL/PostGIS and Redis Testcontainers, security tests, and end-to-end/UAT coverage when behavior crosses modules.

Every completed change MUST pass:

1. Compilation, formatting, static analysis, unit and integration tests.
2. `ArchitectureModularityTest` (ArchUnit) proving domain purity, allowed layer directions, and no cross-module infrastructure access.
3. Security tests for role plus resource ownership, including a literal 403 for UAT-05.
4. UAT-01 through UAT-06:
   - UAT-01: `4,200` VND/kWh is rejected against an active lease committed at `3,500` VND/kWh.
   - UAT-02: two successful co-tenant eKYCs change `occupant_count` from 1 to 3, water becomes VND 300,000 at VND 100,000/person, and `Mau_CT01_P201.docx` contains all three members.
   - UAT-03: `maxDial=100000`, old `99850`, new `120` yields rollover consumption `270 kWh`.
   - UAT-04: an approved VND 250,000 tenant advance reduces VND 4,500,000 to VND 4,250,000 exactly once.
   - UAT-05: Property Manager bank-account mutation returns 403; Owner mutation requires step-up OTP.
   - UAT-06: VND 4,000,000 deposit minus VND 350,000 final utilities and no tenant-fault damage yields a VND 3,650,000 refund.
5. Regression checks for retry/idempotency, concurrency losers, offline replay, R2 ownership, 5-minute private GET expiry, PII redaction, and the 90-day CCCD cleanup.

NEVER declare a feature done with skipped tests, disabled security, placeholder calculations, TODO concurrency controls, mocked ownership, or an upload path that violates the Zero-Bandwidth Media Rule.
