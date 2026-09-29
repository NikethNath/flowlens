# FlowLens

**A process-mining workbench.** Upload an event log (CSV or XES) and FlowLens shows you how your process *actually* runs. It draws the real process map with frequency and waiting-time overlays, ranks process variants and bottlenecks, and flags the cases that break your business rules.

> **Status: in active development.** This README describes the planned design. Measured numbers and screenshots are added only once they come from real runs.

## What it does
- **Ingest** CSV (with column mapping) or XES / XES.gz event logs. The XES parser streams, so memory stays flat on large logs.
- **Discover** the directly-follows process map: frequency and performance (median / p90 wait) per handover, with start and end activities.
- **Filter** with activity and path sliders (Disco-style pruning that never leaves an activity disconnected) and a case date range.
- **Variants:** the distinct paths cases take, with share, cumulative coverage and median duration.
- **Bottlenecks:** handovers ranked by the total waiting time they cost.
- **Conformance rules** (Declare-style): `existence`, `absence`, `exactly_once`, `response`, `precedence`, `not_succession` and `max_duration`, each returning the violating cases.
- **Case timeline:** drill into any single case.

## Stack
| Layer | Tech |
|---|---|
| Backend | Python 3.12, FastAPI, Pydantic, DuckDB (process discovery as SQL window functions), lxml |
| Frontend | React, TypeScript, Redux Toolkit + RTK Query, React Flow + dagre, Vite |
| Testing | pytest (with a **pm4py parity** check on a real 561K-event public log), Jest + React Testing Library, Cypress e2e |
| Delivery | Docker (non-root, arbitrary-UID / OpenShift-compatible images), docker compose, Kubernetes manifests (kustomize), GitHub Actions with a kind cluster smoke test, GHCR |

## How it works (short version)
The whole directly-follows graph is a single SQL window query over the event table:
```sql
SELECT activity, LEAD(activity) OVER w AS next_activity, LEAD(ts) OVER w - ts AS wait
FROM events WINDOW w AS (PARTITION BY case_id ORDER BY ts, seq)
```
Then it groups by `(activity, next_activity)` for frequency and median / p90 wait. Variants are the per-case ordered activity lists, grouped. Conformance rules are one SQL query per template, returning the violating case ids.

## API
FlowLens is API-first. The UI is just one client of a typed OpenAPI contract (`/docs` when running). The main routes are `POST /api/logs`, `GET /api/logs/{id}/dfg`, `/variants`, `/stats`, `/bottlenecks`, `/cases/{case_id}`, `/rules` and `/conformance`. The companion project [SpecProbe](https://github.com/NikethNath/specprobe) runs automated API tests against this contract in CI.

## Datasets
- `samples/order_to_cash_small.csv` is a seeded synthetic order-to-cash log with planted bottlenecks and rule violations, and it ships with a ground-truth file.
- **BPI Road Traffic Fine Management Process** (4TU.ResearchData, CC BY 4.0) is downloaded by script and never committed.

## Roadmap
1. Event-log store, CSV and XES ingest
2. DFG discovery, variants, stats, pm4py parity and benchmarks
3. Path pruning, conformance rules, bottlenecks
4. React + Redux UI: upload, process map, variants, bottlenecks, rules
5. Cypress e2e, Docker images, Kubernetes deploy and kind CI

## References
W. M. P. van der Aalst, *Process Mining: Data Science in Action* (2016) · pm4py · Declare (Pesic & van der Aalst) · BPI Challenge datasets (4TU.ResearchData)

## License
MIT
