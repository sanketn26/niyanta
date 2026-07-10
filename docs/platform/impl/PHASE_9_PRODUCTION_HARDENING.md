# Phase 9: Production Hardening Gate

**Goal**: Qualify Niyanta's durability, isolation, and operational claims with evidence. This phase exits on acceptance results, not a calendar date.

## High Availability And Recovery

- Use Postgres advisory-lock leadership by default unless testing demonstrates a concrete need for etcd. Followers serve APIs; only the leader schedules, sweeps timers, and monitors worker health.
- Test leader churn and mixed coordinator versions against generation fencing and exactly-once child dispatch.
- Define PostgreSQL backup, WAL archiving, point-in-time restore, RPO, and RTO. Run automated restore drills and validate restored state with the integrity checker.
- Console or observability loss must not block scheduling or execution.

## Security Gate

- Threat model trust boundaries: browser, console adapter, coordinator API, workers, database, metrics/tracing, and tenant payloads.
- Authorization-matrix and cross-tenant tests for REST, SSE, metrics views, exports, logs, traces, and saved preferences.
- Validate redaction, sensitive-read audit, sessions, CSRF, CSP, secrets, least-privilege database roles, TLS, and network policy.
- Run dependency, image, and configuration scans with documented severity policy and exception process.

## Upgrade And Rollback

- Publish supported coordinator/worker/console/database schema skew.
- Test rolling upgrade and rollback with running, suspended, version-pinned, signaled, and `ContinueAsNew` activities.
- Prefer expand/migrate/contract migrations; destructive schema contraction occurs only after the supported rollback window.

## Chaos, Integrity, And Capacity

- Inject worker/leader kills, stale workers, Postgres pauses/failover, notification loss, console loss, and observability failure.
- Post-run integrity checker verifies no lost activities, exactly-one child link per completed `RunChild`, no orphan children, fencing, and signal invariants.
- Load in state transitions/sec plus API/console query concurrency. Run a representative 24-hour soak, tune indexes/pools/pagination, and publish the supported envelope and backpressure behavior.
- Add read replicas, blob offload, sticky replay, or a broker only when profiles show a specific bottleneck.
- Enforce the determinism linter in CI.

## Acceptance Tests

1. Leader kill meets the published failover SLO with no lost progress or duplicate child dispatch.
2. Point-in-time restore meets RPO/RTO and passes the integrity checker.
3. One-version rolling upgrade and rollback succeed with live suspended work.
4. One-hour chaos and 24-hour soak complete with zero invariant violations.
5. Cross-tenant and privilege-escalation suites pass across every exposed surface.
6. Capacity, cardinality, retention, version skew, and operational limits are published.

## Done When

- Production claims are backed by repeatable test artifacts and runbooks.
- Unsupported scale is rejected or backpressured predictably rather than accepted on hope.
