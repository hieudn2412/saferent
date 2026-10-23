# PHASE 6 Operations OCR Billing Tickets and Payments

## Reading boundary

This file is the complete implementation spec for TASK-6.1 through TASK-6.4. Read only this file plus root/backend and operations `AGENTS.md` files named in the plan.

## Phase outcomes

- Meter OCR validates serial and handles normal, rollover and replacement paths.
- Fixed and State-tiered-by-capita electricity policies are pure and auditable.
- Maintenance advances are Owner-approved and deducted exactly once.
- Monthly billing is concurrency-safe and passes UAT-01/UAT-04.
- VietQR replay, partial payment, debt and excess credit are reconciled atomically.

## Atomic task contract

### TASK-6.1 meter policies

Required names: `MeterOcrAnalysisService`, `MeterRolloverPolicyEngine`, `EvnTieredTariffService`, `MeterRolloverPolicy`, `EvnTieredTariffByCapitaPolicy`, `ProcessMeterOcrUseCase`, `VisionMeterOcrAdapter`.

MUST use `BigDecimal`, validate `Meter_Serial`, implement the exact rollover formula/threshold, keep meter replacement separate, scale each EVN household tier quota by `occupant_count / 4`, and persist tariff effective version/breakdown.

### TASK-6.2 maintenance ticket

Required names: `MaintenanceTicket`, `MaintenanceTicketService`, `ApproveTicketDeductionUseCase`, `MaintenanceTicketController`.

MUST preserve source enums/columns, direct-R2 evidence, positive advance, Owner-only approval, Manager 403, and guarded `deducted_in_invoice_id`.

### TASK-6.3 monthly billing

Required names: `UtilityInvoice`, `UtilityInvoiceService`, `MonthlyBillingConcurrencyService`, `CreateMonthlyInvoiceUseCase`, `UtilityInvoiceController`, method `createMonthlyInvoice(CreateInvoiceCommand)`.

MUST lock lease and approved-undeducted tickets, enforce unique lease/month, guard electricity rate in domain, compute debt/deduction/credit exactly and pass UAT-01/UAT-04 under parallel execution.

### TASK-6.4 VietQR reconciliation

Required names: `VietQrWebhookReconciler`, `ReconcileVietQrWebhookUseCase`, `PayOsSePaySignatureVerifier`, `WebhookController`.

MUST verify provider, claim Redis idempotency, persist unique transaction, lock invoice+lease, accumulate `paid_amount` and apply exact partial/equal/excess behavior.

## Verbatim source excerpts

### SRS maintenance_tickets and utility_invoices DDL

```sql
CREATE TABLE maintenance_tickets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    lease_id UUID NOT NULL REFERENCES leases(id),
    room_id UUID NOT NULL REFERENCES rooms(id),
    created_by UUID NOT NULL REFERENCES users(id),
    category VARCHAR(30) CHECK (category IN ('AIR_CONDITIONER', 'PLUMBING_WATER', 'ELECTRICAL', 'FIRE_SAFETY', 'OTHER')),
    fault_attribution VARCHAR(30) DEFAULT 'NATURAL_WEAR' CHECK (fault_attribution IN ('NATURAL_WEAR', 'TENANT_FAULT')),
    description TEXT NOT NULL,
    evidence_urls JSONB NOT NULL,
    tenant_advance_amount DECIMAL(12, 2) DEFAULT 0,
    is_deduction_approved BOOLEAN DEFAULT FALSE,
    status VARCHAR(30) DEFAULT 'OPEN' CHECK (status IN ('OPEN', 'IN_PROGRESS', 'RESOLVED', 'REJECTED')),
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE utility_invoices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    lease_id UUID NOT NULL REFERENCES leases(id),
    billing_month VARCHAR(7) NOT NULL,
    elec_old_index DECIMAL(10, 2) NOT NULL,
    elec_new_index DECIMAL(10, 2) NOT NULL,
    is_meter_rollover BOOLEAN DEFAULT FALSE,
    elec_meter_image_url TEXT NOT NULL,
    water_old_index DECIMAL(10, 2) NOT NULL,
    water_new_index DECIMAL(10, 2) NOT NULL,
    previous_debt DECIMAL(12, 2) DEFAULT 0,
    maintenance_deduction DECIMAL(12, 2) DEFAULT 0,
    total_amount DECIMAL(12, 2) NOT NULL,
    paid_amount DECIMAL(12, 2) DEFAULT 0,
    vietqr_payload TEXT,
    payment_status VARCHAR(20) DEFAULT 'UNPAID' CHECK (payment_status IN ('UNPAID', 'PARTIALLY_PAID', 'PAID', 'OVERDUE')),
    paid_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
```

### SADD concurrency DDL

```sql
ALTER TABLE utility_invoices ADD COLUMN version BIGINT NOT NULL DEFAULT 0;

ALTER TABLE utility_invoices
ADD CONSTRAINT uq_invoice_lease_month UNIQUE (lease_id, billing_month);

ALTER TABLE maintenance_tickets
ADD COLUMN deducted_in_invoice_id UUID REFERENCES utility_invoices(id);
```

### SRS BR-3.1a meter serial

> BR-3.1a (Kiểm tra Chéo Mã Công tơ - Anti-Swap): AI đọc cả chỉ số tiêu thụ (`New_Index`) và số Seri công tơ (`Meter_Serial`) để đảm bảo Quản lý không chụp nhầm công tơ phòng 201 gắn vào hóa đơn phòng 202.

### SRS BR-3.1b rollover and replacement

> Khi `New_Index < Old_Index`, thay vì chặn cứng như bản v1.0, hệ thống kiểm tra 2 trường hợp thực tế: (1) Nếu `Old_Index > 90% Max_Dial_Value` (VD: tháng trước là `99850`, tháng này là `00120` trên công tơ 5 chữ số): Hệ thống tự động gợi ý chế độ **Công tơ quay vòng** với công thức: `Sản lượng = (100000 - Old_Index) + New_Index = (100000 - 99850) + 120 = 270 kWh`. (2) Nếu thay công tơ mới giữa tháng: Cho phép nhập `Chỉ số chốt công tơ cũ lúc tháo` + `Chỉ số ban đầu công tơ mới lúc lắp` kèm ảnh biên bản thay đồng hồ.

The UAT/SADD domain guard uses the inclusive threshold `Old_Index >= 90% max_dial`; implementation MUST follow that stricter final rule.

### SRS BR-3.1c two electricity modes

> Phương thức 1 (Đơn giá cam kết cố định - Fixed Committed Rate): Nhân trực tiếp số kWh tiêu thụ với đơn giá đã ký trong hợp đồng (không được tự ý sửa cao hơn hợp đồng).

> Phương thức 2 (Giá bậc thang EVN theo định mức nhân khẩu - State Tiered by Capita): Hệ thống tự động lấy số người đăng ký ở trong phòng (`occupant_count`), tính định mức bậc thang (mỗi người = 1/4 định mức hộ gia đình theo quy định Bộ Công Thương) và áp biểu giá bậc thang EVN hiện hành.

### SRS BR-3.2a maintenance deduction

> Khi đóng một Ticket sửa chữa có gắn cờ `TENANT_PAID_ON_BEHALF` (Người thuê ứng tiền trước kèm ảnh biên lai và được Chủ nhà bấm Duyệt), số tiền đó sẽ tự động được đưa vào mục **`Khoản giảm trừ / Khấu trừ sửa chữa` (`deduction_amount`)** trong hóa đơn tiền phòng tháng kế tiếp.

The final SRS DDL and UAT name the canonical invoice column `maintenance_deduction`; implementation MUST use that final name, not the earlier narrative alias `deduction_amount`.

### SRS BR-3.2b payment outcomes

> Khi Webhook VietQR (PayOS/SePay) trả về số tiền thực nhận (`paid_amount`):

```text
Nếu paid_amount == total_amount:
    Đổi trạng thái sang PAID.

Nếu paid_amount < total_amount:
    Đổi trạng thái sang PARTIALLY_PAID,
    phần còn thiếu tự động kết chuyển sang previous_debt của hóa đơn tháng sau.

Nếu paid_amount > total_amount:
    Ghi nhận khoản dư vào credit_balance của hợp đồng để tự động trừ vào tháng sau.
```

### SADD six-second OCR/tariff race control

> Redis Distributed Lock ngắn hạn (5 giây) + Domain Invariant Guard: Khóa Redis `lock:ocr:room:<roomId>` trong 5 giây để chống double-tap nút chụp ảnh. Trong Domain Entity UtilityInvoice, hàm khởi tạo kiểm tra trực tiếp: nếu `unitPrice > lease.getElectricityRateCommitted()` thì ném ngay `BusinessRuleViolationException` (UAT-01). Nếu `newIndex < oldIndex` và `oldIndex > 0.9 * maxDial`, tự động bật cờ `is_meter_rollover = true` và tính bù vòng (UAT-03).

### SADD ticket deduction concurrency

> Unique Constraint + Pessimistic Write Lock trên Ticket sửa chữa: Đặt `UNIQUE (lease_id, billing_month)` trên bảng utility_invoices để chặn tạo 2 hóa đơn cùng kỳ. Thêm cột `deducted_in_invoice_id UUID REFERENCES utility_invoices(id)` vào bảng maintenance_tickets. Khi tạo hóa đơn tháng, thực thi `SELECT ... FOR UPDATE` trên các Ticket có `is_deduction_approved = TRUE AND deducted_in_invoice_id IS NULL`, tính tổng giảm trừ và cập nhật ngay deducted_in_invoice_id trong cùng 1 giao dịch ACID.

### SADD VietQR concurrency

> Redis Idempotency (SETNX) + Pessimistic Write Lock (SELECT ... FOR UPDATE):

```text
Bước 1: SET webhook:vietqr:<bankTxId> 1 NX EX 86400.
Nếu đã tồn tại, bỏ qua ngay.

Bước 2: Dùng @Lock(LockModeType.PESSIMISTIC_WRITE)
khóa dòng utility_invoices và dòng leases.

Bước 3: Cộng dồn paid_amount.
Nếu paid_amount < total_amount -> PARTIALLY_PAID.
Nếu paid_amount > total_amount -> PAID và cộng tiền dư vào leases.credit_balance.
```

### SADD MonthlyBillingConcurrencyService example

```java
@Service
@RequiredArgsConstructor
public class MonthlyBillingConcurrencyService {
    private final LeaseRepository leaseRepository;
    private final MaintenanceTicketRepository ticketRepository;
    private final UtilityInvoiceRepository invoiceRepository;

    @Transactional(isolation = Isolation.READ_COMMITTED)
    public UtilityInvoice createMonthlyInvoice(CreateInvoiceCommand cmd) {
        Lease lease = leaseRepository.findByIdWithPessimisticLock(cmd.leaseId())
                .orElseThrow(() -> new ResourceNotFoundException("Lease not found"));

        if (cmd.electricityUnitPrice().compareTo(lease.getElectricityRateCommitted()) > 0) {
            throw new BusinessRuleViolationException(String.format(
                "Đơn giá điện không được vượt mức %,.0fđ/kWh đã cam kết trong Hợp đồng",
                lease.getElectricityRateCommitted()
            ));
        }

        BigDecimal elecConsumption;
        boolean isRollover = false;
        BigDecimal maxDial = BigDecimal.valueOf(lease.getRoom().getElectricMeterMaxDial());

        if (cmd.elecNewIndex().compareTo(cmd.elecOldIndex()) < 0) {
            if (cmd.elecOldIndex().compareTo(maxDial.multiply(BigDecimal.valueOf(0.9))) >= 0) {
                isRollover = true;
                elecConsumption = maxDial.subtract(cmd.elecOldIndex()).add(cmd.elecNewIndex());
            } else {
                throw new BusinessRuleViolationException("Chỉ số điện mới nhỏ hơn chỉ số cũ bất thường!");
            }
        } else {
            elecConsumption = cmd.elecNewIndex().subtract(cmd.elecOldIndex());
        }

        List<MaintenanceTicket> approvedTickets = ticketRepository
                .findApprovedUndeductedTicketsWithLock(lease.getId());
        BigDecimal totalMaintenanceDeduction = approvedTickets.stream()
                .map(MaintenanceTicket::getTenantAdvanceAmount)
                .reduce(BigDecimal.ZERO, BigDecimal::add);

        BigDecimal previousDebt = invoiceRepository.calculateUnpaidBalanceFromPreviousMonths(lease.getId());
        BigDecimal grossAmount = calculateGrossAmount(lease, elecConsumption, cmd);
        BigDecimal finalPayable = grossAmount.add(previousDebt)
                .subtract(totalMaintenanceDeduction)
                .subtract(lease.getCreditBalance())
                .max(BigDecimal.ZERO);

        UtilityInvoice savedInvoice = invoiceRepository.save(UtilityInvoice.builder()
                .leaseId(lease.getId())
                .billingMonth(cmd.billingMonth())
                .elecOldIndex(cmd.elecOldIndex())
                .elecNewIndex(cmd.elecNewIndex())
                .isMeterRollover(isRollover)
                .previousDebt(previousDebt)
                .maintenanceDeduction(totalMaintenanceDeduction)
                .totalAmount(finalPayable)
                .paymentStatus(PaymentStatus.UNPAID)
                .build());

        approvedTickets.forEach(t -> t.markDeductedInInvoice(savedInvoice.getId()));
        ticketRepository.saveAll(approvedTickets);
        return savedInvoice;
    }
}
```

### SRS operations APIs

| Method & Endpoint | Rate Limit | Mô tả |
|---|---|---|
| `POST /api/v2/invoices/ocr-meter` | 120 req/phút/User | Nhận ảnh công tơ đã crop từ Client, đọc chỉ số + mã seri, tự động phát hiện công tơ quay vòng (`is_meter_rollover`) và áp dụng khoản khấu trừ sửa chữa. |
| `POST /api/v2/webhooks/vietqr` | Whitelist IP | Nhận callback chuyển khoản, hỗ trợ gạch nợ toàn phần (`PAID`) hoặc gạch nợ một phần (`PARTIALLY_PAID`) và kết chuyển số dư. |

### UAT-01 verbatim Gherkin

```gherkin
UAT-01: Chặn Chủ nhà tự ý tăng giá điện vượt mức cam kết trong Hợp đồng

GIVEN: Hợp đồng phòng P302 đang `ACTIVE` với đơn giá điện cam kết `3.500 VNĐ/kWh`.
WHEN: Chủ nhà/Quản lý tạo hóa đơn tháng và sửa đơn giá điện thành `4.200 VNĐ/kWh`.
THEN: Hệ thống chặn thao tác, báo lỗi đỏ 'Đơn giá điện không được vượt mức 3.500đ/kWh đã cam kết trong Hợp đồng' và giữ nguyên mức 3.500đ/kWh.
```

### UAT-03 verbatim Gherkin

```gherkin
UAT-03: Xử lý Công tơ Điện Quay vòng (Meter Rollover) khi chốt sổ OCR

GIVEN: Công tơ điện phòng P105 là loại 5 chữ số (`max_dial = 100000`), chỉ số tháng trước là `99850 kWh`.
WHEN: Quản lý chụp ảnh công tơ tháng này, AI OCR đọc được chỉ số mới là `00120 kWh`.
THEN: Hệ thống nhận diện `Old_Index > 90% max_dial`, tự động bật cờ gợi ý `Công tơ quay vòng` và tính chính xác sản lượng điện tiêu thụ là `(100000 - 99850) + 120 = 270 kWh` thay vì báo lỗi số âm.
```

### UAT-04 verbatim Gherkin

```gherkin
UAT-04: Tự động Khấu trừ Tiền sửa chữa Người thuê ứng trước vào Hóa đơn tháng

GIVEN: Người thuê phòng P401 tạo Ticket báo hỏng vòi nước, ứng trước `250.000đ` trả thợ kèm ảnh hóa đơn, và Chủ nhà đã bấm `Duyệt khấu trừ`.
WHEN: Chủ nhà phát hành hóa đơn tiền phòng tháng kế tiếp (Tổng tiền thuê + điện nước = `4.500.000đ`).
THEN: Hệ thống tự động điền `maintenance_deduction = 250.000đ` và sinh mã VietQR với số tiền thực thu là `4.250.000đ`.
```

## Derived implementation contract

This section is normative but not verbatim.

- Inclusive rollover decision: normal when new >= old; rollover when new < old AND old >= 0.90 × maxDial; otherwise replacement/correction required.
- Meter replacement records old serial/removal close, new serial/install opening, timestamps and private-R2 evidence. Do not set rollover for replacement.
- For tariff tier `i` with household quota `Q_i`, room quota is `Q_i × occupant_count / 4`. Consume scaled tiers sequentially and use the unbounded final tier. Load dated tariff reference from Caffeine; do not hard-code current prices.
- Canonical invoice equation:

```text
grossAmount = monthlyRent + electricity + water + serviceFees + otherAuthorizedCharges
payableBeforeFloor = grossAmount + previousDebt - maintenanceDeduction - creditBalanceApplied
totalAmount = max(payableBeforeFloor, 0)
```

- If credit exceeds payable, retain unused credit. Persist invoice calculation/tariff breakdown.
- `elec_meter_image_url` stores an R2 object key semantically, never a URL. OCR lock release must compare owner token.
- Webhook Redis is a fast gate; a unique persistent provider transaction record is mandatory. Payment amount is added to cumulative amount, never replaces it.
- Provider signature/timestamp/replay/IP policy is checked before mutation. Duplicate callback returns idempotent success.

## JSON inventory

- Phase 6 owns no LLM JSON schema. Provider webhook DTO schema MUST be derived from the selected PayOS/SePay official contract and isolated behind the signature verifier port; do not invent unsigned trust fields.

## Verification gate

- TASK-6.1 pure tests: serial normalization/mismatch, normal/rollover/exact-90%/invalid/replacement, decimals, tariff tiers N=1..4/effective dates/rounding and UAT-03.
- TASK-6.2: enum/check constraints, evidence verification, Owner approval, Manager 403, invalid advance and immutable applied deduction.
- TASK-6.3: domain rate boundary and UAT-01; parallel monthly issue creates one invoice/deducts once; UAT-04 exact VND values; debt/credit floor/remainder.
- TASK-6.4: signature failure, duplicate retry, two payments 0.5 seconds apart, partial/equal/excess, persistent uniqueness, lock ordering/deadlock-safe retry.
- `mvn test` MUST pass with PostgreSQL/Redis integration tests.
