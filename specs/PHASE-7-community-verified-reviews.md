# PHASE 7 Community Verified Reviews

## Reading boundary

This file is the complete implementation spec for TASK-7.1. Read only this file plus root/backend and smartlease/community `AGENTS.md` files named in the plan.

## Phase outcome

Only verified active Primary/Co-Tenants can review. Negative reviews require evidence and enter a race-safe 48-hour two-way response workflow before publication.

## Atomic task contract

### TASK-7.1 Verified reviews and 48-hour moderation

Required SADD names:

- `VerifiedReview`
- `ReviewModerationPolicy`
- `VerifiedReviewService`
- `TwoWayDisputeService`
- `ReviewAutoPublishJob`
- `CreateVerifiedReviewUseCase`
- `RespondReviewUseCase`
- `PublishReviewJob`
- `JpaVerifiedReviewRepository`
- `VerifiedReviewController`

Implementation MUST:

- query SmartLease through an application port for active, eKYC-verified membership at the reviewed property;
- accept both `PRIMARY` and `CO_TENANT`;
- require evidence for 1–2 stars or a violation allegation;
- start state `PENDING_LANDLORD_RESPONSE` with server deadline +48 hours;
- permit the owning `LANDLORD_OWNER` to submit the official response;
- publish tenant review and landlord response together, or publish with no-response state after deadline;
- use ShedLock plus `@Version`, make pre-deadline Owner response win, and make publish/notify idempotent;
- evict `prop:v2:detail:{propertyId}` and related discovery cache after publication.

## Verbatim source excerpts

### SRS review rule

> Chỉ những tài khoản là `Primary Tenant` hoặc `Co-Tenant` đã xác thực tại tòa nhà mới có quyền viết đánh giá. Đối với các đánh giá tiêu cực (1–2 sao hoặc tố cáo vi phạm), hệ thống bắt buộc đính kèm ảnh bằng chứng, giữ ở trạng thái chờ đối chất 48 giờ (`PENDING_LANDLORD_RESPONSE`) để Chủ nhà phản hồi kèm bằng chứng, sau đó công khai song song cả hai chiều thông tin.

### SRS RBAC row

| Mã CN | Chức năng | Guest | Tenant & Co-Tenant | Property Manager | Landlord Owner | Admin |
|---|---|---|---|---|---|---|
| MOD-08 | Viết Review xác thực & Phản hồi Tranh chấp 2 chiều | Chỉ xem | Viết Review | - | Phản hồi chính thức | Kiểm duyệt |

### SADD module services and behavior

> `VerifiedReviewService` - `TwoWayDisputeService` - `ReviewAutoPublishJob`

> Kiểm tra tư cách Verified Tenant (cho phép cả Primary Tenant và Co-Tenant).

> Bắt buộc đính kèm ảnh bằng chứng với đánh giá 1–2 sao, giữ ở trạng thái chờ đối chất 48 giờ (`PENDING_LANDLORD_RESPONSE`) trước khi công khai song song đánh giá và phản hồi của chủ nhà.

### SADD package names

```text
community/
├── domain/          # VerifiedReview, ReviewModerationPolicy
├── application/     # CreateVerifiedReviewUseCase, RespondReviewUseCase, PublishReviewJob
├── infrastructure/  # JpaVerifiedReviewRepository
└── presentation/    # VerifiedReviewController
```

### SADD concurrency rule

> ShedLock (`@SchedulerLock`) + Optimistic Locking (`@Version`): Dùng ShedLock trên Redis/PostgreSQL để đảm bảo Cron Job chỉ chạy trên duy nhất 1 node. Bảng `handover_records` và `verified_reviews` đều có cột `@Version`. Nếu Chủ nhà vừa bấm phản hồi ở giây 47h59m59s, version tăng lên 1 -> lệnh cập nhật của Cron Job sẽ phát hiện lệch version và nhường quyền ưu tiên cho phản hồi của Chủ nhà.

### SADD Admin rule

> ADMIN: Phân xử tranh chấp Check-out/Review, giám sát hệ thống (không có quyền xóa các Review tiêu cực đã có bằng chứng hợp lệ theo FR-4.1).

### SADD cache eviction

> Xóa cache ngay lập tức khi Review mới được công khai sau 48h.

## Source gap and derived persistence contract

The supplied SRS/SADD do **not** provide a complete `CREATE TABLE verified_reviews` statement or exact API endpoint. The following is normative implementation contract, not a verbatim quotation.

- Add a dedicated Flyway migration in TASK-7.1. The table MUST include UUID primary key; property, lease/member and reviewer FKs; rating; review text; evidence object-key/hash JSON or normalized child rows; status including `PENDING_LANDLORD_RESPONSE`; submitted server time; response deadline; Owner response/evidence/time; published time; moderation audit fields; `version BIGINT NOT NULL DEFAULT 0`.
- Add CHECK rating between 1 and 5, non-null server deadline for pending negative reviews, indexes for property/publication and pending deadline, and an explicit uniqueness policy approved by the application behavior.
- Exact column/endpoint names MUST be documented in migration/API tests and then treated as stable. Do not encode the entire workflow in unvalidated JSON.
- Evidence uploads directly to R2. Public-safe evidence MAY be public; sensitive evidence remains private and is accessed through authorized 5-minute GET. Never expose private object keys/URLs publicly.
- Snapshot eligibility reference at submission for audit, but re-check authoritative property/Owner ownership for response.
- Sanitize displayed content against XSS while preserving an access-controlled immutable audit source.

## Formula JSON and UAT inventory

- Phase 7 owns no mathematical formula or LLM JSON schema.
- The supplied SRS defines no numbered Gherkin UAT for reviews. The 1–2-star evidence, 48-hour state and second-boundary race are mandatory acceptance scenarios even without a UAT ID.

## Acceptance scenarios

```gherkin
Scenario: Active Co-Tenant creates a negative verified review
  Given a TENANT is an eKYC-verified active CO_TENANT of a lease at property A
  When the tenant submits a 2-star review with valid evidence
  Then the review is stored as PENDING_LANDLORD_RESPONSE
  And its response deadline is server submission time plus 48 hours
  And it is not public before the response/deadline workflow completes
```

This scenario is derived for testing and is not a verbatim numbered SRS UAT.

```gherkin
Scenario: Owner response wins the deadline race
  Given a negative review is pending and the Owner response commits at 47:59:59
  When the auto-publish job concurrently attempts a stale update at 48:00:00
  Then optimistic locking rejects the stale job update
  And the published result contains the tenant review and Owner response
  And notification/publication occurs exactly once
```

## Verification gate

- Eligibility tests: active Primary, active Co-Tenant accepted; Guest, Manager, unrelated/inactive/former Tenant rejected.
- Evidence tests: 1–2 stars/allegation required; privacy classification/hash/ownership; public response never exposes private data.
- Time tests with injected clock: before, exactly at and after 48 hours.
- Real concurrency test: response vs scheduler; pre-deadline committed response wins via version.
- Idempotent publication/email and discovery-cache eviction tests.
- Admin moderation audit and “cannot delete substantiated negative review merely for negativity” test.
- `mvn test` MUST pass.
