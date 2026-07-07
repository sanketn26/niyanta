# Phase 6: Activity Manager Console + Integrated Observability

**Goal**: A human can operate the engine from a browser: find any activity, see exactly where it is (attempts, call log, composition tree, signals), act on it, and see cluster and workload health. Observability is **integrated** — the console ships with its own metrics pipeline and works out of the box with zero external observability infrastructure, while remaining fully Prometheus-compatible for teams that already run a stack.

## Architecture

```
┌───────────────────────────────────────────────────────────────┐
│                    cmd/console (single binary)                 │
│  ┌──────────────┐  ┌───────────────────┐  ┌────────────────┐ │
│  │ SPA (React+TS │  │ Read API + actions │  │ Integrated     │ │
│  │  go:embed)    │  │ (RBAC, audit, SSE) │  │ metrics store  │ │
│  └──────────────┘  └─────────┬─────────┘  └───────┬────────┘ │
└──────────────────────────────┼────────────────────┼──────────┘
                               │                    │ scrapes /metrics
                               ▼                    ▼
                         Postgres (state)    coordinator + workers
```

- **Integrated metrics store**: the console embeds a scraper and a local TSDB (Prometheus TSDB library, bounded retention, e.g. 15 days default) that scrapes the coordinator's and workers' `/metrics` endpoints itself. Charts, hot lists, and alert evaluation work with **no external Prometheus deployed**.
- **Prometheus-compatible by construction**: every component exposes standard `/metrics`; the console's query layer speaks PromQL. If the operator configures an external Prometheus URL, the console federates to it instead of (or alongside) its embedded store — same charts, same queries. Alert rules ship as standard Prometheus rule files either way.
- **State reads** come from Postgres via the Phase 5 read API; **actions** go through the Phase 5 endpoints (RBAC: viewer / operator / admin; every action audited; destructive ones need typed confirmation).
- **Live updates** over SSE — watching an activity resume requires no refresh.
- Single static binary via `go:embed`; no CDN assets; works air-gapped.

## Metric Set (exposed by coordinator/workers, consumed by the console)

- Activities by `{customer, activity_type, status}`; attempt outcomes.
- Scheduler queue depth + age by priority; dispatch latency histogram.
- Replay duration histogram; replayed-call counts; call-log size distribution.
- Timer-sweep lag; signal delivery counts; mailbox depth.
- Worker capacity/utilization; heartbeat freshness.
- **`niyanta_fenced_writes_total`** (G3 — a nonzero rate is the trail of a redistribution race).
- **`niyanta_parked_no_compatible_worker`** gauge (G1).
- **`niyanta_nondeterminism_failures_total`** (G6).

## Shipped Alert Rules (`deploy/prometheus/alerts.yaml`)

Leader absent · dispatch-latency SLO breach · dead-worker rate · parked-on-version > 0 for 15m · nondeterminism failures > 0 · queue starvation · timer-sweep lag · DB pool saturation. Evaluated by the integrated store out of the box; loadable into an external Prometheus unchanged.

## Views

1. **Cluster dashboard**: leader status, worker fleet (status, capacity, versions, heartbeat freshness, in-flight), queue depths/ages, live throughput/error sparklines, active alerts.
2. **Activities**: search/filter by status, type, tenant, version pin, time range, root. Saved hot lists: *parked on version* (G1), *nondeterminism failures* (G6), *starving*, *lease expiring*.
3. **Activity detail** (the flagship): status + timings; attempt timeline (worker, generation, error, duration per attempt); **call-log / replay timeline** (every `RunChild`/`Sleep`/`AwaitSignal`/`_now`/`_id` with state and recorded result — for a suspended activity this *is* "exactly where is it"); composition tree (navigable parent/children); signal mailbox; heartbeat state (leaf activities); `ContinueAsNew` execution chain; linked trace.
4. **Workers detail**: per-worker in-flight, capability/version set, lease table, drain button.
5. **Versions**: pinned-version inventory and drain progress.

## Operator Actions

Cancel · pause/resume · send signal · retry-now · force-fail · force-complete · truncate-replay · worker drain. All RBAC-gated, all audited, destructive ones behind typed confirmation.

## Tracing

OpenTelemetry spans: submission → schedule → attempt → each suspend-point call → completion; context propagated through `ActivityContext`; exemplars link latency histograms to traces. Jaeger/Tempo in the dev compose.

## Dev Environment

docker-compose: postgres + coordinator + 2 workers + console (+ optional prometheus + grafana + jaeger to exercise the external-stack path). `deploy/grafana/*.json` dashboards ship for teams that prefer Grafana for deep dives; the console remains sufficient on its own.

## Acceptance Tests

1. **Zero-infra observability**: compose file *without* prometheus/grafana — the console still renders throughput, latency, queue, and replay charts from its integrated store, and the shipped alert rules evaluate and fire.
2. External-stack parity: point the console at an external Prometheus — same charts, same hot lists, no code path divergence.
3. Find a suspended activity, see the call index it is parked on and what it awaits, send it a signal from the UI, watch it resume live (SSE).
4. Composition tree renders a 3-level pipeline with per-node status.
5. Kill a worker: dashboard shows it dead within the heartbeat timeout; the redistribution appears on the affected activity's attempt timeline; `niyanta_fenced_writes_total` ticks when the stale worker returns.
6. G1/G6 hot lists populate; the Phase 5 recovery drill is executable entirely from the UI, leaving an audit trail.
7. Viewer role sees no action buttons; operator actions are audited.

## Done When

- Any activity's exact position is explainable from the console in under a minute.
- Observability works with nothing deployed beyond the engine + console, and plugs into an existing Prometheus stack without change.
- Every operator action is authenticated, authorized, and audited.
