# Platform Implementation Guides

**Status**: Draft
**Last Updated**: 2026-07-10

This folder contains phase-wise implementation details for Niyanta: a durable distributed activity execution engine. The engine is use-case agnostic — these guides never assume anything about what an activity does.

Read these in order. Each phase ends with a runnable system and builds directly on the previous one. Phase numbering matches [../../IMPLEMENTATION_PLAN.md](../../IMPLEMENTATION_PLAN.md) v3.

| Phase | Guide | Result |
|-------|-------|--------|
| 0 | [PHASE_0_SKELETON.md](PHASE_0_SKELETON.md) | Single-process, in-memory activity runner |
| 1 | [PHASE_1_PERSISTENCE_RETRIES.md](PHASE_1_PERSISTENCE_RETRIES.md) | Postgres persistence, framework retries, crash recovery, **write-path fencing** |
| 2 | [PHASE_2_DISTRIBUTED_WORKERS.md](PHASE_2_DISTRIBUTED_WORKERS.md) | Coordinator/worker split, Postgres-native dispatch, failure redistribution |
| 3 | [PHASE_3_CHILD_ACTIVITIES_REPLAY.md](PHASE_3_CHILD_ACTIVITIES_REPLAY.md) | `RunChild`, durable suspend, replay resume, version pinning |
| 4 | [PHASE_4_TIMERS_SIGNALS.md](PHASE_4_TIMERS_SIGNALS.md) | Durable sleep, deterministic primitives, signals, `ContinueAsNew` |
| 5 | [PHASE_5_API_TENANCY.md](PHASE_5_API_TENANCY.md) | Operator/read API, auth, safe mutations, multi-tenancy, priority, version drain |
| 6 | [PHASE_6_CONSOLE_READ_ONLY.md](PHASE_6_CONSOLE_READ_ONLY.md) | Read-only Activity Manager for bounded, redacted state inspection |
| 7 | [PHASE_7_CONSOLE_OPERATIONS.md](PHASE_7_CONSOLE_OPERATIONS.md) | Browser identity, safe operator workflows, reconnectable SSE |
| 8 | [PHASE_8_OBSERVABILITY.md](PHASE_8_OBSERVABILITY.md) | Bounded Prometheus metrics, alerts, dashboards, tracing |
| 9 | [PHASE_9_PRODUCTION_HARDENING.md](PHASE_9_PRODUCTION_HARDENING.md) | HA, restore, security, upgrade, chaos, capacity gate |

By the end of Phase 4 the execution model is complete. Phase 5 makes the API consumable, Phase 6 makes state inspectable, Phase 7 makes browser operations safe, Phase 8 makes the system observable, and Phase 9 qualifies it for production.

## Project Layout

The guides assume this Go module layout:

```text
cmd/
  niyanta/            CLI (submit/get/run)
  coordinator/
  worker/
  console/            Activity Manager (Phase 6)
internal/
  api/
  audit/
  config/
  console/            browser adapter: sessions/read UI (P6), actions/SSE (P7), charts (P8)
  executor/
  metrics/            Prometheus registry + collectors
  registry/
  retry/
  scheduler/
  signal/
  storage/
  worker/
pkg/
  activity/
  models/
```

Keep package names boring and stable. The product can be ambitious; the package graph should not be.
