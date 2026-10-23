# Smart Lease Module Rules

## Scope and responsibility

This module owns lease lifecycle, contract AI scan orchestration, co-living membership, CT01 export, immutable check-in/check-out evidence, notice validation, and deposit settlement. It MUST NOT own user authentication, property discovery, meter billing, or review moderation.

Canonical domain types include `Lease`, `LeaseMember`, `HandoverRecord`, `DepositSettlementPolicy`, and explicit lifecycle/status value objects. Keep policies pure; place LLM, Apache POI, R2 metadata, and persistence adapters in `infrastructure`.

## Contract AI scan and SSE

- `POST /api/v2/contracts/ai-scan` MUST accept a verified R2 object reference or extracted text reference, never a binary file through Spring Boot.
- Contract files MUST upload directly to `saferent-private-vault` with a Presigned PUT. Verify object ownership and SHA-256 before scan.
- `PiiMaskingService` MUST mask CCCD, phone numbers, full names, permanent addresses, bank identifiers, signatures, and other direct identifiers before any text reaches an LLM.
- NEVER include the mask dictionary, raw OCR text, Presigned URL, or plaintext PII in the LLM request, log, trace, SSE event, cache, or exception.
- The LLM output is advisory. Validate it against a strict JSON schema with `safety_score`, `summary`, and `risk_clauses[]` containing `severity`, `category`, `clause_excerpt`, `legal_basis`, `explanation`, and `suggested_revision`.
- Risk rules MUST preserve the SRS thresholds: RED when fixed electricity exceeds VND 4,000/kWh or service pricing is unclear; RED for tenant notice >45 days while owner notice <15 days or missing refund deadline; RED when natural-wear repair is shifted entirely to tenant; YELLOW for unreasonable temporary-residence/housemate restrictions.
- Stream progress and validated findings through `EventSource`/SSE. Emit stable typed events, heartbeat/timeouts, a terminal success/error event, and support client disconnect cancellation. NEVER hold a database connection during OCR/LLM streaming.
- Enforce 1 scan/day for Guest and 3/day for Tenant with the global Redis rate limiter.

## Lease and co-living invariants

Preserve `leases` fields: `room_id`, `primary_tenant_id`, `landlord_id`, `start_date`, `end_date`, `occupant_count`, `notice_period_days`, `termination_requested_at`, `monthly_rent`, `deposit_amount`, `electricity_rate_committed`, `credit_balance`, `contract_pdf_sha256`, `status`, and `version`.

Preserve `lease_members` fields: `lease_id`, `user_id`, `member_role`, `joined_date`, `left_date`, `residence_reg_status`, and `is_active`.

- `member_role` MUST be `PRIMARY` or `CO_TENANT`; one active Primary must correspond to `leases.primary_tenant_id`.
- Residence status MUST be `PENDING_CT01`, `CT01_EXPORTED`, or `VNEID_CONFIRMED`.
- A member MUST NOT be active outside the lease/member date interval. Leaving a lease MUST set `left_date` and `is_active = FALSE`; NEVER erase residence history.
- `occupant_count` MUST equal the count of active members and MUST NOT exceed `rooms.max_occupants`.
- `POST /api/v2/leases/{id}/members` is limited to 10/min/User and MUST require the authorized Primary Tenant/Owner flow plus successful eKYC for activation.

Co-tenant mutation MUST serialize on the parent lease:

```java
@Transactional
public LeaseMember addCoTenant(AddCoTenantCommand command) {
    Lease lease = leaseRepository.findByIdForUpdate(command.leaseId())
            .orElseThrow(ResourceNotFoundException::new); // SELECT ... FOR UPDATE
    int active = memberRepository.countActive(command.leaseId());
    lease.ensureCapacity(active + 1, roomCapacityPort.maxOccupants(lease.roomId()));
    LeaseMember member = memberRepository.save(LeaseMember.coTenant(/* ... */));
    lease.synchronizeOccupantCount(memberRepository.countActive(command.leaseId()));
    return member;
}
```

NEVER perform an unlocked increment. Duplicate invite/eKYC callbacks MUST be idempotent.

## CT01 export

- `GET /api/v2/leases/{id}/export-ct01` is limited to 10/min/User.
- `ApachePoiCt01Exporter` MUST fill `src/main/resources/templates/compliance/Mau_CT01_BoCongAn_Template.docx` corresponding to Mẫu CT01 under Thông tư 66/2023/TT-BCA.
- Export MUST include the landlord/householder, rental address, and every active `PRIMARY` and `CO_TENANT` member with authorized identity fields required by the form.
- Decrypt `users.cccd_encrypted` only inside the authorized export use case. `PROPERTY_MANAGER` requires explicit Owner grant; Owner must own the property. Every export MUST be audited.
- Target generation time is under 200 ms excluding client download. Reuse an immutable parsed template safely; NEVER cache decrypted identities or generated documents containing PII.
- Generated Word/PDF MUST go to `saferent-private-vault` or stream only through the approved document-delivery mechanism. The normal media rule forbids proxying persistent binaries through arbitrary REST controllers; prefer direct R2 upload/download with a 5-minute private GET.
- On successful export, transition applicable members from `PENDING_CT01` to `CT01_EXPORTED`; VNeID confirmation transitions to `VNEID_CONFIRMED`. Make export retry idempotent.

## Digital check-in

- Check-in may capture room/device condition, electricity/water baseline, GPS, server timestamp, lease ID, photos, and video.
- The browser MUST hash and directly upload media to `saferent-private-vault`. Persist object keys plus SHA-256; verify via `HeadObject`.
- Apply a server-side evidence manifest/hash so ordering, metadata, and media hashes cannot be changed silently.
- The dispute window is 48 hours from the persisted server timestamp. After the deadline, if no valid landlord dispute exists, transition the record to `LOCKED` and use its meter values as the opening baseline.
- The auto-lock job MUST use `@SchedulerLock` plus `handover_records.version`. A landlord dispute committed before the deadline wins over a concurrent job.
- A locked record and its evidence are immutable. Corrections require a linked append-only amendment with reason, actor, time, and hashes; NEVER overwrite original evidence.

## Notice and digital check-out

- Termination notice MUST compare `termination_requested_at` with `notice_period_days` (default 30) and communicate whether the full-deposit condition is met.
- Check-out UI/API MUST pair the locked check-in evidence with current check-out evidence.
- Every claimed damage MUST be classified as `NATURAL_WEAR` (Owner bears cost) or `TENANT_FAULT` (potential deduction), and a tenant-fault deduction requires evidence and both-party confirmation or an adjudicated decision.
- `POST /api/v2/leases/{id}/check-out` is limited to 10/min/User and MUST be idempotent.

`DepositSettlementPolicy` MUST use this exact business equation with `BigDecimal`:

```text
Refund = Deposit
       - ProratedFinalRent
       - FinalElectricityAndWater
       - ConfirmedTenantFaultDamage
```

- Natural wear MUST NOT be deducted.
- Every component and evidence reference MUST be written to `handover_records.settlement_summary JSONB` as a versioned, auditable settlement snapshot.
- Refund MUST NOT become negative; any amount owed above the deposit MUST be represented explicitly as debt, not as a negative refund.
- Settlement SLA MUST be no more than 7 days; the owner flow SHOULD support immediate refund and MUST display the Primary Tenant's VietQR for direct refund. SafeRent MUST NOT custody the deposit.
- Contract may transition through `PENDING_CHECKOUT` to `TERMINATED` only after the authorized settlement flow. `TERMINATED` starts the 90-day CCCD-image retention clock.

Required UAT-06 exact result:

```text
Deposit 4,000,000
- final utilities 350,000
- prorated rent 0
- confirmed tenant-fault damage 0
= refund 3,650,000 VND
```

## Lease document integrity

- Electronic signing MUST bind eKYC, verified phone OTP, IP, server timestamp, signers, and the final PDF SHA-256.
- Set `contract_pdf_sha256` only after the exact signed PDF exists and has been verified in private R2.
- `uq_active_lease_per_room` plus `@Version` MUST protect activation. A competing activation returns 409.
- Publish `LeaseSignedEvent` in the transaction; Resend delivery MUST occur with `AFTER_COMMIT`, virtual-thread async execution, retry, and deterministic idempotency.

## Required tests

- MUST prove PII is masked before the mocked LLM adapter and absent from SSE/logs.
- MUST test strict LLM schema rejection, disconnect, retry boundaries, and rate limits without holding a database transaction.
- MUST run a parallel co-tenant test showing two simultaneous eKYCs produce `occupant_count = 3`, not 2, and never exceed `max_occupants`.
- MUST implement UAT-02: VND 100,000/person × 3 = VND 300,000 and `Mau_CT01_P201.docx` contains all three students.
- MUST test CT01 Owner/Manager grant rules, under-200-ms generation benchmark in the controlled environment, and no plaintext identity cache/log.
- MUST test 48-hour boundary races, immutable locked evidence, notice compliance, natural-wear exclusion, settlement rounding, SLA deadline, and UAT-06.
- MUST test double-booking contention and HTTP 409 mapping against PostgreSQL.
