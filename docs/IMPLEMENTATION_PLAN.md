# Niyanta Implementation Plan

**Version**: 3.1
**Last Updated**: 2026-07-10
**Status**: Planning Phase
**Supersedes**: v3.0. This revision keeps the use-case-agnostic engine and six contract-gap resolutions, but grounds delivery of the Activity Manager: read-only inspection, privileged operations, observability, and production hardening are separate acceptance gates.

## Table of Contents
1. [Executive Summary](#executive-summary)
2. [Gap Resolutions](#gap-resolutions)
3. [Development Phases](#development-phases)
   - [Phase 0: End-to-end skeleton, in-memory only](#phase-0-end-to-end-skeleton-in-memory-only)
   - [Phase 1: Persistence, framework-managed retries, write-path fencing](#phase-1-persistence-framework-managed-retries-write-path-fencing)
   - [Phase 2: Multi-process — coordinator + workers, Postgres-native dispatch](#phase-2-multi-process--coordinator--workers-postgres-native-dispatch)
   - [Phase 3: Sub-activities, replay-based resume, version-pinned replay](#phase-3-sub-activities-replay-based-resume-version-pinned-replay)
   - [Phase 4: Deterministic primitives, signals, ContinueAsNew](#phase-4-deterministic-primitives-signals-continueasnew)
   - [Phase 5: Production engine surface — API completeness, multi-tenancy, deployment safety](#phase-5-production-engine-surface--api-completeness-multi-tenancy-deployment-safety)
   - [Phase 6: Read-only Activity Manager console](#phase-6-read-only-activity-manager-console)
   - [Phase 7: Operator workflows + live updates](#phase-7-operator-workflows--live-updates)
   - [Phase 8: Observability distribution](#phase-8-observability-distribution)
   - [Phase 9: Production hardening gate](#phase-9-production-hardening-gate)
4. [Risk Management](#risk-management)
5. [Success Metrics](#success-metrics)
6. [Timeline & Milestones](#timeline--milestones)
7. [Post-Phase 9 Roadmap](#post-phase-9-roadmap)
8. [Appendix](#appendix)
9. [References](#references)
10. [Revision History](#revision-history)

---

## Executive Summary

Niyanta is a **durable activity execution engine**: it runs activities, guarantees their execution (retries, replay-based resume, leases, fencing, durable timers, signals), and holds enough state to resume them after any failure. The engine is agnostic to what an activity does. This plan delivers that engine — nothing else — plus the operational surface an engine needs to be run in production: a complete API, multi-tenant isolation, Prometheus observability, and a web-based Activity Manager console.

### Key Objectives

1. **Phase 1 MVP**: Single-attempt activities with framework-managed retries, persisted in Postgres, with **generation-fenced writes from day one**. Survives crashes.
2. **Phase 2**: Multi-process (coordinator + workers), worker failure detection and redistribution, **Postgres-native dispatch** (no broker dependency), affinity.
3. **Phase 3**: `RunChild` with **exactly-once child dispatch**, replay-based resume, **version-pinned replay**, and a defined recovery path for determinism violations.
4. **Phase 4**: Durable timers, deterministic primitives, signals with sender-side dedupe, **`ContinueAsNew`** for history truncation, and defined cancellation semantics for suspended activities.
5. **Phase 5**: Production engine surface — complete REST API (pause/resume, long-poll wait, operator recovery endpoints), auth, multi-tenant isolation, priority, rate limiting, backpressure, deployment-safety tooling (version drain).
6. **Phase 6**: A read-only **Activity Manager console** over the versioned engine API: activity search/detail, attempt and replay timelines, composition tree, workers, and versions. It has no direct access to core Postgres tables.
7. **Phase 7**: Safe operator workflows and live updates: browser identity, capability-based authorization, optimistic concurrency, transactional audit, and reconnectable SSE.
8. **Phase 8**: Prometheus-compatible observability: bounded-cardinality metrics, alert rules, dashboards, and tracing. External Prometheus is the production default; an embedded store is an optional, explicitly single-node convenience profile.
9. **Phase 9**: Mandatory production gate: coordinator HA, backup/restore, security review, chaos and failover tests, upgrade/rollback, and a published capacity envelope.

### What changed from v2

- **All application content removed.** Former Phases 5/5A/5B (connector registry, ingestion planner, catalog import, CDC, Airbyte compatibility) are out of this plan entirely. Compositions consume the engine's public API and SDK; they never gate or shape engine phases. Engine features are justified only by the engine's own contract.
- **Six contract gaps are now scheduled work** with owning phases and acceptance tests (§[Gap Resolutions](#gap-resolutions)).
- **NATS removed from the critical path.** Phase 2 dispatch is Postgres-native (`SELECT … FOR UPDATE SKIP LOCKED` pull model + `LISTEN/NOTIFY` for latency). A broker is considered only after Phase 9 if the published Postgres envelope is insufficient.
- **Phases 6–9 are deliberately separated.** Read-only inspection, privileged control, observability distribution, and production hardening have different failure and security boundaries and are accepted independently.

### Phases Overview

| Phase | Duration (solo) | Key Deliverables |
|-------|------|------------------|
| Phase 0 | 1 week | Project skeleton, all interfaces stubbed, in-memory impls, single-binary demo |
| Phase 1 | 2 weeks | Postgres-backed execution, framework retries, per-attempt records, generation-fenced writes, survives restart |
| Phase 2 | 2 weeks | Coordinator + worker split, Postgres-native dispatch, failure redistribution, leases, affinity |
| Phase 3 | 4 weeks | `RunChild` + replay, exactly-once dispatch, version-pinned replay, determinism-violation recovery |
| Phase 4 | 4 weeks | `Sleep`/`Now`/`NewID`, `SignalBus` + `AwaitSignal`, signal dedupe, `ContinueAsNew`, cancellation semantics |
| Phase 5 | 4 weeks | Complete operator/read API, auth and tenancy, concurrency controls, transactional audit, version drain |
| Phase 6 | 3 weeks | Read-only Activity Manager: activities, timelines, composition, workers, versions; external Prometheus optional |
| Phase 7 | 3 weeks | Browser sessions/OIDC, capability authorization, safe actions, reconnectable SSE |
| Phase 8 | 3 weeks | Metrics, alert rules, dashboards, tracing; optional embedded single-node metrics profile |
| Phase 9 | 4+ weeks | HA, backup/restore, security, chaos, upgrade/rollback, load envelope; production gate |

**Stop point for prototypes**: end of Phase 4 — the full execution model is expressible and correct.
**Stop point for operational beta**: end of Phase 8 — multi-tenant, inspectable, operable, and observable, but not yet claimed HA.
**Stop point for production**: end of Phase 9 — the durability claims have survived failover, restore, chaos, security, and upgrade drills.

**Total time to model completeness (solo)**: ~13 weeks (Phases 0–4).
**Total time to operational beta (solo)**: ~26 weeks (Phases 0–8).
**Total time to production gate (solo)**: ~30+ weeks (Phases 0–9); Phase 9 exits on evidence, not elapsed time.

---

## Gap Resolutions

Design review identified six gaps in the engine contract. Each is now owned by a phase and has an acceptance test. Follow-up edits to [ADR-005](platform/adr/ADR-005-activity-execution-model.md) and [DATA_MODELS.md](platform/DATA_MODELS.md) are required to match (tracked per phase below).

| # | Gap | Resolution | Owning phase |
|---|-----|------------|--------------|
| G1 | **Versioning under replay.** A suspended parent may resume on a worker running newer code; a reordered/added/removed `RunChild` produces a false "determinism violation." | Workers advertise `{activity_type, version}`. Activities that have suspend-point history are **pinned**: they resume only on workers advertising the version recorded at first dispatch. Version drain tooling lets operators retire old versions safely. | P3 (pinning), P5 (drain tooling), P6 (visibility), P7 (UI action) |
| G2 | **Heartbeat × replay composition undefined.** Branching on `LastHeartbeat()` in code that also suspends breaks replay determinism. | **Mutual exclusivity rule**: heartbeat state is readable only in activities that never suspend. Stated in P1 docs; enforced at runtime in P3 (calling `LastHeartbeat` in an activity with any call-log entries is an immediate, clear error); linted in P9. Heartbeat scope is fixed as **cross-attempt within the current version pin** (retry sees prior attempt's last heartbeat), and the `checkpoints` unique key includes `attempt_id`. | P1 (rule + schema), P3 (guard), P9 (lint) |
| G3 | **Fencing specified on the command path only.** A stale (SIGSTOP'd, partitioned) worker can still *write* completions, heartbeats, or call-log rows. | **Write-path fencing invariant**: every state-mutating write carries the worker's generation and commits only under `WHERE generation = $expected`. No exceptions. Invariant documented in DATA_MODELS transaction boundaries; every store impl tested against it. | P1 (single-process), P2 (multi-worker acceptance test) |
| G4 | **Idempotency bound understated.** ADR-005 says children "may be invoked twice"; the transactional call-log design actually supports exactly-once dispatch. | `RunChild` dispatch is transactional with the call-log append: on replay, an existing incomplete call-log row means **wait, never re-dispatch**. Contract tightened to: *child dispatch is exactly-once; side effects inside a child are at-least-once, bounded by the child's retry policy*. | P3 |
| G5 | **Unbounded history for unbounded-lifetime activities.** `Sleep`/`AwaitSignal` invite immortal activities; replay cost grows monotonically with call count; retention cannot truncate a live activity's log. | **`ctx.ContinueAsNew(input)`**: atomically complete the current execution and start a successor with a fresh call log, preserving the activity's external identity (ID for signal addressing, composition links, console history chain). Designed before Phase 3 code freezes the call-log schema; implemented in Phase 4. | P4 (schema groundwork in P3) |
| G6 | **No recovery path from determinism violation.** Detection exists; the activity is silently poisoned — retry replays into the same wall. | Distinct terminal-ish state `FAILED` with `error_json.type = "nondeterminism"` that **suppresses automatic retry**. Operator recovery endpoints: force-fail, force-complete-with-result, and guarded truncate-replay-to-last-consistent-prefix + resume. Surfaced read-only in P6 and actionable in P7. | P3 (state + suppression), P5 (recovery API), P6–P7 (console) |

Additional review items folded in: attempt-status gets its own enum including `INTERRUPTED` (P1); lease moves to the attempt (P2); call log keyed by `activity_id` (P3, matching DATA_MODELS, since replay spans attempts); `result_json` size cap (P3); `Now()`/`NewID()` recorded per call with segment batching (P4); signal sends take an idempotency key (P4); pause/resume + long-poll wait endpoints (P5).

---

## Development Phases

The phases are designed so that **each one ends with a runnable system**. If you stop after phase N, what you have works for what phase N promises — no half-built state.

### Guiding principles

1. **Interfaces from day one.** Every storage and dispatch concern lives behind an interface starting in Phase 0: `ActivityStore`, `AttemptStore`, `HeartbeatStore`, `ActivityCallStore`, `WorkerRegistry`, `LeaderElector`, `TaskQueue`, `SignalBus`, `EventBus`. Optional substrates can be evaluated after the production envelope is measured without touching activity code.
2. **In-memory before external.** Phase 0 ships an in-memory implementation of every interface. End-to-end runs before any external dependency is introduced.
3. **Vertical slices.** Build one activity end-to-end (submit → schedule → execute → complete), then add capabilities to that slice phase by phase.
4. **Test what's invisible.** Replay, retry, fencing, and resume are invisible when they work. Failing tests come first: kill a worker mid-activity, SIGSTOP a worker and let it wake stale, drop a signal, redeploy changed code under a suspended parent. Make them pass.
5. **Guarantees are written before code.** Each phase that changes the execution contract updates ADR-005 / DATA_MODELS *first*; the docs are the spec the tests are written against.

---

### Phase 0: End-to-end skeleton, in-memory only

**Goal**: One process. One activity. Runs to completion. Everything behind interfaces. Zero external dependencies.

**Duration**: 1 week (solo)

**Key Deliverables**:
- Project skeleton per [impl/PHASE_0_SKELETON.md](platform/impl/PHASE_0_SKELETON.md). Makefile (`build`, `test`, `lint`, `run`). golangci-lint config.
- Domain models in `pkg/models/` — `Activity`, `ActivityAttempt`, `ActivityCall` (placeholder, populated in Phase 3), `Worker`, `Customer`; **two status enums**: `activity_status` (logical lifecycle) and `attempt_status` (physical lifecycle: `PENDING`, `RUNNING`, `COMPLETED`, `FAILED`, `INTERRUPTED`, `FENCED`).
- `Activity` interface and `ActivityContext` in `pkg/activity/` with all methods declared from day one — `RunChild`, `Sleep`, `Now`, `NewID`, `AwaitSignal`, `ContinueAsNew`, `Heartbeat`, `LastHeartbeat`. Unimplemented methods return `ErrNotImplemented` until their phases.
- All nine storage/queue/bus interfaces declared in `internal/storage/`, `internal/broker/`, `internal/signal/`, with in-memory implementations and **shared contract tests** (the same suite later runs against Postgres impls).
- Activity registry (`internal/registry/`) — type-name + **version** → factory function. Sample activity: `sleep`.
- Single-process executor (`internal/executor/`) — pull from `TaskQueue`, run, write result. No retries yet.
- CLI (`cmd/niyanta/main.go`) with `submit`, `run`, `get` subcommands.

**Success Criteria**:
- `niyanta demo sleep 2s` submits, runs, and reports completion using only in-memory state.
- All interfaces have at least one impl and contract tests passing.
- `go test ./...` and `golangci-lint run` clean.

**Explicitly out of scope**: Postgres, HTTP, retries, `RunChild`, signals, multi-tenant, observability beyond stdlib `log`.

---

### Phase 1: Persistence, framework-managed retries, write-path fencing

**Goal**: The activity survives a process restart and retries on transient failure under a declared policy — and **no stale writer can ever corrupt state**, established now while there is one process, so the invariant is load-bearing before there are two.

**Duration**: 2 weeks (solo)

**Key Deliverables**:
- Postgres implementations of `ActivityStore`, `AttemptStore`, `HeartbeatStore` in `internal/storage/postgres/`. `pgx/v5` directly, no ORM.
- Migrations via `golang-migrate`. Initial migration: `customers`, `activities`, `activity_attempts` (own `attempt_status` enum), `checkpoints` (**`UNIQUE(activity_id, attempt_id, sequence_number)`** — G2 schema fix), `workers`, `audit_events`.
- **Write-path fencing (G3)**: every store method that mutates activity/attempt/checkpoint state takes the caller's `generation` and issues conditional writes (`… WHERE generation = $expected`). A failed condition returns a typed `ErrFenced`; the executor treats it as "I am stale — stop." This is a **storage-layer contract test** in the shared suite, so every future impl (in-memory, Postgres, anything) must enforce it.
- Postgres-polling `TaskQueue`: pending attempts claimed with `SELECT … FOR UPDATE SKIP LOCKED LIMIT $n`.
- `RetryPolicy` declared per registered activity type: `MaxAttempts`, `InitialInterval`, `BackoffCoefficient`, `MaxInterval`, `NonRetryableErrors`. Framework enforces — activity code never retries itself. Typed errors: `RetryableError`, `NonRetryableError`.
- Per-attempt records — one `activity_attempts` row per physical execution; the logical activity row never carries a retry counter.
- `ctx.Heartbeat(state)` / `ctx.LastHeartbeat()` with **documented scope (G2)**: heartbeat state is cross-attempt (a retry sees the prior attempt's last heartbeat) and is only legal in activities that never suspend. The rule goes into ADR-005 and the `ActivityContext` godoc now; the runtime guard lands in Phase 3 when suspend points exist.
- **Payload caps**: `input_json` and `result_json` capped (default 256KB, configurable); checkpoint cap 10MB. Oversize is a submit-time / completion-time typed error, never a silent truncation.
- Crash recovery on executor startup: claim `RUNNING` attempts owned by self from a previous run, mark `INTERRUPTED`, enqueue a fresh attempt.
- Doc updates: DATA_MODELS transaction boundaries gain the fencing invariant ("no state-mutating write commits without a generation match"); attempt enum and checkpoint key corrected.

**Success Criteria**:
- Submit a `flaky_sleep` activity (fails attempts 1 and 2, succeeds attempt 3). `psql` shows three `activity_attempts` rows; final state `COMPLETED`; attempt 3 read attempt 2's heartbeat.
- Kill the executor mid-execution. Restart. Interrupted attempt marked `INTERRUPTED`, new attempt enqueued, completes.
- Fencing contract test: a store handle holding a stale generation cannot complete an activity, write a checkpoint, or update attempt state — every such write returns `ErrFenced` and changes nothing.
- Oversize input/result rejected with a clear typed error.
- Migration up/down round-trip clean.

**Explicitly out of scope**: Multi-worker, separate coordinator, `RunChild`, signals, HTTP API.

---

### Phase 2: Multi-process — coordinator + workers, Postgres-native dispatch

**Goal**: Coordinator and workers run as separate processes on different machines. Multiple workers. Worker failure detected, activities redistributed, and stale workers **provably unable to write** after redistribution.

**Duration**: 2 weeks (solo)

**Key Deliverables**:
- Split binaries: `cmd/coordinator/`, `cmd/worker/`. Shared code in `internal/`.
- **Postgres-native dispatch (no broker)**: workers pull work with `FOR UPDATE SKIP LOCKED` against the task queue, filtered by their advertised activity types/tags; `LISTEN/NOTIFY` wakes idle workers so dispatch latency isn't bounded by the poll interval. Worker registration, heartbeats (every 30s, with capacity and in-flight inventory), and completions are rows/notifications in Postgres. The `EventBus`/`TaskQueue` interfaces preserve a measured post-Phase-9 substitution path.
- Coordinator process: scheduler loop (admission: quotas, capability match), health monitor loop; minimal HTTP API for `submit/get/list/cancel` (no auth yet).
- Worker failure detection: mark workers `DEAD` after 60s without heartbeat; mark their in-flight attempts `INTERRUPTED`; re-enqueue.
- **Leases on the attempt** (not the activity): `activity_attempts.lease_expires_at`, renewed by worker heartbeat; expiry triggers redistribution. `generation` on the activity is bumped on every redistribution.
- **Fencing end-to-end (G3 acceptance)**: redistribution bumps generation; the old worker's subsequent writes — completion, heartbeat, checkpoint — all fail with `ErrFenced` at the storage layer, not merely "commands rejected."
- Affinity (hard/soft/anti) in `internal/scheduler/affinity.go` per [impl/PHASE_2_DISTRIBUTED_WORKERS.md](platform/impl/PHASE_2_DISTRIBUTED_WORKERS.md).
- Graceful shutdown: SIGTERM → stop pulling, finish or checkpoint in-flight attempts, deregister.
- docker-compose for local dev: postgres + 1 coordinator + 2 workers.

**Success Criteria**:
- `docker compose up` brings the whole system up.
- Submit a 30s-sleep activity. `kill -9` the worker that picked it up. Activity redistributed within 60s and completes. `psql` shows: attempt 1 `INTERRUPTED` on worker-A, attempt 2 `COMPLETED` on worker-B.
- **Stale-writer test**: `SIGSTOP` a worker mid-attempt; lease expires; work redistributed; `SIGCONT` the original worker — its completion write, heartbeat write, and checkpoint write all fail with `ErrFenced`; the redistributed attempt's result is the one recorded.
- Dispatch latency: p95 submit→running < 1s on an idle system (LISTEN/NOTIFY path, not poll-bound).
- GPU-tagged activity (hard affinity) only lands on GPU-tagged workers.

**Explicitly out of scope**: `RunChild`, signals, coordinator HA (single coordinator), priority, auth, rate limiting, NATS.

---

### Phase 3: Sub-activities, replay-based resume, version-pinned replay

**Goal**: A parent activity calls `ctx.RunChild(...)`; parent suspends durably; child runs; parent resumes via replay. Dispatch is exactly-once, replay is version-safe, and a determinism violation is a *recoverable operational event*, not a silent poison state.

**Duration**: 4 weeks (solo) — the extra week over v2 covers G1/G4/G6, which are contract work, not sugar.

**Key Deliverables**:
- `ActivityCallStore` + Postgres impl. Table `activity_calls` **keyed by `(activity_id, call_index)`** (replay spans attempts; the v2 plan's `parent_attempt_id` keying was wrong — DATA_MODELS is the source of truth). Columns per [DATA_MODELS.md](platform/DATA_MODELS.md): `call_type`, `child_activity_id`, `args_hash`, `result_json`, `completed`.
- **`ctx.RunChild(activityType, input, opts)` with exactly-once dispatch (G4)**:
  - Dispatch transaction atomically: append call-log row + insert child activity row + transition parent to `SUSPENDED(run_child)` + free the slot.
  - On replay, an existing call-log row **short-circuits**: completed → return recorded result; incomplete → re-suspend and wait. **A logged call is never re-dispatched.**
  - Contract statement updated in ADR-005: *child dispatch is exactly-once; side effects inside a child are at-least-once, bounded by the child's own retry policy.*
  - On child completion, coordinator records the result in the call log and re-enqueues the parent atomically.
- **Version-pinned replay (G1)**:
  - Workers advertise `{activity_type: version}` in capabilities (schema exists since Phase 0 registry).
  - The first suspend point stamps `activities.pinned_version`. Scheduling a resume filters to workers advertising that exact version for the activity's type.
  - No compatible worker available → activity parks in `SUSPENDED` with a `no_compatible_worker` condition, visible via API (and the Phase 6 console); it resumes automatically when a compatible worker appears. It is **never** replayed against different code.
  - Unpinned activities (no suspend history) run on any version.
- **Determinism guard + recovery groundwork (G6)**: replay mismatch (call type or `args_hash` at an index) fails the activity with `error_json.type = "nondeterminism"`, which **suppresses automatic retry** — retrying into the same log is wasted work by construction. The operator recovery endpoints land in Phase 5; Phase 3 guarantees the state is inspectable (full call log + divergence index in the error).
- **Heartbeat guard (G2)**: calling `LastHeartbeat()` or `Heartbeat()` in an activity that has any call-log entries returns a typed `ErrHeartbeatWithReplay` immediately — the mutual-exclusivity rule is now enforced, not advisory.
- **`ContinueAsNew` schema groundwork (G5)**: `activities.execution_seq` (int, default 1) and `superseded_by` linkage columns land in this phase's migration so Phase 4 doesn't have to alter the call-log keying it ships. Call-log rows implicitly scope to the current execution (truncation = new execution_seq).
- `result_json` cap enforced on child results at record time (a child result is re-read on every parent replay — unbounded results silently degrade replay).
- Sample composite activities: `sequential_pipeline` (3 children), `flaky_pipeline` (children fail randomly; resume mechanism carries the parent through).
- [impl/PHASE_3_CHILD_ACTIVITIES_REPLAY.md](platform/impl/PHASE_3_CHILD_ACTIVITIES_REPLAY.md) updated **before code** with: exactly-once dispatch semantics, version pinning, the divergence error contract.

**Success Criteria**:
- `sequential_pipeline` timeline in `psql`: parent dispatches child 0 → suspends; child 0 completes; parent replays past child 0 (microseconds), dispatches child 1; …; final replay returns combined result. **Each child dispatched exactly once across all parent attempts and worker kills** (asserted by counting child rows, not just observing success).
- Kill the worker holding the parent mid-replay; new worker resumes correctly.
- **Version-pin test**: suspend a parent on version `1.0.0` workers; deploy only `1.1.0` workers (child call reordered); parent parks with `no_compatible_worker` — no false determinism violation. Bring back a `1.0.0` worker; parent completes on it.
- Determinism-violation test: an activity that randomly skips a `RunChild` on replay → fails with `nondeterminism` error naming the divergent call index, and is **not** retried automatically.
- Heartbeat-guard test: an activity calling `Heartbeat` after a `RunChild` gets `ErrHeartbeatWithReplay`.

**Explicitly out of scope**: `Sleep`, `Now`, `NewID`, signals, `ContinueAsNew` behavior (schema only), in-process child optimization.

---

### Phase 4: Deterministic primitives, signals, ContinueAsNew

**Goal**: Activities can sleep durably, wake on time or on external signals, and run for unbounded lifetimes **without unbounded replay cost**. Cancellation of suspended activities has defined semantics.

**Duration**: 4 weeks (solo)

**Key Deliverables**:
- `ctx.Now()`, `ctx.NewID()` — recorded **per call** as synthetic call-log entries (`_now`, `_id`); replayed thereafter. (The v2 wording "recorded on first call within an attempt" was wrong — each call site gets its own deterministic value.) **Segment batching**: consecutive synthetic entries between real suspend points are written in one batch at the next durable write, so a loop calling `Now()` doesn't issue one Postgres write per iteration.
- `ctx.Sleep(duration)` — records `_sleep` with `wake_at`; suspends; coordinator timer sweep (index on `wake_at`, poll every second) re-enqueues due activities. Multi-day sleeps cost no worker resources.
- `SignalBus` interface in `internal/signal/` + Postgres impl (`signals` mailbox table + `NOTIFY`). Substrate remains abstracted; alternatives require Phase 9 performance evidence.
- `ctx.AwaitSignal(name, timeout)` — records `_await_signal`; suspends; coordinator delivers from the mailbox or fires the timeout. Delivery + call-log record in one transaction (a signal is consumed exactly once by a given call index; redelivery on replay returns the recorded payload).
- **Signal sender dedupe**: `POST /activities/{id}:signal` accepts an `Idempotency-Key`; the `signals` table gets a `(activity_id, dedupe_key)` unique constraint. Engine semantics stay at-least-once, but the common client-retry double-delivery is eliminated at the door.
- **`ctx.ContinueAsNew(input)` (G5)**: atomically — mark current execution `COMPLETED(continued_as_new)`, increment `execution_seq`, start a successor execution with fresh call log and the supplied input. External identity is preserved: same `activity_id`, signals continue to route, pending undelivered signals carry over to the successor, composition links (`parent_activity_id`, `root_activity_id`) are retained. Replay cost is now bounded by the current execution's call count regardless of activity lifetime. Version pin re-evaluates at the boundary (a `ContinueAsNew` is the sanctioned point to pick up new code).
- **Cancellation semantics defined** (closing the API-contract gap): cancelling an activity cancels in-flight children depth-first; a suspended parent is re-dispatched once so its current `RunChild`/`AwaitSignal`/`Sleep` call site returns `ErrCancelled`, giving the body one bounded opportunity to clean up via already-recorded state (no new dispatches allowed after cancellation); then the activity transitions to `CANCELLED`. Documented in ADR-005.
- Signal addressing from inside activities: `ctx.SendSignal(targetID, name, payload)` (recorded in the call log; replay-safe).
- Sample long-lived activity: a loop of `AwaitSignal` → `RunChild` → `Sleep` that calls `ContinueAsNew` every N iterations — demonstrates bounded replay under unbounded lifetime.
- [impl/PHASE_4_TIMERS_SIGNALS.md](platform/impl/PHASE_4_TIMERS_SIGNALS.md) updated before code: delivery semantics, mailbox storage, `ContinueAsNew` atomicity, cancellation state machine.

**Success Criteria**:
- Monitor activity sleeps for hours (fast-forward harness), receives a signal, wakes, runs a child, suspends again. Kill all workers; restart; signal again; wakes correctly.
- Signal delivered before parent suspends (race) — not lost. Signal delivered while no worker holds the parent — persisted, delivered on resume. `AwaitSignal` timeout fires when no signal arrives.
- Replayed parent returns the same signal payload without re-awaiting.
- Duplicate `:signal` POST with the same `Idempotency-Key` → one mailbox entry.
- **`ContinueAsNew` test**: an activity that has made 500 calls continues-as-new; the successor's first resume replays ~0 entries; signals sent to the activity ID still arrive; the console/API history chain links both executions.
- Cancellation test: cancel a parent suspended on `AwaitSignal` with one running child → child cancelled, parent's call site returns `ErrCancelled`, no new children dispatched, terminal state `CANCELLED` with children accounted for.

**Explicitly out of scope**: JetStream `SignalBus`, determinism linter, coordinator HA. Phase 4 relies on author discipline plus the Phase 3 runtime guards.

**Stop point for prototype use**: the execution model is complete and correct here. Phases 5–9 turn it into an operable, then production-qualified system.

---

### Phase 5: Production engine surface — API completeness, multi-tenancy, deployment safety

**Goal**: The engine is safely consumable by untrusted tenants and operable by humans: complete API, auth, isolation, fairness, and the tooling to deploy new activity code without stranding suspended work. Guide: [impl/PHASE_5_API_TENANCY.md](platform/impl/PHASE_5_API_TENANCY.md).

**Duration**: 4 weeks (solo)

**Key Deliverables**:
- **Complete REST API** per [API_SPEC.md](platform/API_SPEC.md) / [niyanta-v1.yaml](../api/openapi/niyanta-v1.yaml), closing the contract gaps:
  - `POST /activities/{id}:pause` and `:resume` (the `PAUSED` state finally has operations that produce it).
  - **Long-poll wait**: `GET /activities/{id}?wait=30s` returns early on terminal state.
  - **Operator recovery endpoints (G6)**: `force-fail`, `force-complete`, and guarded `truncate-replay`.
  - Cursor-paged attempts, calls, signals, executions, audit, workers, versions, leases, and composition-tree reads.
  - Activity filters required by the console: status, type, tenant (admin only), version pin, condition, root, and created/updated time range.
  - Cluster summary, version inventory/drain progress, worker drain, and `retry-now` contracts.
- **One state boundary**: the console and CLIs use the versioned API only; they never read or mutate core Postgres tables directly. Storage migrations remain private implementation details.
- **Auth + tenancy**: hashed API keys for service clients; principal, tenant, and capability extraction middleware; every query tenant-filtered at the storage boundary. Cross-tenant reads return `NOT_FOUND`. Phase 5 defines capabilities (`activity:read`, `payload:read`, `activity:operate`, `activity:recover`, `worker:drain`, `tenant:admin`) so Phase 7 does not invent authorization in the UI.
- **Safe mutation contract**: every mutating endpoint requires `Idempotency-Key`, `expected_generation` (or `If-Match` resource version), and an operator reason where privileged. Stale state returns `409 CONFLICT` with current metadata. Destructive recovery endpoints provide a preview and short-lived server-issued confirmation token.
- **Transactional privileged audit**: the mutation and its audit row commit in the same database transaction. Async audit remains permitted only for non-authoritative telemetry.
- **Sensitive data policy**: metadata reads and payload reads are separate capabilities. Inputs, results, signals, checkpoints, and logs are redacted server-side by default, size-bounded, and never made safe solely by frontend masking.
- **Fairness + protection**: priority scheduling with tier weights and aging (no activity starves > 30 minutes); per-tenant token-bucket rate limits; queue-depth backpressure (503 + `Retry-After`); per-tenant concurrency caps enforced by the scheduler.
- **Deployment safety (G1 tooling)**: `niyanta versions list` (which pinned versions have live activities, counts by state), `niyanta versions drain <type>@<version>` (stop new pins, report until zero), and a `no_compatible_worker` alert condition — operators can retire old worker versions with evidence instead of hope.
- Idempotent submission (`Idempotency-Key`) with a documented retention window for `idempotency_keys` + cleanup job.
- Structured logging (zerolog or zap) with correlation IDs across activity → attempt → call.
- Doc updates: OpenAPI regenerated as the single source of truth; signal route unified (`:signal`, matching the spec — the v2 plan's `/signals/{name}` variant dropped).

**Success Criteria**:
- Full API exercised via `curl`: submit with idempotency key → long-poll wait → completion; duplicate submit returns the same activity.
- Two tenants isolated in queries, metrics, and rate limits; cross-tenant access yields `NOT_FOUND`.
- Premium activity submitted after a free-tier activity executes first; nothing starves past 30 minutes under sustained load.
- Rate-limit and backpressure paths return correct codes and `Retry-After`.
- **Poison-recovery drill**: deliberately break determinism in a running activity → it fails with `nondeterminism`, is not retried; operator truncates replay via the API; activity resumes and completes; audit trail shows the intervention.
- **Drain drill**: activities pinned to `1.0.0`; run `versions drain`; pins complete or continue-as-new onto `1.1.0`; old workers removed with zero parked activities.
- Race drill: two operators mutate the same generation; one succeeds and the other gets a conflict. Retrying the winner's idempotency key does not repeat the action.
- A committed recovery or drain action always has its audit row after a forced process crash.
- OpenAPI contract tests cover every Phase 6–7 user journey; the generated TypeScript client is built in CI.

**Explicitly out of scope**: the web console (Phase 6 consumes this API), coordinator HA, autoscaling.

---

### Phase 6: Read-only Activity Manager console

**Goal**: A human can find an activity and explain its state from a browser without adding a privileged mutation path. Guide: [impl/PHASE_6_CONSOLE_READ_ONLY.md](platform/impl/PHASE_6_CONSOLE_READ_ONLY.md).

**Duration**: 3 weeks (solo)

- React + TypeScript SPA generated against the Phase 5 OpenAPI client and embedded with `go:embed`; no CDN assets.
- Console backend is a browser-facing adapter to the coordinator API. It owns browser sessions and preferences only; it has no credentials for core Postgres.
- Read-only views: cluster summary, activity search, activity detail, paged attempt/call/replay timeline, progressively loaded composition tree, signal/heartbeat metadata, execution chain, workers, versions, and audit history.
- Payloads are collapsed and redacted by default; revealing them requires `payload:read`, is server-side enforced, and emits a sensitive-read audit event.
- Large collections are cursor-paged or virtualized. Composition children load on demand; the browser never downloads an unbounded tree/log/result.
- Baseline UX quality: responsive layout, keyboard navigation, WCAG 2.2 AA automated checks, empty/error/loading states, and no raw HTML rendering from activity data.

**Success Criteria**:
- Find a suspended activity and identify its exact call index and wait reason in under one minute.
- A 10,000-node composition is navigable through progressive loading without freezing the browser or issuing an unbounded query.
- A principal without `payload:read` cannot obtain payloads by UI, direct API call, or export; a permitted reveal is audited.
- Cross-tenant list, detail, audit, and saved-filter access fails without existence leakage.
- The console binary works air-gapped and passes keyboard and automated accessibility checks.

**Explicitly out of scope**: operator mutations, SSE, embedded metrics storage, coordinator HA.

---

### Phase 7: Operator workflows + live updates

**Goal**: Authorized humans can safely change engine state from a browser, and watched resources converge after disconnects. Guide: [impl/PHASE_7_CONSOLE_OPERATIONS.md](platform/impl/PHASE_7_CONSOLE_OPERATIONS.md).

**Duration**: 3 weeks (solo)

**Key Deliverables**:
- OIDC for browser login where available; a documented local-admin/session mode for air-gapped installations. API keys remain service credentials and are never stored in browser local storage.
- Secure server-side sessions: `HttpOnly`, `Secure`, `SameSite` cookies; CSRF tokens on mutations; CSP, frame, and content-type headers; logout and session revocation.
- Capability-gated actions: cancel, pause/resume, signal, retry-now, force-fail, force-complete, truncate replay, and worker/version drain. Recovery capabilities are distinct from routine operation.
- Each action shows target, current generation, expected effect, reason, and preview. The server enforces generation/version checks, idempotency, and transactional audit. Typed confirmation is additional friction, not the concurrency mechanism.
- SSE contract: tenant-scoped endpoint, monotonic event IDs, heartbeat frames, bounded replay window, `Last-Event-ID`, slow-client limits, and proxy-safe headers. Events are invalidations; after reconnect or a sequence gap the UI refetches authoritative state.
- Saved filters/preferences live in a separate console schema or local config and never in engine core tables.

**Success Criteria**:
- Viewer, operator, recovery operator, and tenant admin capabilities are enforced by the API, not button visibility.
- Two operators acting on one generation produce one commit and one explicit conflict; no silent overwrite.
- Kill the console after an action commits: the corresponding audit row exists after restart.
- Disconnect an SSE client beyond the replay window: reconnect detects the gap, refetches, and converges.
- Cross-tenant SSE subscriptions and guessed resource IDs disclose nothing.

---

### Phase 8: Observability distribution

**Goal**: Operators can measure and alert on engine health without making the console a second mandatory stateful control-plane dependency. Guide: [impl/PHASE_8_OBSERVABILITY.md](platform/impl/PHASE_8_OBSERVABILITY.md).

**Duration**: 3 weeks (solo)

**Key Deliverables**:
- Prometheus `/metrics` on coordinator and workers with a cardinality budget. Per-tenant hot lists come from authorized state APIs; raw customer IDs are not unbounded metric labels by default.
- Metrics for attempt outcomes, queue depth/age by bounded priority class, dispatch/replay/timer latency, worker capacity, fenced writes, parked versions, nondeterminism, and DB pools. Histogram buckets and label limits are load-tested.
- Standard Prometheus alert rules and Grafana dashboards as code. The console can query a configured Prometheus URL for charts, but state inspection remains usable without Prometheus.
- OpenTelemetry spans and trace links. The UI displays trace links only when a trace backend is configured; tracing is not falsely advertised as zero-infrastructure.
- Optional embedded metrics profile only if justified by user need. It is explicitly single-console, non-HA, bounded by bytes and time, and documents persistent volume, WAL recovery, disk-full degradation, scrape discovery, query limits, and upgrade compatibility. Metrics failure must not affect execution.

**Success Criteria**:
- A staged leader/worker/queue failure fires the shipped rules in external Prometheus.
- Cardinality and query load stay within the published budget under the Phase 9 load shape.
- Tenant-scoped charts cannot expose another tenant's identifiers or series.
- Prometheus or the optional local metrics store can be unavailable or disk-full while activity execution and state inspection continue.
- If the embedded profile ships, crash/WAL recovery and version upgrade tests pass; otherwise it remains explicitly deferred.

---

### Phase 9: Production hardening gate

**Goal**: Qualify the engine's durability and operational claims for production. Guide: [impl/PHASE_9_PRODUCTION_HARDENING.md](platform/impl/PHASE_9_PRODUCTION_HARDENING.md).

**Duration**: 4+ weeks (solo); exits on evidence, not schedule.

**Key Deliverables**:
- Coordinator HA using Postgres advisory-lock leadership unless testing proves it inadequate; followers serve API, while only the leader schedules, sweeps timers, and monitors health.
- Backup and point-in-time restore runbook plus automated restore drill for Postgres; recovery-point and recovery-time results published.
- Chaos: leader and worker kills, Postgres pauses/failover, dropped notifications, console loss, and stale workers. Post-run integrity checker validates dispatch, fencing, signals, and child linkage.
- Security gate: threat model, dependency/container scanning, authorization matrix tests, secret/redaction tests, session/CSRF/CSP validation, and least-privilege database/network roles.
- Upgrade/rollback tests across supported database schema and coordinator/worker/console version skews.
- Load and soak tests in state transitions/sec and API/console query concurrency. Publish the supported envelope and size Postgres pools, indexes, pagination, and read replicas from evidence.
- Determinism linter in CI; backup/restore, incident, version-drain, and rollback runbooks.

**Success Criteria**:
- Leader kill fails over within the stated SLO with no lost progress or duplicate child dispatch.
- Restore drill meets the published RPO/RTO and the integrity checker passes on restored state.
- A 24-hour representative soak and a one-hour chaos run complete with zero invariant violations.
- Cross-tenant, privilege-escalation, payload-redaction, and SSE authorization suites pass.
- One-version upgrade and rollback succeed with live suspended activities.
- Production capacity and operational limits are published; unsupported scale is rejected or backpressured, not guessed.

---

## Risk Management

Rewritten for v3 — the v2 table predated the activity model and carried stale rows.

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| **Replay semantics subtly wrong** (the engine's core) | Critical | Medium | Docs-before-code for Phases 3–4; failing tests first; integrity checker in chaos runs; exactly-once dispatch asserted by counting, not observing |
| Version pinning gaps strand suspended activities | High | Medium | `no_compatible_worker` condition from Phase 3; drain tooling Phase 5; console hot list Phase 6; alert Phase 8 |
| Postgres becomes the throughput ceiling (transitions/sec) | High | Medium | Measure from Phase 3; segment batching; Phase 9 capacity envelope; add a broker/read replica only from measured evidence |
| Fencing regression in a new store method | High | Low | Fencing lives in the shared storage contract-test suite — every impl, every method, forever |
| Unbounded call logs before `ContinueAsNew` lands | Medium | Low | Schema groundwork in Phase 3; sample long-lived activities use it from Phase 4; replay-duration histogram alerts |
| Privileged UI action lands without durable audit or stale-state protection | High | Medium | Capability checks, expected generation, idempotency, and audit in the same transaction in Phase 5; exercised in Phase 7 |
| Payloads or metrics leak tenant data | Critical | Medium | Separate `payload:read`, server redaction, sensitive-read audit, bounded metric labels, cross-tenant API/SSE/metrics suites |
| Console scope creep delays hardening | Medium | High | Read-only, operations, and observability are separate phases; embedded TSDB is optional and cannot gate engine operation |
| Solo-execution estimate slip | Medium | High | Every phase ends runnable; production is evidence-gated at Phase 9 rather than declared by calendar |

---

## Success Metrics

Cross-phase quality bars (per-phase criteria live in each phase narrative):

- **After Phase 2**: worker failure → redistribution < 2 minutes; stale writers provably fenced at the storage layer.
- **After Phase 3**: children dispatched exactly once across arbitrary parent kills; replay short-circuit in microseconds per logged call; code redeploys never manufacture determinism violations (pinning); violations that do occur are non-retried and inspectable.
- **After Phase 4 (model complete)**: unbounded-lifetime activity with bounded replay via `ContinueAsNew`; signals durable across total worker loss; cancellation leaves no orphaned children.
- **After Phase 5**: API p95 < 200ms; tenant isolation with no existence leaks; nothing starves > 30 min; poison-recovery and version-drain drills executable via API alone.
- **After Phase 6**: any activity's exact position explainable from the read-only console in < 1 minute; large histories remain bounded in API and browser memory.
- **After Phase 7**: every privileged action is authorized, concurrency-safe, idempotent, and transactionally audited; SSE reconnects converge.
- **After Phase 8 (operational beta)**: alert rules cover leader, starvation, parked versions, nondeterminism, fencing, and DB health within a published cardinality budget.
- **After Phase 9 (production gate)**: leader failover, restore, upgrade/rollback, security, 24h load, and chaos suites pass with a published capacity envelope.

---

## Timeline & Milestones

Solo-engineer estimates. Times double under part-time attention; phase boundaries are designed so stopping between phases leaves a working system.

### Solo Gantt (rough)

```
Week 1:      [Phase 0: skeleton + in-memory]
Week 2-3:    [Phase 1: Postgres + retries + write fencing]
Week 4-5:    [Phase 2: coordinator/workers + Postgres dispatch + affinity]
Week 6-9:    [Phase 3: RunChild + replay + version pinning + G4/G6]
Week 10-13:  [Phase 4: Sleep/Now/NewID + signals + ContinueAsNew]   ← model complete
Week 14-17:  [Phase 5: operator API + auth + tenancy + safe mutations]
Week 18-20:  [Phase 6: read-only Activity Manager]
Week 21-23:  [Phase 7: operator workflows + SSE]
Week 24-26:  [Phase 8: metrics + alerts + tracing]                 ← operational beta
Week 27-30+: [Phase 9: HA + restore + security + chaos + scale]    ← production gate
```

### Key Milestones

| Milestone | Phase end | Description |
|-----------|-----------|-------------|
| **M0**: Skeleton | Phase 0 | All interfaces exist; in-memory demo runs |
| **M1**: Persistent + fenced | Phase 1 | Survives restart; framework retries; no stale writes |
| **M2**: Distributed | Phase 2 | Coordinator + workers; redistribution; fencing proven end-to-end |
| **M3**: Composable | Phase 3 | Exactly-once `RunChild`; version-safe replay; violation recovery path |
| **M4**: Model complete | Phase 4 | Timers, signals, `ContinueAsNew`, cancellation semantics |
| **M5**: Consumable | Phase 5 | Complete API, multi-tenant, fair, deploy-safe |
| **M6**: Inspectable | Phase 6 | Read-only console explains activity and cluster state |
| **M7**: Operable | Phase 7 | Safe browser actions and reconnectable live updates |
| **M8**: Observable beta | Phase 8 | Metrics, alerts, dashboards, and tracing with bounded cardinality |
| **M9**: Production-qualified | Phase 9 | HA, restore, security, upgrade, load, and chaos evidence |

---

## Post-Phase 9 Roadmap

Items not on the engine's critical path. Pick up as needed.

- Bounded parallel children (`RunChildren([]...)`) — the call-log mechanics (call-site keying) were reserved in ADR-005 open question 1.
- gRPC API alongside REST.
- Scheduled / cron-like activity submission.
- DAG sugar over `RunChild`.
- Custom worker pools per tenant; multi-region with region-aware scheduling.
- Multi-cluster console federation.
- Billing and usage tracking; self-service tenant dashboard.
- Marketplace for third-party activity types.
- Optional JetStream task/signal substrates, only if the published Postgres envelope is insufficient.
- Optional embedded Prometheus-compatible metrics profile, if Phase 8 user evidence justifies its lifecycle cost.
- Sticky-worker replay cache and blob offload, when profiling shows they are needed.

Application compositions (ingestion platforms, agentic operations, anything else) are separate products with separate plans; they consume the engine API/SDK and never appear on this roadmap.

---

## Appendix

### A. Technology Stack Summary

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Language | Go 1.22+ | Performance, concurrency, ecosystem |
| API framework | stdlib `net/http` + chi (or Echo) | Small surface; the API is not the hard part |
| Database | PostgreSQL 15+ | ACID, `SKIP LOCKED`, `LISTEN/NOTIFY`, JSONB — the engine's required stateful dependency |
| Dispatch/broker | Postgres-native (P2); NATS JetStream post-P9 if measured | Minimal dependencies; interfaces allow substitution |
| Console frontend | React + TypeScript, `go:embed` | Single static binary, air-gap friendly |
| Live updates | SSE | Simpler than websockets for one-way streams |
| Logging | Zerolog or Zap | Structured, high-performance |
| Metrics | Prometheus exposition; external Prometheus production default | Avoids making the console a mandatory stateful monitoring system; embedded profile is optional |
| Dashboards/alerts | Console charts + Prometheus rules + Grafana JSON in `deploy/` | Reviewable, versioned observability; state hot lists come from the API |
| Tracing | OpenTelemetry → Jaeger/Tempo | Exemplar-linked from histograms |
| Migrations | golang-migrate | Versioned up/down pairs |
| Deployment | docker-compose (dev), Kubernetes (prod) | Portable |
| CI/CD | GitHub Actions | Includes determinism lint and security/contract suites by P9 |

### B. Required follow-up doc edits (tracked)

| Doc | Edit | With phase |
|-----|------|-----------|
| ADR-005 | Exactly-once dispatch wording (G4); heartbeat mutual-exclusivity rule (G2); `ContinueAsNew` section (G5); cancellation semantics; version-pinning replaces the "runs to completion on its version" hand-wave (G1) | P1–P4 |
| DATA_MODELS.md | Attempt-status enum; checkpoint unique key; lease on attempt; fencing invariant in transaction boundaries; `execution_seq`/`superseded_by`; signal dedupe key; payload caps | P1–P4 |
| ARCHITECTURE.md | State diagram gains `SUSPENDED`/`FAILED`; broker section rewritten for Postgres-native dispatch with JetStream as P7 option; app-composition sections move to their own docs | P2 |
| niyanta-v1.yaml | pause/resume, wait param, recovery endpoints, executions chain, call-log/mailbox reads, signal idempotency header | P5 |
| API_SPEC.md / niyanta-v1.yaml | console filters, composition/cluster/version/lease reads, action preconditions, capabilities, SSE contract | P5–P7 |
| ARCHITECTURE.md | console uses API only; browser/session boundary; external Prometheus default; production gate | P6–P9 |

---

## References

- [ADR-005](platform/adr/ADR-005-activity-execution-model.md) — Activity execution model (amended per §Gap Resolutions)
- [ARCHITECTURE.md](platform/ARCHITECTURE.md) — System architecture
- [DATA_MODELS.md](platform/DATA_MODELS.md) — Schema (extended in Phases 1, 3, 4)
- [API_SPEC.md](platform/API_SPEC.md) / [niyanta-v1.yaml](../api/openapi/niyanta-v1.yaml) — API contracts (completed in Phase 5)
- [CLAUDE.md](../CLAUDE.md) — Project overview and Go conventions

---

## Revision History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025-10-27 | Initial implementation plan (workload-based model, multi-engineer phasing). |
| 2.0 | 2026-04-28 | Reframed around the activity execution model from ADR-005. |
| 2.1 | 2026-05-01 | Recentered product framing around self-orchestrated ingestion. |
| 2.2 | 2026-05-03 | Split connector ecosystem breadth into Phases 5A/5B. |
| 3.0 | 2026-07-06 | **Use-case-agnostic engine plan.** Removed all application phases (5/5A/5B connector, ingestion, LLMOps content). Scheduled six contract-gap resolutions: write-path fencing (P1/P2), version-pinned replay (P3), exactly-once child dispatch (P3), determinism-violation recovery (P3/P5), heartbeat×replay mutual exclusivity (P1/P3), `ContinueAsNew` history truncation (P4). Replaced NATS-based Phase 2 dispatch with Postgres-native pull dispatch; JetStream deferred to Phase 7 as optional substrate. Added API-completeness work (pause/resume, long-poll wait, recovery endpoints, signal dedupe) to Phase 5. Added Phase 6: Activity Manager console with cluster view, replay/attempt/composition visualizations, RBAC'd operator actions, and Prometheus-backed observability (metrics, alert rules, Grafana dashboards, OTel tracing). Former Phase 6 hardening renumbered to Phase 7 with sticky-worker replay optimization and transition-throughput load model. Risk table rewritten (top risk: replay correctness). |
| 3.1 | 2026-07-10 | Grounded the control-panel delivery plan after critical review. Expanded Phase 5 with the operator API, capability model, mutation preconditions, transactional audit, and payload policy. Split the former Phase 6 into read-only console (P6), safe operator workflows/SSE (P7), and observability distribution (P8). Made external Prometheus the production default and any embedded store optional. Added a mandatory production hardening gate (P9) for HA, restore, security, upgrade/rollback, chaos, and capacity evidence. |

---

**End of Implementation Plan**
