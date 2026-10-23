# Discovery Module Rules

## Scope and responsibility

This module owns PostGIS property search, public room projections, True Monthly Cost, flood-risk filtering, fire-safety badge policy, and discovery caching. It MUST NOT own lease activation, billing, reviews, identity secrets, or media transfer.

## True Monthly Cost policy

`TrueCostPolicy` MUST be pure Java and MUST calculate with `BigDecimal`:

```text
estimatedKwh(N) = (80 kWh + 40 kWh × N) × AEF
estimatedElectricity = estimatedKwh(N) × electricityUnitPrice

TrueMonthlyCost(N) = baseRent
                   + estimatedElectricity
                   + waterCost(N or m³)
                   + serviceCost(N or per-room)

PerPersonCost(N) = TrueMonthlyCost(N) / N
```

- `N` MUST default to 2 for discovery when omitted and MUST be a positive integer within supported room capacity.
- For `water_pricing_type = PER_PERSON`, use `water_unit_price × N`. For `PER_M3`, use the documented water-volume input/estimate; NEVER silently treat a per-m³ price as per-person.
- Use `service_fee_per_person × N` for the SRS field. A future per-room fee requires an explicit versioned field/migration; NEVER overload the per-person column.
- `AEF = 1.00` for `INVERTER_NEW` (<3 years or 5-star label), `1.20` for `STANDARD`, and `1.40` for `OLD_HIGH_CONSUMPTION` when at least two Verified Tenants report high consumption or an AC fault ticket is overdue.
- `NONE` MUST use an explicitly approved no-AC factor/estimate; NEVER pretend it is `INVERTER_NEW`. If the source documents do not define the estimate, surface the missing input instead of inventing a tariff.
- Return both listing price and `estimated_true_monthly_cost`, plus the assumptions (`occupants`, estimated kWh, AEF, unit prices). NEVER present an estimate as a guaranteed invoice.

Required SRS example: for the UI, support the format “Giá niêm yết: 3.500.000đ | Chi phí thực tế ước tính (2 người): ~4.280.000đ/tháng (chia 2 = 2.140.000đ/người)”. Tests MUST derive the result from inputs rather than hard-code display output.

## Fire-safety badge policy

Use the legal label “Xác thực Hiện trạng Trang bị An toàn”. NEVER call the badge an official PCCC certificate.

- Level 0, gray: no verified declaration/evidence.
- Level 1, blue: landlord captures live-camera evidence, not gallery upload, for second exit, ground-floor parking fire door/barrier, fire extinguisher, and smoke alarm.
- Level 2, green/gold: at least three Verified Tenants confirm on-site conditions OR an assigned `INSPECTOR` completes field verification.
- `fire_badge_expires_at` MUST be six months from the qualifying verification. Maintaining Level 2 requires refreshed exit/extinguisher evidence or confirmation from a current tenant.
- Expired Level 2 MUST be downgraded/marked expired by an idempotent `FireBadgeExpiryJob`; use ShedLock when the job may run on multiple instances.
- Fire-safety evidence MUST upload directly to `saferent-public-media` through Presigned PUT. Persist only object keys and hashes.

## PostGIS search

- `properties.location` MUST remain `GEOGRAPHY(Point, 4326)` with `idx_properties_location` GIST index.
- Radius search MUST use `ST_DWithin` in meters and parameter binding; NEVER interpolate coordinates or filters into SQL.
- Results MAY filter by `fire_safety_level`, `flood_risk_score`, availability, rent, and computed True Cost. Stable pagination MUST include a deterministic tie-breaker.
- Public projections MUST omit private contact data, exact private object keys, CCCD, bank data, manager details, and internal moderation fields.

## Redis and Caffeine cache rules

Geo search L2 key and TTL are fixed:

```text
geo:v2:{city}:{geohash6}:{radius}:{filterHash}  TTL = 5 minutes
```

- MUST normalize latitude/longitude to Geohash precision 6 (~1.2 km × 0.6 km), radius, city, sort, and all non-occupant filters before hashing.
- Cache base room inputs (`baseRent`, `elecPrice`, `acAef`, `waterPrice`, `waterType`, `serviceFeePerPerson`), then recompute cost for `N = 1..4` in memory. NEVER make `N` omission return another user's cost.
- Property detail L2 key MUST be `prop:v2:detail:{propertyId}` with a 30-minute TTL.
- Evict property/search caches after room/property changes, lease activation/status change, fire badge change/expiry, review publication, or an overdue AC ticket changing AEF to 1.40.
- Caffeine L1 reference cache MAY hold immutable tariff/administrative reference data for 12 hours. It MUST NOT cache user-specific authorization or PII.
- Prevent cache stampede with bounded single-flight/loading behavior. Cache failure MUST degrade to the authoritative database, not to stale authorization data.

## Required tests

- MUST unit-test the full `TrueCostPolicy` formula for all AEF values, `PER_PERSON`, `PER_M3`, `N=1..4`, invalid N, exact scale, and rounding.
- MUST test Level 0/1/2 qualification, exactly-three-tenant boundary, Inspector path, six-month expiry, and concurrent expiry/refresh.
- MUST integration-test `ST_DWithin`, GIST-compatible query behavior, radius units, and deterministic pagination on PostgreSQL/PostGIS Testcontainers.
- MUST test Geohash6 cache key normalization, TTLs, cache eviction triggers, and recalculation when only `N` changes.
