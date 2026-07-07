# Niyanta Implementation Plan

**Version**: 3.0
**Last Updated**: 2026-07-06
**Status**: Planning Phase
**Supersedes**: v2.2. This revision makes the plan **use-case agnostic**: Niyanta is planned, built, and hardened purely as a durable activity execution engine. Application compositions (ingestion, LLMOps, or anything else) are built *on* the engine and are planned in their own documents — they no longer appear in this plan. This revision also resolves six engine-contract gaps identified in design review (see §[Gap Resolutions](#gap-resolutions)) and adds a first-class **Activity Manager console** with built-in cluster and Prometheus observability.

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
   - [Phase 6: Activity Manager console + observability stack](#phase-6-activity-manager-console--observability-stack)
   - [Phase 7: HA, scale, hardening](#phase-7-ha-scale-hardening)
4. [Risk Management](#risk-management)
5. [Success Metrics](#success-metrics)
6. [Timeline & Milestones](#timeline--milestones)
7. [Post-Phase 7 Roadmap](#post-phase-7-roadmap)
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
6. **Phase 6**: **Activity Manager console with integrated observability** — a web UI over the engine with activity search/detail/composition-tree/replay-timeline views, operator actions, and a live cluster view, shipping its **own built-in metrics pipeline** (embedded scraper + local TSDB) so observability works with zero external infrastructure, while every component stays Prometheus-compatible (`/metrics`, PromQL, standard alert rule files) for teams that run their own stack.
7. **Phase 7**: Coordinator HA, optional NATS JetStream substrates, determinism linter, chaos testing, scale tuning to 10K concurrent activities.

### What changed from v2

- **All application content removed.** Former Phases 5/5A/5B (connector registry, ingestion planner, catalog import, CDC, Airbyte compatibility) are out of this plan entirely. Compositions consume the engine's public API and SDK; they never gate or shape engine phases. Engine features are justified only by the engine's own contract.
- **Six contract gaps are now scheduled work** with owning phases and acceptance tests (§[Gap Resolutions](#gap-resolutions)).
- **NATS removed from the critical path.** Phase 2 dispatch is Postgres-native (`SELECT … FOR UPDATE SKIP LOCKED` pull model + `LISTEN/NOTIFY` for latency). This honors the minimal-external-dependencies principle; NATS JetStream returns in Phase 7 as an optional scale substrate behind the existing interfaces. Rationale: the v2 plan built a Postgres task queue in Phase 1 and then added a second, push-based dispatch path over NATS in Phase 2 — two dispatch mechanisms, one extra stateful dependency, no correctness gain (the broker held no durable state and the state store remained the source of truth).
- **New Phase 6** delivers the Activity Manager console and observability stack as a first-class engine deliverable, not an afterthought.

### Phases Overview

| Phase | Duration (solo) | Key Deliverables |
|-------|------|------------------|
| Phase 0 | 1 week | Project skeleton, all interfaces stubbed, in-memory impls, single-binary demo |
| Phase 1 | 2 weeks | Postgres-backed execution, framework retries, per-attempt records, generation-fenced writes, survives restart |
| Phase 2 | 2 weeks | Coordinator + worker split, Postgres-native dispatch, failure redistribution, leases, affinity |
| Phase 3 | 4 weeks | `RunChild` + replay, exactly-once dispatch, version-pinned replay, determinism-violation recovery |
| Phase 4 | 4 weeks | `Sleep`/`Now`/`NewID`, `SignalBus` + `AwaitSignal`, signal dedupe, `ContinueAsNew`, cancellation semantics |
| Phase 5 | 3 weeks | Complete REST API, auth, multi-tenant isolation, priority + aging, rate limits, backpressure, version drain |
| Phase 6 | 3 weeks | Activity Manager web console with integrated observability (built-in metrics store, cluster view, Prometheus-compatible), alert rules, OTel tracing |
| Phase 7 | open-ended | Coordinator HA, JetStream substrates (optional), determinism lint, chaos, scale to targets |

**Stop point for prototypes**: end of Phase 4 — the full execution model is expressible and correct.
**Stop point for production**: end of Phase 6 — multi-tenant, observable, operable engine with a console.
**Phase 7** is incremental hardening driven by actual scale needs.

**Total time to model completeness (solo)**: ~13 weeks (Phases 0–4).
**Total time to production engine (solo)**: ~19 weeks (Phases 0–6).

---

## Gap Resolutions

Design review identified six gaps in the engine contract. Each is now owned by a phase and has an acceptance test. Follow-up edits to [ADR-005](platform/adr/ADR-005-activity-execution-model.md) and [DATA_MODELS.md](platform/DATA_MODELS.md) are required to match (tracked per phase below).

| # | Gap | Resolution | Owning phase |
|---|-----|------------|--------------|
| G1 | **Versioning under replay.** A suspended parent may resume on a worker running newer code; a reordered/added/removed `RunChild` produces a false "determinism violation." | Workers advertise `{activity_type, version}`. Activities that have suspend-point history are **pinned**: they resume only on workers advertising the version recorded at first dispatch. Version drain tooling lets operators retire old versions safely. | P3 (pinning), P5 (drain tooling), P6 (console visibility) |
| G2 | **Heartbeat × replay composition undefined.** Branching on `LastHeartbeat()` in code that also suspends breaks replay determinism. | **Mutual exclusivity rule**: heartbeat state is readable only in activities that never suspend. Stated in P1 docs; enforced at runtime in P3 (calling `LastHeartbeat` in an activity with any call-log entries is an immediate, clear error); linted in P7. Heartbeat scope is fixed as **cross-attempt within the current version pin** (retry sees prior attempt's heartbeat), and the `checkpoints` unique key includes `attempt_id`. | P1 (rule + schema), P3 (guard), P7 (lint) |
| G3 | **Fencing specified on the command path only.** A stale (SIGSTOP'd, partitioned) worker can still *write* completions, heartbeats, or call-log rows. | **Write-path fencing invariant**: every state-mutating write carries the worker's generation and commits only under `WHERE generation = $expected`. No exceptions. Invariant documented in DATA_MODELS transaction boundaries; every store impl tested against it. | P1 (single-process), P2 (multi-worker acceptance test) |
| G4 | **Idempotency bound understated.** ADR-005 says children "may be invoked twice"; the transactional call-log design actually supports exactly-once dispatch. | `RunChild` dispatch is transactional with the call-log append: on replay, an existing incomplete call-log row means **wait, never re-dispatch**. Contract tightened to: *child dispatch is exactly-once; side effects inside a child are at-least-once, bounded by the child's retry policy*. | P3 |
| G5 | **Unbounded history for unbounded-lifetime activities.** `Sleep`/`AwaitSignal` invite immortal activities; replay cost grows monotonically with call count; retention cannot truncate a live activity's log. | **`ctx.ContinueAsNew(input)`**: atomically complete the current execution and start a successor with a fresh call log, preserving the activity's external identity (ID for signal addressing, composition links, console history chain). Designed before Phase 3 code freezes the call-log schema; implemented in Phase 4. | P4 (schema groundwork in P3) |
| G6 | **No recovery path from determinism violation.** Detection exists; the activity is silently poisoned — retry replays into the same wall. | Distinct terminal-ish state `FAILED` with `error_json.type = "nondeterminism"` that **suppresses automatic retry**. Operator recovery endpoints: force-fail, force-complete-with-result, and guarded truncate-replay-to-last-consistent-prefix + resume. Surfaced in the console. | P3 (state + suppression), P5 (recovery API), P6 (console) |

Additional review items folded in: attempt-status gets its own enum including `INTERRUPTED` (P1); lease moves to the attempt (P2); call log keyed by `activity_id` (P3, matching DATA_MODELS, since replay spans attempts); `result_json` size cap (P3); `Now()`/`NewID()` recorded per call with segment batching (P4); signal sends take an idempotency key (P4); pause/resume + long-poll wait endpoints (P5).

---

## Development Phases

The phases are designed so that **each one ends with a runnable system**. If you stop after phase N, what you have works for what phase N promises — no half-built state.

### Guiding principles

1. **Interfaces from day one.** Every storage and dispatch concern lives behind an interface starting in Phase 0: `ActivityStore`, `AttemptStore`, `HeartbeatStore`, `ActivityCallStore`, `WorkerRegistry`, `LeaderElector`, `TaskQueue`, `SignalBus`, `EventBus`. Phase 1's Postgres impls can be swapped for Phase 7's JetStream impls without touching activity code.
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
- **Postgres-native dispatch (no broker)**: workers pull work with `FOR UPDATE SKIP LOCKED` against the task queue, filtered by their advertised activity types/tags; `LISTEN/NOTIFY` wakes idle workers so dispatch latency isn't bounded by the poll interval. Worker registration, heartbeats (every 30s, with capacity and in-flight inventory), and completions are rows/notifications in Postgres. The `EventBus`/`TaskQueue` interfaces are unchanged — a NATS/JetStream impl remains a Phase 7 drop-in if scale demands it.
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

**Explicitly out of scope**: `Sleep`, `Now`, `NewID`, signals, `ContinueAsNew` behavior (schema only), in-process child optimization (children always dispatch through the queue; sticky-worker optimization is Phase 7).

---

### Phase 4: Deterministic primitives, signals, ContinueAsNew

**Goal**: Activities can sleep durably, wake on time or on external signals, and run for unbounded lifetimes **without unbounded replay cost**. Cancellation of suspended activities has defined semantics.

**Duration**: 4 weeks (solo)

**Key Deliverables**:
- `ctx.Now()`, `ctx.NewID()` — recorded **per call** as synthetic call-log entries (`_now`, `_id`); replayed thereafter. (The v2 wording "recorded on first call within an attempt" was wrong — each call site gets its own deterministic value.) **Segment batching**: consecutive synthetic entries between real suspend points are written in one batch at the next durable write, so a loop calling `Now()` doesn't issue one Postgres write per iteration.
- `ctx.Sleep(duration)` — records `_sleep` with `wake_at`; suspends; coordinator timer sweep (index on `wake_at`, poll every second) re-enqueues due activities. Multi-day sleeps cost no worker resources.
- `SignalBus` interface in `internal/signal/` + Postgres impl (`signals` mailbox table + `NOTIFY`). Substrate fully abstracted; JetStream alternative is Phase 7.
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

**Stop point for prototype use**: the execution model is complete and correct here. Phases 5–7 are productionization.

---

### Phase 5: Production engine surface — API completeness, multi-tenancy, deployment safety

**Goal**: The engine is safely consumable by untrusted tenants and operable by humans: complete API, auth, isolation, fairness, and the tooling to deploy new activity code without stranding suspended work. Guide: [impl/PHASE_5_API_TENANCY.md](platform/impl/PHASE_5_API_TENANCY.md).

**Duration**: 3 weeks (solo)

**Key Deliverables**:
- **Complete REST API** per [API_SPEC.md](platform/API_SPEC.md) / [niyanta-v1.yaml](../api/openapi/niyanta-v1.yaml), closing the contract gaps:
  - `POST /activities/{id}:pause` and `:resume` (the `PAUSED` state finally has operations that produce it).
  - **Long-poll wait**: `GET /activities/{id}?wait=30s` returns early on terminal state — the most common client interaction stops being a poll loop. (SSE stream endpoint reserved for Phase 6 console.)
  - **Operator recovery endpoints (G6)**: `POST /activities/{id}:force-fail`, `:force-complete` (operator-supplied result), and `:truncate-replay` (drop the call-log suffix from the divergent index, guarded by a `confirm` token + audit event) — the documented escape hatches for nondeterminism poisoning.
  - Attempts, call-log, and signal-mailbox read endpoints (the console's data source, also useful for debugging via `curl`).
  - Executions chain endpoint for `ContinueAsNew` history.
- **Auth + tenancy**: API-key auth (hashed at rest), customer extraction middleware, every query tenant-filtered at the storage boundary; contract tests assert cross-tenant reads return `NOT_FOUND`, not `FORBIDDEN` (no existence leaks).
- **Fairness + protection**: priority scheduling with tier weights and aging (no activity starves > 30 minutes); per-tenant token-bucket rate limits; queue-depth backpressure (503 + `Retry-After`); per-tenant concurrency caps enforced by the scheduler.
- **Deployment safety (G1 tooling)**: `niyanta versions list` (which pinned versions have live activities, counts by state), `niyanta versions drain <type>@<version>` (stop new pins, report until zero), and a `no_compatible_worker` alert condition — operators can retire old worker versions with evidence instead of hope.
- Idempotent submission (`Idempotency-Key`) with a documented retention window for `idempotency_keys` + cleanup job.
- Structured logging (zerolog or zap) with correlation IDs across activity → attempt → call; audit events written async (channel + writer goroutine) so audit never blocks the hot path.
- Doc updates: OpenAPI regenerated as the single source of truth; signal route unified (`:signal`, matching the spec — the v2 plan's `/signals/{name}` variant dropped).

**Success Criteria**:
- Full API exercised via `curl`: submit with idempotency key → long-poll wait → completion; duplicate submit returns the same activity.
- Two tenants isolated in queries, metrics, and rate limits; cross-tenant access yields `NOT_FOUND`.
- Premium activity submitted after a free-tier activity executes first; nothing starves past 30 minutes under sustained load.
- Rate-limit and backpressure paths return correct codes and `Retry-After`.
- **Poison-recovery drill**: deliberately break determinism in a running activity → it fails with `nondeterminism`, is not retried; operator truncates replay via the API; activity resumes and completes; audit trail shows the intervention.
- **Drain drill**: activities pinned to `1.0.0`; run `versions drain`; pins complete or continue-as-new onto `1.1.0`; old workers removed with zero parked activities.

**Explicitly out of scope**: the web console (Phase 6 consumes this API), coordinator HA, autoscaling.

---

### Phase 6: Activity Manager console + integrated observability

**Goal**: A human can operate the engine from a browser: find any activity, understand exactly where it is (attempts, call log, composition tree, signals), act on it, and see cluster and workload health. Observability is **integrated**: the console ships its own metrics pipeline and is fully functional with zero external observability infrastructure, while staying Prometheus-compatible end to end. Guide: [impl/PHASE_6_CONSOLE_OBSERVABILITY.md](platform/impl/PHASE_6_CONSOLE_OBSERVABILITY.md).

**Duration**: 3 weeks (solo)

**Key Deliverables**:

**Metrics exposition (Prometheus-compatible)** — lands first:
- `/metrics` on coordinator and workers. Engine metric set per [ARCHITECTURE.md](platform/ARCHITECTURE.md): activities by `{customer, activity_type, status}`; attempt outcomes; scheduler queue depth and age by priority; dispatch latency histogram; replay duration histogram and replayed-call counts; timer-sweep lag; signal delivery counts and mailbox depth; worker capacity/utilization; fenced-write counter (G3 — a nonzero rate is a redistribution event trail); `no_compatible_worker` parked-activity gauge (G1); nondeterminism-failure counter (G6).
- **Shippable alert rules** (`deploy/prometheus/alerts.yaml`, standard Prometheus rule format): leader absent, dispatch latency SLO breach, dead-worker rate, parked-on-version activities > 0 for > 15m, nondeterminism failures > 0, queue starvation, timer-sweep lag, DB pool saturation. Evaluated by the console's integrated store out of the box; loadable into an external Prometheus unchanged.
- **Grafana dashboards as code** (`deploy/grafana/*.json`) for teams that prefer Grafana for deep dives — optional; the console is sufficient on its own.
- OpenTelemetry tracing: span per submission → schedule → attempt → each `RunChild`/`Sleep`/`AwaitSignal` → completion; trace context propagated through `ActivityContext`; exemplars linked from latency histograms.

**Activity Manager console** (`cmd/console/`, or served by coordinator followers):
- **Architecture**: Go backend that (a) serves the engine's read API + operator actions with RBAC (viewer / operator / admin roles on top of Phase 5 auth), (b) runs an **integrated metrics store** — an embedded scraper + local TSDB (Prometheus TSDB library, bounded retention, 15 days default) that scrapes coordinator/worker `/metrics` itself and answers PromQL for charts, hot lists, and alert evaluation with **no external Prometheus deployed**; if an external Prometheus URL is configured, the console federates to it instead (same queries, same charts, no code-path divergence), (c) streams live updates over SSE. Frontend: React + TypeScript SPA embedded via `go:embed` — single static binary, no separate deploy, works air-gapped (no CDN assets).
- **Views**:
  - **Cluster dashboard**: coordinator leader + health, worker fleet (status, capacity, versions, last heartbeat, in-flight counts), queue depths and ages, live throughput/error sparklines (Prometheus-backed), active alerts.
  - **Activities**: search/filter by status, type, tenant, version pin, time range, root; saved filters for the operational hot lists — parked on version (G1), nondeterminism failures (G6), starving, lease-expiring.
  - **Activity detail** — the flagship view: status + timings; **attempt timeline** (every physical attempt with worker, generation, error, duration); **call-log / replay timeline** (each `RunChild`/`Sleep`/`AwaitSignal`/`_now`/`_id` entry with completion state and recorded result — for a suspended activity this *is* "exactly where is it"); **composition tree** (parent/children graph, navigable); signal mailbox (pending + delivered); heartbeat state (for leaf activities); `ContinueAsNew` execution chain; linked trace.
  - **Workers detail**: per-worker in-flight activities, capability/version set, lease table, drain button.
  - **Versions**: pinned-version inventory and drain progress (the Phase 5 tooling, visualized).
- **Operator actions** (RBAC-gated, every action writes an audit event, destructive ones require typed confirmation): cancel, pause/resume, send signal, retry-now, force-fail, force-complete, truncate-replay, worker drain.
- **Live behavior**: SSE-driven updates on dashboards and detail views (no manual refresh while watching an activity resume).
- docker-compose: postgres + coordinator + 2 workers + console — one command to a fully observable local cluster with **no other observability services**; an extended profile adds prometheus + grafana + jaeger to exercise the external-stack path.

**Success Criteria**:
- **Zero-infra observability**: with nothing deployed beyond engine + console, the console renders throughput, latency, queue, and replay charts from its integrated store, and the shipped alert rules evaluate and fire in a staged failure (leader stop, queue starvation).
- **External-stack parity**: pointed at an external Prometheus, the console shows the same charts and hot lists with no behavioral difference.
- From a browser: find a suspended activity, see which call index it is parked on and what it's waiting for, send it a signal, watch it resume live.
- Composition tree renders a 3-level parent/child pipeline correctly, with per-node status.
- Kill a worker: cluster dashboard shows it dead within the heartbeat timeout, the redistribution is visible on the affected activity's attempt timeline, and the fenced-write counter ticks when the stale worker returns.
- The G1/G6 hot lists work: a parked-on-version activity and a nondeterminism failure each appear in their saved filter, and the recovery drill from Phase 5 can be executed entirely from the UI, leaving an audit trail.
- Console binary serves the SPA with no external network dependency; viewer role cannot see action buttons; operator actions are audited.

**Explicitly out of scope**: coordinator HA, autoscaling, JetStream, multi-cluster console.

---

### Phase 7: HA, scale, hardening

**Goal**: Coordinator HA, optional broker substrates, determinism lint, chaos testing, scale to the targets in [ARCHITECTURE.md](platform/ARCHITECTURE.md) §Performance Targets. Guide: [impl/PHASE_7_HA_SCALE_HARDENING.md](platform/impl/PHASE_7_HA_SCALE_HARDENING.md).

**Duration**: open-ended; tackle items selectively based on actual scale needs.

**Key Deliverables**:
- Coordinator HA via `LeaderElector` (pick one: Postgres advisory lock *or* etcd; align docs). Followers serve API + console; leader runs scheduler, health monitor, timer sweep, signal listener. Generation tokens already prevent split-brain (Phase 1/2); HA failover tests prove it under leader churn.
- **Optional NATS JetStream substrates** behind the existing interfaces: `SignalBus` (removes the LISTEN/NOTIFY single-listener constraint) and/or `TaskQueue` — adopted only if measured Postgres dispatch throughput demands it. This is where the broker enters the system, if ever.
- **Determinism linter (G2/ADR-005)**: static analysis flagging `time.Now()`, `rand.*`, map iteration ordering dependence, goroutine spawns, direct I/O, and `Heartbeat`/`LastHeartbeat` in any activity that uses suspend points. CI-enforced.
- **Sticky-worker replay optimization**: prefer resuming a parent on the worker that last held it, with a bounded in-memory replay cache — cuts the per-suspend-point scheduler round-trip for hot compositions. (The suspend-and-free path remains the durable fallback; correctness never depends on stickiness.)
- Chaos testing: random worker kills, leader kills, Postgres pauses/failovers, dropped notifications. Verify: no lost activities, exactly-once child dispatch holds, fencing holds, signal delivery within SLO.
- Performance: load test to targets — and the meaningful load metric is **state transitions/sec** (each suspend point is several transactions), not concurrent-activity count. Profile; index per real query patterns; document the engine's throughput envelope as part of its contract.
- Heartbeat/result blob offload via a `BlobStore` interface (S3) for payloads above threshold, pointer in Postgres.
- Read replicas for console/list/audit queries at scale.

**Success Criteria**:
- Kill the leader: failover < 30s, no activity progress lost, no duplicate child dispatch.
- Sustained 1,000 activities/min for 24h with zero correctness errors; transition throughput envelope published.
- Determinism lint catches a deliberately-broken activity in CI.
- 1-hour chaos run: system recovers within SLO; call-log invariants verified post-run by an integrity checker (every completed call has exactly one child; no orphan children).
- HPA (K8s) scales workers on queue-depth metric up and down.

---

## Risk Management

Rewritten for v3 — the v2 table predated the activity model and carried stale rows.

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| **Replay semantics subtly wrong** (the engine's core) | Critical | Medium | Docs-before-code for Phases 3–4; failing tests first; integrity checker in chaos runs; exactly-once dispatch asserted by counting, not observing |
| Version pinning gaps strand suspended activities | High | Medium | `no_compatible_worker` gauge + alert from Phase 3; drain tooling Phase 5; console hot list Phase 6 |
| Postgres becomes the throughput ceiling (transitions/sec) | High | Medium | Measure transitions/sec from Phase 3; segment batching; sticky-worker cache and JetStream substrates staged in Phase 7 |
| Fencing regression in a new store method | High | Low | Fencing lives in the shared storage contract-test suite — every impl, every method, forever |
| Unbounded call logs before `ContinueAsNew` lands | Medium | Low | Schema groundwork in Phase 3; sample long-lived activities use it from Phase 4; replay-duration histogram alerts |
| Console scope creep delays engine hardening | Medium | Medium | Console is read-API + Prometheus proxy only; no console-private state beyond RBAC/saved filters |
| Solo-execution estimate slip | Medium | High | Every phase ends runnable; stop points at 4 and 6 are real products |

---

## Success Metrics

Cross-phase quality bars (per-phase criteria live in each phase narrative):

- **After Phase 2**: worker failure → redistribution < 2 minutes; stale writers provably fenced at the storage layer.
- **After Phase 3**: children dispatched exactly once across arbitrary parent kills; replay short-circuit in microseconds per logged call; code redeploys never manufacture determinism violations (pinning); violations that do occur are non-retried and inspectable.
- **After Phase 4 (model complete)**: unbounded-lifetime activity with bounded replay via `ContinueAsNew`; signals durable across total worker loss; cancellation leaves no orphaned children.
- **After Phase 5**: API p95 < 200ms; tenant isolation with no existence leaks; nothing starves > 30 min; poison-recovery and version-drain drills executable via API alone.
- **After Phase 6 (production engine)**: any activity's exact position explainable from the console in < 1 minute; every operator action audited; alert rules cover leader, starvation, parked versions, nondeterminism, fencing anomalies.
- **After Phase 7**: leader failover < 30s with zero duplicate dispatch; 24h sustained load clean; published transition-throughput envelope.

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
Week 14-16:  [Phase 5: full API + auth + tenancy + drain tooling]
Week 17-19:  [Phase 6: Activity Manager console + Prometheus stack] ← production engine
Week 20+:    [Phase 7: HA + JetStream option + lint + chaos + scale]
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
| **M6**: Operable | Phase 6 | Console + cluster view + Prometheus observability |
| **M7**: Hardened | Phase 7 | HA, scale targets met, chaos-proven |

---

## Post-Phase 7 Roadmap

Items not on the engine's critical path. Pick up as needed.

- Bounded parallel children (`RunChildren([]...)`) — the call-log mechanics (call-site keying) were reserved in ADR-005 open question 1.
- gRPC API alongside REST.
- Scheduled / cron-like activity submission.
- DAG sugar over `RunChild`.
- Custom worker pools per tenant; multi-region with region-aware scheduling.
- Multi-cluster console federation.
- Billing and usage tracking; self-service tenant dashboard.
- Marketplace for third-party activity types.

Application compositions (ingestion platforms, agentic operations, anything else) are separate products with separate plans; they consume the engine API/SDK and never appear on this roadmap.

---

## Appendix

### A. Technology Stack Summary

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Language | Go 1.22+ | Performance, concurrency, ecosystem |
| API framework | stdlib `net/http` + chi (or Echo) | Small surface; the API is not the hard part |
| Database | PostgreSQL 15+ | ACID, `SKIP LOCKED`, `LISTEN/NOTIFY`, JSONB — the only required external dependency through Phase 6 |
| Dispatch/broker | Postgres-native (P2); NATS JetStream optional (P7) | Minimal dependencies; interfaces allow substitution |
| Console frontend | React + TypeScript, `go:embed` | Single static binary, air-gap friendly |
| Live updates | SSE | Simpler than websockets for one-way streams |
| Logging | Zerolog or Zap | Structured, high-performance |
| Metrics | Prometheus exposition + embedded TSDB in console | Integrated out of the box; PromQL-compatible; external Prometheus optional via federation |
| Dashboards/alerts | Console built-in + Prometheus rule files (+ optional Grafana JSON), as code in `deploy/` | Reviewable, versioned observability that works with zero extra infra |
| Tracing | OpenTelemetry → Jaeger/Tempo | Exemplar-linked from histograms |
| Migrations | golang-migrate | Versioned up/down pairs |
| Deployment | docker-compose (dev), Kubernetes (prod) | Portable |
| CI/CD | GitHub Actions | Includes determinism lint (P7) |

### B. Required follow-up doc edits (tracked)

| Doc | Edit | With phase |
|-----|------|-----------|
| ADR-005 | Exactly-once dispatch wording (G4); heartbeat mutual-exclusivity rule (G2); `ContinueAsNew` section (G5); cancellation semantics; version-pinning replaces the "runs to completion on its version" hand-wave (G1) | P1–P4 |
| DATA_MODELS.md | Attempt-status enum; checkpoint unique key; lease on attempt; fencing invariant in transaction boundaries; `execution_seq`/`superseded_by`; signal dedupe key; payload caps | P1–P4 |
| ARCHITECTURE.md | State diagram gains `SUSPENDED`/`FAILED`; broker section rewritten for Postgres-native dispatch with JetStream as P7 option; app-composition sections move to their own docs | P2 |
| niyanta-v1.yaml | pause/resume, wait param, recovery endpoints, executions chain, call-log/mailbox reads, signal idempotency header | P5 |

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

---

**End of Implementation Plan**
