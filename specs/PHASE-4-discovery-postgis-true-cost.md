# PHASE 4 Discovery PostGIS and True Cost

## Reading boundary

This file is the complete implementation spec for TASK-4.1 and TASK-4.2. Read only this file plus root, backend and discovery `AGENTS.md` files named in the plan.

## Phase outcomes

- Pure domain calculation returns transparent True Monthly Cost for occupants N.
- PostGIS radius search uses `ST_DWithin` and the GIST index.
- Redis Geohash6 and property-detail caches meet exact keys/TTLs.
- Fire-safety verification uses the legal three-level wording and six-month expiry.

## Atomic task contract

### TASK-4.1 TrueCostPolicy

Required names from SADD:

- `TrueCostPolicy` (directory structure) / `TrueMonthlyCostCalculator` (service list).
- Domain value types for money, occupants, AEF and pricing breakdown.

Use `TrueCostPolicy` as the pure domain policy name. An application facade MAY expose calculator semantics. The policy MUST have zero framework imports and use `BigDecimal` only.

### TASK-4.2 spatial search cache and fire badge

Required names:

- `SpatialPropertySearchService`
- `FireSafetyBadgeService`
- `FireBadgeExpiryJob`
- `SpatialSearchUseCase`
- `VerifyFireBadgeUseCase`
- `PostGisPropertyRepository`
- `RedisGeoSearchCacheAdapter`
- `PropertyDiscoveryController`
- `GET /api/v2/properties/search`

Search MUST accept `occupants=N`, radius and filters, use `ST_DWithin`, return `estimated_true_monthly_cost` with assumptions, and enforce 60 requests/minute/IP.

## Verbatim source excerpts

### SRS True Monthly Cost formula

> QUY TẮC NGHIỆP VỤ CẬP NHẬT (BR-1.1 v2.0: CÔNG THỨC TÍNH TRUE MONTHLY COST THEO NHÂN KHẨU)

> Khi người dùng chọn bộ lọc 'Số người ở dự kiến = N' (mặc định N = 2 đối với sinh viên ở ghép), hệ thống tính toán Chi phí Thực tế Ước tính/tháng cho từng phòng như sau:

```text
True Cost = Giá Thuê Cơ Bản
          + Tiền Điện Ước Tính
          + Tiền Nước (theo N người hoặc m³)
          + Phí Dịch Vụ (theo N người hoặc theo phòng).

Tiền Điện Ước Tính
    = (Định mức cơ bản 80 kWh + 40 kWh × N người)
    × Hệ số tiêu hao điều hòa (AEF)
    × Đơn giá điện
```

> AEF = 1.00: Điều hòa Inverter đời mới (< 3 năm) hoặc nhãn năng lượng 5 sao.

> AEF = 1.20: Điều hòa tiêu chuẩn (Non-Inverter).

> AEF = 1.40: Điều hòa đời cũ bị >= 2 Verified Tenants phản ánh 'Ngốn điện' hoặc có Ticket báo hỏng quá hạn.

> Hiển thị Minh bạch trên Thẻ Tin đăng: Hiển thị rõ 'Giá niêm yết: 3.500.000đ | Chi phí thực tế ước tính (2 người): ~4.280.000đ/tháng (chia 2 = 2.140.000đ/người)'.

### SRS PCCC three levels and expiry

> Cấp 0 (Chưa xác thực - Màu Xám): Chủ nhà chưa tải lên bằng chứng trang bị an toàn PCCC.

> Cấp 1 (Chủ nhà Tự kê khai qua Camera trực tiếp - Màu Xanh Dương): Chủ nhà chụp ảnh trực tiếp trên Web App (chặn upload ảnh cũ trong thư viện) đủ 4 hạng mục: Lối thoát hiểm thứ 2, vách/cửa ngăn cháy khu để xe tầng 1, bình chữa cháy, chuông báo khói.

> Cấp 2 (Đã Xác thực Cộng đồng / Thực địa - Huy hiệu Xanh Lá Vàng): Được tối thiểu 03 Verified Tenants xác nhận thực tế HOẶC Kiểm định viên SafeRent VN kiểm tra thực địa.

> Quy tắc Mới trong v2.0 (BR-1.2 - Hạn sử dụng Huy hiệu 6 tháng): Bình chữa cháy có thể hết hạn hoặc lối thoát hiểm có thể bị chủ nhà tận dụng chất đồ đạc gây cản trở sau một thời gian. Do đó, cứ mỗi 06 tháng, hệ thống yêu cầu Chủ nhà chụp cập nhật lại ảnh lối thoát hiểm & tem bình chữa cháy, hoặc lấy xác nhận nhanh từ người thuê đang ở để duy trì Huy hiệu Cấp 2.

### SRS legal label

> Sử dụng thuật ngữ chuẩn: 'Xác thực Hiện trạng Trang bị An toàn (Safety Equipment Verified)'. Phân làm 3 cấp (Chưa kê khai / Chủ nhà chụp ảnh Camera trực tiếp / Xác thực bởi >= 3 Người thuê thực tế hoặc Kiểm định viên).

### SRS relevant DDL

```sql
CREATE TABLE properties (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    owner_id UUID NOT NULL REFERENCES users(id),
    manager_id UUID REFERENCES users(id),
    name VARCHAR(255) NOT NULL,
    address_detail TEXT NOT NULL,
    ward_code VARCHAR(20),
    district_code VARCHAR(20) NOT NULL,
    city_code VARCHAR(20) NOT NULL,
    location GEOGRAPHY(Point, 4326) NOT NULL,
    fire_safety_level SMALLINT DEFAULT 0 CHECK (fire_safety_level IN (0, 1, 2)),
    fire_badge_expires_at TIMESTAMPTZ,
    has_second_exit BOOLEAN DEFAULT FALSE,
    has_fire_door BOOLEAN DEFAULT FALSE,
    has_smoke_detector BOOLEAN DEFAULT FALSE,
    electricity_pricing_type VARCHAR(30) CHECK (electricity_pricing_type IN ('STATE_TIERED_BY_CAPITA', 'FIXED_RATE')),
    electricity_unit_price DECIMAL(10, 2) NOT NULL,
    water_unit_price DECIMAL(10, 2) NOT NULL,
    water_pricing_type VARCHAR(20) CHECK (water_pricing_type IN ('PER_M3', 'PER_PERSON')),
    service_fee_per_person DECIMAL(10, 2) DEFAULT 0,
    flood_risk_score SMALLINT DEFAULT 0 CHECK (flood_risk_score BETWEEN 0 AND 5),
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_properties_location ON properties USING GIST (location);

CREATE TABLE rooms (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id UUID NOT NULL REFERENCES properties(id) ON DELETE CASCADE,
    room_number VARCHAR(50) NOT NULL,
    floor_number INT NOT NULL,
    area_sqm DECIMAL(5, 2),
    max_occupants SMALLINT DEFAULT 3,
    base_rent_price DECIMAL(12, 2) NOT NULL,
    ac_condition_type VARCHAR(30) DEFAULT 'STANDARD' CHECK (ac_condition_type IN ('INVERTER_NEW', 'STANDARD', 'OLD_HIGH_CONSUMPTION', 'NONE')),
    electric_meter_serial VARCHAR(100),
    electric_meter_max_dial INT DEFAULT 100000,
    status VARCHAR(30) DEFAULT 'AVAILABLE' CHECK (status IN ('AVAILABLE', 'OCCUPIED', 'MAINTENANCE'))
);
```

### SRS API contract

| Method & Endpoint | Phân hệ | Rate Limit | Mô tả |
|---|---|---|---|
| `GET /api/v2/properties/search` | Discovery | 60 req/phút/IP | Tìm kiếm PostGIS theo bán kính, nhận tham số `occupants=N` để tính chính xác `estimated_true_monthly_cost` theo đầu người. |

### SADD service names and behavior

> `SpatialPropertySearchService` - `TrueMonthlyCostCalculator` - `FireSafetyBadgeService` - `FireBadgeExpiryJob`

> Tìm kiếm phòng theo bán kính địa lý PostGIS (ST_DWithin), lọc rủi ro ngập lụt và cấp độ PCCC.

> Tính toán động True Monthly Cost theo số lượng người ở dự kiến (Occupants = N) và hệ số tiêu hao điều hòa (AEF = 1.0, 1.2, 1.4) (BR-1.1 v2.0).

> Quản lý quy trình cấp huy hiệu PCCC 3 cấp và chạy Cron Job kiểm tra hạn hiệu lực định kỳ 6 tháng (BR-1.2).

### SADD cache keys TTLs and eviction

```text
L2 Geo Spatial Search Cache (Redis 7)
Key: geo:v2:{city}:{geohash6}:{radius}:{filterHash}
TTL: 5 phút
```

> Quy đổi tọa độ tìm kiếm (lat, lng) của người dùng về ô lưới địa lý Geohash cấp 6 (~1.2km x 0.6km) để tăng tỷ lệ Cache Hit lên > 85% cho các khu vực nóng quanh các trường Đại học (Cầu Giấy, Hòa Lạc, Thủ Đức, Bình Thạnh).

> Redis lưu danh sách thông số cơ sở của phòng (baseRent, elecPrice, acAef, waterPrice, waterType, serviceFeePerPerson). Khi người dùng đổi bộ lọc 'Số người ở dự kiến = N' (1, 2, 3, 4 người), Backend lấy danh sách từ Redis trong 2ms và tính toán lại công thức True Cost ngay trên RAM mà không cần truy vấn lại PostGIS.

```text
L2 Property Detail & Fire Badge Cache (Redis 7)
Key: prop:v2:detail:{propertyId}
TTL: 30 phút
```

> Cơ chế Eviction (@CacheEvict): Xóa cache ngay lập tức khi Chủ nhà cập nhật thông tin phòng, khi có hợp đồng mới được ký (phòng chuyển sang OCCUPIED), khi Review mới được công khai sau 48h, hoặc khi Ticket báo hỏng điều hòa quá hạn 7 ngày làm hệ số tiêu hao AEF tăng lên 1.40.

```text
L1 In-Memory Reference Cache (Caffeine)
TTL: 12 giờ
```

> Cache biểu giá bậc thang điện sinh hoạt EVN hiện hành (theo Thông tư 09/2023/TT-BCT) và danh mục mã hành chính Tỉnh/Thành, Quận/Huyện, Phường/Xã.

## Derived implementation contract

This section is normative but not verbatim.

- `N` defaults to 2 for discovery and must be a positive integer; supported UI/cache variants are 1–4. Reject nonsensical or capacity-breaking inputs rather than divide by zero.
- Return a calculation breakdown: base rent, estimated kWh, AEF, unit price, water, service fee, total, per-person and assumptions.
- `PER_PERSON` water is `water_unit_price × N`. `PER_M3` requires a documented volume/estimate input; never reinterpret it as per-person.
- `service_fee_per_person` is multiplied by N. A per-room service fee requires a future explicit field; do not overload the column.
- `ac_condition_type = NONE` has no exact source formula. TASK-4.1 MUST model it explicitly and document the chosen no-AC consumption assumption in tests; do not silently map it to another condition.
- PostGIS geography radius is meters, parameters are bound, pagination has deterministic tie-breaker, and public DTOs exclude private details.
- Level 1 evidence uses live capture and direct public R2 upload. Level 2 requires three distinct current verified tenants or assigned Inspector.
- Expiry job must be idempotent and ShedLock-protected; refreshed verification committed before expiry update wins according to version/transaction semantics.

## JSON and UAT inventory

- Phase 4 owns no LLM JSON schema.
- The supplied SRS defines no standalone UAT ID for Discovery/PCCC. Formula and badge rules are mandatory domain acceptance tests even without a numbered UAT.

## Verification gate

- TASK-4.1 reaches 100% branch coverage on the policy and tests N=1..4, all AEF values, water modes, invalid N, scale/rounding and exact breakdown.
- PostgreSQL/PostGIS integration test proves `ST_DWithin`, correct meter units, GIST-backed query plan expectation where stable, filters and deterministic pagination.
- Cache tests prove normalized Geohash6 key, filter hash completeness, exact 5/30-minute TTLs, recomputation when only N changes and all eviction triggers.
- Badge tests cover Levels 0/1/2, two vs three tenants, Inspector path, exactly-six-month boundary and concurrent refresh/expiry.
- `mvn test` MUST pass.
