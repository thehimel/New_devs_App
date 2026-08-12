# Solution

Summary of what I found and fixed for the Property Revenue Dashboard assignment.

## Results after Fixes

* [Sunset Properties](sunset/results.md)
* [Ocean Rentals](ocean/results.md)

## Setup

- Cloned the repo, set up the environment, started with [`docker-compose.yml`](../docker-compose.yml) (`docker-compose up --build`)
- Setup notes: [`setup.md`](setup.md)
- Logged in as both clients, saved screenshots, compared UI vs API vs DB

## Reports

| Client / area | Folder | Status |
|---|---|---|
| Sunset Properties | [`sunset/analysis.md`](sunset/analysis.md) | Symptoms documented |
| Ocean Rentals | [`ocean/analysis.md`](ocean/analysis.md) | Symptoms documented |
| Finance | [`finance/analysis.md`](finance/analysis.md) | Symptoms documented |
| Code issues | [`code-issues.md`](code-issues.md) | Root causes + broken code |
| Code fixes | [`code-fixes.md`](code-fixes.md) | Fixes with before/after code |

## Findings

### 1. Sunset — March / revenue mismatch

- UI and API matched each other
- Beach House Alpha (`prop-001`) did **not** match the DB: UI showed **1,000.00 / 3 bookings**, DB had **2,250.000 / 4 bookings**
- Missing amount lined up with reservation `res-tz-1` (1250)
- Sunset’s dropdown also showed Ocean properties: Lakeside Cottage (`prop-004`), Urban Loft Modern (`prop-005`)

**Fix applied:** real Postgres connection + removed mock totals. Beach House Alpha now returns **2250 / 4**. Details in [`code-fixes.md`](code-fixes.md) (Bug 2). Commit: chore(backend): refactor database pool initialization and standardize reservation queries.

Timezone note: `res-tz-1` is March 1 in Paris. The live API sums all bookings (not a separate monthly query), so including that row fixed the March gap without changing timezone code. Details in [`code-fixes.md`](code-fixes.md) (Bug 4).

### 2. Ocean — privacy / wrong company data

- Ocean saw the **same numbers as Sunset** for every property
- Beach House Alpha (`prop-001`) data showed up for Ocean; in the DB Ocean’s `prop-001` is Mountain Lodge Beta with **0** reservations
- Ocean also saw Sunset properties: City Apartment Downtown (`prop-002`), Country Villa Estate (`prop-003`)

**Fix applied:** tenant in Redis cache key + tenant-scoped property list (Mountain Lodge for Ocean). Refresh no longer leaks totals across clients. Details in [`code-fixes.md`](code-fixes.md) (Bug 1). Commit: chore: add tenant context handling and property filtering for multi-tenant support.

### 3. Finance — cents off

- Amounts are stored with 3 decimal places
- API converted totals with `float(...)`
- UI used `Math.round` on floats — bad path for money

**Fix applied:** API returns money as a Decimal-quantized string; UI formats with a string helper (no float). Details in [`code-fixes.md`](code-fixes.md) (Bug 3). Commit: chore(frontend/backend): implement precise decimal handling for revenue values.

## Evidence

Checked three places for each property:

1. UI screenshots (in each folder’s `assets/`)
2. `GET /api/v1/dashboard/summary` with each client token
3. Postgres reservation totals

## Root causes (code path)

Traced: [`Dashboard`](../frontend/src/components/Dashboard.tsx) → [`RevenueSummary`](../frontend/src/components/RevenueSummary.tsx) → [`dashboard.py`](../backend/app/api/v1/dashboard.py) → [`cache.py`](../backend/app/services/cache.py) → [`reservations.py`](../backend/app/services/reservations.py).

| Bug | Root cause |
|---|---|
| Ocean leak | Redis key was `revenue:{property_id}` with no tenant; dropdown listed everyone’s properties. **Fixed** — see [`code-fixes.md`](code-fixes.md). |
| Sunset wrong total | [`database_pool.py`](../backend/app/core/database_pool.py) used missing `supabase_db_*` settings. Query failed → mock returned 1000 / 3. **Fixed** — see [`code-fixes.md`](code-fixes.md). |
| Finance cents | [`dashboard.py`](../backend/app/api/v1/dashboard.py) used `float(...)`; [`RevenueSummary.tsx`](../frontend/src/components/RevenueSummary.tsx) used `Math.round`. **Fixed** — see [`code-fixes.md`](code-fixes.md). |
| Dates / March | Live path is total revenue; after DB fix, `res-tz-1` is included. No timezone code change needed — see [`code-fixes.md`](code-fixes.md) (Bug 4). |

Auth is fine — JWTs already carry the right `tenant_id` for Sunset / Ocean.

## Result after fixes

| Check | Before | After |
|---|---|---|
| Sunset — Beach House Alpha (`prop-001`) | 1000 / 3 | **2250 / 4** (`"2250.00"`) |
| Ocean vs Sunset numbers | identical (mock/cache) | **isolated** (Sunset 2250 / 4, Ocean 0 / 0) |
| Finance totals vs DB | float / rounding risk | **string totals** match DB at 2 dp |
| March / `res-tz-1` | missing from UI total | included; Paris local time is March 1 |
