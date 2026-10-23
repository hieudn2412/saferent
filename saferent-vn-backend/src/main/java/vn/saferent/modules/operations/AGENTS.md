# Operations Module Rules

## Scope and responsibility

This module owns meter OCR validation, meter rollover/replacement, electricity tariff calculation, monthly invoices, maintenance tickets/deductions, VietQR payloads, and payment reconciliation. Financial correctness, idempotency, and locks are mandatory.

`MeterRolloverPolicy`, `EvnTieredTariffByCapitaPolicy`, invoice arithmetic, and ticket-deduction rules MUST be pure domain code using `BigDecimal`. OCR, PayOS/SePay, JPA, Redis, and R2 metadata checks belong in `infrastructure`.

## Meter capture and BR-3.1a anti-swap

- `POST /api/v2/invoices/ocr-meter` is limited to 120/min/User and MUST acquire `lock:ocr:room:<roomId>` for 5 seconds to reject double-taps.
- The browser MUST send only the verified R2 object key/hash and capture metadata. Meter images MUST NOT pass through Spring Boot.
- OCR MUST return both `New_Index` and `Meter_Serial` with confidence values.
- Normalize and compare OCR `Meter_Serial` with `rooms.electric_meter_serial`. A mismatch MUST block automatic posting and require an audited correction/review; NEVER attach room P201 evidence to P202.
- Low-confidence index or serial MUST require human confirmation while preserving original OCR output and evidence hash.
- Invoice persistence MUST use server-verified object metadata. Store an object key, not a public URL or expiring Presigned URL, even if the legacy column name is `elec_meter_image_url`.

## Meter rollover and replacement BR-3.1b

For ordinary meters:

```text
if New_Index >= Old_Index:
    Consumption = New_Index - Old_Index
    is_meter_rollover = false
```

For a rollover candidate:

```text
if New_Index < Old_Index AND Old_Index >= 0.90 × Max_Dial_Value:
    Consumption = (Max_Dial_Value - Old_Index) + New_Index
    is_meter_rollover = true
```

Required UAT-03 example:

```text
Max_Dial_Value = 100000
Old_Index = 99850
New_Index = 120
Consumption = (100000 - 99850) + 120 = 270 kWh
```

- Use `rooms.electric_meter_max_dial` (default `100000`) and `BigDecimal`; NEVER hard-code 100000 inside the policy.
- Treat the 90% threshold as inclusive (`>=`) to match the concurrency/domain guard.
- If `New_Index < Old_Index` below the rollover threshold, reject automatic rollover. Require the meter-replacement flow or correction.
- Meter replacement MUST capture old meter serial, removal closing index, new meter serial, installation opening index, timestamp, and signed evidence images. Consumption is the sum of the valid old-meter segment and new-meter segment; NEVER force replacement through `is_meter_rollover`.

## Electricity pricing BR-3.1c and UAT-01

Only two pricing modes exist:

1. `FIXED_RATE`: multiply consumption by the committed lease unit price.
2. `STATE_TIERED_BY_CAPITA`: allocate current EVN household tier quotas proportionally by registered occupants, where each person receives `1/4` of a household quota, then price each slice at the effective tariff.

Fixed-rate invariant:

```java
if (requestedUnitPrice.compareTo(lease.electricityRateCommitted()) > 0) {
    throw new BusinessRuleViolationException(
        "Đơn giá điện không được vượt mức đã cam kết trong Hợp đồng");
}
```

- The guard MUST exist inside the domain creation path, not only in the request validator/UI.
- UAT-01 MUST reject VND 4,200/kWh for an active lease committed at VND 3,500/kWh and keep 3,500 unchanged.
- `STATE_TIERED_BY_CAPITA` MUST use `leases.occupant_count` synchronized from active `lease_members`.
- For each EVN tier `i` with household quota `Q_i`, the room quota is `Q_i × occupant_count / 4`. Apply consumption sequentially across scaled tiers, then the final unbounded tier.
- Tariff version/effective dates MUST come from immutable reference data cached in Caffeine for 12 hours. NEVER hard-code “current” EVN prices in domain code and NEVER apply a future tariff to an earlier billing month.
- Persist the tariff version and a calculation breakdown with the invoice so the amount remains auditable after tariff changes.

## Maintenance tickets and UAT-04

Preserve `maintenance_tickets`: `lease_id`, `room_id`, `created_by`, `category`, `fault_attribution`, `description`, `evidence_urls`, `tenant_advance_amount`, `is_deduction_approved`, `status`, `created_at`, and `deducted_in_invoice_id`.

- Category MUST be `AIR_CONDITIONER`, `PLUMBING_WATER`, `ELECTRICAL`, `FIRE_SAFETY`, or `OTHER`.
- Fault attribution MUST be `NATURAL_WEAR` or `TENANT_FAULT`.
- Tenant advance requires receipt/evidence in private R2 and a positive `tenant_advance_amount`.
- Only a resource-owning `LANDLORD_OWNER` MAY set `is_deduction_approved = TRUE`. `PROPERTY_MANAGER` MAY process technical work but MUST receive 403 when approving the rent deduction.
- Once applied, `deducted_in_invoice_id` MUST point to the exact invoice and MUST NOT be reassigned or cleared without a compensating audited workflow.

Invoice creation MUST:

1. Lock the parent lease with `PESSIMISTIC_WRITE`.
2. Rely on `uq_invoice_lease_month UNIQUE (lease_id, billing_month)`.
3. Lock all approved tickets with `deducted_in_invoice_id IS NULL` using `FOR UPDATE`.
4. Sum `tenant_advance_amount` into `utility_invoices.maintenance_deduction`.
5. Create the invoice and set every ticket's `deducted_in_invoice_id` in the same transaction.

Required UAT-04 example:

```text
Gross rent + utilities = 4,500,000
maintenance_deduction = 250,000
VietQR payable = 4,250,000 VND
```

Parallel Owner/Manager issue attempts MUST create one invoice and deduct the ticket exactly once.

## Invoice calculation

Preserve `utility_invoices`: `lease_id`, `billing_month` (`YYYY-MM`), `elec_old_index`, `elec_new_index`, `is_meter_rollover`, meter evidence object key, `water_old_index`, `water_new_index`, `previous_debt`, `maintenance_deduction`, `total_amount`, `paid_amount`, `vietqr_payload`, `payment_status`, `paid_at`, `created_at`, and `version`.

Use an explicit auditable equation:

```text
grossAmount = monthlyRent + electricity + water + serviceFees + otherAuthorizedCharges
payableBeforeFloor = grossAmount + previousDebt - maintenanceDeduction - creditBalanceApplied
totalAmount = max(payableBeforeFloor, 0)
```

- All operands MUST be `BigDecimal`, non-negative unless the domain explicitly represents a credit, and use one documented rounding policy.
- If credit exceeds payable, retain only the unused remainder in `leases.credit_balance`; NEVER silently discard it.
- `previous_debt` MUST reflect authoritative unpaid balance, not a caller-provided value.
- Invoice issue MUST publish an AFTER_COMMIT event. Resend batch chunks MUST be at most 100 and idempotent.

## VietQR webhook BR-3.2b

- Verify PayOS/SePay signature and provider replay/timestamp/IP controls before mutation.
- First claim Redis key `webhook:vietqr:<bankTxId>` with `SET NX EX 86400`. Duplicate callbacks return a successful idempotent response without posting twice.
- Also persist a unique immutable bank transaction ID; Redis is the fast gate, not the sole ledger.
- Lock both target `utility_invoices` and `leases` with `PESSIMISTIC_WRITE` in one transaction.
- Add the transfer amount to cumulative `paid_amount`.
- If cumulative `paid_amount < total_amount`, set `PARTIALLY_PAID`; carry `total_amount - paid_amount` to the later invoice's `previous_debt` exactly once.
- If equal, set `PAID` and `paid_at`.
- If greater, set `PAID`, set `paid_at`, and add the excess to `leases.credit_balance` exactly once.
- Never trust an invoice ID/amount from an unsigned callback field. Resolve the payment reference and validate property/lease/invoice linkage.

## Required tests

- MUST test meter serial normalization/mismatch, OCR confidence review, double-tap lock ownership, and direct-R2-only evidence.
- MUST unit-test normal meter arithmetic, exactly-90% boundary, rollover, invalid regression, replacement, decimal indices, and UAT-03.
- MUST unit-test fixed rate cap and UAT-01 at below/equal/above committed price.
- MUST unit-test tier allocation for occupant counts 1–4, tier boundaries, tariff effective dates, and exact invoice breakdown.
- MUST run real parallel PostgreSQL tests for duplicate monthly invoice and ticket deduction; assert UAT-04 is applied once.
- MUST test webhook duplicate delivery, two co-tenants paying 0.5 seconds apart, partial/equal/excess amounts, unique bank transaction persistence, debt carry, and credit carry.
- MUST test Property Manager 403 on deduction approval and revenue endpoints.
