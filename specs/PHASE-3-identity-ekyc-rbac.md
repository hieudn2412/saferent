# PHASE 3 Identity eKYC and RBAC

## Reading boundary

This file is the complete implementation spec for TASK-3.1 and TASK-3.2. Read only this file plus root, backend and identity `AGENTS.md` files named in the plan.

## Phase outcomes

- Stateless JWT authentication uses a 15-minute access token and Redis-backed 7-day refresh rotation.
- Exactly six persisted roles exist; Primary/Co-Tenant remains a lease-membership attribute.
- Every protected action combines RBAC and resource ownership.
- VietQR bank changes are Owner-only and require step-up OTP; UAT-05 passes.

## Atomic task contract

### TASK-3.1 JWT authentication and refresh rotation

Required SADD service/class names:

- `AuthTokenService`
- `EkycCccdOcrService`
- `ConsentAuditService`
- `User`, `UserRole`, `CccdProfile`
- `AuthenticateUserUseCase`, `VerifyEkycUseCase`
- `JpaUserRepository`, `VisionCccdOcrAdapter`
- `AuthController`

Implementation MUST:

- issue signed access JWT valid for 15 minutes;
- rotate refresh tokens stored/digested in Redis for at most 7 days;
- detect refresh reuse and revoke the token family;
- represent only `TENANT`, `LANDLORD_OWNER`, `PROPERTY_MANAGER`, `INSPECTOR`, `PARTNER`, `ADMIN` as user roles;
- never encode mutable property ownership/manager assignment or Primary/Co-Tenant membership as long-lived JWT facts;
- use Phase 2 encryption/digest for CCCD and consent/audit the eKYC purpose;
- mask landlord phone last four digits for public views.

### TASK-3.2 Step-up OTP ownership and bank API

Required names:

- `BankAccountStepUpService`
- `UpdateBankConfigUseCase`
- `LandlordBankConfigController`
- `ResourceOwnershipEvaluator`
- `@RequireStepUpOtp`
- action string `UPDATE_VIETQR_BANK_ACCOUNT`

Implementation MUST:

- scope Owner by `properties.owner_id == principal.userId`;
- scope Manager by `properties.manager_id == principal.userId` but still deny bank mutation;
- bind OTP to principal, property, action and a short TTL; digest it; enforce attempts and one-time use;
- return 403 for `PROPERTY_MANAGER`, wrong-property Owner, missing/invalid/expired/reused step-up proof;
- audit success and denial without full bank/OTP/CCCD data;
- pass the exact UAT-05 scenario below.

## Verbatim source excerpts

### SRS role constraint and ownership columns

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    full_name VARCHAR(150) NOT NULL,
    date_of_birth DATE,
    phone_number VARCHAR(20) UNIQUE NOT NULL,
    email VARCHAR(150) UNIQUE,
    role VARCHAR(30) NOT NULL CHECK (role IN ('TENANT', 'LANDLORD_OWNER', 'PROPERTY_MANAGER', 'INSPECTOR', 'PARTNER', 'ADMIN')),
    cccd_encrypted TEXT,
    cccd_hash VARCHAR(64) UNIQUE,
    ekyc_verified BOOLEAN DEFAULT FALSE,
    reputation_points INT DEFAULT 0,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE properties (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    owner_id UUID NOT NULL REFERENCES users(id),
    manager_id UUID REFERENCES users(id),
    -- Quản lý/Bảo vệ được ủy quyền chụp công tơ, xử lý Ticket
    name VARCHAR(255) NOT NULL,
    address_detail TEXT NOT NULL,
    ward_code VARCHAR(20),
    district_code VARCHAR(20) NOT NULL,
    city_code VARCHAR(20) NOT NULL,
    location GEOGRAPHY(Point, 4326) NOT NULL,
    -- PCCC & Hạn hiệu lực xác thực định kỳ 6 tháng
    fire_safety_level SMALLINT DEFAULT 0 CHECK (fire_safety_level IN (0, 1, 2)),
    fire_badge_expires_at TIMESTAMPTZ,
    has_second_exit BOOLEAN DEFAULT FALSE,
    has_fire_door BOOLEAN DEFAULT FALSE,
    has_smoke_detector BOOLEAN DEFAULT FALSE,
    -- Cấu hình Điện / Nước / Dịch vụ
    electricity_pricing_type VARCHAR(30) CHECK (electricity_pricing_type IN ('STATE_TIERED_BY_CAPITA', 'FIXED_RATE')),
    electricity_unit_price DECIMAL(10, 2) NOT NULL,
    water_unit_price DECIMAL(10, 2) NOT NULL,
    water_pricing_type VARCHAR(20) CHECK (water_pricing_type IN ('PER_M3', 'PER_PERSON')),
    service_fee_per_person DECIMAL(10, 2) DEFAULT 0,
    -- Phí rác, thang máy, vệ sinh chung/người
    flood_risk_score SMALLINT DEFAULT 0 CHECK (flood_risk_score BETWEEN 0 AND 5),
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
```

### SRS role descriptions

> Landlord Owner (Chủ đầu tư / Chủ nhà chính): Toàn quyền quản lý tòa nhà, cấu hình tài khoản ngân hàng nhận tiền VietQR (chỉ Owner mới sửa được thông tin nhận tiền để chống nhân viên gian lận), phê duyệt hợp đồng và xem báo cáo tài chính.

> Property Manager (Quản lý vận hành / Bảo vệ được ủy quyền): Tài khoản phụ do Chủ nhà cấp cho từng tòa nhà. Chỉ có quyền đi chụp ảnh công tơ chốt điện nước, dẫn khách xem phòng, lập biên bản Check-in/Check-out và xử lý Ticket sửa chữa. Không có quyền đổi số tài khoản nhận tiền.

> Primary Tenant (Người thuê đại diện) & Co-Tenant (Thành viên ở ghép): Primary Tenant là người đại diện ký hợp đồng và chịu trách nhiệm pháp lý chính về tiền cọc. Co-Tenant là các thành viên ở cùng phòng (đã khai báo CCCD đăng ký tạm trú), được quyền xem hóa đơn, quét QR thanh toán, tạo Ticket sửa chữa và viết đánh giá xác thực.

### SRS RBAC matrix

| Mã CN | Chức năng Nghiệp vụ | Guest | Tenant & Co-Tenant | Property Manager | Landlord Owner | Admin |
|---|---|---|---|---|---|---|
| MOD-01 | Tìm kiếm bản đồ, lọc PCCC & Giá thực tế | Xem | Xem | Xem | Xem | Toàn quyền |
| MOD-02 | Cấu hình Tài khoản Ngân hàng nhận tiền VietQR | - | - | Chỉ xem | Tạo/Sửa (2FA) | Giám sát |
| MOD-03 | AI Quét Hợp đồng & Ký Hợp đồng Điện tử | Quét 1 lần/ngày | Ký (Primary) / Xem | Chuẩn bị HĐ | Ký duyệt HĐ | Cấu hình |
| MOD-04 | Khai báo Thành viên Ở ghép & Xuất Mẫu tạm trú CT01 | - | Khai báo CCCD | Tải file CT01 | Tải file CT01 | - |
| MOD-05 | Biên bản Bàn giao (Check-in) & Quyết toán Cọc (Check-out) | - | Tạo / Ký xác nhận | Kiểm tra thực tế | Duyệt hoàn cọc | Phân xử |
| MOD-06 | Chốt Điện/Nước AI OCR & Phát hành Hóa đơn | - | Xem / Khiếu nại | Chụp ảnh OCR | Duyệt & Phát hành | Đối soát |
| MOD-07 | Ticket Sửa chữa & Duyệt Khấu trừ vào Hóa đơn | - | Tạo Ticket / Ứng tiền | Xử lý kỹ thuật | Duyệt trừ tiền phòng | Giám sát |
| MOD-08 | Viết Review xác thực & Phản hồi Tranh chấp 2 chiều | Chỉ xem | Viết Review | - | Phản hồi chính thức | Kiểm duyệt |

### SADD JWT and RBAC details

> Hệ thống sử dụng Spring Security 6 kết hợp Stateless JWT Access Token (hiệu lực 15 phút) và Refresh Token Rotation lưu trên Redis (hiệu lực 7 ngày).

> ROLE_TENANT (member_role = PRIMARY trong lease_members)

> ROLE_TENANT (member_role = CO_TENANT trong lease_members)

> ROLE_PROPERTY_MANAGER (properties.manager_id == userId)

> ROLE_LANDLORD_OWNER (properties.owner_id == userId)

> ROLE_INSPECTOR ROLE_PARTNER ROLE_ADMIN

### SADD resource rules

> Property Manager: Chỉ được thao tác trên đúng các tòa nhà mà mình được gán làm manager_id. Được quyền chụp ảnh công tơ điện/nước bằng AI OCR, hỗ trợ kiểm tra thực tế khi Check-in/Check-out, xử lý kỹ thuật Ticket sửa chữa và tải file tạm trú Mẫu CT01 (nếu được Owner cấp phép). BỊ CHẶN TUYỆT ĐỐI (403 Forbidden): Không được phép sửa số tài khoản ngân hàng nhận tiền VietQR (UAT-05), không được duyệt khấu trừ tiền phòng và không xem được báo cáo tổng doanh thu.

> Landlord Owner: Toàn quyền quản lý tòa nhà, phân quyền cho Property Manager, ký duyệt hợp đồng, duyệt khấu trừ tiền sửa chữa vào hóa đơn (UAT-04), duyệt phát hành hóa đơn và phản hồi tranh chấp Review 2 chiều. Khi thay đổi cấu hình Tài khoản Ngân hàng nhận tiền VietQR (MOD-02 & UAT-05), bắt buộc phải vượt qua bước xác thực bổ sung Step-Up 2FA OTP (@RequireStepUpOtp).

### SADD bank endpoint code example

```java
@RestController
@RequestMapping("/api/v2/properties/{propertyId}/bank-account")
@RequiredArgsConstructor
public class LandlordBankConfigController {
    private final BankAccountConfigUseCase bankAccountUseCase;

    @PutMapping
    @PreAuthorize("hasRole('LANDLORD_OWNER') and @resourceGuard.isPropertyOwner(#propertyId, authentication)")
    @RequireStepUpOtp(action = "UPDATE_VIETQR_BANK_ACCOUNT")
    // Bắt buộc kèm header X-StepUp-Otp hợp lệ
    public ResponseEntity<BankConfigResponse> updateVietQrBankAccount(
            @PathVariable UUID propertyId,
            @Valid @RequestBody UpdateBankAccountRequest request) {
        // Nếu PROPERTY_MANAGER gọi vào đây -> Spring Security chặn ngay lập tức với HTTP 403 Forbidden
        return ResponseEntity.ok(bankAccountUseCase.updateBankAccount(propertyId, request));
    }
}
```

The SADD module list names `UpdateBankConfigUseCase`, while the code sample uses `BankAccountConfigUseCase`. The implementation MUST choose one canonical application port name and keep controller/tests consistent. Prefer `UpdateBankConfigUseCase` from the directory structure and expose an `updateBankAccount(...)` method.

### UAT-05 verbatim Gherkin

```gherkin
UAT-05: Phân quyền Bảo mật: Chặn Quản lý (Property Manager) đổi tài khoản nhận tiền

GIVEN: Người dùng đăng nhập bằng tài khoản `PROPERTY_MANAGER` được Chủ nhà ủy quyền quản lý tòa nhà A.
WHEN: Người dùng truy cập màn hình 'Cấu hình Tài khoản Ngân hàng VietQR' và thử thay đổi số tài khoản nhận tiền.
THEN: Hệ thống trả về mã lỗi `403 Forbidden`, ẩn nút chỉnh sửa và chỉ cho phép tài khoản `LANDLORD_OWNER` (kèm xác thực OTP 2FA) thay đổi thông tin ngân hàng.
```

## Derived implementation contract

This section is normative but not verbatim.

- Access token claims MUST include immutable subject/token identifiers and current roles only. Resource authorization queries authoritative property/lease membership.
- Refresh tokens MUST be opaque random values; store only a digest and family/parent metadata. A reused rotated token revokes the family.
- OTP MUST be cryptographically random, short-lived, attempt-limited, one-time and stored as a digest. Success proof MUST not be a reusable boolean in the client.
- A bank account entity/table is not specified in the source DDL. TASK-3.2 MAY introduce a dedicated migration with property FK, encrypted/masked account fields, version/audit columns and no plaintext secret logs. The exact schema must be documented in the test and must not overload `properties` with unstructured JSON.
- API errors use 401 for absent/invalid authentication and 403 for authenticated but unauthorized/failed step-up. Do not return 404/403 inconsistently in a way that leaks resource existence.

## Formula and JSON inventory

- Phase 3 owns no business monetary formula or LLM JSON schema.
- Token/OTP TTLs and RBAC facts above are exact normative values.

## Verification gate

- TASK-3.1: JWT expiry/signature/tamper, exact six roles, refresh rotation, reuse family revocation, TTL, parallel refresh winner, log redaction.
- TASK-3.2: Owner same property + valid OTP succeeds; Owner wrong property, Manager same/other property, Tenant, expired/wrong/reused OTP all fail; attempts/rate limit/audit are tested.
- UAT-05 MUST be an integration/security test asserting literal HTTP 403 for Property Manager.
- `mvn test` MUST pass without disabled security or mocked ownership.
