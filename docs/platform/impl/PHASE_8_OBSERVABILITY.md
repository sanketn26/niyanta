# Phase 8: Observability Distribution

**Goal**: Operators can measure and alert on engine health without turning the console into a mandatory stateful dependency.

## Production Default

Coordinator and workers expose Prometheus `/metrics`. Shipped Prometheus rules and Grafana dashboards are the production default. The console may query a configured Prometheus endpoint for charts, but activity inspection remains functional when metrics or tracing are unavailable.

Operational hot lists—parked activities, nondeterminism failures, starving activities, expiring leases—come from authorized, indexed engine APIs. They are state queries, not reconstructed from metrics.

## Cardinality Contract

- Do not label general-purpose metrics with activity IDs, worker-generated free text, raw errors, or other unbounded values.
- Raw customer IDs are not default labels. Per-tenant views use authorized state queries or a deliberately bounded/hashed aggregation with a published series budget.
- Bound activity-type/version labels by configured registry limits and publish the worst-case series calculation.
- Define histogram buckets from measured SLOs and test scrape/query cost under the Phase 9 load shape.

## Metric Set

- Attempt outcomes and activity state transitions.
- Queue depth and oldest age by bounded priority class.
- Dispatch, replay, timer-sweep, and signal-delivery latency.
- Replayed-call counts and bounded call-log size distribution.
- Worker capacity, utilization, and heartbeat freshness.
- Fenced writes, parked incompatible versions, nondeterminism failures, and database pool health.

## Alerts And Tracing

Ship rules for leader absence, dispatch/starvation SLOs, worker loss, parked versions, nondeterminism, timer lag, fencing anomalies, and database saturation. OpenTelemetry spans link submission, scheduling, attempts, and suspend points. Trace links appear only when a backend is configured.

## Optional Embedded Metrics Profile

Implement only if user evidence justifies it. It is a single-console convenience profile, not the production HA path. Before shipping, specify:

- Persistent-volume and disk/memory budgets.
- Byte and time retention; query concurrency/time limits.
- WAL crash recovery and upgrade compatibility.
- Target discovery and stale-target cleanup.
- Disk-full behavior that drops/degrades metrics without affecting execution.
- Clear prohibition or coordination model for multiple console scrapers.

## Acceptance Tests

1. Staged coordinator, worker, queue, and database failures trigger the shipped Prometheus rules.
2. Cardinality and query latency remain inside the published budget under representative load.
3. Tenant-scoped charts cannot expose another tenant's identifiers or series.
4. Loss of Prometheus, tracing, or the optional local store does not affect activity execution or state inspection.
5. If the embedded profile ships, crash/WAL recovery, disk-full, retention, and version-upgrade tests pass.

## Done When

- Metrics have a bounded label contract and operational alerts are executable as code.
- The engine and read-only console remain useful when the observability stack is down.
