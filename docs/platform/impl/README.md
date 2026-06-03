# Platform Implementation Guides

**Status**: Draft
**Last Updated**: 2026-05-01

This folder contains phase-wise implementation details for the Niyanta **platform**: the durable distributed activity execution engine that the data-connector application is built on.

Read these in order. Each phase ends with a runnable system and builds directly on the previous one.

| Phase | Guide | Result |
|-------|-------|--------|
| 0 | [PHASE_0_SKELETON.md](PHASE_0_SKELETON.md) | Single-process, in-memory activity runner |
| 1 | [PHASE_1_PERSISTENCE_RETRIES.md](PHASE_1_PERSISTENCE_RETRIES.md) | Postgres persistence, retries, crash recovery |
| 2 | [PHASE_2_DISTRIBUTED_WORKERS.md](PHASE_2_DISTRIBUTED_WORKERS.md) | Coordinator/worker split, NATS, failure redistribution |
| 3 | [PHASE_3_CHILD_ACTIVITIES_REPLAY.md](PHASE_3_CHILD_ACTIVITIES_REPLAY.md) | `RunChild`, durable suspend, replay resume |
| 4 | [PHASE_4_TIMERS_SIGNALS.md](PHASE_4_TIMERS_SIGNALS.md) | Durable sleep, deterministic primitives, signals |

By the end of Phase 4 the platform is a general-purpose durable activity engine. The data-connector application is then built on top of it; see [../../apps/dataconnector/impl/README.md](../../apps/dataconnector/impl/README.md) for phases 5–6.

## Project Layout

The guides assume this Go module layout:

```text
cmd/
  niyanta/
  coordinator/
  worker/
internal/
  api/
  audit/
  broker/
  config/
  connector/
  executor/
  ingest/
  registry/
  scheduler/
  signal/
  storage/
pkg/
  activity/
  models/
  tenant/
```

Keep package names boring and stable. The product can be ambitious; the package graph should not be.
