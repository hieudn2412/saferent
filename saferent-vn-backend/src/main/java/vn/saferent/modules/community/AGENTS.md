# Community Module Rules

## Scope and responsibility

This module owns verified reviews, evidence requirements, landlord response, two-way dispute presentation, and the 48-hour publication workflow. It MUST NOT determine lease membership by itself; query Smart Lease through an application port.

## Verified reviewer eligibility

- Only a user who is an active verified member of the property through `lease_members` MAY create a Verified Review.
- Both `PRIMARY` and `CO_TENANT` are eligible. A Co-Tenant MUST NOT be rejected merely because the account role is the shared `TENANT` role.
- Verify `ekyc_verified`, active membership, property linkage through lease/room, and relevant stay interval at submission time.
- Store an immutable eligibility snapshot/reference for audit. NEVER expose CCCD, lease financials, private contact data, or private R2 keys in the public review.
- One-review/eligibility limits MUST be explicit and database-enforced when introduced; NEVER rely only on a UI button state.

## Rating and evidence rules

- Ratings 1–2 stars or a violation allegation MUST include evidence uploaded directly to the appropriate R2 bucket through a Presigned PUT. Spring Boot MUST receive only verified object keys and SHA-256 hashes.
- Validate evidence ownership, MIME, size, hash, and safe public/private classification before accepting the review.
- Evidence containing CCCD, phone, faces requiring consent, signed leases, bank data, or handover material MUST remain in `saferent-private-vault` and MUST NOT be embedded publicly.
- Sanitize review and response text against stored/presented XSS while preserving original audit content safely.
- Admin MUST NOT delete a substantiated negative review solely because it is negative. Moderation actions require a reason, actor, timestamp, and audit record.

## 48-hour two-way response workflow

For a 1–2-star review or qualifying allegation:

1. Create the review in `PENDING_LANDLORD_RESPONSE`.
2. Persist `response_deadline = submitted_server_time + 48 hours` and optimistic `version`.
3. Notify the resource-owning `LANDLORD_OWNER` asynchronously after commit.
4. Allow the Owner to submit one official evidence-backed response before the deadline.
5. Publish the review and Owner response together after response or deadline, preserving both sides.

- `PROPERTY_MANAGER` MUST NOT author the official landlord response unless an explicit future SRS change grants it.
- The publication job MUST use `@SchedulerLock` plus `verified_reviews.version`.
- If Owner response commits at 47:59:59, the optimistic version change MUST defeat a concurrent stale auto-publish update; Owner response wins.
- If no response arrives, publish the tenant review after 48 hours with a clear no-response state.
- Publication and notifications MUST be idempotent. A retry MUST NOT duplicate a public review or email.
- Cache `prop:v2:detail:{propertyId}` and related discovery results MUST be evicted when a review becomes public.

## State and data requirements

`verified_reviews` MUST support at minimum:

- reviewer user reference and verified membership/lease reference;
- property reference;
- rating and review text;
- evidence object references and SHA-256 values;
- `PENDING_LANDLORD_RESPONSE` and final moderation/publication state;
- server submission time and 48-hour response deadline;
- landlord response text/evidence/time;
- moderation/audit reason;
- optimistic `version`.

Exact new column names require a Flyway migration and API/schema review. NEVER invent a second review table or encode workflow state only inside JSON.

## Required tests

- MUST test eligibility for active Primary and active Co-Tenant, and rejection for Guest, inactive/former unrelated Tenant, Property Manager, and a tenant of another property.
- MUST test mandatory evidence for 1–2 stars/allegation and allowed behavior for ordinary 3–5-star reviews.
- MUST test exactly-before, exactly-at, and after the 48-hour boundary using an injected clock.
- MUST run a concurrent Owner-response vs scheduler test and assert that a committed pre-deadline response wins.
- MUST test idempotent publication/notification, evidence privacy, XSS sanitization, audit trail, and discovery-cache eviction.
