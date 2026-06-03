# ADR-005: Activity Execution Model

**Status**: Accepted
**Date**: 2026-04-28
**Supersedes**: Workload interface in [ARCHITECTURE.md](../ARCHITECTURE.md) §Component Architecture (Worker → Workload Plugin Interface)

## Context

Niyanta's original (v1) design defined a single `Workload` interface with `Init/Execute/Checkpoint/Close` methods. Workload code was responsible for its own checkpoint design, retry loops, and progress reporting. This ADR supersedes that interface; the current `Activity` interface is documented in [ARCHITECTURE.md](../ARCHITECTURE.md) §Worker.

The target use cases have since been clarified:

1. **Long-running ingestion pipelines** — extract/transform/load workflows running for hours, with natural sequential stages.
2. **Long-running monitors for agentic flows** — orchestrators that run for days to weeks, mostly idle, punctuated by LLM calls, tool invocations, and sub-agent runs. State accumulates across the run; external events (alerts, completions, human approvals) drive transitions.

Both share characteristics that the original `Workload` interface handles poorly:

- **Multi-day suspend/resume.** A monitor waiting 6 hours for a signal cannot hold a worker slot. Replaying its history on every resume is also too expensive once the call count grows large.
- **Sequential sub-activity composition.** A parent activity dispatches children (tool calls, sub-stages); children may run in the same process or a different one. The parent should not need to know.
- **External event-driven progression.** Monitors must wake on signals (alert fired, PR merged, human approved). Polling is unacceptable for multi-day waits.
- **Idempotency under retry without authoring effort.** Activity authors should declare retry policy; the framework should enforce it. Authors should not write retry loops.

Three execution models were considered.

## Decision

Adopt a **Temporal-inspired activity execution model**, scoped to what the use cases require:

1. **Sequential activities only.** No fan-out, no parallel sub-activities. A parent activity invokes children one at a time.
2. **Activity / orchestrator distinction at the call-site, not the type.** There is one `Activity` interface. Activities that call `ctx.RunChild` are *de facto* orchestrators; activities that don't are *de facto* leaves. The framework treats them uniformly.
3. **Sub-activity calls (`RunChild`) are durable suspend points.** When a parent calls `RunChild`, the framework records the call, frees the parent's worker slot, and re-dispatches the parent when the child completes. The child runs as a normal activity (in-process or remote — framework's choice).
4. **Replay-based resume across `RunChild` boundaries.** On parent resume, the activity body re-executes from the top. Each `RunChild` call checks the durable call log; calls with recorded results return immediately without re-dispatching. The parent fast-forwards to its previous suspension point.
5. **Determinism rule on parent code.** Parent activity code (anything outside `RunChild`-dispatched children) must be deterministic. Direct use of `time.Now()`, `rand`, I/O, and concurrent goroutines is forbidden in parent code. Equivalent primitives are exposed on `ActivityContext`: `ctx.Now()`, `ctx.NewID()`, `ctx.Sleep()`. All side-effecting work goes through `RunChild`.
6. **Signals as the external-event mechanism.** Signals are addressed to a running activity by ID and delivered to a durable per-activity mailbox. `ctx.AwaitSignal(name, timeout)` suspends the activity until a matching signal arrives or the timeout fires. Signals abstracted behind a `SignalBus` interface; backing impl is decoupled from the API (Phase 1 may use Postgres LISTEN/NOTIFY or a dedicated table; Phase 2 may swap to NATS JetStream).
7. **Heartbeat-based intra-activity checkpointing remains optional.** Activities with long in-process loops between `RunChild` calls may call `ctx.Heartbeat(state)` to persist fine-grained progress. On retry of the same attempt, `ctx.LastHeartbeat()` returns the last persisted state.
8. **Per-attempt records replace `retry_count` integer.** Each retry is a row in `activity_attempts`, capturing attempt number, start/end times, error, and worker. The activity itself (`activities` table) records the logical execution; attempts record the physical retries. Provides audit trail for autonomous systems where retries happen unattended.

### What is explicitly out of scope

- **Workflows as a separate type from activities.** No workflow engine, no workflow-specific scheduling.
- **Parallel sub-activities, fan-out/fan-in.** Sequential only.
- **Queries (synchronous reads of running activity state).** Signals only.
- **Cross-activity transactions, sagas, compensations.** Out of scope for v1.
- **Workflow versioning / hot code updates mid-run.** Activities run to completion on the version they started with.

## Rationale

### Why activities, not workflows-and-activities

Temporal separates *workflows* (deterministic orchestrators) from *activities* (side-effecting leaves) as distinct types with different runtimes. For our use cases this distinction adds complexity without value — every parent in our system is also a candidate leaf if invoked directly, and every leaf could in principle gain orchestration logic later. Collapsing to one `Activity` type with `RunChild` as the orchestration primitive keeps the model uniform. The determinism rule applies only to code that calls `RunChild` (or `Sleep`, `AwaitSignal`); pure leaves are unconstrained.

### Why replay-based resume, not full state serialization

Serializing a running goroutine's stack is not portable, language-runtime-coupled, and brittle across worker version changes. Replay reconstructs state by re-executing deterministic code against a recorded call log. The cost of replay is bounded by the call count, not the wall-clock duration — a 30-day monitor that made 50 `RunChild` calls replays 50 log lookups, not 30 days of state.

### Why the determinism rule is acceptable

The parent-as-orchestrator constraint (every side-effecting operation goes through `RunChild`) means parent code is naturally small. The determinism rule restricts a small surface, and the violations are mechanically detectable (we can lint for `time.Now()`, `rand.*`, direct package-level I/O in any package whose activities use `RunChild`). The cost is a learning curve for activity authors writing orchestrators; the benefit is multi-day suspend/resume without bespoke state-management code.

### Why signals are essential, not optional

The agentic monitor use case is unworkable without them. "Run when alert X fires" cannot be expressed with `RunChild` (the framework has no child to dispatch); polling at multi-day timescales is wasteful and high-latency. Signals must be in MVP, not deferred.

### Why abstract signals behind an interface

The signal delivery substrate is the most likely component to be swapped out as scale changes. Postgres LISTEN/NOTIFY is operationally simple but has known limits (single-database, payload size cap, no durable replay). NATS JetStream is more scalable but adds a dependency. The `SignalBus` interface lets the MVP use Postgres and migrate to JetStream without changing activity code.

## Consequences

### Positive

- **Multi-day suspend/resume is genuinely free for parents** — a monitor sleeping for 6 hours holds no worker slot, no memory, no resources beyond a row in storage.
- **Activity authors don't write retry loops or checkpoint serialization.** Framework enforces retry policy; replay handles cross-call resume.
- **Sub-activity calls are uniform whether local or remote** — caller code is identical.
- **Audit trail for unattended retries** — per-attempt rows capture every failure with worker, error, and timing.
- **Idempotency requirement is bounded.** Only `RunChild` dispatches must be idempotent (children may be invoked twice if a parent crashes between dispatch and result-record). Other side effects in children are bounded by the child's own attempt boundary.

### Negative

- **Determinism tax on parent code.** New rule for activity authors to learn; subtle bugs (e.g., iterating a map) won't surface until the second resume. Mitigation: lint, plus integration tests that resume every activity at least once.
- **Replay cost grows with call count, not wall time.** A parent that makes 10,000 `RunChild` calls in a single attempt will be slow to resume. Mitigation: practical activities make tens to hundreds of children, not thousands; if needed, intra-activity batching reduces call count.
- **Signal delivery must be durable.** A signal arriving while no worker holds the parent must be persisted until the parent resumes. Adds a `signals` table (per-activity mailbox).
- **Schema is larger than original `Workload` model.** Adds `activity_calls` (call log) and `signals` (mailbox); promotes `retry_count` to a separate `activity_attempts` table. These are the canonical schema in [DATA_MODELS.md](../DATA_MODELS.md) (v2.0).
- **Existing `Checkpoint()` method on the `Workload` interface goes away.** Replaced by `Heartbeat` (intra-attempt) + replay (cross-attempt). Migration is a doc-only rename until code exists.

### Neutral

- **Affinity, priority, capacity, lease/fence semantics from the existing design carry over unchanged.** They apply to the dispatch of each activity (parent and child alike), not to the resume mechanism.

## Implementation Phasing

This ADR describes the **engine capability sequence** — the order in which the activity-execution features land. These are *capability milestones*, not the system-wide delivery phases in [IMPLEMENTATION_PLAN.md](../../IMPLEMENTATION_PLAN.md) (which numbers 0–6 across the whole platform and apps). The cross-walk below is the single source of truth for which system phase delivers each capability; where this ADR and the plan ever disagree on numbering, the plan's phase numbers win and this column is the mapping.

| Capability milestone | Scope | Delivered in system phase |
|---|---|---|
| **M1 — Single-attempt activities** | `Activity` interface, `ctx.Heartbeat`, framework-enforced retry policy. No `RunChild`, no signals, no resume. Equivalent to the old `Workload` capability but with framework-managed retries. | Plan Phase 1 (persistence + retries) |
| **M2 — Composition + replay** | `RunChild` with sequential dispatch. Replay-based resume across `RunChild` boundaries. `activity_calls` table. | Plan Phase 3 (child activities + replay) |
| **M3 — Deterministic primitives + signals** | `ctx.Sleep`, `ctx.Now`, `ctx.NewID`. `SignalBus` interface + Postgres-backed impl. `ctx.AwaitSignal`. `signals` table. Signal-to-activity routing. | Plan Phase 4 (timers + signals) |
| **M4 — Hardening** | Determinism linter for parent code. NATS JetStream-backed `SignalBus` alternative. Per-call-site replay-key tuning and replay performance for large call counts. | Plan Phase 6 (HA, scale, hardening) |

**Signals must ship before any self-orchestrated ingestion (Plan Phase 5).** They land in Plan Phase 4 (capability M3), ahead of ingestion in Phase 5 — the agentic-monitor and ingestion-supervisor use cases are blocked without them. (This corrected an earlier draft that deferred signals to Phase 3 of an older, since-superseded plan; the duplicate "Phase 2" rows in that draft conflated M2 and M3 and have been split.)

## Alternatives Considered

### A. Keep the original `Workload` interface; add framework-managed retries only

Cheapest delta. But leaves the suspend/resume problem unsolved — long monitors hold worker slots or require activity authors to design their own checkpoint-and-restart logic. Rejected because the agentic-monitor use case is unworkable.

### B. Self-checkpointing only (no replay, no `RunChild` durability)

Activity calls `ctx.Heartbeat(state)` periodically; on resume, framework provides last heartbeat; activity is responsible for skip-already-done logic. Simpler framework. Rejected because every author writes the same skip-children boilerplate, and sub-activity composition is awkward (parent must manage child IDs, poll for results, handle child failure modes).

### C. Full Temporal model — workflows distinct from activities, full event-history replay, signals + queries + timers + child workflows

Most powerful. Rejected on scope: queries are not needed; workflow/activity type distinction adds conceptual overhead; full event history (every operation, not just `RunChild`) is overkill for sequential code.

### D. Replay with explicit `Suspend` points, not `RunChild` as suspend points

Activity calls `ctx.Suspend()` at known points; framework only frees the slot at `Suspend`. Rejected because agentic flows have no natural suspend points — every tool call could be one. Would converge on calling `Suspend` after every `RunChild`, at which point the design is equivalent but with extra ceremony.

## References

- [ARCHITECTURE.md](../ARCHITECTURE.md) — system context this ADR modifies
- [DATA_MODELS.md](../DATA_MODELS.md) — schema to be updated for `activity_calls`, `activity_attempts`, `signals`
- [IMPLEMENTATION_PLAN.md](../../IMPLEMENTATION_PLAN.md) — system-wide delivery phases (0–6); see the capability cross-walk above for how engine milestones map onto them
- [impl/PHASE_3_CHILD_ACTIVITIES_REPLAY.md](../impl/PHASE_3_CHILD_ACTIVITIES_REPLAY.md) — child activity and replay interface details
- [impl/PHASE_4_TIMERS_SIGNALS.md](../impl/PHASE_4_TIMERS_SIGNALS.md) — timer and signal delivery design
- Temporal documentation, *Workflows* and *Activities* sections — primary inspiration

## Open Questions

1. **Replay ordering key.** Children are matched to call-log entries by call-index (sequential position). Robust as long as parent code is deterministic and sequential. If we ever relax to parallel children, this becomes call-site-keyed (file:line) — flagged for future ADR if needed.
2. **Signal delivery semantics.** At-least-once vs. exactly-once. Leaning at-least-once with idempotent signal handlers; see [PHASE_4_TIMERS_SIGNALS.md](../impl/PHASE_4_TIMERS_SIGNALS.md).
3. **Maximum activity duration.** Multi-day is supported; multi-month is not specifically tested. Practical limit likely set by storage retention policy on `activity_calls`. To be load-tested in Phase 4.
