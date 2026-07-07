# Phase 7: HA, Scale, Hardening

**Goal**: Coordinator HA, optional broker substrates, determinism lint, chaos testing, and load-tested scale to the targets in [../ARCHITECTURE.md](../ARCHITECTURE.md) §Performance Targets. Open-ended; pick items by measured need.

## Coordinator HA

- `LeaderElector` — choose **one** mechanism (Postgres advisory lock *or* etcd) and align all docs.
- Followers serve API + console reads; leader runs scheduler, health monitor, timer sweep, signal listener.
- Generation fencing (Phases 1–2) already prevents split-brain; HA adds failover tests under leader churn: kill the leader → failover < 30s, no progress lost, **no duplicate child dispatch** (the exactly-once invariant holds across failover).

## Optional Broker Substrates

Only if measured Postgres dispatch throughput demands it:

- NATS JetStream `SignalBus` (removes the LISTEN/NOTIFY single-listener constraint).
- JetStream `TaskQueue` behind the same interface.

Adoption is a config change, not an activity-code change — that was the point of the Phase 0 interfaces.

## Determinism Linter (G2 / ADR-005)

Static analysis, CI-enforced, for any activity package that uses suspend points: flags `time.Now()`, `rand.*`, map-iteration-order dependence, goroutine spawns, direct I/O, and `Heartbeat`/`LastHeartbeat`.

## Sticky-Worker Replay Optimization

Prefer resuming a parent on the worker that last held it, with a bounded in-memory replay cache — cuts the per-suspend-point scheduler round-trip for hot compositions. Suspend-and-free remains the durable fallback; **correctness never depends on stickiness**.

## Chaos Testing

Random worker kills, leader kills, Postgres pauses/failovers, dropped notifications. Post-run integrity checker verifies: no lost activities; every completed call has exactly one child; no orphan children; fencing held (no stale write landed); signal delivery within SLO.

## Performance

- Load-test in **state transitions/sec** (each suspend point is several transactions), not concurrent-activity count. Publish the engine's transition-throughput envelope as part of its contract.
- Blob offload (`BlobStore`/S3) for heartbeat and result payloads above threshold, pointer in Postgres.
- Read replicas for console/list/audit queries; writes stay on the primary.
- Index tuning against real query patterns; partitioning for `audit_events`.

## Acceptance Tests

1. Leader kill → failover < 30s, zero lost progress, zero duplicate dispatch.
2. Sustained 1,000 activities/min for 24h with zero correctness errors.
3. Determinism lint catches a deliberately-broken activity in CI.
4. 1-hour chaos run passes the integrity checker; SLOs hold.
5. Kubernetes HPA scales workers on the queue-depth metric, up and down.

## Done When

- The architecture performance targets are met under load test and the envelope is documented.
- Every invariant the engine claims survives the chaos suite.
