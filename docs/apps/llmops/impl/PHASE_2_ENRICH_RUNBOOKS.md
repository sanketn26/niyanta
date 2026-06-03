# LLMOps Phase 2 — Enrichment, Runbooks, Cost Control

**Status**: Draft
**Last Updated**: 2026-06-03

**Goal**: Make diagnosis cheaper, smarter, and config-driven. Replace the single built-in prompt with operator-authored **runbooks**, add replay-safe **tool-lookup** children for targeted context, ground the model in **playbooks**, and enforce per-tenant **budgets**. Still suggest-only — no automated action.

**Depends on**: Phase 1.

## Scope

In:

- `tool_lookup` child activity: a single bounded, tenant-scoped read against Data Connector telemetry (e.g. last N failed runs, lag trend, active controls). Runs as `RunChild` so it is recorded in the call log and replay-safe; never returns raw payloads.
- **Runbooks** (`llmops_runbooks`): declarative YAML/JSON authored by operators. `sweep` evaluates `when` predicates to select candidates; `diagnose` runs the runbook's `context` lookups and `intent`. The activities become generic interpreters; behavior lives in runbooks. This is LLMOps' own config language — the engine does not parse it.
- **Playbooks** (`llmops_playbooks`): curated failure-mode knowledge used as retrieval context and to bound which actions a category may later propose. Public (per connector definition) or tenant-private.
- **Cost control**: `llmops_model_calls` ledger aggregated per tenant per day; `llmops_policies.budget_json` caps tokens/cost. On exhaustion the composition stops calling the model and falls back to suggest-only from deterministic signals, emitting an event.
- API: runbook CRUD, playbook list/create, policy get/put, `GET cost`.

Out (Phase 3):

- Applying controls, verification, rollback — still no `remediate`/`apply_control`.

## Key Deliverables

- Runbook parser/validator owned by this composition: validates `when` predicates reference real telemetry and `respond.action.control_type` is a real Data Connector control; stores parsed spec as JSONB. Editing a runbook changes behavior with no code or engine change.
- Runbook precedence: tenant-private overrides public for the same trigger.
- `diagnose` builds context from the runbook's declared `tool_lookup` children; each is replay-safe and audited, so a parent resume re-spends neither queries nor tokens.
- Prompt grounding: applicable playbooks injected as retrieval context; tenant-private playbooks never cross tenants.
- Budget enforcement verified: once a tenant's daily budget is hit, `model_call` is not dispatched; runs fall back to suggest-only and emit `budget_exhausted`.
- Metrics: `niyanta_llmops_model_cost_usd_total` per customer/model; runbook-match counters.

## Done When

- An operator adds a runbook for a new issue category (YAML), and the next sweep diagnoses matching connections using that runbook — with no deployment.
- A `diagnose` that makes three `tool_lookup` children and one `model_call`, when its worker is killed mid-flight, resumes and re-runs none of the completed children (verified by query/token counts).
- Exceeding a tenant's daily token budget flips that tenant to suggest-only and records `budget_exhausted`, while other tenants are unaffected.
