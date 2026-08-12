# Analysis for Finance Team

No separate UI login for finance. I checked the same dashboard numbers, the API, and the reservation amounts in the DB.

## What finance reported

Totals look “slightly off” by a few cents.

## What I checked

Beach House Alpha (`prop-001`, Sunset) has these amounts in the DB:

| Reservation | Amount |
|---|---|
| res-dec-1 | 333.333 |
| res-dec-2 | 333.333 |
| res-dec-3 | 333.334 |
| **sum** | **1000.000** |

In Decimal math that adds up cleanly. The API turns the total into a float (`float(...)`), and the UI rounds it for display.

## Comparison

| Source | Beach House Alpha (`prop-001`) total | Notes |
|---|---|---|
| DB (all 4 rows) | 2250.000 | includes res-tz-1 (1250) |
| DB (3 x 333.xxx only) | 1000.000 | exact in Decimal |
| API / UI | 1000.0 → shown as 1,000.00 | float + display rounding |

## Notes

- Money is stored with 3 decimal places (`NUMERIC(10,3)`), then converted to float in the API. That’s a bad fit for money and can create small cent-level drift.
- On top of that, Beach House Alpha (`prop-001`) is already missing a whole reservation in the UI (1250), so the total is wrong by more than cents there.
- Need to keep amounts as Decimal (or cents as integers) end to end, not float.
