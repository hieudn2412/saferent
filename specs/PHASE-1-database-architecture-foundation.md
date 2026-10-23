# PHASE 1 Database and Architecture Foundation

## Reading boundary

This file is the complete implementation spec for TASK-1.1 through TASK-1.3. During a Phase 1 task, read only this file plus the applicable `AGENTS.md` files named in `IMPLEMENTATION_PLAN.md`. Do not read another Phase spec or the source DOCX files.

## Phase outcomes

- A Java 21 / Spring Boot 3.3 Maven application exists under `saferent-vn-backend`.
- Production configuration enables virtual threads, disables OSIV, uses Hikari max 20 and Hibernate batch 30.
- Flyway creates the PostgreSQL 16 + PostGIS baseline and concurrency constraints.
- ArchUnit enforces the Modular Monolith and Clean Architecture boundaries before feature work begins.

## Atomic task contract

### TASK-1.1 Bootstrap Maven and runtime configuration

MUST provide:

- Java release 21 and Spring Boot 3.3.x.
- Spring Web, Validation, Security, Data JPA, AOP, Cache, Retry and Actuator.
- PostgreSQL driver, Flyway PostgreSQL support, Hibernate Spatial and PostGIS-compatible tests.
- Spring Data Redis/Lettuce, AWS SDK Java v2 S3 + URL Connection/Apache client as required, Apache POI OOXML, ShedLock Spring + chosen provider, Thymeleaf, ArchUnit JUnit 5 and Testcontainers PostgreSQL/JUnit/Redis strategy.
- `SafeRentApplication` at `vn.saferent`.
- `application.yml` with no committed secret.

Required configuration semantics:

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
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        jdbc:
          batch_size: 30
        order_inserts: true
        order_updates: true
```

### TASK-1.2 Flyway V1 and V2

MUST create:

- `src/main/resources/db/migration/V1__init_saferent_v2_postgis_schema.sql`.
- `src/main/resources/db/migration/V2__add_concurrency_locks_and_shedlock.sql`.
- A PostgreSQL 16/PostGIS migration integration test. H2 is forbidden.

V1 MUST preserve all verbatim SRS columns/checks below. V2 MUST preserve the verbatim SADD concurrency DDL below. Because the source DDL excerpt omits complete `handover_records`, `verified_reviews`, and `shedlock` definitions, use the explicit derived contract in “Source gaps resolved for implementation”.

### TASK-1.3 ArchUnit

MUST create the package skeleton and `ArchitectureModularityTest.java` with rules that:

- only allow module packages `identity`, `discovery`, `smartlease`, `operations`, `community`;
- require each module to use `domain`, `application`, `infrastructure`, `presentation`;
- prevent `domain` from importing Spring, Jakarta Persistence, Hibernate, web, AWS, Redis, Resend, OCR or LLM SDKs;
- enforce `presentation -> application -> domain` and `infrastructure -> application/domain ports`;
- prevent one module from importing another module's `infrastructure` or `presentation`;
- prevent controllers from exposing classes in an infrastructure/persistence package;
- keep `vn.saferent.common` technical, not a bypass for business dependencies.

## Verbatim source excerpts

The following excerpts are transcribed from the supplied SRS v2.0 and SADD v2.0. Whitespace is normalized for executable Markdown code blocks; identifiers, values and rules are unchanged.

### SADD runtime and architecture baseline

> Java 21 LTS (Virtual Threads) | Spring Boot 3.3+ | Spring Security 6 | PostgreSQL 16 + PostGIS 3.4 | HikariCP | Redis 7

> Modular Monolith kết hợp Hexagonal / Clean Architecture (Spring Modulith Ready)

> Dự án được tổ chức theo nguyên tắc Package-by-Module ở cấp ngoài cùng (chia đúng theo các phân hệ nghiệp vụ của SRS v2.0) và áp dụng Clean / Hexagonal Architecture (4 lớp: domain, application, infrastructure, presentation) ở bên trong từng module.

### SADD Hikari and Hibernate requirements

> Kích thước Connection Pool: Với máy chủ Database 4 vCPU chạy ổ cứng NVMe SSD, cấu hình maximum-pool-size = 20, minimum-idle = 5, connection-timeout = 10000ms, max-lifetime = 1800000ms.

> Tắt Open-Session-In-View (spring.jpa.open-in-view = false): Đây là nguyên tắc sống còn. Nếu để mặc định bằng true, kết nối Database sẽ bị giữ suốt thời gian chờ gọi AI LLM hoặc chờ render template, làm cạn kiệt 20 kết nối chỉ sau vài giây.

> Bật Batch Insert/Update trong Hibernate: Cấu hình hibernate.jdbc.batch_size = 30 và hibernate.order_inserts = true để khi chủ nhà bấm 'Phát hành hóa đơn tất cả 50 phòng' vào cuối tháng, Hibernate gom thành 2 gói batch thay vì bắn 50 câu INSERT rời rạc.

### SRS V1 baseline DDL

```sql
-- Kích hoạt PostGIS cho truy vấn không gian địa lý
CREATE EXTENSION IF NOT EXISTS postgis;

-- 1. Bảng Người dùng (Tuân thủ Nghị định 13/2023/NĐ-CP & Hỗ trợ Xuất Mẫu CT01)
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    full_name VARCHAR(150) NOT NULL,
    date_of_birth DATE,
    phone_number VARCHAR(20) UNIQUE NOT NULL,
    email VARCHAR(150) UNIQUE,
    role VARCHAR(30) NOT NULL CHECK (role IN ('TENANT', 'LANDLORD_OWNER', 'PROPERTY_MANAGER', 'INSPECTOR', 'PARTNER', 'ADMIN')),
    cccd_encrypted TEXT,
    -- Mã hóa AES-256-GCM (Số CCCD + Địa chỉ thường trú phục vụ xuất CT01)
    cccd_hash VARCHAR(64) UNIQUE,
    -- SHA-256 chống trùng lặp
    ekyc_verified BOOLEAN DEFAULT FALSE,
    reputation_points INT DEFAULT 0,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- 2. Bảng Tòa nhà / Khu trọ (Properties - Bổ sung Quản lý Ủy quyền & Cấu hình Công tơ)
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

CREATE INDEX idx_properties_location ON properties USING GIST (location);

-- 3. Bảng Phòng trọ & Thiết bị (Rooms)
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
    -- Mốc quay vòng công tơ (VD: 99999 -> 0)
    status VARCHAR(30) DEFAULT 'AVAILABLE' CHECK (status IN ('AVAILABLE', 'OCCUPIED', 'MAINTENANCE'))
);

-- 4. Bảng Hợp đồng Thuê Điện tử (Leases - Hỗ trợ Ở ghép & Báo trước Chấm dứt HĐ)
CREATE TABLE leases (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    room_id UUID NOT NULL REFERENCES rooms(id),
    primary_tenant_id UUID NOT NULL REFERENCES users(id),
    landlord_id UUID NOT NULL REFERENCES users(id),
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,
    occupant_count SMALLINT DEFAULT 1 NOT NULL,
    -- Số người thực tế đang ở để tính phí/định mức điện
    notice_period_days SMALLINT DEFAULT 30,
    -- Số ngày phải báo trước khi nghỉ thuê
    termination_requested_at TIMESTAMPTZ,
    monthly_rent DECIMAL(12, 2) NOT NULL,
    deposit_amount DECIMAL(12, 2) NOT NULL,
    electricity_rate_committed DECIMAL(10, 2) NOT NULL,
    credit_balance DECIMAL(12, 2) DEFAULT 0,
    -- Số dư tiền thừa kết chuyển
    contract_pdf_sha256 VARCHAR(64),
    -- Mã băm bảo toàn tính toàn vẹn của Hợp đồng đã ký
    status VARCHAR(30) DEFAULT 'DRAFT' CHECK (status IN ('DRAFT', 'ACTIVE', 'PENDING_CHECKOUT', 'TERMINATED', 'EXPIRED')),
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- 5. [MỚI v2.0] Bảng Thành viên Ở ghép & Đăng ký Tạm trú (Lease Members)
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

-- 6. [MỚI v2.0] Bảng Ticket Sửa chữa & Ứng tiền Khấu trừ (Maintenance Tickets)
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
    -- Tiền người thuê ứng trước trả thợ
    is_deduction_approved BOOLEAN DEFAULT FALSE,
    -- Chủ nhà duyệt cho trừ vào hóa đơn tháng tới
    status VARCHAR(30) DEFAULT 'OPEN' CHECK (status IN ('OPEN', 'IN_PROGRESS', 'RESOLVED', 'REJECTED')),
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- 7. Bảng Hóa đơn Hàng tháng (Utility Invoices - Hỗ trợ Quay vòng Công tơ, Khấu trừ & Trả góp)
CREATE TABLE utility_invoices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    lease_id UUID NOT NULL REFERENCES leases(id),
    billing_month VARCHAR(7) NOT NULL,
    -- YYYY-MM
    elec_old_index DECIMAL(10, 2) NOT NULL,
    elec_new_index DECIMAL(10, 2) NOT NULL,
    is_meter_rollover BOOLEAN DEFAULT FALSE,
    elec_meter_image_url TEXT NOT NULL,
    water_old_index DECIMAL(10, 2) NOT NULL,
    water_new_index DECIMAL(10, 2) NOT NULL,
    previous_debt DECIMAL(12, 2) DEFAULT 0,
    -- Nợ cũ kỳ trước chuyển sang
    maintenance_deduction DECIMAL(12, 2) DEFAULT 0,
    -- Tiền sửa chữa được khấu trừ
    total_amount DECIMAL(12, 2) NOT NULL,
    paid_amount DECIMAL(12, 2) DEFAULT 0,
    vietqr_payload TEXT,
    payment_status VARCHAR(20) DEFAULT 'UNPAID' CHECK (payment_status IN ('UNPAID', 'PARTIALLY_PAID', 'PAID', 'OVERDUE')),
    paid_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
```

### SADD V2 concurrency DDL

```sql
-- [1. BỔ SUNG RÀNG BUỘC CONCURRENCY VÀO SCHEMA POSTGRESQL v2.0]
ALTER TABLE rooms ADD COLUMN version BIGINT NOT NULL DEFAULT 0;
ALTER TABLE leases ADD COLUMN version BIGINT NOT NULL DEFAULT 0;
ALTER TABLE handover_records ADD COLUMN version BIGINT NOT NULL DEFAULT 0;
ALTER TABLE utility_invoices ADD COLUMN version BIGINT NOT NULL DEFAULT 0;

-- Chặn tuyệt đối Double Booking: 1 phòng chỉ có tối đa 1 hợp đồng đang ACTIVE hoặc PENDING_CHECKOUT
CREATE UNIQUE INDEX uq_active_lease_per_room
ON leases (room_id)
WHERE status IN ('ACTIVE', 'PENDING_CHECKOUT');

-- Chặn tạo trùng 2 hóa đơn cho cùng 1 hợp đồng trong 1 tháng
ALTER TABLE utility_invoices
ADD CONSTRAINT uq_invoice_lease_month UNIQUE (lease_id, billing_month);

-- Liên kết chặt chẽ Ticket sửa chữa đã khấu trừ vào đúng hóa đơn để chống trừ đúp 2 lần
ALTER TABLE maintenance_tickets
ADD COLUMN deducted_in_invoice_id UUID REFERENCES utility_invoices(id);
```

### SADD directory and test names

> `V1__init_saferent_v2_postgis_schema.sql` # Bảng core từ SRS v2.0 (users, properties, rooms, leases...)

> `V2__add_concurrency_locks_and_shedlock.sql` # Cột @Version, Partial Unique Index, bảng shedlock

> `ArchitectureModularityTest.java` # ArchUnit đảm bảo các module không gọi chéo tầng Infrastructure

> `domain/` # Pure Unit Tests (<1s) cho công thức True Cost, Rollover, Bậc thang điện

> `uat/` # Integration Tests (Testcontainers PostGIS + Redis) cho UAT-01 -> UAT-06

## Source gaps resolved for implementation

This section is normative for implementation but is **not a verbatim quotation**.

1. The SRS data dictionary mentions `handover_records` and `settlement_summary JSONB`, while the displayed SRS DDL omits the table. V1 MUST create a minimally sufficient `handover_records` table before V2 adds `version`. It MUST contain UUID primary key, `lease_id` FK, check-in/check-out type, lifecycle status including `LOCKED`, evidence manifest JSONB/object references and hashes, server timestamp, 48-hour deadline, meter baselines, GPS evidence, `settlement_summary JSONB`, actor/audit timestamps. Exact names MUST be frozen in the migration test and reused in Phase 5.
2. SADD concurrency requires `verified_reviews.version` even though its sample V2 DDL omits that line. The table may be introduced in a later Phase 7 migration, but that same migration MUST include `version BIGINT NOT NULL DEFAULT 0`. V2 MUST NOT reference a table that does not yet exist.
3. `V2__add_concurrency_locks_and_shedlock.sql` MUST create the ShedLock provider table using the official ShedLock JDBC schema for PostgreSQL, with primary key `name` and non-null `lock_until`, `locked_at`, `locked_by`. Timestamp types MUST preserve correct instant semantics.
4. Add indexes for every FK and hot lookup: property owner/manager/location, room property, lease room/status/tenants, active lease members, invoice lease/month/status, approved-undeducted tickets, handover deadline/status.
5. Add safe data checks omitted by the excerpt where they do not change behavior: non-negative money and meter indices, `occupant_count >= 1`, `max_occupants >= 1`, valid `billing_month` format, end date not before start date. Any check that could reject existing production data requires a staged migration.
6. `elec_meter_image_url` is a legacy name. Persist an R2 object key, never a Presigned URL. Renaming requires an approved migration; semantics are authoritative even if the source column name remains.

## Formula JSON and UAT inventory

- Phase 1 defines no business formula or LLM JSON schema.
- Phase 1 defines no standalone Gherkin UAT. Its schema/locking constraints are prerequisites for UAT-01 through UAT-06 and MUST be exercised by migration tests now and feature tests in later phases.

## Verification gate

- `mvn test` MUST compile with Java 21.
- Migration test MUST start clean PostgreSQL 16/PostGIS, apply V1 then V2, and assert tables, columns, checks, GIST index, partial unique index, unique invoice constraint, FK and ShedLock schema.
- A second migration validation run MUST report no pending/failed migration.
- ArchUnit tests MUST pass against production classes and MUST include negative fixture checks or equivalent proof that each core rule detects a violation.
- No application context may enable OSIV or schema auto-update.
