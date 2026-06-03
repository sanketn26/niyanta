# LLMOps — Operations And Failure Semantics

**Version**: 0.1
**Last Updated**: 2026-06-03
**Status**: Draft

## Overview

These are the correctness and safety rules for the LLMOps composition ([ARCHITECTURE.md](ARCHITECTURE.md)). They constrain a system that reasons with a non-deterministic model and can take action on a live fleet. They are the composition's own rules, enforced by its activities — Niyanta only guarantees those activities run and resume; it does not enforce these.

The guiding principle: **augment operations, never endanger the fleet.** If LLMOps is wrong, slow, or down, the Data Connector composition's deterministic health loop must remain correct on its own.

## Action Safety

Automated remediation is off by default and gated by three independent checks, all of which must pass:

1. **Runbook** says `mode: auto` for the matched response.
2. **Tenant policy** opts in to automation for that `control_type` and sets a confidence floor.
3. **Guardrails** (below) are satisfied at action time.

A permissive runbook can never override a conservative policy. This is defense in depth: the runbook expresses intent; the policy and guardrails express the outer safety bounds.

Action rules:

- Actions map 1:1 to existing Data Connector controls (`rate_limit_override`, `temporary_disable`, `pause`, `resume`, `quarantine`). LLMOps invents no new mutation.
- Every action is applied through the Data Connector control API as the `apply_control` child activity — the same path an operator uses.
- **Reversible-first.** Prefer `rate_limit_override` / `temporary_disable` over `pause`; prefer `pause` over `quarantine`. Severe or hard-to-reverse actions require human approval regardless of confidence.
- **Idempotency.** `apply_control` must be idempotent on (`connection_id`, `control_type`, `finding_id`) so a parent resume that re-dispatches it does not double-apply.

## Confidence And Inconclusive Outcomes

- Each `Finding` carries a model-estimated `confidence` in `[0,1]`, plus evidence references.
- Below the policy's confidence floor for a control type, the action is **suggest-only** — never applied automatically.
- A diagnosis that cannot produce a parseable, validated `Finding` ends `inconclusive`. Inconclusive is a first-class outcome: it is recorded, surfaced to operators, and never silently dropped or guessed past.
- The model does not self-certify safety. `auto_actionable` on a finding is computed by the guardrail evaluator, not taken from the model's output.

## Guardrails

Enforced at action time, from the tenant policy:

- **Rate cap**: at most `max_auto_actions_per_hour` automated actions per tenant.
- **Blast-radius cap**: no automated action affecting more than `max_connections_affected_per_window` connections; any action touching a **shared parser or destination** is escalated to a human, never auto-applied (a shared resource fault can affect many tenants).
- **Cooldown**: no repeated automated action on the same connection within `cooldown_seconds`.
- **Budget**: model spend is capped per tenant per day; on exhaustion the composition stops calling the model and falls back to suggest-only (using cached/deterministic signals), emitting an event. It never silently overspends.

Blocked actions increment `niyanta_llmops_auto_action_blocked_total` with the reason and are surfaced as suggestions.

## Verification And Rollback

Every applied action is provisional until verified.

```text
apply control (via Data Connector API)
sleep cooldown
run verification diagnosis
  improved   -> mark resolved
  no change  -> rollback (clear the control) OR escalate, per runbook on_no_improvement
  worse      -> rollback immediately and escalate
```

- Verification reuses `diagnose`; the action's effect is judged from the same telemetry, not from the model's prior belief.
- Rollback clears the control via the Data Connector control API (e.g. `clear-control`), restoring the connection to its prior effective state.
- An action that flaps (applied, rolled back, re-proposed) within a window is escalated to a human rather than retried indefinitely.

## Model-Call Failure Semantics

- Provider 429/5xx/timeout on `model_call` are retryable via the platform retry policy; the author writes no retry loop.
- Because `model_call` is a `RunChild` recorded in the call log, a parent (`diagnose`) that resumes after a worker crash does **not** re-issue a completed model call — tokens are not re-spent. This is a platform guarantee the composition leans on.
- A `model_call` that exhausts retries fails the diagnosis attempt; the diagnosis ends `inconclusive` rather than fabricating a finding.
- Structured-output parse/validation failure is treated as a failed attempt (retryable a bounded number of times), never as a low-quality finding.

## Tenant Isolation

- Every diagnosis reads only the tenant's own telemetry; context retrieval is tenant-scoped at the storage boundary, not filtered after a cross-tenant query.
- Prompts contain only redacted, tenant-scoped data. Raw record payloads and secrets never reach a model. Redaction is applied before prompt assembly and verified during onboarding. The enforceable rules, forbidden-field list, prompt-logging policy, model-provider failure matrix, and test fixtures are in [REDACTION_CONTRACT.md](REDACTION_CONTRACT.md).
- Tenant-private runbooks and playbooks are never used in another tenant's prompt or decision.
- Per-tenant budgets prevent one tenant's flapping fleet from consuming shared model quota.

## Fail-Safe Toward The Fleet

- LLMOps is never on the ingestion critical path. If its activities are paused, failing, or its budget is exhausted, ingestion and the Data Connector health loop continue unaffected.
- LLMOps has read access to telemetry and write access only through the Data Connector control API — it cannot corrupt connector state even when wrong.
- A bug in a runbook can, at worst, propose or (if all three gates pass) apply a reversible control on a bounded number of connections, which verification then rolls back. Blast-radius caps and reversible-first keep the worst case small and self-healing.

## Audit Requirements

Every applied or rolled-back action, and every policy/runbook change, writes an audit event through the existing audit trail:

- diagnosis triggered (with trigger source)
- finding produced (category, confidence)
- remediation suggested
- remediation applied (control_type, config, model_id, prompt_hash, confidence, evidence refs, policy/runbook id)
- remediation verified / rolled back / escalated
- runbook created/updated/disabled
- policy changed
- budget exhausted / automation auto-disabled

The audit answers who/what/when/why. For automated actions the "who" is the named policy + runbook, with the model id and prompt hash attached, so an unattended action is fully reconstructable.
