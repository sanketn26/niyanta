# LLMOps Implementation Guides

**Status**: Draft
**Last Updated**: 2026-06-03

This folder contains phase-wise implementation details for the **LLMOps composition**: a set of activities on Niyanta that diagnose and operationalize the data-connector fleet. See [../ARCHITECTURE.md](../ARCHITECTURE.md) for the composition and [../OPERATIONS_AND_FAILURE_SEMANTICS.md](../OPERATIONS_AND_FAILURE_SEMANTICS.md) for the safety rules each phase must preserve.

This composition builds on:

- the [platform](../../../platform/impl/README.md) (the activity runner — phases 0–4), and
- the [data connector](../../dataconnector/impl/README.md) composition (the system it operates — phases 5–6), whose telemetry and control API are LLMOps' inputs and outputs.

Read these in order. The phases follow the risk progression **observe → enrich → act**.

| Phase | Guide | Result |
|-------|-------|--------|
| 1 | [PHASE_1_DIAGNOSE.md](PHASE_1_DIAGNOSE.md) | Read-only diagnosis: `sweep` + `diagnose` + `model_call`, findings, suggestions only |
| 2 | [PHASE_2_ENRICH_RUNBOOKS.md](PHASE_2_ENRICH_RUNBOOKS.md) | Tool-lookup children, runbooks + playbooks, cost ledger and budgets |
| 3 | [PHASE_3_REMEDIATE.md](PHASE_3_REMEDIATE.md) | Guarded automated remediation, verification, rollback, full audit |

## Target End State

By the end of Phase 3:

1. A per-tenant `sweep` activity wakes on a timer or significant-event signal and dispatches diagnoses for at-risk connections, holding no worker slot while idle.
2. `diagnose` gathers tenant-scoped, redacted context via replay-safe `tool_lookup` children and a `model_call` child, and produces a structured, validated `Finding`.
3. Operators author **runbooks** (YAML/JSON) that drive behavior with no code change; a tenant **policy** sets the outer automation/budget/blast-radius bounds.
4. `remediate` applies controls only through the Data Connector control API, only when runbook + policy + guardrails all allow, then verifies and rolls back if it did not help.
5. Every model call and action is costed, audited, and tenant-isolated; if LLMOps is down, ingestion is unaffected.

Nothing in these phases changes the Niyanta engine. They register activity types and add LLMOps' own state (findings, remediations, model-call ledger, runbooks, playbooks, policy).
