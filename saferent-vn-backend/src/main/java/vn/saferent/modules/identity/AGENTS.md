# Identity Module Rules

## Scope and responsibility

This module owns authentication, JWT/refresh rotation, eKYC identity, consent/audit, the six stored roles, Owner/Manager authority, and step-up OTP for bank configuration. It MUST NOT own lease membership lifecycle, invoices, reviews, or binary storage.

Use these packages only: `domain`, `application`, `infrastructure`, and `presentation`. Keep `User`, `UserRole`, `CccdProfile`, consent, and authentication policies in `domain`; keep JPA, Redis, OCR, KMS, and provider adapters in `infrastructure`.

## Canonical roles and tenant subtype

- `users.role` MUST be one of `TENANT`, `LANDLORD_OWNER`, `PROPERTY_MANAGER`, `INSPECTOR`, `PARTNER`, `ADMIN`.
- Primary vs Co-Tenant is NEVER a seventh/eighth user role. Resolve it from an active `lease_members.member_role` of `PRIMARY` or `CO_TENANT` in the Smart Lease module.
- JWT authorities MUST use `ROLE_TENANT`, `ROLE_LANDLORD_OWNER`, `ROLE_PROPERTY_MANAGER`, `ROLE_INSPECTOR`, `ROLE_PARTNER`, and `ROLE_ADMIN`.
- JWT MUST NOT embed permanent property/lease ownership claims that can become stale. Resolve `properties.owner_id`, `properties.manager_id`, and active `lease_members` at authorization time or via a safely invalidated authorization view.

## User and CCCD persistence

Preserve the canonical `users` fields: `id`, `full_name`, `date_of_birth`, `phone_number`, `email`, `role`, `cccd_encrypted`, `cccd_hash`, `ekyc_verified`, `reputation_points`, and `created_at`.

- `phone_number` MUST be unique and normalized before hashing/comparison. `email` MUST be unique when present.
- `cccd_encrypted` MUST contain a versioned AES-256-GCM envelope with a fresh 12-byte IV and 128-bit tag. The JPA `AttributeConverter` MUST use `AES/GCM/NoPadding`.
- `cccd_hash` MUST be the 64-character lowercase hexadecimal SHA-256 digest of the canonical 12-digit identifier and MUST remain `UNIQUE`.
- NEVER store plaintext CCCD in another column, OCR response table, cache, search index, log, event, exception, metric label, or JWT.
- CCCD decryption MUST be explicit, authorized, request-scoped, and limited to eKYC correction or CT01 export. Clear mutable buffers where practical.
- The module MUST publish only a minimal identity result such as user ID, verified flag, and masked display name. NEVER publish plaintext `CccdProfile` on the shared event bus.

Example converter boundary:

```java
@Converter
public final class AesGcmCccdConverter implements AttributeConverter<String, String> {
    private static final String ALGORITHM = "AES/GCM/NoPadding";
    private static final int IV_BYTES = 12;
    private static final int TAG_BITS = 128;
    // Key material MUST be injected from KMS/Key Vault configuration.
}
```

NEVER use a static IV, AES/ECB, a hard-coded key, reversible “encoding,” deterministic encryption, MD5, or SHA-1.

## Authentication and token lifecycle

- Access JWT MUST expire after 15 minutes.
- Refresh tokens MUST rotate and live in Redis for at most 7 days. Reuse of an already rotated token MUST revoke the token family and produce an audit event.
- OTP values MUST be random, single-use, short-lived, rate-limited, attempt-limited, stored as a digest, and never logged.
- Guest AI scan allowance MUST be 1/day using IP plus device fingerprint. Authenticated Tenant allowance MUST be 3/day/User.
- Landlord phone display for unauthenticated users MUST mask the last four digits.

## RBAC plus resource ownership

- Role checks MUST be combined with resource checks. Identity supplies principals and common guards; the owning module supplies the ownership facts.
- `PROPERTY_MANAGER` authority is limited to properties where `properties.manager_id == principal.userId`.
- `LANDLORD_OWNER` authority is limited to properties where `properties.owner_id == principal.userId`.
- `INSPECTOR` and `PARTNER` MUST be scoped to explicit assignments/referrals.
- `ADMIN` operations MUST be audited and MUST NOT bypass legal retention or delete substantiated negative reviews.

Bank configuration is a high-risk action:

```java
@PreAuthorize("hasRole('LANDLORD_OWNER') and @resourceGuard.isPropertyOwner(#propertyId, authentication)")
@RequireStepUpOtp(action = "UPDATE_VIETQR_BANK_ACCOUNT")
public BankConfigResponse updateBankAccount(UUID propertyId, UpdateBankAccountCommand command) {
    return updateBankConfigUseCase.handle(command);
}
```

- A `PROPERTY_MANAGER` attempt MUST always return 403 even when assigned to the property.
- Owner change MUST require a newly validated step-up OTP bound to principal, property, action, and short time window.
- NEVER accept role, owner ID, manager ID, or OTP-validation state from the request body.
- Audit successful and denied bank-change attempts without logging the full account number or OTP.

## eKYC and retention coordination

- eKYC OCR MAY extract full name, date of birth, 12-digit identifier, place of origin, and permanent address only with a valid consent record and stated purpose.
- Original CCCD images MUST go directly to `saferent-private-vault`; this module receives only object keys and hashes after `HeadObject` verification.
- The common retention job MUST receive an eligibility decision derived from Smart Lease: lease status `TERMINATED` for more than 90 days. Deletion MUST be idempotent and audited.
- `PROPERTY_MANAGER` MUST NOT view plaintext CCCD. CT01 access requires explicit Owner grant and is performed through the Smart Lease export use case.

## Required tests

- MUST test every role code, token expiry/rotation/reuse, OTP expiry/reuse/attempt limit, and redacted logs.
- MUST test randomized encryption: identical CCCD plaintext produces different ciphertext but the same canonical SHA-256 hash.
- MUST test GCM tamper detection and key/version failure without leaking plaintext.
- MUST test duplicate `cccd_hash`, phone, and email constraints against PostgreSQL.
- MUST implement UAT-05 as a security integration test: Property Manager receives 403; wrong-property Owner receives 403; correct Owner without step-up receives 403; correct Owner with valid step-up succeeds.
