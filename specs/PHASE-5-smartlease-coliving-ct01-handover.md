# PHASE 5 SmartLease Co-living CT01 and Handover

## Reading boundary

This file is the complete implementation spec for TASK-5.1 through TASK-5.3. Read only this file plus the root/backend and directly applicable identity/smartlease `AGENTS.md` files named in the plan.

## Phase outcomes

- Contract AI scan masks PII before LLM and streams validated results with SSE.
- Co-tenant activation is race-safe and keeps `occupant_count` authoritative.
- Apache POI produces Mẫu CT01 for all active members under the performance/authorization contract.
- Check-in evidence locks after 48 hours; check-out settles deposits transparently within seven days.

## Atomic task contract

### TASK-5.1 PII-masked contract AI scan

Required names:

- `AiContractScannerService`, `PiiMaskingService`
- `ScanContractAiUseCase`
- `LlmContractScannerAdapter`, `PiiMasker`
- `ContractAiController`
- endpoint `POST /api/v2/contracts/ai-scan`

MUST accept only a verified private-R2 object reference, mask CCCD/phone/full name and other direct identifiers before any LLM call, validate the exact JSON shape below, stream typed SSE events, and enforce 1/day Guest or 3/day Tenant. No DB transaction may remain open during OCR/LLM/SSE I/O.

### TASK-5.2 co-living and CT01

Required names:

- `Lease`, `LeaseMember`
- `CoLivingMemberService`, `Ct01DocumentExportService`
- `AddCoTenantUseCase`, `ExportCt01UseCase`
- `ApachePoiCt01Exporter`
- endpoint `POST /api/v2/leases/{id}/members`
- endpoint `GET /api/v2/leases/{id}/export-ct01`

MUST lock the parent lease before member mutation, enforce `rooms.max_occupants`, recompute active member count, authorize CCCD decrypt, fill the official CT01 template and pass UAT-02.

### TASK-5.3 check-in checkout settlement

Required names:

- `HandoverRecord`, `DepositSettlementPolicy`
- `HandoverCheckInOutService`, `DepositSettlementService`
- `SubmitCheckInUseCase`, `AutoLockCheckInJob`, `SettleCheckOutUseCase`
- `HandoverController`
- endpoint `POST /api/v2/leases/{id}/check-out`

MUST use direct private-R2 evidence, SHA-256 manifest, server time/GPS, 48-hour ShedLock + optimistic version, immutable locked records, notice validation, natural-wear classification and the exact settlement formula/UAT-06.

## Verbatim source excerpts

### SRS leases and lease_members DDL

```sql
CREATE TABLE leases (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    room_id UUID NOT NULL REFERENCES rooms(id),
    primary_tenant_id UUID NOT NULL REFERENCES users(id),
    landlord_id UUID NOT NULL REFERENCES users(id),
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,
    occupant_count SMALLINT DEFAULT 1 NOT NULL,
    notice_period_days SMALLINT DEFAULT 30,
    termination_requested_at TIMESTAMPTZ,
    monthly_rent DECIMAL(12, 2) NOT NULL,
    deposit_amount DECIMAL(12, 2) NOT NULL,
    electricity_rate_committed DECIMAL(10, 2) NOT NULL,
    credit_balance DECIMAL(12, 2) DEFAULT 0,
    contract_pdf_sha256 VARCHAR(64),
    status VARCHAR(30) DEFAULT 'DRAFT' CHECK (status IN ('DRAFT', 'ACTIVE', 'PENDING_CHECKOUT', 'TERMINATED', 'EXPIRED')),
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE lease_members (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    lease_id UUID NOT NULL REFERENCES leases(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id),
    member_role VARCHAR(20) CHECK (member_role IN ('PRIMARY', 'CO_TENANT')),
    joined_date DATE NOT NULL,
    left_date DATE,
    residence_reg_status VARCHAR(30) DEFAULT 'PENDING_CT01' CHECK (residence_reg_status IN ('PENDING_CT01', 'CT01_EXPORTED', 'VNEID_CONFIRMED')),
    is_active BOOLEAN DEFAULT TRUE
);
```

### SRS AI scan behavior

> Người dùng tải ảnh chụp hoặc file PDF hợp đồng lên hệ thống. Hệ thống thực hiện OCR, che thông tin nhạy cảm (PII Masking: CCCD, SĐT, Họ tên) trước khi gửi sang LLM phân tích theo Luật Nhà ở 2023, Bộ luật Dân sự 2015 và Thông tư 09/2023/TT-BCT về giá điện nhà trọ, trả về Điểm an toàn (0–100) kèm gợi ý đàm phán lại.

### SRS AI endpoint

| Method & Endpoint | Phân hệ | Rate Limit | Mô tả |
|---|---|---|---|
| `POST /api/v2/contracts/ai-scan` | Smart Lease | 3 req/ngày/User | Nhận file/ảnh hợp đồng, chạy PII Masking, phân tích rủi ro pháp lý qua LLM và trả kết quả bằng Server-Sent Events (SSE). |

The architecture's complete rate-limit table additionally sets Guest to 1/day and Tenant to 3/day.

### SRS LLM system prompt and JSON schema

```text
[SYSTEM PROMPT - SAFERENT VN LEGAL AI ANALYZER v2.0]

Bạn là Chuyên gia Pháp lý Bất động sản tại Việt Nam, am hiểu Bộ luật Dân sự 2015, Luật Nhà ở 2023, Luật Cư trú 2020 và Thông tư 09/2023/TT-BCT quy định về cách tính giá điện sinh hoạt cho người thuê nhà.

Hãy phân tích văn bản Hợp đồng thuê trọ dưới đây và đánh giá rủi ro theo 4 nhóm tiêu chí:

1. Giá Điện/Nước & Phí dịch vụ: Cảnh báo ĐỎ (RED) nếu giá điện cố định > 4.000đ/kWh hoặc không ghi rõ đơn giá dịch vụ theo đầu người/theo phòng.

2. Điều khoản Đặt cọc, Báo trước & Hoàn cọc: Cảnh báo ĐỎ (RED) nếu thời gian báo trước của người thuê dài bất hợp lý (> 45 ngày) trong khi chủ nhà được lấy lại nhà báo trước < 15 ngày, hoặc không ghi rõ thời hạn hoàn trả tiền cọc sau khi bàn giao phòng (chuẩn an toàn là <= 7 ngày).

3. Trách nhiệm Sửa chữa Thiết bị (Điều hòa, Bình nóng lạnh): Theo Điều 477 Bộ luật Dân sự 2015, chủ nhà phải chịu chi phí bảo dưỡng và sửa chữa các hư hỏng do hao mòn tự nhiên. Cảnh báo ĐỎ (RED) nếu đẩy mọi chi phí sửa chữa hao mòn tự nhiên cho người thuê.

4. Quyền Đăng ký Tạm trú & Ở ghép: Cảnh báo VÀNG (YELLOW) nếu hợp đồng cấm hoặc thu phí vô lý khi người thuê yêu cầu chủ nhà xác nhận đăng ký tạm trú theo Luật Cư trú.

TRẢ VỀ ĐỊNH DẠNG JSON CHUẨN:
```

```json
{
  "safety_score": 62,
  "summary": "Hợp đồng thiếu cam kết thời hạn hoàn trả tiền cọc khi trả phòng và đẩy trách nhiệm sửa điều hòa hao mòn tự nhiên cho người thuê.",
  "risk_clauses": [
    {
      "severity": "RED",
      "category": "DEPOSIT_AND_CHECKOUT",
      "clause_excerpt": "Bên A sẽ hoàn trả tiền cọc sau khi đã tìm được người thuê mới.",
      "legal_basis": "Điều 328 & Điều 482 Bộ luật Dân sự 2015.",
      "explanation": "Việc hoàn trả tiền cọc phụ thuộc vào việc bàn giao phòng và thanh toán hết nghĩa vụ tài chính của bạn, hoàn toàn không được phụ thuộc vào việc chủ nhà đã tìm được khách mới hay chưa.",
      "suggested_revision": "Bên A hoàn trả lại toàn bộ tiền đặt cọc (sau khi trừ tiền điện, nước kỳ cuối nếu có) cho Bên B ngay tại thời điểm bàn giao trả phòng hoặc chậm nhất trong vòng 05 ngày làm việc."
    }
  ]
}
```

### SRS co-living and CT01 main flow

> 1. Khi hợp đồng được tạo, Người thuê đại diện (Primary Tenant) gửi link mời các bạn cùng phòng (Co-Tenants) tham gia vào phòng trên hệ thống.

> 2. Từng thành viên chụp ảnh 2 mặt CCCD gắn chip. AI OCR tự động trích xuất: Họ tên, Ngày sinh, Số định danh cá nhân (12 số), Quê quán, Nơi thường trú.

> 3. Hệ thống tự động cập nhật `occupant_count` (Số nhân khẩu thực tế) của hợp đồng để tính đúng tiền nước/dịch vụ và định mức điện.

> 4. Tính năng '1-Click Xuất Hồ sơ Tạm trú (Mẫu CT01)': Chủ nhà chỉ cần bấm 1 nút, hệ thống tự động điền toàn bộ thông tin Chủ hộ, Địa chỉ nhà trọ và Danh sách thành viên cùng phòng vào đúng biểu mẫu **Tờ khai thay đổi thông tin cư trú (Mẫu CT01 ban hành kèm Thông tư 66/2023/TT-BCA)** dưới dạng file Word/PDF chuẩn, sẵn sàng để chủ nhà nộp lên Cổng dịch vụ công / VNeID hoặc in nộp Công an phường.

### SRS membership and CT01 API

| Method & Endpoint | Rate Limit | Mô tả |
|---|---|---|
| `POST /api/v2/leases/{id}/members` | 10 req/phút/User | Thêm thành viên ở ghép (Co-Tenant) qua eKYC CCCD, tự động cập nhật `occupant_count` của hợp đồng. |
| `GET /api/v2/leases/{id}/export-ct01` | 10 req/phút/User | Trích xuất dữ liệu CCCD của các thành viên trong phòng ra file Word/PDF chuẩn Mẫu CT01 (Bộ Công an) để đăng ký tạm trú. |

### SADD co-tenant concurrency rule

> Parent-Row Pessimistic Write Lock trên bảng leases: Trước khi thêm bản ghi mới vào bảng lease_members, lấy khóa PESSIMISTIC_WRITE trên bản ghi cha Lease (SELECT * FROM leases WHERE id = :leaseId FOR UPDATE). Kiểm tra số lượng thành viên hiện tại < rooms.max_occupants, chèn thành viên mới và cập nhật lại occupant_count bằng câu đếm trực tiếp số thành viên active trong phòng.

### SADD CT01 exporter requirement

> Trong module smartlease, ApachePoiCt01Exporter đọc file phôi Mau_CT01_BoCongAn_Template.docx, giải mã dữ liệu cccd_encrypted của tất cả thành viên active trong bảng lease_members (gồm cả Primary Tenant và Co-Tenants) để điền tự động vào bảng danh sách thành viên cùng cư trú và xuất ra luồng byte cho Chủ nhà tải về chỉ trong < 200ms (Đáp ứng UAT-02).

### SRS Digital Check-in and Check-out

| Giai đoạn | Quy trình Nghiệp vụ & Ràng buộc Hệ thống |
|---|---|
| 1. Nhận phòng (Digital Check-in) [Trong 48h đầu] | Người thuê chụp ảnh/video hiện trạng thiết bị (điều hòa, nóng lạnh, tường, nội thất) và chốt chỉ số công tơ điện/nước gốc. Hệ thống đóng dấu Watermark (Timestamp NTP Server + GPS + Mã HĐ) và lưu mã băm SHA-256. Sau 48h nếu Chủ nhà không bấm 'Khiếu nại hiện trạng', biên bản tự động khóa (`LOCKED`) và lấy chỉ số công tơ đó làm mốc đầu kỳ. |
| 2. Báo kết thúc HĐ (Notice of Termination) | Người thuê hoặc Chủ nhà bấm nút 'Thông báo chấm dứt hợp đồng / Không gia hạn' trên hệ thống. Hệ thống tự động đối chiếu số ngày báo trước (`notice_period_days`, thường là 30 ngày) trong hợp đồng để thông báo ngay cho hai bên biết có đủ điều kiện hoàn 100% cọc hay vi phạm thời hạn báo trước. |
| 3. Bàn giao trả phòng (Digital Check-out) & So sánh Ảnh | Vào ngày trả phòng, Quản lý/Chủ nhà và Người thuê mở màn hình So sánh Song song (Side-by-Side Comparison): Bên trái là ảnh `Check-in` gốc (đã khóa từ lúc mới vào ở), bên phải là ảnh `Check-out` hiện tại. Nếu có hư hỏng, hệ thống bắt buộc phân loại rõ: (a) Hao mòn tự nhiên (Chủ nhà chịu) hoặc (b) Hư hỏng do lỗi người thuê (Khấu trừ cọc kèm ảnh bằng chứng). |
| 4. Quyết toán Cọc (Deposit Settlement) [SLA <= 7 ngày] | Hệ thống tự động lập Phiếu Quyết toán Kết thúc Hợp đồng: `Số tiền hoàn trả = Tiền cọc ban đầu - (Tiền phòng lẻ ngày + Tiền điện/nước kỳ cuối + Chi phí đền bù hư hỏng được 2 bên xác nhận)`. Hệ thống sinh mã VietQR của Người thuê để Chủ nhà quét hoàn cọc ngay tại chỗ (hoặc cam kết hoàn trả trong vòng tối đa 05–07 ngày). Lịch sử hoàn cọc đúng hạn được cộng điểm trực tiếp vào chỉ số Uy tín của Chủ nhà. |

### SADD 48-hour concurrency rule

> ShedLock (@SchedulerLock) + Optimistic Locking (@Version): Dùng ShedLock trên Redis/PostgreSQL để đảm bảo Cron Job chỉ chạy trên duy nhất 1 node. Bảng handover_records và verified_reviews đều có cột @Version. Nếu Chủ nhà vừa bấm phản hồi ở giây 47h59m59s, version tăng lên 1 -> lệnh cập nhật của Cron Job sẽ phát hiện lệch version và nhường quyền ưu tiên cho phản hồi của Chủ nhà.

### SRS Check-out endpoint

| Method & Endpoint | Rate Limit | Mô tả |
|---|---|---|
| `POST /api/v2/leases/{id}/check-out` | 10 req/phút/User | Tạo biên bản bàn giao trả phòng, đối chiếu ảnh Check-in/Check-out và tự động tính bảng quyết toán hoàn trả tiền cọc. |

### UAT-02 verbatim Gherkin

```gherkin
UAT-02: Tự động tính đúng tiền dịch vụ khi thêm Thành viên Ở ghép & Xuất Mẫu CT01

GIVEN: Phòng P201 có phí nước `100.000đ/người/tháng`, ban đầu chỉ có 1 Primary Tenant (`occupant_count = 1`).
WHEN: Primary Tenant thêm 2 bạn cùng phòng (`Co-Tenants`) qua eKYC CCCD thành công.
THEN: Hệ thống tự động cập nhật `occupant_count = 3`, tính tiền nước kỳ tới là `300.000đ`, và cho phép Chủ nhà bấm nút xuất file `Mau_CT01_P201.docx` chứa đầy đủ thông tin định danh của cả 3 sinh viên.
```

### UAT-06 verbatim Gherkin

```gherkin
UAT-06: Quyết toán Hoàn cọc tự động khi Trả phòng (Digital Check-out)

GIVEN: Hợp đồng có tiền cọc `4.000.000đ`, người thuê đã báo nghỉ đúng hạn 30 ngày. Khi Check-out, tiền điện nước kỳ cuối là `350.000đ` và không có hư hỏng do lỗi người thuê.
WHEN: Hai bên bấm xác nhận Biên bản Check-out.
THEN: Hệ thống lập Phiếu Quyết toán yêu cầu Chủ nhà hoàn trả đúng `3.650.000đ` và hiển thị mã VietQR của Người thuê để Chủ nhà chuyển khoản hoàn cọc.
```

## Derived implementation contract

This section is normative but not verbatim.

- AI input travels by verified R2 reference. SSE event contract must have typed progress/finding/completed/failed events and opaque scan ID; no PII in URL or events.
- Validate LLM output with a strict schema: score range 0–100, enum severity/category allowlist, bounded strings. Never trust LLM output as executable or authoritative legal fact.
- Active membership uniqueness/idempotency and one active Primary must be database-enforced in a feature migration where feasible.
- Add-member transaction order is: lock lease, authorize, verify eKYC, count active, enforce capacity, insert/idempotently resolve, recount, update lease count, commit.
- CT01 decrypt exists only inside authorized export. Owner owns property; Manager requires explicit Owner grant. Never cache plaintext or generated private document.
- Check-in media is direct private R2. Persist an ordered evidence manifest whose SHA-256 binds object hashes, GPS, server time and lease ID. Locked evidence is immutable; correction is append-only amendment.
- `DepositSettlementPolicy` uses `BigDecimal` and floors refund at zero; excess obligation is separate debt. Only confirmed `TENANT_FAULT` damage is deductible; `NATURAL_WEAR` is not.
- `TERMINATED` starts the 90-day original-CCCD-image deletion clock only after completed settlement.

## Verification gate

- TASK-5.1 tests masking-before-adapter, JSON validation, prompt thresholds, SSE order/dedup/cancel/error, rate limits, no transaction during LLM and log/trace redaction.
- TASK-5.2 runs concurrent eKYC activations against PostgreSQL and proves count 3, capacity, idempotency, all active members in CT01, authorization and controlled <200 ms generation. UAT-02 passes.
- TASK-5.3 tests exactly-before/at/after 48 hours, scheduler/response race, immutable evidence, notice period, damage classes, settlement rounding/debt/SLA and UAT-06.
- `mvn test` MUST pass.
