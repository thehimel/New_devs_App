# Steps I Followed to Solve the Problem

* Cloned the repository.
* Set up the environment. Configured the IDE. Notes in [`setup.md`](setup.md).
* Started the servers with Docker Compose ([`docker-compose.yml`](../docker-compose.yml)).
* Logged into the UI with both the clients.
* Saved the screenshots for all the properties for both clients in `sunset/assets/` and `ocean/assets/`.
* Analysed the results in the UI with the API and the DB. Notes in [`sunset/`](sunset/analysis.md), [`ocean/`](ocean/analysis.md), and [`finance/`](finance/analysis.md).
* Traced the code path ([`Dashboard.tsx`](../frontend/src/components/Dashboard.tsx) → [`RevenueSummary.tsx`](../frontend/src/components/RevenueSummary.tsx) → [`dashboard.py`](../backend/app/api/v1/dashboard.py) → [`cache.py`](../backend/app/services/cache.py) → [`reservations.py`](../backend/app/services/reservations.py)) and wrote down the root causes in [`README.md`](README.md) and [`code-issues.md`](code-issues.md).
* Fixed the database connection (Bug 2). Notes in [`code-fixes.md`](code-fixes.md). Commit: chore(backend): refactor database pool initialization and standardize reservation queries.
* Fixed tenant cache isolation + property list (Bug 1). Notes in [`code-fixes.md`](code-fixes.md). Commit: chore: add tenant context handling and property filtering for multi-tenant support.
* Fixed money float drift (Bug 3). Notes in [`code-fixes.md`](code-fixes.md). Commit: chore(frontend/backend): implement precise decimal handling for revenue values.
* Checked timezone / March (`res-tz-1`) — no extra code change needed on the live total-revenue path. Notes in [`code-fixes.md`](code-fixes.md) (Bug 4).
* Full retest with both clients vs DB — passed.
