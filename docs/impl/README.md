# Implementation Guides

**Status**: Draft
**Last Updated**: 2026-05-01

This folder contains phase-wise implementation details for Niyanta: a self-orchestrated, multi-tenant, high-density, isolated ingestion platform built on a durable activity execution engine.

Read these in order. Each phase ends with a runnable system and builds directly on the previous one.

| Phase | Guide | Result |
|-------|-------|--------|
| 0 | [PHASE_0_SKELETON.md](PHASE_0_SKELETON.md) | Single-process, in-memory activity runner |
| 1 | [PHASE_1_PERSISTENCE_RETRIES.md](PHASE_1_PERSISTENCE_RETRIES.md) | Postgres persistence, retries, crash recovery |
| 2 | [PHASE_2_DISTRIBUTED_WORKERS.md](PHASE_2_DISTRIBUTED_WORKERS.md) | Coordinator/worker split, NATS, failure redistribution |
| 3 | [PHASE_3_CHILD_ACTIVITIES_REPLAY.md](PHASE_3_CHILD_ACTIVITIES_REPLAY.md) | `RunChild`, durable suspend, replay resume |
| 4 | [PHASE_4_TIMERS_SIGNALS.md](PHASE_4_TIMERS_SIGNALS.md) | Durable sleep, deterministic primitives, signals |
| 5 | [PHASE_5_INGESTION_PLATFORM.md](PHASE_5_INGESTION_PLATFORM.md) | Connector registry, ingestion planner, multi-tenant isolation |
| 6 | [PHASE_6_HA_SCALE_HARDENING.md](PHASE_6_HA_SCALE_HARDENING.md) | HA, JetStream signals, autoscaling, determinism lint |

## Target End State

By the end of Phase 5, the product flow is:

1. A customer creates a connector definition or uses a public one.
2. A customer creates a connector connection with tenant-scoped config, secret refs, destination refs, and isolation policy.
3. The ingestion planner continuously creates connector runs from schedules, source lag, checkpoint state, backfills, and health policy.
4. Workers execute connector runtime activities in dense shared pools while enforcing tenant isolation.
5. Records are read, parsed, transformed, deduped, delivered, and checkpointed only after delivery succeeds.
6. Health, lag, checkpoint age, delivery failures, rate-limit backoff, and parser drops are observable per tenant and connection.

Operational correctness rules are captured in [../OPERATIONS_AND_FAILURE_SEMANTICS.md](../OPERATIONS_AND_FAILURE_SEMANTICS.md). Implementation phases must preserve those semantics.

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
