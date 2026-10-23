# SafeRent VN v2.0 Incremental Implementation Plan

Tài liệu này là hàng đợi thực thi duy nhất cho phương pháp Incremental Spec-Driven Development. Danh sách gồm 8 Phase và 22 Atomic Tasks, giữ nguyên thứ tự phụ thuộc từ nền tảng dữ liệu đến frontend.

## QUY TẮC VẬN HÀNH DÀNH CHO CODEX (EXECUTION PROTOCOL)

1. Mỗi lần được gọi để triển khai, Codex MUST chọn **DUY NHẤT 1 Task** có trạng thái `[ ] TODO`: Task đầu tiên từ trên xuống chưa hoàn thành và có toàn bộ dependency đã `[x] DONE`. NEVER làm đồng thời hai Task, chuẩn bị trước Task sau, hoặc sửa code ngoài phạm vi Task đã chọn.
2. Codex MUST chỉ đọc:
   - mục của Task đang chọn trong file này;
   - đúng một file `specs/PHASE-X-...md` được Task trỏ tới;
   - `AGENTS.md` gốc và các `AGENTS.md` nằm trên đường dẫn thư mục trực tiếp chứa file sẽ sửa.
   Codex MUST NOT đọc spec của Phase khác và MUST NOT đọc lại hai DOCX nguồn trong lúc triển khai Task. Nếu spec thiếu dữ kiện bắt buộc, STOP, giữ `[ ] TODO`, báo rõ khoảng trống và chờ quyết định của người dùng.
3. Trước khi code, Codex MUST nêu chính xác `TASK-X.Y`, phạm vi file dự kiến sửa và tiêu chí pass. Codex MUST bảo toàn mọi thay đổi sẵn có không liên quan.
4. Codex MUST triển khai đầy đủ code, migration/config và automated tests thuộc đúng Task. NEVER dùng placeholder, TODO/FIXME, test bị disable, mock để né invariant, hoặc hạ chuẩn bảo mật/kiến trúc cho test xanh.
5. Sau khi code xong, Codex MUST chạy lệnh kiểm thử ghi tại Task:
   - Backend: `mvn test` từ `saferent-vn-backend/`.
   - Frontend: `npm run build` từ `saferent-vn-frontend/`; nếu Task thêm test suite thì MUST chạy cả test suite được khai báo trong `package.json` trước build.
   - Nếu Task yêu cầu coverage, ArchUnit, Testcontainers hoặc UAT, các kiểm tra đó MUST nằm trong lệnh trên hoặc MUST được chạy bổ sung và báo kết quả.
6. Chỉ khi **toàn bộ** compile, lint/static checks liên quan, unit/integration/UAT tests và build pass 100%, Codex mới được đổi đúng một dòng trạng thái của Task từ `[ ] TODO` thành `[x] DONE`.
7. Nếu bất kỳ kiểm thử nào fail, môi trường thiếu dependency/service, hoặc acceptance criterion chưa đạt, Codex MUST giữ `[ ] TODO`, không đổi Task khác, không nới lỏng test, và dừng với báo cáo lỗi có thể tái hiện.
8. Sau khi đánh dấu đúng một Task `[x] DONE`, Codex MUST dừng và chờ lệnh tiếp theo. NEVER tự động bắt đầu Task kế tiếp trong cùng phiên.
9. Việc sửa trạng thái chỉ được phép thay `[ ] TODO` bằng `[x] DONE`. NEVER đổi nội dung/ID/thứ tự/dependency/spec link của Task trong một phiên triển khai, trừ khi người dùng yêu cầu cập nhật kế hoạch.
10. Khi người dùng yêu cầu “tiếp tục”, Codex MUST quét trạng thái từ đầu file này và chọn Task TODO hợp lệ đầu tiên; NEVER dựa vào trí nhớ của phiên trước.

## Trạng thái và Definition of Done

- `[ ] TODO`: chưa hoàn thành hoặc chưa pass toàn bộ kiểm thử.
- `[x] DONE`: code và automated tests của Task đã pass 100%; không còn placeholder hay công việc bắt buộc trong scope.
- Không sử dụng trạng thái trung gian trong checklist. Lỗi/blocker được báo trong phản hồi, không sửa nhãn Task.

Mỗi Task chỉ được DONE khi:

- Tuân thủ toàn bộ `AGENTS.md` áp dụng.
- Khớp contract, đoạn nguồn nguyên văn và acceptance criteria trong Phase spec.
- Không phá build/test của các Task đã DONE.
- Không phát sinh đường upload media xuyên Backend, rò rỉ PII, sai RBAC/resource ownership, arithmetic bằng floating point, hoặc race condition đã bị SADD cấm.
- Lệnh verification của Task kết thúc exit code 0 và không có test bị skip ngoài danh sách skip được người dùng phê duyệt rõ ràng.

## Phase 1 Database and Architecture Foundation

Spec duy nhất được phép đọc: [`specs/PHASE-1-database-architecture-foundation.md`](specs/PHASE-1-database-architecture-foundation.md)

- [ ] TODO **TASK-1.1 — Bootstrap Maven and runtime configuration**
  - Dependencies: none.
  - Read AGENTS: `AGENTS.md`, `saferent-vn-backend/AGENTS.md`.
  - Deliver: `pom.xml`, application entry point, `application.yml`, profiles/test bootstrap; Java 21, Spring Boot 3.3, Hibernate Spatial, AWS S3 v2, Apache POI, ShedLock, ArchUnit, Flyway, PostgreSQL/PostGIS, Redis and test dependencies; virtual threads, OSIV false, Hikari max 20, Hibernate batch 30.
  - Acceptance: application context/test skeleton compiles without weakening production settings; no secrets committed.
  - Verify: `cd saferent-vn-backend && mvn test`.

- [ ] TODO **TASK-1.2 — Create Flyway V1 and V2 database migrations**
  - Dependencies: TASK-1.1.
  - Read AGENTS: `AGENTS.md`, `saferent-vn-backend/AGENTS.md`.
  - Deliver: `V1__init_saferent_v2_postgis_schema.sql`, `V2__add_concurrency_locks_and_shedlock.sql`, and migration integration tests against PostgreSQL 16 + PostGIS.
  - Acceptance: exact SRS tables/columns/checks; `version` columns; `uq_active_lease_per_room`; `uq_invoice_lease_month`; `deducted_in_invoice_id`; ShedLock table; FK/index behavior; repeatable clean migration.
  - Verify: `cd saferent-vn-backend && mvn test`.

- [ ] TODO **TASK-1.3 — Enforce modular Clean Architecture with ArchUnit**
  - Dependencies: TASK-1.1, TASK-1.2.
  - Read AGENTS: `AGENTS.md`, `saferent-vn-backend/AGENTS.md`.
  - Deliver: package skeleton for `common` and five modules; `ArchitectureModularityTest.java`.
  - Acceptance: four layers per module; domain has zero Spring/JPA/Hibernate; no cross-module infrastructure/presentation imports; controllers cannot expose persistence entities; test fails on deliberate fixture violations.
  - Verify: `cd saferent-vn-backend && mvn test`.

## Phase 2 Shared Kernel and Common Infrastructure

Spec duy nhất được phép đọc: [`specs/PHASE-2-common-infrastructure.md`](specs/PHASE-2-common-infrastructure.md)

- [ ] TODO **TASK-2.1 — AES-GCM CCCD converter and SHA-256 utility**
  - Dependencies: Phase 1 DONE.
  - Read AGENTS: `AGENTS.md`, `saferent-vn-backend/AGENTS.md`.
  - Deliver: `AesGcmCccdConverter`, key-provider boundary, versioned ciphertext envelope, `Sha256DigestUtil`, unit/integration tests.
  - Acceptance: AES-256-GCM, fresh 12-byte IV, 128-bit tag, randomized ciphertext, tamper detection, canonical lowercase 64-hex SHA-256, no plaintext logging.
  - Verify: `cd saferent-vn-backend && mvn test`.

- [ ] TODO **TASK-2.2 — Cloudflare R2 Presigned URL service**
  - Dependencies: TASK-2.1.
  - Read AGENTS: `AGENTS.md`, `saferent-vn-backend/AGENTS.md`.
  - Deliver: R2 configuration, category/bucket policy, `CloudflareR2StorageService`, PUT/GET presign DTOs and metadata verification tests.
  - Acceptance: public/private buckets; PUT exactly 10 minutes; private GET exactly 5 minutes; server-generated key; allowlisted MIME/size <=15 MB; signed SHA-256 metadata; no binary controller.
  - Verify: `cd saferent-vn-backend && mvn test`.

- [ ] TODO **TASK-2.3 — Async Resend email pipeline**
  - Dependencies: TASK-2.2.
  - Read AGENTS: `AGENTS.md`, `saferent-vn-backend/AGENTS.md`.
  - Deliver: virtual-thread executor, Resend client port/adapter, transactional/outbox event shape, `TransactionalEmailEventListener`, retry and batch tests.
  - Acceptance: `@Async("virtualThreadExecutor")`; `AFTER_COMMIT`; no send on rollback; deterministic Idempotency-Key; transient retry only; batch chunks <=100.
  - Verify: `cd saferent-vn-backend && mvn test`.

- [ ] TODO **TASK-2.4 — Redis sliding-window rate limiter**
  - Dependencies: TASK-2.1.
  - Read AGENTS: `AGENTS.md`, `saferent-vn-backend/AGENTS.md`.
  - Deliver: `RedisSlidingWindowRateLimitFilter`, atomic Lua/Bucket4j implementation, endpoint policy map, stable 429 response and tests.
  - Acceptance: key `rl:v2:{apiGroup}:{subjectId}` and every quota from SRS/SADD; no protected-resource existence leak; concurrency-safe boundary tests.
  - Verify: `cd saferent-vn-backend && mvn test`.

## Phase 3 Identity eKYC and RBAC

Spec duy nhất được phép đọc: [`specs/PHASE-3-identity-ekyc-rbac.md`](specs/PHASE-3-identity-ekyc-rbac.md)

- [ ] TODO **TASK-3.1 — JWT authentication and refresh rotation**
  - Dependencies: Phase 1 DONE, TASK-2.1, TASK-2.4.
  - Read AGENTS: `AGENTS.md`, `saferent-vn-backend/AGENTS.md`, `saferent-vn-backend/src/main/java/vn/saferent/modules/identity/AGENTS.md`.
  - Deliver: identity domain/application/adapters/controllers, JWT access token, Redis refresh token family rotation/reuse detection, six roles and tests.
  - Acceptance: 15-minute access token, 7-day rotating refresh token, exact role enum, no stale resource ownership embedded in JWT, redacted logs.
  - Verify: `cd saferent-vn-backend && mvn test`.

- [ ] TODO **TASK-3.2 — Step-up OTP resource ownership and VietQR bank API**
  - Dependencies: TASK-3.1.
  - Read AGENTS: `AGENTS.md`, `saferent-vn-backend/AGENTS.md`, identity module `AGENTS.md`.
  - Deliver: `@RequireStepUpOtp`, aspect, `ResourceOwnershipEvaluator`, Owner bank configuration use case/controller, security/integration tests.
  - Acceptance: Owner must own property and pass bound one-time OTP; manager of same property still receives 403; audit is redacted; UAT-05 passes exactly.
  - Verify: `cd saferent-vn-backend && mvn test`.

## Phase 4 Discovery PostGIS and True Cost

Spec duy nhất được phép đọc: [`specs/PHASE-4-discovery-postgis-true-cost.md`](specs/PHASE-4-discovery-postgis-true-cost.md)

- [ ] TODO **TASK-4.1 — Pure True Cost domain policy**
  - Dependencies: Phase 1 DONE.
  - Read AGENTS: `AGENTS.md`, backend `AGENTS.md`, discovery module `AGENTS.md`.
  - Deliver: `TrueCostPolicy` and pure unit tests.
  - Acceptance: exact formula by occupants N; AEF 1.00/1.20/1.40; `PER_PERSON` vs `PER_M3`; `BigDecimal`; 100% branch coverage for policy; zero framework imports.
  - Verify: `cd saferent-vn-backend && mvn test`.

- [ ] TODO **TASK-4.2 — PostGIS search Geohash6 cache and PCCC badge**
  - Dependencies: TASK-4.1, TASK-2.4.
  - Read AGENTS: `AGENTS.md`, backend `AGENTS.md`, discovery module `AGENTS.md`.
  - Deliver: `ST_DWithin` repository, search use case/API, Redis Geohash6/detail cache, PCCC 3-level policy and six-month expiry job with ShedLock.
  - Acceptance: GIST-backed geography query; exact cache keys/TTLs/evictions; legal badge wording; three-tenant or Inspector Level 2; expiry race tests.
  - Verify: `cd saferent-vn-backend && mvn test`.

## Phase 5 SmartLease Co-living CT01 and Handover

Spec duy nhất được phép đọc: [`specs/PHASE-5-smartlease-coliving-ct01-handover.md`](specs/PHASE-5-smartlease-coliving-ct01-handover.md)

- [ ] TODO **TASK-5.1 — PII-masked AI contract scan over SSE**
  - Dependencies: TASK-2.2, TASK-2.4, TASK-3.1.
  - Read AGENTS: `AGENTS.md`, backend `AGENTS.md`, smartlease module `AGENTS.md`.
  - Deliver: `PiiMasker`, LLM port/adapter, `ScanContractAiUseCase`, strict JSON validation, SSE controller and tests.
  - Acceptance: mask before outbound call; exact risk schema; no transaction held during LLM/SSE; typed progress/final/error events; rate limits; PII absent from logs/events.
  - Verify: `cd saferent-vn-backend && mvn test`.

- [ ] TODO **TASK-5.2 — Co-living membership and CT01 export**
  - Dependencies: TASK-3.1, TASK-2.1, TASK-2.2.
  - Read AGENTS: `AGENTS.md`, backend `AGENTS.md`, identity and smartlease module `AGENTS.md`.
  - Deliver: `Lease`/`LeaseMember`, locked add-member use case, repositories, `ApachePoiCt01Exporter`, authorization/audit and integration tests.
  - Acceptance: parent Lease `PESSIMISTIC_WRITE`; capacity and active count; CT01 all active members; authorized decrypt only; controlled generation <200 ms; UAT-02 passes.
  - Verify: `cd saferent-vn-backend && mvn test`.

- [ ] TODO **TASK-5.3 — Digital check-in checkout and deposit settlement**
  - Dependencies: TASK-5.2, TASK-2.2, TASK-2.3.
  - Read AGENTS: `AGENTS.md`, backend `AGENTS.md`, smartlease module `AGENTS.md`.
  - Deliver: `HandoverRecord`, immutable evidence manifest, 48-hour auto-lock, notice/check-out use cases, `DepositSettlementPolicy`, APIs/jobs/tests.
  - Acceptance: ShedLock + `@Version`; pre-deadline response wins; natural wear excluded; exact refund formula; SLA <=7 days; UAT-06 passes.
  - Verify: `cd saferent-vn-backend && mvn test`.

## Phase 6 Operations OCR Billing Tickets and Payments

Spec duy nhất được phép đọc: [`specs/PHASE-6-operations-ocr-billing-payments.md`](specs/PHASE-6-operations-ocr-billing-payments.md)

- [ ] TODO **TASK-6.1 — Meter rollover and EVN tariff domain policies**
  - Dependencies: TASK-4.1, TASK-5.2.
  - Read AGENTS: `AGENTS.md`, backend `AGENTS.md`, operations module `AGENTS.md`.
  - Deliver: `MeterRolloverPolicy`, `EvnTieredTariffByCapitaPolicy`, tariff version value types and pure unit tests.
  - Acceptance: serial validation contract; >=90% rollover threshold; replacement path separation; 1/4 household quota per occupant; dated tariff; UAT-03 passes.
  - Verify: `cd saferent-vn-backend && mvn test`.

- [ ] TODO **TASK-6.2 — Maintenance ticket and Owner deduction approval**
  - Dependencies: TASK-3.2, TASK-5.2.
  - Read AGENTS: `AGENTS.md`, backend `AGENTS.md`, operations module `AGENTS.md`.
  - Deliver: maintenance ticket domain/use cases/repository/API, R2 evidence references and authorization tests.
  - Acceptance: exact categories/fault attribution; positive tenant advance; Owner-only approval; Manager 403; append-only evidence/audit; `deducted_in_invoice_id` lifecycle guarded.
  - Verify: `cd saferent-vn-backend && mvn test`.

- [ ] TODO **TASK-6.3 — Atomic monthly billing and ticket deduction**
  - Dependencies: TASK-6.1, TASK-6.2.
  - Read AGENTS: `AGENTS.md`, backend `AGENTS.md`, operations module `AGENTS.md`.
  - Deliver: `MonthlyBillingConcurrencyService`, invoice domain/repositories, locked deduction, VietQR amount, parallel integration tests.
  - Acceptance: fixed rate cannot exceed lease; unique lease/month; `FOR UPDATE` approved-undeducted tickets; exact debt/deduction/credit arithmetic; one winner under concurrency; UAT-01 and UAT-04 pass.
  - Verify: `cd saferent-vn-backend && mvn test`.

- [ ] TODO **TASK-6.4 — Idempotent VietQR reconciliation**
  - Dependencies: TASK-6.3, TASK-2.4.
  - Read AGENTS: `AGENTS.md`, backend `AGENTS.md`, operations module `AGENTS.md`.
  - Deliver: signature verifier port, persistent bank transaction idempotency, Redis SETNX fast gate, locked reconciliation use case/webhook/tests.
  - Acceptance: `webhook:vietqr:<bankTxId>` NX EX 86400; invoice + lease locks; cumulative `paid_amount`; exact PARTIALLY_PAID/debt/PAID/excess credit behavior; replay and concurrent payer tests.
  - Verify: `cd saferent-vn-backend && mvn test`.

## Phase 7 Verified Reviews and Two-Way Moderation

Spec duy nhất được phép đọc: [`specs/PHASE-7-community-verified-reviews.md`](specs/PHASE-7-community-verified-reviews.md)

- [ ] TODO **TASK-7.1 — Verified reviews and 48-hour two-way moderation**
  - Dependencies: TASK-5.2, TASK-2.2, TASK-2.3.
  - Read AGENTS: `AGENTS.md`, backend `AGENTS.md`, smartlease and community module `AGENTS.md`.
  - Deliver: review domain/state/version, membership eligibility port, evidence validation, Owner response, ShedLock publication job, APIs and tests.
  - Acceptance: active Primary and Co-Tenant eligible; 1–2 stars require evidence; `PENDING_LANDLORD_RESPONSE`; 48-hour boundary; Owner response committed before deadline wins; idempotent publish/notify/cache eviction.
  - Verify: `cd saferent-vn-backend && mvn test`.

## Phase 8 React 18 PWA

Spec duy nhất được phép đọc: [`specs/PHASE-8-frontend-react-pwa.md`](specs/PHASE-8-frontend-react-pwa.md)

- [ ] TODO **TASK-8.1 — Vite PWA foundation and direct R2 upload**
  - Dependencies: Backend TASK-2.2, TASK-3.1.
  - Read AGENTS: `AGENTS.md`, `saferent-vn-frontend/AGENTS.md`.
  - Deliver: React/TypeScript/Vite PWA, Tailwind/Shadcn, TanStack Query/Zustand boundaries, direct R2 upload client, SHA-256 utility and tests.
  - Acceptance: strict TS; >=48px actions; >=16–18px critical text; exact hash of final bytes; PUT direct to R2; no FormData/base64/backend media proxy; private URL not persisted.
  - Verify: `cd saferent-vn-frontend && npm run build` plus configured frontend tests.

- [ ] TODO **TASK-8.2 — Meter camera preprocessing and offline queue**
  - Dependencies: TASK-8.1, Backend TASK-6.1.
  - Read AGENTS: `AGENTS.md`, frontend `AGENTS.md`.
  - Deliver: camera bounding box, orientation-aware Canvas crop, grayscale WebP <50 KB, OCR review, Dexie queue, Workbox background sync and tests.
  - Acceptance: stable idempotency key; processed blob/hash saved before offline success; fresh presign on replay; ordered upload/register; bounded retry; NEEDS_REVIEW for domain/auth conflict; blob cleanup only after confirmation.
  - Verify: `cd saferent-vn-frontend && npm run build` plus configured frontend tests.

- [ ] TODO **TASK-8.3 — Map AI SSE and check-in checkout comparison UI**
  - Dependencies: TASK-8.1, Backend Phase 4, TASK-5.1, TASK-5.3, TASK-7.1.
  - Read AGENTS: `AGENTS.md`, frontend `AGENTS.md`.
  - Deliver: Mapbox/MapLibre search and True Cost cards; SSE contract scan; `react-compare-image` handover comparison; settlement/review states and UI tests.
  - Acceptance: accessible list alternative; exact N/AEF assumptions; typed SSE lifecycle; no PII in URL/log; natural-wear vs tenant-fault selection; 48-hour/7-day deadlines from server; frontend outcomes for UAT-01 through UAT-06.
  - Verify: `cd saferent-vn-frontend && npm run build` plus configured frontend tests.

## Completion gate

Project implementation is complete only when all 22 Tasks are `[x] DONE`, backend `mvn test` passes including ArchUnit/Testcontainers/UAT-01 through UAT-06, frontend tests plus `npm run build` pass, and no rule in any applicable `AGENTS.md` is waived.
