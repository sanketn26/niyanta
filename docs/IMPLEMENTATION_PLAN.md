# Niyanta Implementation Plan

**Version**: 2.2
**Last Updated**: 2026-05-03
**Status**: Planning Phase
**Supersedes**: v1.0 (workload-based model). Reframed around the activity execution model defined in [ADR-005](platform/adr/ADR-005-activity-execution-model.md).

## Table of Contents
1. [Executive Summary](#executive-summary)
2. [Development Phases](#development-phases)
   - [Phase 0: End-to-end skeleton, in-memory only](#phase-0-end-to-end-skeleton-in-memory-only)
   - [Phase 1: Persistence and framework-managed retries](#phase-1-persistence-and-framework-managed-retries)
   - [Phase 2: Multi-process — coordinator + workers, NATS for control plane](#phase-2-multi-process--coordinator--workers-nats-for-control-plane)
   - [Phase 3: Sub-activities and replay-based resume](#phase-3-sub-activities-and-replay-based-resume)
   - [Phase 4: Deterministic primitives and signals](#phase-4-deterministic-primitives-and-signals)
   - [Phase 5: Production ingestion shape — connectors, planner, API, observability](#phase-5-production-ingestion-shape--connectors-planner-api-observability)
   - [Phase 5A: Connector packaging and compatibility foundation](#phase-5a-connector-packaging-and-compatibility-foundation)
   - [Phase 5B: Connector ecosystem breadth](#phase-5b-connector-ecosystem-breadth)
   - [Phase 6: HA, scale, hardening](#phase-6-ha-scale-hardening)
3. [Resource Requirements](#resource-requirements) (multi-engineer view; preserved from v1)
4. [Risk Management](#risk-management)
5. [Success Metrics](#success-metrics)
6. [Timeline & Milestones](#timeline--milestones)
7. [Post-Phase 6 Roadmap](#post-phase-6-roadmap)
8. [Appendix](#appendix)
9. [References](#references)
10. [Revision History](#revision-history)

---

## Executive Summary

This document outlines a phased approach to implementing Niyanta as a self-orchestrated ingestion platform backed by a distributed activity execution system. The plan is structured to deliver incremental value while managing technical risk, and is reframed around the activity model defined in [ADR-005](platform/adr/ADR-005-activity-execution-model.md). The ingestion control-plane shape is defined in [INGESTION_ARCHITECTURE.md](apps/dataconnector/INGESTION_ARCHITECTURE.md).

### Key Objectives

1. **Phase 1 MVP**: Single-attempt activities with framework-managed retries, persisted in Postgres. End-to-end: submit → execute → complete, with crash recovery and minimal attempt observability.
2. **Phase 2**: Multi-process (coordinator + workers over NATS), with worker failure detection, redistribution, and connector runtime capability advertising.
3. **Phase 3**: Sub-activities via `RunChild` with replay-based resume across `RunChild` boundaries — the mechanism needed for ingestion supervisors to compose plan/read/transform/deliver/commit steps.
4. **Phase 4**: Deterministic primitives (`ctx.Sleep`, `ctx.Now`, `ctx.NewID`) and `SignalBus` (Postgres-backed). After Phase 4 the self-orchestrated ingestion loop is expressible: supervisors can sleep, wake on signals, run child partitions, and resume after crashes.
5. **Phase 5**: Production ingestion shape — connector definitions/connections, ingestion planner, full REST API, multi-tenant auth, high-density shared execution, isolation policies, rate limiting, observability stack, audit, backpressure.
6. **Phase 6**: Coordinator HA, adaptive planner sharding, auto-scaling, NATS JetStream `SignalBus`, determinism linter, chaos testing, scale tuning to 10K concurrent activities.

### What changed from v1

- The `Workload` interface (with author-managed `Checkpoint()`) is replaced by the `Activity` interface with framework-managed retries, replay-based resume, and suspend-on-`RunChild`. Authors do not write retry loops or design checkpoint state in the common case.
- **Signals are now Phase 4, not Phase 3.** The agentic-monitor use case is unworkable without them; deferring is no longer an option. They are abstracted behind a `SignalBus` interface so the substrate (Postgres LISTEN/NOTIFY → NATS JetStream) can change without touching activity code.
- **Affinity and priority are deferred.** v1 made these Phase 1; the activity model is itself the Phase 1 lift, and squeezing both in produced an 8-week MVP with no real safety margin. Affinity moves to Phase 2 (alongside multi-worker, where it actually matters), priority to Phase 5 (alongside multi-tenant, where SLA tiers become meaningful).
- **Phases 0–4 are sized for a single engineer.** The team-size and cost columns from v1 reflected a multi-engineer org plan; that view is preserved in §[Resource Requirements](#resource-requirements) but the build sequence below assumes solo or small-team execution.

### Phases Overview

| Phase | Duration (solo) | Key Deliverables |
|-------|------|------------------|
| Phase 0 | 1 week | Project skeleton, all interfaces stubbed, in-memory impls, single-binary demo |
| Phase 1 | 2 weeks | Postgres-backed `Activity` execution. Framework-managed retries. Survives restart. |
| Phase 2 | 2 weeks | Coordinator + worker split, NATS broker, multi-worker, failure redistribution. Affinity. |
| Phase 3 | 3 weeks | `RunChild` + replay-based resume. Sequential sub-activities. |
| Phase 4 | 3 weeks | `ctx.Sleep`, `ctx.Now`, `ctx.NewID`. `SignalBus` interface + Postgres impl. `ctx.AwaitSignal`. |
| Phase 5 | 3 weeks | Connector API, ingestion planner, customer/auth, isolation policy, dense shared execution, observability, audit, priority scheduling, connector discovery/catalog foundation. |
| Phase 5A | 2 weeks | Connector packaging and compatibility foundation: import API, Airbyte manifest/image metadata preservation, command adapter skeleton, selected-catalog lifecycle. |
| Phase 5B | 3-4 weeks | Connector breadth: `database_source`, `cdc_source`, complex `destination_connector`, stream-provider expansion, and production-grade Airbyte adapter execution. |
| Phase 6 | open-ended | Coordinator HA, adaptive planner sharding, NATS JetStream `SignalBus`, auto-scaling, determinism lint, chaos, scale tuning. |

**Stop point for prototypes**: end of Phase 4. The activity substrate can express self-orchestrated ingestion + agentic monitors at this point, but connector registry/planner APIs arrive in Phase 5.

**Stop point for production foundation**: end of Phase 5. This supports the first native connector class and the production control plane. Airbyte-scale connector breadth arrives in Phase 5A/5B.

**Stop point for connector ecosystem parity**: end of Phase 5B. Phase 6 is incremental hardening.

**Total time to use-case parity (solo)**: ~11 weeks (Phases 0–4).
**Total time to production foundation (solo)**: ~14 weeks (Phases 0–5).
**Total time to connector ecosystem parity (solo)**: ~19-20 weeks (Phases 0–5B).

---

## Development Phases

The phases are designed so that **each one ends with a runnable system**. If you stop after phase N, what you have works for what phase N promises — no half-built state.

### Guiding principles

1. **Interfaces from day one.** Every storage and broker concern lives behind an interface starting in Phase 0. Phase 1's Postgres impl can be swapped for Phase 6's JetStream impl without touching activity code. Six storage interfaces (`ActivityStore`, `AttemptStore`, `HeartbeatStore`, `ActivityCallStore`, `WorkerRegistry`, `LeaderElector`), one queue (`TaskQueue`), one signal bus (`SignalBus`), one event bus (`EventBus`). See [ADR-005](platform/adr/ADR-005-activity-execution-model.md).
2. **In-memory before external.** Phase 0 ships an in-memory implementation of every interface. End-to-end runs before any external dependency is introduced; design problems surface before infrastructure problems can hide them.
3. **Vertical slices.** Don't build "all of storage" then "all of scheduler." Build one activity end-to-end (submit → schedule → execute → complete), then add features to that slice phase by phase.
4. **Test what's invisible.** Replay, retry, and resume are invisible when they work. Failing tests come first: kill a worker mid-activity, drop a signal, time out a heartbeat. Make them pass.

---

### Phase 0: End-to-end skeleton, in-memory only

**Goal**: One process. One activity. Runs to completion. Everything behind interfaces. Zero external dependencies.

**Duration**: 1 week (solo)

**Key Deliverables**:
- Project skeleton per [impl/PHASE_0_SKELETON.md](platform/impl/PHASE_0_SKELETON.md). Makefile (`build`, `test`, `lint`, `run`). golangci-lint config.
- Domain models in `pkg/models/` — `Activity` (renamed from `Workload`), `ActivityAttempt`, `ActivityCall` (placeholder, populated in Phase 3), `Worker`, `Customer`, status enums.
- `Activity` interface and `ActivityContext` in `pkg/activity/` with all methods declared from day one. `RunChild`, `Sleep`, `Now`, `NewID`, `AwaitSignal` stub-return `ErrNotImplemented` until their phases.
- All eight storage/queue/bus interfaces declared in `internal/storage/`, `internal/broker/`, `internal/signal/`.
- In-memory implementations of all interfaces (`internal/storage/memory/`, etc.) with shared contract tests.
- Activity registry (`internal/registry/`) — type-name → factory function. Sample activity: `sleep`.
- Single-process executor (`internal/executor/`) — pull from `TaskQueue`, run, write result. No retries yet, one at a time.
- CLI (`cmd/niyanta/main.go`) with `submit`, `run`, `get` subcommands; Phase 0's demo runs all three in one binary.

**Success Criteria**:
- `niyanta demo sleep 2s` submits, runs, and reports completion using only in-memory state.
- All interfaces have at least one impl and contract tests passing.
- `go test ./...` and `golangci-lint run` clean.

**Explicitly out of scope**: Postgres, NATS, HTTP, retries, `RunChild`, signals, multi-tenant, observability beyond stdlib `log`.

---

### Phase 1: Persistence and framework-managed retries

**Goal**: Same end-to-end flow as Phase 0, but the activity survives a process restart and retries on transient failure under a declared policy.

**Duration**: 2 weeks (solo)

**Key Deliverables**:
- Postgres implementations of `ActivityStore`, `AttemptStore`, `HeartbeatStore` in `internal/storage/postgres/`. `pgx/v5` directly, no ORM. Connection pool config in `internal/config/`.
- Migrations via `golang-migrate`. Initial migration: `activities`, `activity_attempts`, `activity_heartbeats`, `workers`. Schema starts simpler than [DATA_MODELS.md](platform/DATA_MODELS.md) — only what Phase 1 uses; later phases extend.
- Postgres-polling `TaskQueue`. Pending attempts queried with `SELECT … FOR UPDATE SKIP LOCKED LIMIT 1` every 2s. Crude but adequate.
- `RetryPolicy` declared per registered activity type: `MaxAttempts`, `InitialInterval`, `BackoffCoefficient`, `MaxInterval`, `NonRetryableErrors`. Framework enforces — activity code never retries itself.
- Typed errors: `RetryableError`, `NonRetryableError`, classification helpers.
- Per-attempt records (`activity_attempts` rows, not a `retry_count` column) — replaces the integer counter from v1's data model.
- `ctx.Heartbeat(state)` writes to `activity_heartbeats`; `ctx.LastHeartbeat()` reads the latest for the current attempt.
- Crash recovery on executor startup: claim any `state='RUNNING' AND worker_id=<self>` rows from a previous run, mark `INTERRUPTED`, enqueue a fresh attempt.

**Success Criteria**:
- Submit a `flaky_sleep` activity (fails attempts 1 and 2, succeeds attempt 3). `psql` shows three `activity_attempts` rows; final state is `COMPLETED`.
- Kill the executor mid-execution. Restart. Interrupted attempt is marked, new attempt enqueued, completes.
- Migration up/down round-trip clean.

**Explicitly out of scope**: Multi-worker, separate coordinator, NATS, `RunChild`, signals.

---

### Phase 2: Multi-process — coordinator + workers, NATS for control plane

**Goal**: Coordinator and workers run as separate processes, possibly on different machines. Multiple workers. Worker failure detected and activities redistributed.

**Duration**: 2 weeks (solo)

**Key Deliverables**:
- Split binaries: `cmd/coordinator/`, `cmd/worker/`. Shared code in `internal/`.
- NATS-backed `EventBus` in `internal/broker/nats/`. Channels per [ARCHITECTURE.md:280](platform/ARCHITECTURE.md#L280): `worker.{id}.commands`, `worker.{id}.status`, `coordinator.events`. Reconnect logic, pub-with-retry.
- Worker process: registers on startup, subscribes to its command channel, heartbeats every 30s with capacity + running-activity inventory.
- Coordinator process: HTTP API for `submit/get/list/cancel` (minimal, no auth yet); scheduler loop every 5s; health monitor loop every 30s.
- Worker failure detection: marks workers `DEAD` after 60s without heartbeat; `INTERRUPTED` their in-flight attempts; re-enqueues.
- Lease + fence tokens: `lease_expires_at` on the attempt; `generation` on the activity; workers reject commands with stale generation. Resolves split-brain.
- Affinity (hard, soft, anti-affinity) — moved here from v1 Phase 1 because affinity is meaningless with one worker. Implementation lives in `internal/scheduler/affinity.go` per [impl/PHASE_2_DISTRIBUTED_WORKERS.md](platform/impl/PHASE_2_DISTRIBUTED_WORKERS.md). Soft affinity is scoring; hard is filter; anti-affinity needs `GetActivitiesByWorker` query.
- docker-compose for local dev: postgres + nats + 1 coordinator + 2 workers.

**Success Criteria**:
- `docker compose up` brings the whole system up.
- Submit a 30s-sleep activity. `kill -9` the worker that picked it up. Activity is redistributed to another worker within 60s and completes.
- `psql` shows the timeline: attempt 1 INTERRUPTED on worker-A, attempt 2 COMPLETED on worker-B.
- Lease-expiration test: `SIGSTOP` a worker mid-attempt, lease expires, redistribution happens, `SIGCONT` the original worker, its commands are fenced off.
- GPU-tagged activity (hard affinity) only lands on GPU-tagged workers.

**Explicitly out of scope**: `RunChild`, signals, coordinator HA (single coordinator), priority, auth, rate limiting.

---

### Phase 3: Sub-activities and replay-based resume

**Goal**: A parent activity calls `ctx.RunChild(...)`. Parent suspends durably; child runs (in-process or remote — framework's choice); parent resumes via replay across attempts. Children with recorded results short-circuit on resume.

**Duration**: 3 weeks (solo)

**Key Deliverables**:
- `ActivityCallStore` interface + Postgres impl. New table `activity_calls(parent_attempt_id, call_index, child_activity_id, recorded_result, created_at)`.
- `ctx.RunChild(activityType, input, opts)` semantics:
  - Records call in `activity_calls`, dispatches child via `TaskQueue`, transitions parent attempt to `WAITING_CHILD`, frees parent's worker slot.
  - On child completion, coordinator records result in `activity_calls`, transitions parent to `PENDING_RESUME`, enqueues a new parent attempt.
  - On the new parent attempt, framework loads the call log; activity body re-executes from the top; each `RunChild` call checks the log and short-circuits with the recorded result, fast-forwarding to the next un-recorded call.
- Call-index determinism: sequential `nextCallIndex()` counter per attempt. Runtime check on replay: if a logged call's type doesn't match the activity's call at the same index, fail loudly with a determinism-violation error.
- Locality default: always remote (always dispatched via `TaskQueue`). In-process optimization deferred to a later phase.
- Sample composite activities: `sequential_pipeline` (3 children combined), `flaky_pipeline` (children fail randomly, parent retries each via the resume mechanism).
- Document the call-log + replay semantics fully in [impl/PHASE_3_CHILD_ACTIVITIES_REPLAY.md](platform/impl/PHASE_3_CHILD_ACTIVITIES_REPLAY.md) before writing code — replay correctness is subtle.

**Success Criteria**:
- Submit `sequential_pipeline` with 3 children. `psql` timeline shows: parent attempt 1 dispatches child 0 → suspends; child 0 completes; parent attempt 2 replays past child 0 (instant), dispatches child 1 → suspends; ... ; parent attempt 4 replays past all three, returns the combined result.
- Kill the worker holding the parent during attempt N. New worker picks up the parent, replays through completed children (each in microseconds), continues from where it left off.
- Each child runs exactly once across all parent attempts.
- Determinism-violation test: an activity that randomly skips a `RunChild` on replay → framework detects and fails the activity with a clear error.

**Explicitly out of scope**: `Sleep`, `Now`, `NewID`, signals, in-process child optimization.

---

### Phase 4: Deterministic primitives and signals

**Goal**: Activities can sleep for hours, wake on time, and respond to external signals. After this phase, multi-day agentic monitors and ingestion pipelines are fully expressible.

**Duration**: 3 weeks (solo)

**Key Deliverables**:
- `ctx.Now()`, `ctx.NewID()` — recorded into the call log on first call within an attempt; replayed thereafter. Implemented as synthetic call-log entries (`_now`, `_id`).
- `ctx.Sleep(duration)` — durable. Records `_sleep` with `wake_at = now + d`. Suspends parent. Coordinator-side timer wheel (a `pending_timers(activity_id, wake_at)` table with index on `wake_at`, polled every second) wakes activities at their scheduled time. Multi-day sleeps cost no worker resources.
- `SignalBus` interface in `internal/signal/`:
  ```go
  type SignalBus interface {
      Send(ctx context.Context, activityID, name string, payload []byte) error
      AwaitForActivity(ctx context.Context, activityID, name string) ([]byte, error)
  }
  ```
  Signal substrate is fully abstracted — Phase 4 ships a Postgres-backed impl, Phase 6 adds a JetStream-backed alternative without API changes.
- Postgres `SignalBus` impl: `signals(id, activity_id, name, payload, delivered_at, created_at)`. `Send` inserts + `NOTIFY signal_arrived, '<activity_id>'`. Coordinator owns the LISTEN connection; pushes deliveries to waiting parents via internal channels. (Single-coordinator constraint; HA in Phase 6 requires the JetStream variant.)
- `ctx.AwaitSignal(name, timeout)` — same suspend-and-resume pattern as `RunChild`. Records `_await_signal(name, timeout)` in call log. Coordinator parks the parent until signal arrives or timeout fires.
- Signal addressing: external clients send via API `POST /v1/activities/{id}/signals/{name}`. Activities can also send via `ctx.SendSignal(target_id, name, payload)`.
- Sample agentic-monitor activity: `for { ctx.AwaitSignal("alert", 24*time.Hour); ctx.RunChild("evaluate_and_decide", ...); }`. Demonstrates multi-day lifetime.
- [impl/PHASE_4_TIMERS_SIGNALS.md](platform/impl/PHASE_4_TIMERS_SIGNALS.md) written before code — delivery semantics, mailbox storage, race handling.

**Success Criteria**:
- Submit a monitor activity. Don't touch it for hours. Send a signal. Activity wakes, runs a child, suspends again. Kill all workers. Restart. Send another signal. Wakes correctly.
- Signal delivered before parent suspends (race) — not lost.
- Signal delivered while no worker holds the parent — persisted, delivered on resume.
- Timeout: `AwaitSignal` with 5s timeout, no signal sent → timeout error returned.
- Replay correctness: parent that already received signal X, on replay, returns the same payload X without re-awaiting.
- Long sleep test: `ctx.Sleep(48h)` — fast-forward variant exercises the timer wheel without waiting 48 hours.

**Explicitly out of scope**: NATS-backed `SignalBus`, determinism linter, coordinator HA. Phase 4 relies on author discipline for the determinism rule.

**Stop point for prototype use**: The system supports your stated use cases (ingestion + agentic monitors) at this point. Phases 5 and 6 are productionization.

---

### Phase 5: Production ingestion shape — connectors, planner, API, observability

**Goal**: Turn the durable activity substrate into the actual ingestion product: connector definitions, customer connections, ingestion planner, self-orchestrated runs, customer/auth, dense shared execution, isolation policies, structured logs/metrics/traces, rate limiting, audit, backpressure, and priority scheduling.

**Duration**: 3 weeks (solo)

**Key Deliverables**:
- Connector definition and connection APIs per [DATA_CONNECTOR_SPEC.md](apps/dataconnector/DATA_CONNECTOR_SPEC.md): create/list/get/update/test/run/pause/resume.
- Connector registry and connection manager tables: definitions, versions, direction, packaging metadata, compatibility metadata, customer connections, secret refs, destinations, selected catalog, health state.
- Connector discovery/catalog foundation: `GetSpec`, `Discover`, selected-catalog storage, catalog versioning, and run references to a selected catalog version.
- Isolation policy model: shared, pooled, and dedicated worker placement; tenant-scoped state, secret refs, destination refs, checkpoints, dedupe state, logs, metrics, and traces.
- Customer ingestion operations: CID-scoped ingestion view, significant event filters, active connection controls, rate-limit overrides, and temporary disable controls for erring connections.
- Operational failure semantics from [OPERATIONS_AND_FAILURE_SEMANTICS.md](apps/dataconnector/OPERATIONS_AND_FAILURE_SEMANTICS.md): at-least-once delivery, checkpoint-after-confirmed-ingestion, dead-letter sinks, connector spec versioning, secret-rotation wakeups, backfill isolation, and blast-radius controls.
- Ingestion planner loop from [INGESTION_ARCHITECTURE.md](apps/dataconnector/INGESTION_ARCHITECTURE.md): schedules, lag, checkpoint state, backfills, source rate limits, destination backpressure, and health policy produce `ConnectorRun` records and activity attempts.
- Initial declarative `poll_http` runtime: API key/basic/OAuth2 client credentials, JSON response extraction, time-window checkpoints, batching, and at-least-once delivery.
- Native `poll_http` implements `GetSpec`, `Test`, `Discover`, and `Run`; Airbyte-compatible command execution is represented in the model but not executed in Phase 5.
- Connection supervisor semantics using `ctx.Sleep` and `ctx.AwaitSignal`: config changes, backfills, and manual run requests wake the supervisor.
- Full REST API per [API_SPEC.md](platform/API_SPEC.md), with `workloads` renamed to `activities` throughout. Activity submit/get/list/cancel/pause/resume; logs; audit trail; customer info; usage stats; health/ready/metrics endpoints.
- API key authentication per [API_SPEC.md:33-50](platform/API_SPEC.md#L33-L50). Hashed in DB. Customer extraction middleware. All queries filter by `customer_id`.
- Backpressure: queue-depth check on submit; 503 with `Retry-After` when at customer's queue limit.
- Rate limiting: token bucket per customer, tier-based limits per [DATA_MODELS.md:120-126](platform/DATA_MODELS.md#L120-L126).
- Priority scheduler with SLA tier weights (Free=1, Standard=10, Premium=50, Enterprise=100) and aging to prevent starvation (boost +5 priority per minute of wait). Moved here from v1 Phase 2 because tiers are meaningless without multi-tenant.
- Structured logging (zap or zerolog) with correlation IDs across connection → run → activity → attempt → call.
- Prometheus metrics from [ARCHITECTURE.md](platform/ARCHITECTURE.md). Per-customer, per-connector, per-connection, per-activity-type counters/histograms. Queue depth, source lag, checkpoint age, destination latency, and worker utilization gauges (sets up Phase 6 auto-scaling).
- OpenTelemetry tracing. Span per connector supervisor, planner cycle, activity, attempt, source read, transform, destination flush, checkpoint commit, and `RunChild`. Trace context propagation through `ActivityContext`.
- Audit events table per [DATA_MODELS.md:357-372](platform/DATA_MODELS.md#L357-L372). Async writer (channel + worker goroutine) so audit doesn't block hot path.

**Success Criteria**:
- Create a `poll_http` connector definition and customer connection. The planner creates runs without manual workload submission.
- Discover a catalog for the connection, store the selected catalog version, and run ingestion against that selected catalog.
- A connection supervisor sleeps, wakes for its next poll, processes data, commits checkpoint, and schedules the next run.
- Manual backfill uses an isolated checkpoint namespace and does not corrupt live ingestion.
- Checkpoints are committed only after confirmed destination delivery.
- Dead-letter records can be written to file/object storage or stream destinations according to policy.
- Secret updates wake affected connection supervisors and the next run resolves the rotated secret.
- Source 429 triggers backoff; repeated auth failure quarantines the connection; both are visible in health state.
- Submit via `curl` with API key. Poll. Receive completion.
- Two customers' activities are isolated in queries, metrics, and rate limits.
- Shared worker pool runs multiple tenants densely while respecting per-tenant concurrency, queue, source, and destination limits.
- Dedicated isolation mode routes a selected connection only to an approved worker pool.
- A CID ingestion view shows connection health, active controls, significant events, source lag, checkpoint age, and run metrics.
- Significant event filters can mark source throttling, auth failure, parser drops, destination failure, and checkpoint-age events.
- Operators can apply a temporary rate limit or temporary disable to an erring connection, and the planner respects it before creating new runs.
- Premium tier activity submitted after free tier activity executes first.
- No activity in the queue starves longer than 30 minutes (aging).
- Rate-limit and backpressure paths exercised — correct status codes, headers, `Retry-After`.
- Grafana dashboard shows per-customer and per-connection throughput/lag/drop rate; Jaeger trace spans full lifecycle including planner, source read, destination flush, checkpoint commit, and `RunChild`.

**Explicitly out of scope**: production Airbyte image execution; database snapshot connectors; CDC connectors; complex destination connectors; Coordinator HA; NATS JetStream `SignalBus`; auto-scaling integration (the metrics that enable it ship here, the integration ships in Phase 6).

---

### Phase 5A: Connector packaging and compatibility foundation

**Goal**: Make the connector registry able to represent imported connector ecosystems, especially Airbyte-style manifests and images, without yet committing to every runtime behavior.

**Duration**: 2 weeks (solo)

**Key Deliverables**:
- `POST /connector-definitions:import` accepts an imported connector bundle or metadata document and stores upstream identity, image tag/digest, release stage, support level, docs URL, allowed hosts, resource requirements, breaking-change notes, and test-suite references.
- Compatibility model for `declarative`, `native_plugin`, `oci_image`, `external_command`, `airbyte_manifest`, and `airbyte_image` packaging modes.
- Airbyte-compatible adapter skeleton maps `spec`, `check`, `discover`, `read`, state, log, and trace messages to Niyanta interfaces, with `spec/check/discover` implemented for fixture-backed test connectors.
- Selected-catalog lifecycle: discover, diff, approve/update selected catalog, audit catalog changes, and reject runs against stale or missing catalog versions.
- Runtime sandbox policy design for OCI/external command connectors: allowed hosts, resource limits, temp filesystem, secret scope, network isolation, and redaction.

**Success Criteria**:
- Import an Airbyte-style low-code HTTP connector metadata/manifest fixture and preserve upstream metadata in `connector_definitions`.
- Run `spec`, `check`, and `discover` through the adapter skeleton for a fixture connector and store the selected catalog.
- Catalog changes are audited and do not silently alter existing runs.
- Unsupported Airbyte custom components are detected and routed to `airbyte_image` or `worker_plugin` instead of being mistranslated.

**Explicitly out of scope**: production container execution; broad Airbyte catalog import; database/CDC implementation; destination connector implementation.

---

### Phase 5B: Connector ecosystem breadth

**Goal**: Add the high-value connector families needed to handle the Airbyte connector set beyond basic HTTP polling.

**Duration**: 3-4 weeks (solo)

**Key Deliverables**:
- `database_source` runtime for at least one native database, starting with PostgreSQL snapshot and cursor-based incremental reads.
- `cdc_source` runtime spike for PostgreSQL logical replication with LSN checkpointing, schema history, and snapshot-to-CDC handoff policy.
- `destination_connector` runtime for one complex destination, starting with S3/object-lake writes using durable commit receipts.
- Stream subscription expansion beyond Kafka/Event Hubs/Pub/Sub/OCI to include Kinesis, Pulsar, or RabbitMQ based on customer demand.
- Production Airbyte adapter execution for selected `airbyte_image` connectors with sandboxing, resource limits, allowed-host enforcement, state translation, and log/trace redaction.

**Success Criteria**:
- PostgreSQL database source discovers tables, stores a selected catalog, runs a chunked snapshot, and resumes from a per-table checkpoint.
- PostgreSQL CDC source advances LSN checkpoints only after delivery and preserves transaction ordering within the replication stream.
- S3/object-lake destination returns durable commit receipts and source checkpoints advance only after those receipts.
- One Airbyte image connector can run through `spec/check/discover/read` in the sandbox with state normalized into Niyanta checkpoints.

**Explicitly out of scope**: full Airbyte catalog parity, all database vendors, exactly-once across arbitrary destinations, marketplace signing at scale.

---

### Phase 6: HA, scale, hardening

**Goal**: Coordinator HA, NATS JetStream `SignalBus`, auto-scaling, determinism linter, chaos testing, scale to the targets in [ARCHITECTURE.md:693-702](platform/ARCHITECTURE.md#L693-L702).

**Duration**: open-ended; tackle items selectively based on actual scale needs.

**Key Deliverables**:
- Coordinator HA via `LeaderElector`. Pick one mechanism (Postgres advisory lock *or* etcd) and align architecture/data-model docs with [impl/PHASE_6_HA_SCALE_HARDENING.md](apps/dataconnector/impl/PHASE_6_HA_SCALE_HARDENING.md). Followers serve API; leader runs scheduler, health monitor, timer wheel, signal listener. Generation tokens prevent split-brain.
- NATS JetStream-backed `SignalBus` impl as alternative behind the same interface. Removes the LISTEN/NOTIFY single-coordinator bottleneck — required for true HA.
- Auto-scaling: custom Prometheus metrics adapter, Kubernetes HPA manifests, queue-depth- and utilization-based scaling per [impl/PHASE_6_HA_SCALE_HARDENING.md](apps/dataconnector/impl/PHASE_6_HA_SCALE_HARDENING.md).
- Determinism linter: static analysis catching direct use of `time.Now()`, `rand.*`, file/network I/O in any package whose activities call `RunChild` (or other suspend points). Run in CI.
- Chaos testing: random worker kills, leader kills, NATS message drops, brief Postgres pauses. Verify SLOs hold — no lost activities, bounded retry duplication, exactly-once-per-instance signal delivery.
- Performance work: load test to architecture targets (10K concurrent activities, p95 latency, etc.). Profile and fix hot spots. Database indexes per actual query patterns.
- Heartbeat S3 offload via a `BlobStore` interface — for activities with large heartbeat payloads (> threshold), payload goes to S3 with pointer in `activity_heartbeats`. Resolves the Postgres-bytea concern from architectural review.
- Database read replicas for query offloading at scale (> 5K concurrent activities). Read/write splitting belongs in [impl/PHASE_6_HA_SCALE_HARDENING.md](apps/dataconnector/impl/PHASE_6_HA_SCALE_HARDENING.md) — `GetPendingActivities`, list endpoints, `GetActivity` go to replica; writes go to primary.

**Success Criteria**:
- Kill the leader coordinator: failover within 30s, no activity progress lost, no duplicate execution.
- Sustained 1000 activities/min with no errors over 24h.
- HPA scales workers up under load and down when idle.
- Determinism lint catches a deliberately-broken activity in CI.
- Chaos run: 1-hour scenario with random faults injected; system recovers within SLO; activity correctness preserved.

---


## Resource Requirements

### Team Composition

**Phase 0-1 (Weeks 1-10)**:
- 1 Tech Lead
- 2-3 Backend Engineers (Go)
- 1 DevOps Engineer
- 1 QA Engineer (from week 6)

**Phase 2 (Weeks 11-20)**:
- 1 Tech Lead
- 3-4 Backend Engineers
- 1 DevOps Engineer
- 1 QA Engineer
- 0.5 DBA (consultant)

**Phase 3-4 (Weeks 21+)**:
- 1 Tech Lead
- 3 Backend Engineers
- 1 DevOps Engineer
- 1 QA Engineer

### Infrastructure Costs (Estimated)

**Phase 1 (Dev/Staging)**:
- PostgreSQL (RDS db.t3.medium): $70/month
- NATS cluster (3x t3.small): $40/month
- Kubernetes cluster (EKS/GKE): $150/month
- Monitoring (Grafana Cloud): $100/month
- **Total**: ~$360/month

**Phase 2 (Production + Staging)**:
- PostgreSQL (RDS db.m5.large Multi-AZ): $400/month
- NATS cluster (3x t3.medium): $120/month
- Kubernetes cluster (3 nodes c5.xlarge): $450/month
- Monitoring: $200/month
- Load balancer, storage, etc.: $100/month
- **Total**: ~$1,270/month

**Phase 4 (Scale)**:
- Database scaling: +$500/month
- Worker auto-scaling (avg 20 nodes): +$2,000/month
- Multi-region: +$2,000/month
- **Total**: ~$5,770/month

---

## Risk Management

### High-Priority Risks

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| Database becomes bottleneck | High | Medium | Early load testing, read replicas, query optimization |
| NATS message loss during failures | High | Low | Use NATS JetStream for persistence (Phase 2) |
| Checkpoint size exceeds limits | Medium | Medium | Implement compression and S3 offload (Phase 4) |
| Worker failure detection too slow | Medium | Low | Tune heartbeat intervals, add active health checks |
| Multi-region data consistency issues | High | Medium | Defer to Phase 4, careful design of eventual consistency |
| Team availability/attrition | Medium | Medium | Cross-training, documentation, 2-person knowledge rule |

### Risk Monitoring

- Weekly risk review in team meetings
- Maintain risk register (spreadsheet or JIRA)
- Escalate high-impact risks to leadership

---

## Success Metrics

Each phase has its own success criteria embedded in the §[Development Phases](#development-phases) narrative. The summary below captures the cross-phase quality bars.

### After Phase 1
- End-to-end activity execution succeeds ≥95% of the time in dev.
- Activity survives a process restart; in-flight attempts marked INTERRUPTED, fresh attempts enqueued, completion observed.
- Framework-managed retries respect declared `RetryPolicy`; non-retryable errors fail fast.

### After Phase 2
- Worker failure detection < 60s.
- Redistribution within 2 minutes of detection.
- Hard, soft, and anti-affinity rules placed correctly in test scenarios.
- Lease/fence tokens prevent split-brain when SIGSTOP'd workers resume.

### After Phase 3
- Sub-activities run exactly once across parent attempts.
- Replay short-circuit measurable: parent re-runs in microseconds per logged call.
- Determinism violations detected at runtime with clear errors.

### After Phase 4 (use-case parity)
- Multi-day monitor activity submitted, sleeping, signaled, woken, and completing children — observed end-to-end.
- Signal delivered while no worker holds the parent is durably persisted and delivered on resume.
- `AwaitSignal` timeout fires correctly when no signal arrives.

### After Phase 5 (production shape)
- API p95 latency < 200ms.
- Multi-tenant isolation: customer A cannot observe or affect customer B.
- Premium-tier activity scheduled before free-tier when both are pending.
- No activity in queue starves longer than 30 minutes (aging).
- Rate limit and backpressure paths return correct status codes and `Retry-After` headers.
- Connector definitions include direction, packaging, and compatibility metadata.
- Connections can discover, store, select, and run against a versioned catalog.

### After Phase 5A (compatibility foundation)
- Airbyte-style metadata and manifest fixtures import without losing upstream version, image, support, docs, allowed-host, resource, and breaking-change data.
- Adapter skeleton maps `spec/check/discover/read` concepts to Niyanta interfaces.
- Unsupported imported connector features are rejected or routed to plugin/image execution explicitly.

### After Phase 5B (connector ecosystem breadth)
- Native database source, CDC source, and complex destination connector have end-to-end checkpoint-after-delivery tests.
- One sandboxed Airbyte image connector runs through the adapter with state normalized into Niyanta checkpoints.
- Destination commit receipts gate source checkpoint advancement.

### After Phase 6 (hardened)
- Coordinator failover < 30s with no progress lost.
- Sustained 1000 activities/min for 24h with no errors.
- HPA scales workers up under load, down when idle.
- Determinism lint catches deliberately-broken activity in CI.
- Architecture targets from [ARCHITECTURE.md:693-702](platform/ARCHITECTURE.md#L693-L702) met under load test.

---

## Timeline & Milestones

Solo-engineer estimates per the §[Phases Overview](#phases-overview) table. Times double under part-time attention; phase boundaries are designed so stopping between phases leaves a working system.

### Solo Gantt (rough)

```
Week 1:    [Phase 0: skeleton + in-memory]
Week 2-3:  [Phase 1: Postgres + retries]
Week 4-5:  [Phase 2: coordinator/worker + NATS + affinity]
Week 6-8:  [Phase 3: RunChild + replay]
Week 9-11: [Phase 4: Sleep/Now/NewID + SignalBus + AwaitSignal]   ← use-case parity
Week 12-14:[Phase 5: API + auth + observability + priority]       ← production shape
Week 15-16:[Phase 5A: packaging + compatibility + catalog import]
Week 17-20:[Phase 5B: DB/CDC + destination connectors + Airbyte execution]
Week 21+:  [Phase 6: HA + auto-scaling + JetStream + chaos + scale]
```

### Key Milestones

| Milestone | Phase end | Description |
|-----------|-----------|-------------|
| **M0**: Skeleton | Phase 0 | All interfaces exist; in-memory demo runs |
| **M1**: Persistent | Phase 1 | Activities survive restart; framework retries |
| **M2**: Distributed | Phase 2 | Coordinator + workers; failure redistribution |
| **M3**: Composable | Phase 3 | `RunChild` + replay-based resume |
| **M4**: Use-case parity | Phase 4 | Signals + Sleep — agentic monitors and ingestion expressible |
| **M5**: Production foundation | Phase 5 | REST API, multi-tenant, observable, catalog-aware connector platform |
| **M5A**: Compatibility foundation | Phase 5A | Imported connector metadata and Airbyte command mapping represented |
| **M5B**: Connector ecosystem breadth | Phase 5B | DB/CDC, destination connectors, and Airbyte image execution MVP |
| **M6**: Hardened | Phase 6 | HA, auto-scale, scale targets met |

---

## Post-Phase 6 Roadmap

Items not on the critical path for the stated use cases. Pick up as needed.

- gRPC API alongside REST.
- Activity dependencies as a higher-level construct (DAG sugar over `RunChild`).
- Scheduled / cron-like activity submission.
- Activity versioning and A/B testing.
- Custom worker pools per customer.
- Integration with external schedulers (Kubernetes Jobs, Airflow).
- Self-service customer dashboard.
- Billing and usage tracking.
- Multi-region deployment with region-aware scheduling.
- Marketplace for third-party activity types.

---

## Appendix

### A. Technology Stack Summary

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Language | Go 1.21+ | Performance, concurrency, ecosystem |
| API Framework | Gin or Echo | Fast, lightweight, good docs |
| Database | PostgreSQL 15+ | ACID, JSON support, mature |
| Broker | NATS 2.9+ | Lightweight, request-reply, clustering |
| Logging | Zap or Zerolog | Structured, high-performance |
| Metrics | Prometheus | Standard for Go, Kubernetes-native |
| Tracing | Jaeger or Tempo | OpenTelemetry compatible |
| Deployment | Kubernetes | Portable, scaling, ecosystem |
| CI/CD | GitHub Actions | Integrated with GitHub |
| IaC | Terraform | Multi-cloud, declarative |

### B. Staffing Plan

**Hiring Priorities**:
1. **Immediately**: Backend Go engineer (for Phase 1 parallelization)
2. **Week 10**: QA engineer (for Phase 2 testing)
3. **Week 20**: Additional backend engineer (for Phase 3 features)

**Skills Required**:
- Strong Go programming (goroutines, channels, context)
- Distributed systems knowledge
- PostgreSQL and SQL optimization
- Kubernetes and Docker
- REST API design
- Testing and quality assurance

---

## References

- [ADR-005](platform/adr/ADR-005-activity-execution-model.md) — Activity execution model (the basis for this plan's reframing)
- [ARCHITECTURE.md](platform/ARCHITECTURE.md) — System architecture (note: `Workload` references throughout will be renamed to `Activity` after Phase 3 lands)
- [DATA_MODELS.md](platform/DATA_MODELS.md) — Schema (will be extended in Phase 1, 3, and 4 — see each phase's Key Deliverables)
- [API_SPEC.md](platform/API_SPEC.md) — API contracts (full surface lands in Phase 5)
- [CLAUDE.md](../CLAUDE.md) — Project overview and Go conventions

---

## Revision History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025-10-27 | Initial implementation plan (workload-based model, multi-engineer phasing). |
| 2.0 | 2026-04-28 | Reframed around the activity execution model from [ADR-005](platform/adr/ADR-005-activity-execution-model.md). Replaced workload + manual checkpoint with activity + framework-managed retries + replay-based resume. Signals promoted from Phase 3 to Phase 4 (now mandatory for stated use cases). Affinity moved from Phase 1 to Phase 2 (meaningless without multi-worker). Priority moved from Phase 2 to Phase 5 (meaningless without multi-tenant). Phasing rescaled for solo execution; old multi-engineer phasing preserved in §Resource Requirements. Removed redundant per-task work-item lists — phase narratives are the source of truth. |
| 2.1 | 2026-05-01 | Recentered product framing around self-orchestrated ingestion: connector definitions/connections, ingestion planner, supervisor semantics, adaptive health, and ingestion observability. |
| 2.2 | 2026-05-03 | Split connector ecosystem breadth out of Phase 5. Phase 5 now includes catalog/discovery foundations; Phase 5A covers packaging and Airbyte-compatible import metadata; Phase 5B covers database, CDC, destination connectors, and Airbyte image execution MVP. |

---

**End of Implementation Plan**
