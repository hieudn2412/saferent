# SafeRent VN Frontend Rules

## Scope and stack

These rules apply to `saferent-vn-frontend/**` and extend the repository-level `AGENTS.md`.

- MUST use React 18+, TypeScript in strict mode, and Vite PWA.
- MUST use Tailwind CSS and Shadcn UI for accessible primitives, TanStack Query v5 for server state, Zustand for small client-only UI/session state, Dexie.js for IndexedDB, and Workbox for service worker/background sync.
- MUST use Mapbox GL JS or MapLibre GL with Turf.js for map/radius UX.
- MUST use native `EventSource` or an equivalent standards-compliant SSE client for contract AI scan.
- MUST use `react-compare-image` for check-in vs check-out evidence comparison.
- NEVER duplicate server state into Zustand. NEVER store JWT, OTP, plaintext CCCD, bank secrets, or private Presigned URLs in persistent Zustand/localStorage.
- All TypeScript must pass strict type checking. Avoid `any`; validate untrusted API/SSE/IndexedDB payloads at runtime.

## Mobile-first UX for landlords aged 45–65

- Design primary workflows for one hand and “Chạm & Chụp” zero-typing behavior.
- Primary action touch targets MUST be at least `48px × 48px`, with visible focus, pressed, disabled, loading, success, and error states.
- Meter values, money totals, and critical labels MUST render at no less than 16–18 px. Do not rely on color alone.
- Use Vietnamese labels, plain-language errors, formatted VND, and explicit room/meter identity before confirmation.
- Provide a review screen before irreversible issue/sign/settlement actions. Financial confirmation MUST show gross amount, old debt, deduction, credit, and final payable/refund.
- Camera permission denial, offline state, slow upload, expired URL, retry, duplicate tap, and browser incompatibility MUST have recoverable UX.
- Disable duplicate submit immediately, but assume the user can bypass the UI; preserve backend idempotency keys.
- Meet WCAG AA contrast, keyboard navigation, screen-reader labels, reduced-motion preference, and safe-area insets. NEVER hide security information in hover-only UI.

Example primary button baseline:

```tsx
<Button className="min-h-12 min-w-12 px-5 text-base font-semibold">
  Chụp công tơ
</Button>
```

## Zero-bandwidth direct R2 transfer

The browser MUST NEVER send media bytes to Spring Boot. The mandatory sequence is:

1. Capture/select locally according to category policy.
2. Crop/compress/convert/hash locally.
3. Request a Presigned PUT using metadata only.
4. PUT bytes directly to Cloudflare R2.
5. Submit only `{objectKey, sha256}` to the business API.

```ts
type PresignRequest = {
  category: StorageCategory;
  targetEntityId: string;
  contentType: string;
  contentLength: number;
  sha256Hex: string;
};

const { presignedPutUrl, objectKey } = await api.post<PresignedPut>(
  "/api/v2/storage/presigned-put",
  metadata,
);

await fetch(presignedPutUrl, {
  method: "PUT",
  headers: {
    "Content-Type": blob.type,
    "x-amz-meta-sha256": sha256Hex,
  },
  body: blob,
});

await api.post(businessEndpoint, { objectKey, sha256: sha256Hex });
```

- Public listing/PCCC evidence uses `saferent-public-media`; CCCD, contracts, CT01, meter originals, repair receipts, and handover media use `saferent-private-vault`. The client sends a category, NEVER a bucket name or object key.
- PUT URLs expire after 10 minutes. On expiry, request a new URL and retry the same immutable blob/hash; NEVER reuse another file's URL.
- Private GET URLs expire after exactly 5 minutes. Keep them in component memory only, revoke object URLs, and request a fresh URL after expiry. NEVER persist or log them.
- Treat R2 success as provisional until the business API verifies `HeadObject`. Show completion only after business registration succeeds.
- NEVER use `FormData`, base64 JSON, Axios upload-to-backend endpoints, Next.js/Vite server proxying, or a Spring media relay for these files.

## Client-side meter vision and cost control

Meter capture MUST use a live camera overlay with a rectangular bounding box around the numeric display.

- Capture at device resolution, map the visible bounding box into source-pixel coordinates, crop with HTML5 Canvas, correct orientation, convert to grayscale, and iteratively reduce dimensions/quality until the OCR blob is below 50 KB.
- Preserve sufficient contrast and digits; if the 50 KB target cannot be met without illegibility, show a recapture/review flow instead of silently sending a bad image.
- Display the crop, OCR index, OCR serial, and confidence for human confirmation.
- Compute SHA-256 over the exact final bytes that will be uploaded, not the original image or a data URL string.

```ts
export async function sha256Hex(blob: Blob): Promise<string> {
  const bytes = await blob.arrayBuffer();
  const digest = await crypto.subtle.digest("SHA-256", bytes);
  return [...new Uint8Array(digest)]
    .map((byte) => byte.toString(16).padStart(2, "0"))
    .join("");
}
```

- Room/handover images MUST be converted to WebP before hashing and upload.
- Video MUST use an approved bounded codec/size and direct upload; NEVER load an entire large video as base64.
- Keep the original local blob only as long as required for retry/offline sync and delete it after confirmed registration.

## Offline-first basement synchronization

Meter work MUST remain usable without connectivity. Define a versioned Dexie queue with enough data to replay safely:

```ts
type PendingMeterCapture = {
  localId: string;
  idempotencyKey: string;
  propertyId: string;
  roomId: string;
  billingMonth: string;
  capturedAt: string;
  meterSerialDraft?: string;
  meterIndexDraft?: string;
  cropBlob: Blob;
  contentType: "image/webp";
  sha256Hex: string;
  state: "PENDING" | "UPLOADING" | "REGISTERING" | "SYNCED" | "NEEDS_REVIEW";
  attemptCount: number;
  lastErrorCode?: string;
};
```

- Persist the processed crop blob, hash, room, billing month, capture time, draft index/serial, and a stable idempotency key in IndexedDB before reporting “Đã lưu ngoại tuyến”.
- Workbox Background Sync MUST replay in order: obtain fresh Presigned PUT, direct PUT to R2, then register/submit OCR with the same idempotency key.
- A service worker MUST NOT rely on an expired Presigned URL stored in IndexedDB.
- Retry transient network/5xx/429 errors with bounded exponential backoff. Authentication, ownership, serial mismatch, billing-month conflict, or domain validation MUST move to `NEEDS_REVIEW`, not loop forever.
- Sync MUST be idempotent across app close, service-worker restart, duplicate connectivity events, and a user also retrying in the foreground.
- Show per-room sync status and a visible unresolved queue count. NEVER claim the invoice is issued while only local capture is saved.
- Delete the local blob only after R2 upload and business registration are both confirmed. Provide explicit consent before clearing unresolved offline work.
- Protect IndexedDB from accidental PII: meter crops are allowed for operational retry; CCCD and raw signed contracts MUST NOT be cached offline unless a separately approved encrypted design exists.

## Server state, forms, and authorization UX

- TanStack Query keys MUST include every server-state input: principal scope where relevant, property/lease/room IDs, billing month, occupants `N`, map filters, radius, and pagination.
- Mutations MUST invalidate exact property/room/lease/invoice/review keys. NEVER display stale bank, invoice, or settlement data after mutation.
- UI permission checks improve usability only. The server remains authoritative. Handle `401`, `403`, `409`, `422`, and `429` distinctly.
- For UAT-05, hide/disable the VietQR bank edit control for `PROPERTY_MANAGER`, but still display a safe read-only value if permitted. Owner edit MUST collect step-up OTP and send it only in the designated header/body contract; NEVER persist it.
- Format money from decimal strings or minor-unit-safe representations. NEVER use floating-point arithmetic to calculate invoice, debt, credit, deposit, or refund in the browser. Server values are authoritative.
- True Cost UI MUST show listing rent, estimated total for `N`, per-person estimate, estimated kWh, AEF, and assumptions. Label it as an estimate.

## Contract AI scan with SSE

- Upload the contract directly to private R2, register/scan the object reference, then open an SSE stream.
- Parse only known event types such as `scan.started`, `ocr.completed`, `risk.found`, `scan.completed`, and `scan.failed`; reject malformed payloads.
- Render findings incrementally and announce progress accessibly. Provide reconnect behavior using server event IDs when supported and prevent duplicate findings.
- Close `EventSource` on terminal event, route change, user cancellation, logout, or component unmount.
- NEVER place raw PII or Presigned URLs in the SSE URL/query string. Use an opaque scan ID and authenticated same-origin/session mechanism approved by the backend.
- Clearly mark AI analysis as advisory and keep the validated legal-basis/category data from the API.

## Check-in and check-out UI

- Check-in MUST guide users through room/device photos, video if required, electricity/water baseline, GPS permission, and a final evidence manifest confirmation.
- Display the 48-hour dispute deadline using the server timestamp; never use the device clock as legal truth.
- Once server state is `LOCKED`, disable editing and show hash/timestamp evidence. Corrections use an amendment flow, never local overwrite.
- Check-out MUST use `react-compare-image` to compare each locked check-in image against the corresponding check-out image.
- Every damage item requires `NATURAL_WEAR` or `TENANT_FAULT`. Only tenant-fault items may enter a proposed deposit deduction, with evidence and confirmation.
- Settlement UI MUST show:

```text
Refund = Deposit
       - Prorated Final Rent
       - Final Electricity/Water
       - Confirmed Tenant-Fault Damage
```

- Show the no-more-than-7-day settlement deadline and the Primary Tenant VietQR for the Owner's direct refund. NEVER imply SafeRent holds the deposit.
- UAT-06 UI MUST render VND 3,650,000 for a VND 4,000,000 deposit minus VND 350,000 final utilities with no other deduction.

## PCCC, reviews, and maps

- Fire badge copy MUST say “Xác thực Hiện trạng Trang bị An toàn”, with Level 0 gray, Level 1 blue, and Level 2 green/gold. Never call it an official certificate.
- Level 1 capture MUST use live camera UX and block gallery upload for required evidence.
- Map cards MUST show listing rent and True Monthly Cost for the selected occupant count. Use Mapbox/MapLibre accessibly and offer a list alternative.
- Verified Review composer MUST be available to eligible Primary and Co-Tenants.
- A 1–2-star review/violation report MUST require evidence and explain the 48-hour `PENDING_LANDLORD_RESPONSE` period. Once published, display tenant review and landlord response side by side without obscuring either.

## PWA security and privacy

- Service worker MUST NOT cache authenticated API responses, private R2 responses, CCCD, contracts, CT01, bank settings, OTP routes, or Presigned URLs.
- Cache only hashed static assets and explicitly public GET resources with a versioned strategy. Logout MUST clear principal-scoped query and transient state.
- Apply a restrictive CSP and avoid unsafe HTML. If rich text is unavoidable, sanitize with an approved allowlist.
- NEVER log full phone, CCCD, bank account, JWT, OTP, contract text, exact private object key, or signed URL to console, analytics, replay tools, or error monitoring.
- Do not ask for camera, location, or notification permission until the user starts the relevant flow; explain purpose and provide fallback.

## Required tests and quality gates

- MUST run lint, formatting, TypeScript strict check, unit/component tests, service-worker/offline tests, and production build.
- MUST test touch target size, 16–18 px critical text, keyboard/focus behavior, Vietnamese labels, error recovery, and screen-reader announcements.
- MUST test Canvas crop coordinates/orientation, grayscale output, <50 KB target behavior, WebP conversion, and SHA-256 against known vectors.
- MUST test that no request sends media bytes to the backend and that direct PUT contains the signed type/hash headers.
- MUST test offline save, restart, reconnect, expired-URL renewal, duplicate replay, authentication failure, conflict/serial mismatch, and local blob cleanup.
- MUST test SSE incremental events, malformed data, reconnect/deduplication, cancellation, and PII absence.
- MUST test role/ownership UX including UAT-05, while retaining explicit API 403 handling.
- MUST cover frontend outcomes for UAT-01 through UAT-06, especially exact money/meter values, co-tenant count, deduction, and settlement presentation.
