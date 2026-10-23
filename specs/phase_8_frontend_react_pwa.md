# PHASE 8: Frontend React 18 PWA & Mobile-First UX Specification

## 1. Reading Boundary & Execution Guardrails

This file is the complete, self-contained implementation specification for **`TASK-8.1`** through **`TASK-8.3`**. 
- **Context Isolation Rule:** While executing a Phase 8 task, the AI Coding Agent **MUST** read only this file (`specs/PHASE-8-frontend-pwa.md`), the root `./AGENTS.md`, and `./saferent-vn-frontend/AGENTS.md`. Do **NOT** load the full SRS/SADD source documents or backend phases unless verifying an endpoint path.
- **Verification Gate:** Every task in Phase 8 **MUST** pass `npm run typecheck`, `npm run lint`, `npm run test -- --run`, and `npm run build` before marking the task `[x] DONE` in `IMPLEMENTATION_PLAN.md`.

---

## 2. Phase Outcomes

1. **Dual-Persona Accessible PWA:** React 18 + TypeScript (strict mode) Vite PWA tailored for two distinct user groups:
   - **Landlords & Property Managers (Aged 45–65):** High-contrast, Zero-Typing ("Touch & Shoot") interface inspired by **KiotViet Mobile**, featuring $\ge 16\text{px}-18\text{px}$ tabular numerals for meter readings/money and $\ge 48 \times 48\text{px}$ full-width primary action buttons.
   - **Tenants & Students:** Transparent spatial search and legal/financial clarity inspired by **Airbnb** (Split-screen map + True Cost toggle) and **Stripe Dashboard** (Deposit settlement & SSE contract audit).
2. **Zero-Bandwidth Media Pipeline:** All media files (cropped meter images, PCCC photos, 2-sided CCCD, contract PDFs, Check-in/Check-out handover photos) are processed client-side, hashed with `SHA-256` via Web Crypto API (`crypto.subtle.digest`), and uploaded directly to **Cloudflare R2** using Presigned PUT URLs.
3. **Basement-Proof Offline Sync:** Meter captures taken in zero-signal basements are stored in **Dexie.js (IndexedDB)** and replayed idempotently via Workbox/Background Sync (`Fresh Presign -> Direct R2 PUT -> Backend OCR/Invoice Registration`).
4. **Full UAT-01 to UAT-06 Compliance:** All 6 acceptance scenarios from SRS v2.0 are enforced and testable in the UI.

---

## 3. UI/UX Design System & Styling Mandate (Normative)

All components built in `TASK-8.1` through `TASK-8.3` **MUST** adhere to the following Design Tokens configured in `tailwind.config.ts` and `src/index.css`:

### 3.1. Reference Design Archetypes
- **Discovery & Map (`PropertyDiscoveryMap`, `TrueCostCard`):** Follow **Airbnb** split-screen layout (list on left, interactive MapLibre/Mapbox map on right; floating Map/List toggle on mobile) combined with **Zillow** flood-risk polygon overlays and PCCC safety pins.
- **Smart Lease, Settlement & Invoice (`ContractScanStream`, `DepositSettlementSummary`):** Follow **Stripe Dashboard** financial clarity (crisp borders, `tabular-nums` alignment, clear status pills for `PAID`, `PARTIALLY_PAID`, `UNPAID`, `OVERDUE`) and **MoMo/VietQR** 1-touch shareable receipt cards.
- **Landlord Operations (`MeterCameraCapture`, `MonthlyInvoiceBuilder`, `BankAccountConfigForm`):** Follow **KiotViet Mobile** ergonomics—zero ambiguous icon-only buttons; every button has a bold Vietnamese label + icon, high contrast, and large touch targets.

### 3.2. Semantic Color Tokens (Tailwind CSS Classes)
| Semantic Token | Hex / Tailwind Equivalent | Usage in SafeRent VN v2.0 |
| :--- | :--- | :--- |
| `brand-primary` | `#0F766E` (`teal-700`) | Primary CTA buttons, verified headers, active states |
| `pccc-level-0` | `#64748B` (`slate-500`) | **Cấp 0 (Chưa xác thực - Màu Xám):** Unverified fire safety |
| `pccc-level-1` | `#0284C7` (`sky-600`) | **Cấp 1 (Màu Xanh Dương):** Landlord live-camera self-declaration |
| `pccc-level-2` | `#16A34A` + `#EAB308` ring (`green-600` + `yellow-500`) | **Cấp 2 (Huy hiệu Xanh Lá Vàng):** Verified by $\ge 3$ Tenants or Inspector |
| `risk-red` | Bg `#FEF2F2`, Border `#EF4444`, Text `#991B1B` | **Cảnh báo ĐỎ (RED):** Illegal contract clause / Elec rate $> 4.000\text{đ/kWh}$ |
| `risk-yellow` | Bg `#FEFCE8`, Border `#EAB308`, Text `#854D0E` | **Cảnh báo VÀNG (YELLOW):** Clause needing renegotiation (e.g., CT01 fee) |
| `risk-safe` | Bg `#F0FDF4`, Border `#22C55E`, Text `#166534` | **An toàn (SAFE):** Compliant with Housing Law 2023 & Civil Code 2015 |

### 3.3. Typography & Ergonomic Rules (Mandatory)
- **Fonts:** `Be Vietnam Pro` or `Plus Jakarta Sans` for UI copy; `JetBrains Mono` for meter serial numbers, meter indices, `SHA-256` hashes, and currency amounts.
- **Landlord 45–65 Ergonomics:**
  - Meter indices (`elecOldIndex`, `elecNewIndex`) and currency totals **MUST** use `text-lg` to `text-2xl` (`18px`–`24px`), `font-bold`, and `tabular-nums`.
  - Primary operational buttons (`Chụp công tơ`, `Gửi hóa đơn`, `Xuất mẫu tạm trú CT01`, `Duyệt khấu trừ`) **MUST** enforce `min-h-[48px] px-5 text-base font-semibold`.
  - **NEVER** render text smaller than `14px` (`text-sm`) for financial figures, legal citations, or error banners.

---

## 4. Atomic Task Contracts

### TASK-8.1: PWA Foundation, Golden UI Primitives & Direct Cloudflare R2 Upload
**Scope:**
1. Configure React 18+, TypeScript (`strict: true`), Vite + `vite-plugin-pwa` (Workbox strategies), Tailwind CSS, Shadcn UI primitives, TanStack Query v5, Zustand, Zod runtime validation, and Vitest + React Testing Library.
2. Implement the **Design System Tokens** (`tailwind.config.ts`) and **Golden Base Components** in `src/components/ui/`:
   - `ActionTouchButton`: Enforces `min-h-[48px] min-w-[48px]` with icon + explicit Vietnamese label.
   - `MoneyDisplay`: Formats `number` as Vietnamese currency (`3.500.000đ`) with `font-mono tabular-nums font-bold`.
   - `FireSafetyBadge`: Renders Level 0 (Gray), Level 1 (Blue), or Level 2 (Green-Gold) with 6-month expiry warning (`fireBadgeExpiresAt`).
   - `OfflineSyncBanner`: Displays online/offline status and count of pending items in IndexedDB.
3. Implement the **Zero-Bandwidth Direct R2 Upload Pipeline** in `src/lib/storage/r2UploadClient.ts`:

**Normative Types & Function Signatures (`TASK-8.1`):**
```typescript
export type StorageCategory =
  | 'PROPERTY_PUBLIC_IMAGE'
  | 'PCCC_LIVE_CAMERA'
  | 'CCCD_PRIVATE'
  | 'CONTRACT_DOCUMENT'
  | 'HANDOVER_MEDIA'
  | 'METER_OCR_CROP'
  | 'MAINTENANCE_EVIDENCE'
  | 'REVIEW_EVIDENCE';

export interface PresignRequest {
  category: StorageCategory;
  targetEntityId: string;
  contentType: string;
  contentLength: number;
  sha256Hex: string;
}

export interface PresignedUpload {
  presignedPutUrl: string;
  objectKey: string;
  expiresAt: string;
}

export interface UploadedObjectReceipt {
  objectKey: string;
  sha256Hex: string;
  contentType: string;
  contentLength: number;
}

// 1. Compute lowercase hex SHA-256 using window.crypto.subtle.digest('SHA-256', buffer)
export async function sha256Hex(blob: Blob): Promise<string>;

// 2. Call POST /api/v2/storage/presigned-put (validates size <= 15MB before calling)
export async function requestPresignedPut(req: PresignRequest): Promise<PresignedUpload>;

// 3. Execute direct HTTP PUT binary stream to Cloudflare R2 with headers:
//    'Content-Type': blob.type, 'x-amz-meta-sha256': sha256Hex
export async function uploadDirectToR2(
  presignedPutUrl: string,
  blob: Blob,
  sha256Hex: string
): Promise<void>;

// 4. High-level orchestrator: hash -> requestPresignedPut -> uploadDirectToR2 -> return receipt
export async function registerUploadedObject(
  category: StorageCategory,
  targetEntityId: string,
  blob: Blob
): Promise<UploadedObjectReceipt>;
```

---

### TASK-8.2: Meter Camera Bounding-Box Crop (< 50KB Grayscale) & Dexie Offline Queue
**Scope:**
1. Implement client-side vision utility `cropMeterDisplayToGrayscaleWebp` using HTML5 Canvas API:
   - Accepts raw camera frame/image and normalized bounding box `{ x, y, width, height }` covering only the meter digits + serial number.
   - Applies orientation correction, crops to the bounding box, converts pixels to **luminance grayscale** ($Y = 0.299R + 0.587G + 0.114B$), and encodes to `image/webp` (iteratively stepping down quality from `0.82` to `0.45` if needed) until **`blob.size < 50 * 1024` (50 KB)**.
2. Implement `MeterCameraCapture` component:
   - Live camera viewfinder (`facingMode: 'environment'`) with rectangular **Bounding Box overlay** for meter digits.
   - Local preview showing cropped grayscale image, byte size badge (e.g., `34.2 KB - Đạt chuẩn < 50KB`), extracted/editable `meterSerial` (BR-3.1a anti-swap check), `elecOldIndex`, and `elecNewIndex`.
   - Automatic detection and UI badge for **Meter Rollover (UAT-03)**: When `elecNewIndex < elecOldIndex` and `elecOldIndex >= 0.9 * maxDial` (e.g., `99850` -> `00120` on a `100000` dial), automatically enable the `"Công tơ quay vòng"` toggle and display the rollover formula: `(100000 - 99850) + 120 = 270 kWh`.
3. Implement **Dexie.js Offline Queue (`meterCaptureQueue`) & Background Sync**:
   - When offline in a basement (or when network request fails), store `PendingMeterCapture` in IndexedDB (`Dexie.js`).
   - When connectivity is restored (`window` `online` event or Workbox Background Sync), `replayPendingMeterCapture` **MUST** execute in strict order:
     1. Request a **fresh** Presigned PUT URL (`requestPresignedPut`) because any previously cached presigned URL may have expired after 10 minutes.
     2. Upload cropped grayscale WebP directly to Cloudflare R2 (`uploadDirectToR2`).
     3. Call `POST /api/v2/invoices/ocr-meter` with header `Idempotency-Key: <capture.idempotencyKey>` and `{ objectKey, sha256Hex, roomId, leaseId, billingMonth, elecOldIndex, elecNewIndex, meterSerial }`.

**Normative Types & Function Signatures (`TASK-8.2`):**
```typescript
export interface NormalizedBoundingBox {
  x: number;      // 0..1 relative to source width
  y: number;      // 0..1 relative to source height
  width: number;  // 0..1
  height: number; // 0..1
}

export interface PendingMeterCapture {
  id: string;                 // UUID
  idempotencyKey: string;     // Stable key across offline retries
  propertyId: string;
  roomId: string;
  roomNumber: string;
  leaseId: string;
  billingMonth: string;       // YYYY-MM
  meterSerial: string;
  elecOldIndex: number;
  elecNewIndex: number;
  maxDial: number;            // Default 100000
  isMeterRollover: boolean;
  croppedBlob: Blob;          // Grayscale WebP < 50KB
  sha256Hex: string;
  status: 'QUEUED_OFFLINE' | 'SYNCING' | 'SYNCED' | 'FAILED';
  createdAt: string;
  lastError?: string;
}

export async function cropMeterDisplayToGrayscaleWebp(
  source: ImageBitmap | HTMLImageElement | HTMLVideoElement,
  box: NormalizedBoundingBox,
  maxBytes?: number // Default 50 * 1024 (50 KB)
): Promise<Blob>;

export async function replayPendingMeterCapture(
  captureId: string
): Promise<void>;
```

---

### TASK-8.3: PostGIS Map + True Cost, AI Contract SSE, Handover Comparison, Reviews & UAT-01..UAT-06 Flows
**Scope:**
1. **Discovery Map & True Cost Calculator (`PropertyDiscoveryMap`, `TrueCostCard`):**
   - Interactive MapLibre GL / Mapbox GL map with radius search (`GET /api/v2/properties/search?lat=...&lng=...&radiusMeters=...&occupants=N`), Flood Risk Score (`0–5`) badge/overlay, and Fire Safety Level (`0, 1, 2`) pins.
   - Accessible list view synchronized with map bounds and occupant selector (`occupants = 1..4`, default `2`).
   - `TrueCostCard` **MUST** compute and display side-by-side:
     $$\text{Estimated Electricity} = (80 + 40 \times N) \times \text{AEF} \times \text{electricityUnitPrice}$$
     $$\text{True Monthly Cost} = \text{baseRentPrice} + \text{Estimated Electricity} + \text{Water Cost}(N) + \text{Service Fee}(N)$$
     Where $\text{AEF} \in \{1.00, 1.20, 1.40\}$.
   - Exact required copy pattern:
     `Giá niêm yết: 3.500.000đ | Chi phí thực tế ước tính (2 người): ~4.280.000đ/tháng (chia 2 = 2.140.000đ/người)`.
2. **AI Contract Scanner via SSE (`ContractScanStream`):**
   - Streams real-time analysis events over `EventSource` (or fetch SSE stream) from `/api/v2/contracts/ai-scan`.
   - Renders overall `safety_score` (`0–100`), `summary`, and progressive `risk_clauses` cards color-coded by `severity` (`RED` | `YELLOW`), showing `clause_excerpt`, `legal_basis` (Luật Nhà ở 2023, Bộ luật Dân sự 2015, TT 09/2023/TT-BCT), `explanation`, and a copyable `suggested_revision`.
3. **Co-Living Members & 1-Click CT01 Export (`CoLivingMembersPanel` - UAT-02):**
   - Allows Primary Tenant to add Co-Tenants with eKYC CCCD upload (encrypted/hashed).
   - Dynamically updates `occupant_count` from `1` to `3`, recalculates per-person water/service fees (`100.000đ × 3 = 300.000đ`), and provides the **`Xuất Tờ khai Tạm trú (Mẫu CT01)`** button (`GET /api/v2/leases/{id}/export-ct01`) downloading `Mau_CT01_<roomNumber>.docx`.
4. **Digital Check-in/Check-out & Deposit Settlement (`HandoverComparison`, `DepositSettlementSummary` - UAT-06):**
   - Side-by-side visual comparison slider using `react-compare-image` between locked Check-in image (`SHA-256` verified timestamp) and current Check-out image.
   - Mandatory damage attribution radio selector: `NATURAL_WEAR` (*Hao mòn tự nhiên - Chủ nhà chịu*, 0đ deduction) vs `TENANT_FAULT` (*Hư hỏng do lỗi người thuê*, requires photo evidence + deduction amount).
   - `DepositSettlementSummary` calculates:
     $$\text{Refund Amount} = \text{depositAmount} - (\text{proRatedRent} + \text{finalUtilities} + \text{tenantFaultDeductions})$$
     Displays the tenant's VietQR code for instant refund within the $\le 7\text{-day}$ SLA.
5. **Landlord Billing, Ticket Deduction & RBAC Guards (`MonthlyInvoiceBuilder`, `BankAccountConfigForm` - UAT-01, UAT-04, UAT-05):**
   - **UAT-01 Guard:** In `MonthlyInvoiceBuilder`, if the user edits `electricityUnitPrice` above `lease.electricityRateCommitted` (e.g., entering `4.200đ/kWh` when contract committed `3.500đ/kWh`), immediately block submission, show red validation error:
     `"Đơn giá điện không được vượt mức 3.500đ/kWh đã cam kết trong Hợp đồng"`, and offer a 1-tap reset to `3.500đ/kWh`.
   - **UAT-04 Offset:** Automatically pull approved tenant-advanced maintenance tickets (`is_deduction_approved = true`, e.g., `250.000đ` plumbing repair) into `maintenance_deduction` and reduce the final VietQR payable amount (`4.500.000đ - 250.000đ = 4.250.000đ`).
   - **UAT-05 RBAC + Step-Up 2FA Guard:** In `BankAccountConfigForm` (`/api/v2/properties/{propertyId}/bank-account`):
     - If `currentUser.role === 'PROPERTY_MANAGER'`, hide edit buttons, disable inputs, and display a `403 Forbidden` security banner explaining that only `LANDLORD_OWNER` can modify VietQR payout accounts.
     - If `currentUser.role === 'LANDLORD_OWNER'`, require entering a 6-digit Step-Up 2FA OTP (`X-StepUp-Otp` header) before submitting changes.
6. **Verified Review & 48h Two-Way Moderation (`VerifiedReviewStatus`):**
   - Restricts review submission to verified `PRIMARY` and `CO_TENANT` members.
   - Forces photo/receipt evidence upload (via R2) when rating is `1` or `2` stars.
   - Displays the `PENDING_LANDLORD_RESPONSE` 48-hour countdown state and renders the Tenant's review alongside the Landlord's official rebuttal once published.

---

## 5. Verbatim Source Excerpts & Complete Schemas

### 5.1. SADD Frontend Technology Table
| Tầng Frontend | Công nghệ Đề xuất | Lý do Lựa chọn & Ánh xạ vào Nghiệp vụ SRS v2.0 |
|---|---|---|
| Core Framework & Build Tool | React 18+ & TypeScript (Vite PWA / Next.js App Router) | Xây dựng Progressive Web App (PWA) cài đặt trực tiếp ra màn hình chính điện thoại của Chủ nhà và Sinh viên mà không tốn phí phát triển App Native iOS/Android. Tối ưu UI cho chủ nhà trung niên: Font chữ chỉ số điện/thành tiền >= 16px–18px, nút bấm chính ('Chụp công tơ', 'Gửi hóa đơn') tối thiểu 48x48px. |
| Offline Storage & Background Sync | Dexie.js (IndexedDB) + Workbox Service Worker | Khi Quản lý tòa nhà đi chốt điện nước ở tầng hầm mất sóng, ảnh công tơ đã crop và chỉ số tạm được lưu vào IndexedDB. Ngay khi điện thoại bắt được sóng 4G/WiFi, Service Worker tự động xin Presigned URL, đẩy ảnh lên Cloudflare R2 và đồng bộ hóa đơn ngầm. |
| Client-Side Vision & Compression | HTML5 Canvas API + browser-image-compression + Web Crypto API | Giao diện Camera hiển thị khung ngắm chữ nhật (Bounding Box) chỉ lấy vùng dãy số công tơ, tự động crop và chuyển sang ảnh xám (Grayscale) < 50KB trước khi gửi OCR. Nén toàn bộ ảnh phòng/bàn giao sang định dạng WebP ngay trên máy khách và băm SHA-256 (`crypto.subtle.digest`) trước khi upload lên R2. |
| Bản đồ Không gian & Địa lý | Mapbox GL JS / MapLibre GL + Turf.js | Hiển thị bản đồ tìm kiếm theo bán kính quanh trường Đại học, phủ lớp đa giác cảnh báo ngập lụt (Flood Risk Score) và hiển thị song song [Giá thuê niêm yết] cùng [True Monthly Cost theo N người ở ghép]. |
| State, Caching & Real-time Stream | TanStack Query v5 + Zustand + EventSource (SSE) | Quản lý Server State, nhận luồng Server-Sent Events (SSE) hiển thị từng điều khoản rủi ro khi AI đang quét hợp đồng (FR-2.1) và tự động cập nhật trạng thái PAID ngay khi Webhook VietQR gạch nợ thành công. |
| UI Components & Comparison | Tailwind CSS + Shadcn UI + react-compare-image | Cung cấp thanh trượt đối chiếu trực quan Side-by-Side giữa ảnh Check-in gốc (đã khóa SHA-256 từ lúc nhận phòng) và ảnh Check-out hiện tại để phân định hao mòn tự nhiên vs lỗi người thuê. |

### 5.2. SADD Direct Upload Sequence to Cloudflare R2
```text
1. Xử lý tại trình duyệt (Client-side):
   - Crop vùng số công tơ (<50KB Grayscale)
   - Hoặc nén ảnh WebP / đóng dấu Watermark
   - Tính mã băm SHA-256 (Web Crypto API)

2. POST /api/v2/storage/presigned-put
   {category, targetEntityId, contentType, contentLength, sha256Hex}

3. Backend kiểm tra JWT, RBAC trên Entity, MIME Type & Size <= 15MB.
   Presigned PUT hiệu lực 10 phút, khóa Content-Type & metadata sha256.

4. Nhận {presignedPutUrl, objectKey}.

5. HTTP PUT <Binary Stream> trực tiếp từ trình duyệt lên Cloudflare R2.
   Headers: Content-Type, x-amz-meta-sha256

6. R2 trả 200 OK.

7. Gọi API nghiệp vụ kèm {objectKey, sha256}.

8. Backend HeadObject xác minh tồn tại và SHA-256.

9. Backend lưu objectKey & sha256 vào PostgreSQL.
```

### 5.3. Complete SRS AI Contract Scanner JSON Schema (Used by `ContractScanStream`)
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
    },
    {
      "severity": "RED",
      "category": "UTILITY_PRICING",
      "clause_excerpt": "Tiền điện tính theo giá cố định 4.500đ/kWh, chủ nhà có quyền điều chỉnh theo thị trường.",
      "legal_basis": "Thông tư 09/2023/TT-BCT quy định về cách tính giá điện sinh hoạt cho người thuê nhà.",
      "explanation": "Mức giá 4.500đ/kWh vượt trần quy định và điều khoản cho phép tự ý tăng giá gây rủi ro tài chính lớn.",
      "suggested_revision": "Đơn giá điện áp dụng cố định 3.500đ/kWh trong suốt thời hạn hợp đồng (hoặc tính theo biểu giá bậc thang EVN chia theo định mức nhân khẩu đăng ký tạm trú)."
    },
    {
      "severity": "YELLOW",
      "category": "RESIDENCE_AND_COLIVING",
      "clause_excerpt": "Người thuê tự làm thủ tục tạm trú và đóng phí hỗ trợ giấy tờ 500.000đ/lần cho chủ nhà.",
      "legal_basis": "Luật Cư trú 2020 & Thông tư 66/2023/TT-BCA (Mẫu CT01).",
      "explanation": "Khai báo tạm trú là trách nhiệm phối hợp của chủ sở hữu chỗ ở hợp pháp, không được thu phụ phí bất hợp lý.",
      "suggested_revision": "Bên A có trách nhiệm hỗ trợ xuất tờ khai tạm trú Mẫu CT01 trên hệ thống SafeRent VN miễn phí ngay khi Bên B cung cấp đủ thông tin CCCD."
    }
  ]
}
```

---

## 6. Frontend Acceptance Criteria Matrix (UAT-01 through UAT-06 Verification)

Automated Vitest + React Testing Library tests in `src/__tests__/uat/` **MUST** verify all 6 scenarios below:

| UAT ID | Scenario Name | Frontend Given / When / Then Verification Contract |
| :--- | :--- | :--- |
| **UAT-01** | Block landlord from raising electricity price above contract rate | **GIVEN** active lease for Room `P302` with `electricityRateCommitted = 3500`.<br>**WHEN** Landlord/Manager opens `MonthlyInvoiceBuilder` and inputs `electricityUnitPrice = 4200`.<br>**THEN** submit button is disabled, error alert renders `"Đơn giá điện không được vượt mức 3.500đ/kWh đã cam kết trong Hợp đồng"`, and rate cannot exceed `3500`. |
| **UAT-02** | Recalculate per-capita water fee on Co-Tenant addition & export CT01 | **GIVEN** Room `P201` with `waterPricingType = 'PER_PERSON'`, `waterUnitPrice = 100000`, `occupantCount = 1`.<br>**WHEN** Primary Tenant adds 2 Co-Tenants via eKYC.<br>**THEN** UI updates `occupantCount = 3`, projects next water charge as `300.000đ`, and enables the button to download `Mau_CT01_P201.docx` with all 3 members listed. |
| **UAT-03** | Handle electric meter rollover (`99850` -> `00120`) during OCR capture | **GIVEN** Room `P105` has `maxDial = 100000` and `elecOldIndex = 99850`.<br>**WHEN** Manager captures meter photo in `MeterCameraCapture` and OCR reads `00120`.<br>**THEN** UI detects `elecOldIndex >= 0.9 * maxDial`, toggles `isMeterRollover = true`, and displays consumption as `270 kWh` (`(100000 - 99850) + 120`) instead of a negative index error. |
| **UAT-04** | Auto-deduct approved tenant repair advance from monthly invoice | **GIVEN** Room `P401` has an approved maintenance ticket (`isDeductionApproved = true`, `tenantAdvanceAmount = 250000`) and gross rent + utilities = `4500000`.<br>**WHEN** Landlord issues the monthly invoice.<br>**THEN** UI populates `maintenanceDeduction = 250.000đ` and generates the VietQR code for the net payable amount of `4.250.000đ`. |
| **UAT-05** | Block `PROPERTY_MANAGER` from modifying VietQR bank account (`403`) | **GIVEN** user logged in with `role = 'PROPERTY_MANAGER'`.<br>**WHEN** navigating to `BankAccountConfigForm`.<br>**THEN** edit controls are hidden/disabled, a `403 Forbidden` notice is shown, and only `LANDLORD_OWNER` with a valid 6-digit Step-Up 2FA OTP can submit updates. |
| **UAT-06** | Automated deposit settlement & VietQR refund on Digital Check-out | **GIVEN** lease with `depositAmount = 4000000`, valid 30-day termination notice, final utilities = `350000`, and all handover items marked `NATURAL_WEAR` (`0đ` fault deduction).<br>**WHEN** both parties confirm `HandoverComparison`.<br>**THEN** `DepositSettlementSummary` displays net refund `3.650.000đ` and renders the Tenant's VietQR code for instant payout. |

---

## 7. Recommended Frontend Directory Structure

```text
saferent-vn-frontend/
├── index.html
├── package.json
├── tsconfig.json
├── vite.config.ts
├── tailwind.config.ts
└── src/
    ├── main.tsx
    ├── App.tsx
    ├── index.css
    ├── components/
    │   └── ui/
    │       ├── ActionTouchButton.tsx         # >= 48x48px button for landlords 45-65
    │       ├── MoneyDisplay.tsx              # Tabular-nums VND currency formatter
    │       ├── FireSafetyBadge.tsx           # 3-tier PCCC badge + 6-month expiry indicator
    │       └── OfflineSyncBanner.tsx         # Basement offline queue indicator
    ├── lib/
    │   ├── storage/
    │   │   └── r2UploadClient.ts             # sha256Hex, requestPresignedPut, uploadDirectToR2, registerUploadedObject
    │   ├── vision/
    │   │   └── meterCropProcessor.ts         # cropMeterDisplayToGrayscaleWebp (< 50KB)
    │   └── offline/
    │       └── meterCaptureQueue.ts          # Dexie IndexedDB table + replayPendingMeterCapture
    ├── features/
    │   ├── discovery/
    │   │   ├── PropertyDiscoveryMap.tsx      # MapLibre/Mapbox + accessible list + flood/PCCC filters
    │   │   └── TrueCostCard.tsx              # True Monthly Cost formula by N occupants & AEF
    │   ├── smartlease/
    │   │   ├── ContractScanStream.tsx        # EventSource SSE AI legal scanner
    │   │   ├── CoLivingMembersPanel.tsx      # Add Co-Tenants + 1-Click CT01 export (UAT-02)
    │   │   ├── HandoverComparison.tsx        # react-compare-image + NATURAL_WEAR vs TENANT_FAULT
    │   │   └── DepositSettlementSummary.tsx  # 7-day SLA refund calculator + Tenant VietQR (UAT-06)
    │   ├── operations/
    │   │   ├── MeterCameraCapture.tsx        # Bounding box camera + rollover UAT-03 + offline queue
    │   │   ├── MonthlyInvoiceBuilder.tsx     # Rate guard UAT-01 + ticket offset UAT-04
    │   │   └── BankAccountConfigForm.tsx     # RBAC 403 guard for Manager + 2FA OTP for Owner (UAT-05)
    │   └── community/
    │       └── VerifiedReviewStatus.tsx      # Verified badge, 1-2 star evidence & 48h dispute timer
    └── __tests__/
        ├── r2UploadClient.test.ts
        ├── meterCropAndQueue.test.ts
        └── uat/
            └── FrontendUatScenarios.test.tsx # Automated tests covering UAT-01 through UAT-06
```