# Platform Implementation Guides

**Status**: Draft
**Last Updated**: 2026-07-06

This folder contains phase-wise implementation details for Niyanta: a durable distributed activity execution engine. The engine is use-case agnostic — these guides never assume anything about what an activity does.

Read these in order. Each phase ends with a runnable system and builds directly on the previous one. Phase numbering matches [../../IMPLEMENTATION_PLAN.md](../../IMPLEMENTATION_PLAN.md) v3.

| Phase | Guide | Result |
|-------|-------|--------|
| 0 | [PHASE_0_SKELETON.md](PHASE_0_SKELETON.md) | Single-process, in-memory activity runner |
| 1 | [PHASE_1_PERSISTENCE_RETRIES.md](PHASE_1_PERSISTENCE_RETRIES.md) | Postgres persistence, framework retries, crash recovery, **write-path fencing** |
| 2 | [PHASE_2_DISTRIBUTED_WORKERS.md](PHASE_2_DISTRIBUTED_WORKERS.md) | Coordinator/worker split, Postgres-native dispatch, failure redistribution |
| 3 | [PHASE_3_CHILD_ACTIVITIES_REPLAY.md](PHASE_3_CHILD_ACTIVITIES_REPLAY.md) | `RunChild`, durable suspend, replay resume, version pinning |
| 4 | [PHASE_4_TIMERS_SIGNALS.md](PHASE_4_TIMERS_SIGNALS.md) | Durable sleep, deterministic primitives, signals, `ContinueAsNew` |
| 5 | [PHASE_5_API_TENANCY.md](PHASE_5_API_TENANCY.md) | Complete REST API, auth, multi-tenancy, priority, version drain |
| 6 | [PHASE_6_CONSOLE_OBSERVABILITY.md](PHASE_6_CONSOLE_OBSERVABILITY.md) | Activity Manager console with integrated cluster + Prometheus observability |
| 7 | [PHASE_7_HA_SCALE_HARDENING.md](PHASE_7_HA_SCALE_HARDENING.md) | Coordinator HA, optional broker substrates, lint, chaos, scale |

By the end of Phase 4 the execution model is complete. Phase 5 makes the engine consumable, Phase 6 makes it operable, Phase 7 makes it hardened.

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
  console/            console backend: read API, metrics store, SSE (Phase 6)
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
