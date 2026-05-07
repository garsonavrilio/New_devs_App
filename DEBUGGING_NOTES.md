# Debugging Notes — Property Revenue Dashboard

## Reported Issues

| # | Client | Symptom |
|---|---|---|
| 1 | Ocean Rentals (Client B) | Sometimes sees revenue belonging to another company |
| 2 | Sunset Properties (Client A) | March revenue totals do not match internal records |
| 3 | Finance | Revenue totals are off by a few cents |

---

## Bug 1 — Cross-Tenant Revenue Cache Leakage

**File:** `backend/app/services/cache.py`

### Symptom
Client B (Ocean Rentals) intermittently sees Client A's revenue figures on the dashboard for the same `property_id`.

### Root Cause
The Redis cache key was scoped only to `property_id`:
```python
cache_key = f"revenue:{property_id}"
```
Both `tenant-a` and `tenant-b` share `prop-001` as a valid property ID (composite PK in the DB is `property_id + tenant_id`). Whichever tenant loaded first would write their revenue to the shared key; the other tenant would read it on a cache hit.

### Fix
```diff
- cache_key = f"revenue:{property_id}"
+ cache_key = f"tenant:{tenant_id}:property:{property_id}:revenue_summary"
```
The cache key now includes `tenant_id`, so each tenant gets an isolated Redis namespace.

### Verification
```bash
redis-cli FLUSHDB

# Log in as Sunset (tenant-a), request prop-001
redis-cli GET "tenant:tenant-a:property:prop-001:revenue_summary"
# → Sunset's revenue JSON

# Log in as Ocean (tenant-b), request prop-001
redis-cli GET "tenant:tenant-b:property:prop-001:revenue_summary"
# → Ocean's revenue JSON (different value)

# Old unscoped key must not exist
redis-cli KEYS "revenue:*"
# → (empty)
```

---

## Bug 2 — Financial Precision Loss

**Files:** `backend/app/api/v1/dashboard.py`, `frontend/src/components/RevenueSummary.tsx`

### Symptom
Finance reports revenue totals off by a few cents. The database stores `total_amount` as `NUMERIC(10, 3)` (3 decimal places), and the seed data contains sub-cent amounts such as `333.333`.

### Root Cause
The dashboard endpoint converted the `Decimal` total to a Python `float` before returning it:
```python
total_revenue_float = float(revenue_data['total'])
```
IEEE 754 binary floating-point cannot represent all decimal fractions exactly. Once serialised to JSON as a float, precision is permanently lost.

On the frontend, `RevenueSummary.tsx` compounded the issue by re-rounding using `Math.round(data.total_revenue * 100) / 100` — float arithmetic on an already-imprecise float.

### Fix

**Backend** — quantize with `ROUND_HALF_UP`, return as a string:
```python
from decimal import Decimal, ROUND_HALF_UP

total_revenue = Decimal(str(revenue_data['total'])).quantize(
    Decimal("0.01"), rounding=ROUND_HALF_UP
)
return { ..., "total_revenue": str(total_revenue), ... }
```

**Frontend** — update the TypeScript type and display logic:
```diff
- total_revenue: number;
+ total_revenue: string;

- const displayTotal = Math.round(data.total_revenue * 100) / 100;
+ // parseFloat is safe here — used only for locale display formatting, not arithmetic
+ const displayTotal = parseFloat(data.total_revenue);
```
The float-drift precision warning block was also removed as it no longer applies.

### Verification
```bash
curl -H "Authorization: Bearer <token>" \
  "http://localhost:8000/api/v1/dashboard/summary?property_id=prop-001"

# Response should contain a string, e.g.:
# { "total_revenue": "999.00", ... }
# Not a float like 999.0000000000001
```
For `prop-001` (tenant-a), the three seed reservations sum to exactly `333.333 + 333.333 + 333.334 = 999.000`, which rounds to `"999.00"`.

---

## Bug 3 — Monthly Revenue Timezone Boundary

**File:** `backend/app/services/reservations.py`

### Symptom
`calculate_monthly_revenue` used naive `datetime` objects with no timezone, so month boundaries were evaluated in server/UTC time rather than the property's local time. For `prop-001` (Paris, UTC+1), the March boundary at midnight local time is `2024-02-29T23:00:00Z` in UTC — one hour before UTC midnight. Without the fix, reservation `res-tz-1` (`2024-02-29 23:30 UTC`) would be classified as February instead of March.

### Root Cause
```python
start_date = datetime(year, month, 1)   # naive — no timezone
```
`reservations.check_in_date` is stored as `TIMESTAMP WITH TIME ZONE` (UTC). Comparing UTC timestamps against naive local-time boundaries produces incorrect month attribution for any property not in UTC.

### Fix
Added `property_timezone` parameter; boundaries are built in the property's local timezone using `zoneinfo.ZoneInfo` and converted to UTC before querying:
```python
from zoneinfo import ZoneInfo

tz = ZoneInfo(property_timezone)          # e.g. "Europe/Paris"
start_local = datetime(year, month, 1, tzinfo=tz)
end_local   = datetime(year, month + 1, 1, tzinfo=tz)

start_utc = start_local.astimezone(ZoneInfo("UTC"))
end_utc   = end_local.astimezone(ZoneInfo("UTC"))
# → use start_utc / end_utc in the SQL filter
```

> **Note:** `calculate_monthly_revenue` is currently dead code — it has no callers in the active dashboard path. The fix was applied so the function is correct when it is eventually wired to an endpoint. The actual March mismatch reported by Client A was caused by Bugs 1 and 2 above.

### Verification
When the function is wired to a live DB session, the debug log will confirm the UTC window:
```
DEBUG: Monthly revenue for prop-001 (tenant=tenant-a) month=3/2024 tz=Europe/Paris
→ UTC [2024-02-29T23:00:00+00:00, 2024-03-31T22:00:00+00:00)
```
`res-tz-1` (`check_in_date = 2024-02-29 23:30 UTC`) falls inside this window and is correctly attributed to March.

---

## Final Manual Testing Checklist

- [ ] **Cache isolation** — Log in as Sunset and Ocean separately for `prop-001`. Confirm each sees their own revenue. Confirm `redis-cli KEYS "revenue:*"` returns empty.
- [ ] **Precision** — Confirm `total_revenue` in the API response is a string with exactly 2 decimal places. Confirm no float scientific notation in the JSON.
- [ ] **Tenant guard** — Call `/api/v1/dashboard/summary` without a valid token. Confirm `403 Forbidden` with `"Tenant context is required"`.
- [ ] **Timezone** — When monthly revenue is wired: confirm `res-tz-1` is included in tenant-a March totals for `prop-001 (Europe/Paris)`.
- [ ] **No regression** — `prop-002` through `prop-005` return correct all-time totals for their respective tenants with no cross-tenant data.
