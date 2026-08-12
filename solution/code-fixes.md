# Code Fixes

Pair with [`code-issues.md`](code-issues.md). Same bug numbers, side by side.

---

## Bug 1 — Ocean sees Sunset’s numbers (privacy)

Commit: chore: add tenant context handling and property filtering for multi-tenant support

### What I changed

1. Put `tenant_id` in the Redis cache key
2. Require a real tenant on the dashboard API (no `default_tenant` fallback)
3. Show only that client’s properties in the dropdown (correct names too)

### Fix in [`backend/app/services/cache.py`](../backend/app/services/cache.py)

Before (line 13):

```python
cache_key = f"revenue:{property_id}"
```

After (line 13):

```python
cache_key = f"revenue:{tenant_id}:{property_id}"
```

### Fix in [`backend/app/api/v1/dashboard.py`](../backend/app/api/v1/dashboard.py)

Before:

```python
tenant_id = getattr(current_user, "tenant_id", "default_tenant") or "default_tenant"
```

After (lines 14–16):

```python
tenant_id = getattr(current_user, "tenant_id", None)
if not tenant_id:
    raise HTTPException(status_code=403, detail="Tenant context required")
```

### Fix in [`frontend/src/components/Dashboard.tsx`](../frontend/src/components/Dashboard.tsx)

Before: one hardcoded list of all 5 properties for every user.

After (lines 5–28): properties come from the logged-in tenant.

```tsx
const PROPERTIES_BY_TENANT: Record<string, { id: string; name: string }[]> = {
  "tenant-a": [
    { id: "prop-001", name: "Beach House Alpha" },
    { id: "prop-002", name: "City Apartment Downtown" },
    { id: "prop-003", name: "Country Villa Estate" },
  ],
  "tenant-b": [
    { id: "prop-001", name: "Mountain Lodge Beta" },
    { id: "prop-004", name: "Lakeside Cottage" },
    { id: "prop-005", name: "Urban Loft Modern" },
  ],
};

const { user } = useAuth();
const tenantId =
  user?.tenant_id ||
  (user as { app_metadata?: { tenant_id?: string } } | null)?.app_metadata?.tenant_id ||
  null;

const properties = useMemo(
  () => (tenantId ? PROPERTIES_BY_TENANT[tenantId] ?? [] : []),
  [tenantId]
);
```

Also updated [`frontend/src/utils/jwtUtils.ts`](../frontend/src/utils/jwtUtils.ts) so JWT tenant is read from `app_metadata.tenant_id`, and [`frontend/src/contexts/AuthContext.new.tsx`](../frontend/src/contexts/AuthContext.new.tsx) so an existing `user.tenant_id` from login is kept.

### Check

| Order | Sunset `prop-001` | Ocean `prop-001` |
|---|---|---|
| Sunset first, then Ocean | 2250 / 4 | **0 / 0** |
| Ocean first, then Sunset | **2250 / 4** | 0 / 0 |

Redis keys after both calls: `revenue:tenant-a:prop-001` and `revenue:tenant-b:prop-001`.

---

## Bug 2 — Sunset’s Beach House Alpha total is wrong

Commit: chore(backend): refactor database pool initialization and standardize reservation queries

### What I changed

1. Connect with the real database URL from [`docker-compose.yml`](../docker-compose.yml) / `settings.database_url`
2. Removed the fake mock totals
3. Keep filtering by both `property_id` and `tenant_id`

### Fix in [`backend/app/core/database_pool.py`](../backend/app/core/database_pool.py)

Before: built a URL from missing `supabase_db_*` settings.

After (lines 8–16, 30):

```python
def _async_database_url(url: str) -> str:
    """SQLAlchemy async needs the asyncpg driver in the URL."""
    if url.startswith("postgresql+asyncpg://"):
        return url
    if url.startswith("postgresql://"):
        return url.replace("postgresql://", "postgresql+asyncpg://", 1)
    if url.startswith("postgres://"):
        return url.replace("postgres://", "postgresql+asyncpg://", 1)
    return url

# ...

database_url = _async_database_url(settings.database_url)
```

### Fix in [`backend/app/services/reservations.py`](../backend/app/services/reservations.py)

Before: on any DB error, return a hardcoded mock map (Beach House Alpha → 1000 / 3).

After (lines 36–83): use the shared pool and real query only. No mock.

```python
async def calculate_total_revenue(property_id: str, tenant_id: str) -> Dict[str, Any]:
    """
    Aggregates revenue from database.
    """
    from app.core.database_pool import db_pool
    from sqlalchemy import text

    await db_pool.initialize()

    if not db_pool.session_factory:
        raise RuntimeError(
            f"Database pool not available for {property_id} (tenant: {tenant_id})"
        )

    async with db_pool.get_session() as session:
        query = text("""
            SELECT
                property_id,
                SUM(total_amount) as total_revenue,
                COUNT(*) as reservation_count
            FROM reservations
            WHERE property_id = :property_id AND tenant_id = :tenant_id
            GROUP BY property_id
        """)

        result = await session.execute(query, {
            "property_id": property_id,
            "tenant_id": tenant_id,
        })
        row = result.fetchone()

        if row:
            total_revenue = Decimal(str(row.total_revenue))
            return {
                "property_id": property_id,
                "tenant_id": tenant_id,
                "total": str(total_revenue),
                "currency": "USD",
                "count": row.reservation_count,
            }

        return {
            "property_id": property_id,
            "tenant_id": tenant_id,
            "total": "0.00",
            "currency": "USD",
            "count": 0,
        }
```

### Check

| Client | Property | Before | After |
|---|---|---|---|
| Sunset | Beach House Alpha (`prop-001`) | 1000 / 3 (mock) | **2250 / 4** (DB) |
| Ocean | Mountain Lodge Beta (`prop-001`) | 1000 / 3 (mock) | **0 / 0** (DB) |

---

## Bug 3 — Money is a few cents off (Finance)

Commit: chore(frontend/backend): implement precise decimal handling for revenue values

### What I changed

1. Keep money as `Decimal` in the API — return a string, never `float`
2. Format on the UI with a string helper — no `Math.round` on floats

### Fix in [`backend/app/api/v1/dashboard.py`](../backend/app/api/v1/dashboard.py)

Before:

```python
total_revenue_float = float(revenue_data['total'])

return {
    "total_revenue": total_revenue_float,
    ...
}
```

After (lines 21–28):

```python
# Keep money exact — never use float
total = Decimal(str(revenue_data["total"])).quantize(
    Decimal("0.01"), rounding=ROUND_HALF_UP
)

return {
    "property_id": revenue_data['property_id'],
    "total_revenue": format(total, "f"),
    "currency": revenue_data['currency'],
    "reservations_count": revenue_data['count']
}
```

### Fix in [`frontend/src/components/RevenueSummary.tsx`](../frontend/src/components/RevenueSummary.tsx)

Before:

```tsx
const displayTotal = Math.round(data.total_revenue * 100) / 100;
```

After: `formatMoney()` works on the string/digits (half-up to 2 decimals, no float). Display uses that string directly.

```tsx
function formatMoney(amount: string | number): string {
  // round half-up to 2 decimal places using digits / BigInt — no float Math.round
  ...
}

const displayTotal = formatMoney(data.total_revenue);
// ...
{data.currency} {displayTotal}
```

### Check

| Property | API `total_revenue` | Type | Matches DB (2 dp) |
|---|---|---|---|
| Beach House Alpha (`prop-001`) | `"2250.00"` | string | yes |
| City Apartment Downtown (`prop-002`) | `"4975.50"` | string | yes |
| Country Villa Estate (`prop-003`) | `"6100.50"` | string | yes |

---

## Bug 4 — March dates / timezones

### Decision

No code change on this path.

The live dashboard uses **total** revenue (`calculate_total_revenue`), not the monthly helper. After the DB fix (Bug 2), Beach House Alpha includes `res-tz-1` (**1250**), so the total is **2250.00 / 4**.

`res-tz-1` is `2024-02-29 23:30:00+00` UTC = **2024-03-01 00:30** in `Europe/Paris`. Counting it in Sunset’s March books matches the property timezone. The unused `calculate_monthly_revenue` placeholder was not wired to the dashboard, so it did not need a fix for the reported UI issue.

### Check

| Item | Result |
|---|---|
| Beach House Alpha total | `"2250.00"` / 4 (includes `res-tz-1`) |
| `res-tz-1` in Paris | 2024-03-01 00:30 |
