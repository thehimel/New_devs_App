# Code Issues

Simple notes on what is broken in the code, where it is, and how to fix it.

## Code path I traced

[`Dashboard.tsx`](../frontend/src/components/Dashboard.tsx) → [`RevenueSummary.tsx`](../frontend/src/components/RevenueSummary.tsx) → [`dashboard.py`](../backend/app/api/v1/dashboard.py) → [`cache.py`](../backend/app/services/cache.py) → [`reservations.py`](../backend/app/services/reservations.py)

---

## Bug 1 — Ocean sees Sunset’s numbers (privacy)

### Root cause

The cache saves revenue with only the property id. It does **not** include which company (tenant) asked for it. The cache key needs the company id too.

Two companies share the same id `prop-001`:

- Sunset → Beach House Alpha
- Ocean → Mountain Lodge Beta

So one company can get the other company’s saved answer.

File: [`backend/app/services/cache.py`](../backend/app/services/cache.py#L13) (line 13)

```python
cache_key = f"revenue:{property_id}"
```

Same problem on the screen list. The dropdown shows **all** properties for everyone, not only that client’s properties. It should list only that client’s homes.

File: [`frontend/src/components/Dashboard.tsx`](../frontend/src/components/Dashboard.tsx#L4-L10) (lines 4–10)

```tsx
const PROPERTIES = [
  { id: 'prop-001', name: 'Beach House Alpha' },
  { id: 'prop-002', name: 'City Apartment Downtown' },
  { id: 'prop-003', name: 'Country Villa Estate' },
  { id: 'prop-004', name: 'Lakeside Cottage' },
  { id: 'prop-005', name: 'Urban Loft Modern' }
];
```

### What to change

1. Put the tenant in the cache key, for example: `revenue:{tenant_id}:{property_id}`
2. Only show properties that belong to the logged-in client

---

## Bug 2 — Sunset’s Beach House Alpha total is wrong

### Root cause

The app tries to talk to the database with the wrong settings (`supabase_db_*`). Those settings are not set in this project. The real database URL from [`docker-compose.yml`](../docker-compose.yml) is not used.

File: [`backend/app/core/database_pool.py`](../backend/app/core/database_pool.py#L18) (line 18)

```python
database_url = f"postgresql+asyncpg://{settings.supabase_db_user}:{settings.supabase_db_password}@{settings.supabase_db_host}:{settings.supabase_db_port}/{settings.supabase_db_name}"
```

When that fails, the code does not stop. It returns **fake** numbers instead. For Beach House Alpha (`prop-001`) the fake total is `1000` with `3` bookings. The real database has `2250` with `4` bookings. The app should use the real DB, not fake numbers.

File: [`backend/app/services/reservations.py`](../backend/app/services/reservations.py#L93-L99) (lines 93–99)

```python
mock_data = {
    'prop-001': {'total': '1000.00', 'count': 3},
    'prop-002': {'total': '4975.50', 'count': 4},
    'prop-003': {'total': '6100.50', 'count': 2},
    'prop-004': {'total': '1776.50', 'count': 4},
    'prop-005': {'total': '3256.00', 'count': 3}
}
```

The fake map also only uses `property_id`, so both companies get the same fake answer for the same id.

### What to change

1. Connect with the real `DATABASE_URL` / local Postgres from [`docker-compose.yml`](../docker-compose.yml)
2. Stop returning mock totals when the database works
3. Always filter by both `property_id` **and** `tenant_id`

---

## Bug 3 — Money is a few cents off (Finance)

### Root cause

Money should stay exact (like counting coins). The API turns the total into a `float` (a number that can be a tiny bit wrong). Then the UI rounds it again. Do not use float for money.

File: [`backend/app/api/v1/dashboard.py`](../backend/app/api/v1/dashboard.py#L18) (line 18)

```python
total_revenue_float = float(revenue_data['total'])
```

File: [`frontend/src/components/RevenueSummary.tsx`](../frontend/src/components/RevenueSummary.tsx#L64) (line 64)

```tsx
const displayTotal = Math.round(data.total_revenue * 100) / 100;
```

### What to change

1. Keep money as `Decimal` (or as a string / integer cents) in the API — do not use `float`
2. On the UI, format the number for display without turning it into a risky float first

---

## Bug 4 — March dates / timezones

### Root cause

Some bookings are stored in UTC. Properties have their own timezone (for example Paris). The monthly helper builds month start/end in plain datetime with **no** timezone, and it is not fully wired yet. Month boundaries should use the property timezone.

File: [`backend/app/services/reservations.py`](../backend/app/services/reservations.py#L10) (lines 10 and 32)

```python
start_date = datetime(year, month, 1)  # line 10
# ...
return Decimal('0') # line 32 — Placeholder for now until DB connection is finalized
```

Example: reservation `res-tz-1` is `2024-02-29 23:30:00+00`. In Paris that can count as March. If we group by UTC month only, March totals can look wrong.

### What I concluded

No code change on the live path. The dashboard uses **total** revenue, not `calculate_monthly_revenue`. After the DB fix, `res-tz-1` is included in Beach House Alpha’s total (**2250 / 4**), and in Paris that booking is March 1. See [`code-fixes.md`](code-fixes.md) (Bug 4).

If monthly filtering gets wired later, convert check-in times to the property timezone before deciding the month, and use a real database query instead of the placeholder `0`.
