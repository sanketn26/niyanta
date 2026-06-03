# Data Connector Implementation Guides

**Status**: Draft
**Last Updated**: 2026-05-01

This folder contains phase-wise implementation details for the Niyanta **data-connector application**: the self-orchestrated, multi-tenant, high-density ingestion platform built on the durable activity execution engine.

These phases build on the platform engine. Read the [platform implementation guides](../../../platform/impl/README.md) (phases 0–4) first.

| Phase | Guide | Result |
|-------|-------|--------|
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
