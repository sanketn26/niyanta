# LLMOps Phase 3 — Guarded Remediation

**Status**: Draft
**Last Updated**: 2026-06-03

**Goal**: Let high-confidence findings drive action — safely. Add `remediate` and `apply_control`, the three-gate authorization (runbook + policy + guardrails), verification, rollback, and full audit. Automation is off by default and opt-in per tenant per control type.

**Depends on**: Phases 1–2, and the Data Connector control API (`rate_limit_override`, `temporary_disable`, `pause`, `resume`, `quarantine`, `clear-control`).

## Scope

In:

- `remediate` activity (child of `sweep`/dispatched from a finding): evaluate the three gates; if all pass, `RunChild("apply_control", …)`, then `Sleep(verify_after)`, then `RunChild("diagnose", …)` to verify; else emit a suggestion.
- `apply_control` child: a single idempotent call to the Data Connector control API — the same path an operator uses. Idempotent on (`connection_id`, `control_type`, `finding_id`).
- **Three-gate authorization**: runbook `mode: auto` AND tenant policy opt-in + confidence floor AND guardrails (rate cap, blast-radius cap, cooldown, budget). All must pass; a permissive runbook cannot override a conservative policy.
- **Verification + rollback**: judge effect from telemetry via `diagnose`; `improved → resolved`, `no_change → rollback|escalate` per runbook `on_no_improvement`, `worse → rollback + escalate`. Flapping actions escalate to a human.
- Storage: `llmops_remediation_actions` lifecycle (`suggested → applied → verifying → resolved|rolled_back|escalated`).
- API: `GET remediations`, `POST remediations/{id}:apply` (apply a suggestion), `POST remediations/{id}:rollback`.
- Full audit: every applied/rolled-back action and policy/runbook change writes an audit event with model id, prompt hash, confidence, evidence, policy + runbook id.

Out:

- Cross-tenant or shared-resource auto-actions remain escalate-only (never auto), by design.

## Key Deliverables

- Guardrail evaluator computes `auto_actionable` independently of the model's output; the model never self-certifies safety.
- Reversible-first ordering enforced: severe/irreversible controls (`quarantine`) require human approval regardless of confidence.
- `apply_control` idempotency verified: a parent resume that re-dispatches it does not double-apply (relies on the platform's at-least-once child dispatch + idempotency key).
- Verification scheduled durably via `Sleep` + `RunChild("diagnose")`; holds no worker slot during the cooldown.
- Blast-radius cap: an action touching a shared parser/destination is escalated, never auto-applied; `niyanta_llmops_auto_action_blocked_total` records blocked attempts with reason.
- Fail-safe verified: with LLMOps paused or budget-exhausted, ingestion and the Data Connector health loop continue unaffected.
- Metrics: `niyanta_llmops_remediations_total` (control_type, mode, outcome), `niyanta_llmops_auto_action_blocked_total`.

## Done When

- A tenant opts in to auto `rate_limit_override`; a high-confidence `source_rate_limited` finding applies the override via the Data Connector API, a verification diagnosis after the cooldown sees lag recover, and the action is marked `resolved` — fully audited with model id and prompt hash.
- The same scenario with `no_change` at verification rolls the control back automatically (or escalates per the runbook).
- A finding whose action would touch a shared destination is blocked from auto-apply and surfaced as a suggestion, with a blocked-action metric and reason.
- Killing the `remediate` worker between `apply_control` and verify resumes without re-applying the control (idempotency holds).
