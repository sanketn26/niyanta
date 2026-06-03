# LLMOps For Data Connectors — Activity Composition

**Version**: 0.2
**Last Updated**: 2026-06-03
**Status**: Draft

## What This Is

This document describes a **composition of activities** on Niyanta that uses LLMs to operate the data-connector fleet: detect issues, diagnose lag and failures, explain root cause, and operationalize connectors through suggested or automated remediation.

It is **not** a new service, engine, or platform layer. There is no "LLMOps runtime." It is a set of activity types wired together on Niyanta, leaning on the platform's execution guarantees, plus a small amount of its own state. Like the Data Connector composition, it may expose a higher-level interface over those activities (a playbook/policy language) so operators configure behavior without writing activities — but that interface, its parser, and its state are **this composition's own responsibility**, built on unchanged platform primitives. The platform does not change to support it. See [../../platform/ARCHITECTURE.md](../../platform/ARCHITECTURE.md) for the runner and the composition model.

It reads from the Data Connector composition's telemetry and acts only through its existing control API — it never builds a parallel control plane and never writes connector state directly.

**A note on what belongs to whom.** The runbook and policy languages defined here are **this composition's config, not Niyanta's.** Niyanta does not parse or store them; it only runs the activities that interpret them and guarantees those activities resume if they get stuck. To the engine, a runbook is an opaque payload inside an activity's state. This is why LLMOps can change its runbook language freely without any platform change.

## The Activities

The composition is these activity types. Each is a normal Niyanta activity; the parent/child relationships use `RunChild`; periodic work uses `ctx.Sleep`; event-driven work uses `ctx.AwaitSignal`.

| Activity | Role | Composition | Platform guarantees it leans on |
|----------|------|-------------|---------------------------------|
| `sweep` | Long-lived supervisor per tenant (or per shard). Wakes on a timer or on a significant-event signal, decides which connections deserve a look, and dispatches diagnoses. | Parent. `Sleep` between sweeps; `AwaitSignal` for significant events; `RunChild("diagnose", …)` per candidate connection. | Durable timers, signals, replay-based resume (a multi-hour sweep holds no worker slot while sleeping). |
| `diagnose` | Analyze one connection: gather context, call the model, produce a `Finding`. | Parent of tool lookups. `RunChild("tool_lookup", …)` for each targeted query; one `RunChild("model_call", …)` (or more) for the LLM. | Retries on model/provider failure, per-attempt audit, isolation (tenant-scoped). |
| `tool_lookup` | A single bounded, side-effecting read against connector telemetry (e.g. "last 5 failed runs", "24h lag trend"). | Leaf child of `diagnose`. | `RunChild` durability — recorded in the call log, so replay does not re-query or re-spend. |
| `model_call` | One LLM invocation with a bounded, redacted prompt; returns structured output. | Leaf child of `diagnose`. | Retry/backoff on 429/timeout; recorded in the call log so a parent resume does not re-spend tokens. |
| `remediate` | Turn a high-confidence finding into an action: suggest, or (if policy allows) apply a connector control via the Data Connector API, then schedule verification. | Parent. `RunChild("apply_control", …)` calling the existing control API; `Sleep` then `RunChild("diagnose", …)` to verify. | Idempotency bounded to the `apply_control` child; audit trail; durable scheduled verify. |
| `apply_control` | A single call to the Data Connector control API (`rate_limit_override`, `temporary_disable`, `pause`, `resume`, `quarantine`). | Leaf child of `remediate`. | At-most-the-child-attempt side effect; recorded in the call log. |

Nothing here is a new platform primitive. `sweep` is to LLMOps what `IngestionSupervisor` is to the Data Connector — a parent activity that sleeps, wakes, and dispatches children.

These activities are **generic interpreters of runbooks**. The behavior — what triggers a look, which context to gather, which control to propose, how to verify — is not hardcoded in the activity bodies; it comes from declarative **runbooks** (YAML/JSON) that operators author. `sweep` evaluates runbook `when` predicates; `diagnose` runs the runbook's context lookups and diagnosis intent; `remediate` applies the runbook's guarded responses. Adding a new behavior means writing a runbook, not an activity. This makes LLMOps a config-driven platform in its own right — the same way the Data Connector exposes a YAML connector language — and, like that, the runbook language is owned by this composition, not the engine. See the Runbooks section in [LLMOPS_SPEC.md](LLMOPS_SPEC.md).

## Composition Shape

```go
// One supervisor per tenant (or per shard of connections).
func Sweep(ctx activity.Context, scope LLMOpsScope) error {
    for {
        // Wake on a timer OR on a significant-event signal, whichever comes first.
        candidates := selectCandidates(ctx, scope) // deterministic from retrieved state
        for _, c := range candidates {
            finding := ctx.RunChild("diagnose", c.ConnectionID)
            if finding.AutoActionable {
                ctx.RunChild("remediate", finding.ID)
            }
        }
        ctx.Sleep(scope.SweepInterval) // or AwaitSignal — holds no worker slot
    }
}

func Diagnose(ctx activity.Context, connectionID string) Finding {
    facts := ctx.RunChild("tool_lookup", contextQueryFor(connectionID)) // recorded; replay-safe
    resp := ctx.RunChild("model_call", buildPrompt(facts))              // recorded; tokens not re-spent on resume
    return parseFinding(resp) // structured-output parse + validation
}

func Remediate(ctx activity.Context, findingID string) error {
    f := loadFinding(ctx, findingID)
    if !guardrailsAllow(ctx, f) {
        suggestToOperator(ctx, f) // default path: human-in-the-loop
        return nil
    }
    ctx.RunChild("apply_control", controlFor(f)) // calls Data Connector control API
    ctx.Sleep(f.VerifyAfter)
    ctx.RunChild("diagnose", f.ConnectionID)     // verification; rolls back / escalates if no improvement
    return nil
}
```

The actual implementation may optimize the physical shape, but the durable semantics — sleep/wake supervisor, replay-safe tool and model calls, action-then-verify — must hold.

## What It Reads (Inputs)

All inputs come from the Data Connector composition, tenant-scoped at the storage boundary. See [../dataconnector/DATA_CONNECTOR_SPEC.md](../dataconnector/DATA_CONNECTOR_SPEC.md) and [../dataconnector/INGESTION_ARCHITECTURE.md](../dataconnector/INGESTION_ARCHITECTURE.md).

- `ingestion_events` flagged `significant = TRUE` (the existing `ingestion_event_filters` decide significance).
- Connection health transitions (`degraded`, `lagging`, `failing`, `quarantined`) and current `connection_controls`.
- Metrics: `niyanta_ingestion_source_lag_seconds`, `niyanta_ingestion_checkpoint_age_seconds`, `niyanta_ingestion_delivery_failures_total`, `niyanta_ingestion_records_dropped_total`, `niyanta_ingestion_rate_limit_backoffs_total`.
- Recent runs, the customer ingestion view, the connector definition + version, and prior findings for the connection.

The `sweep` supervisor is woken either by a durable timer (periodic drift detection) or by a signal raised when a significant event is recorded — the same signal mechanism the platform already provides.

## What It Writes (Outputs)

- **Its own state** (new tables, namespaced to this composition): `Finding`, `RemediationAction`, `ModelCall` (cost ledger), and `Playbook` knowledge. Schemas in [LLMOPS_SPEC.md](LLMOPS_SPEC.md). The platform stores arbitrary activity state; these tables are this composition's domain state, not engine changes.
- **Connector controls**, only via the Data Connector control API — exactly the calls an operator would make. The "actuator" is `apply_control`, a leaf child activity. LLMOps adds no new mutation path to connector state.
- **Audit events**, through the existing audit trail, recording that the action was taken by the LLMOps composition under a named policy, with model id, prompt hash, confidence, and evidence.

## Guarantees It Relies On (and why this composition is correct)

- **Replay-based resume** makes tool and model calls replay-safe: a `diagnose` whose worker dies after the model call resumes from the call log without re-querying telemetry or re-spending tokens.
- **Framework retries** handle provider 429s/timeouts on `model_call` without the author writing a retry loop; an exhausted diagnosis is left `inconclusive`, never silently dropped.
- **Durable sleep + signals** let a per-tenant `sweep` run for days holding no resources between wakes, and wake immediately on a significant event.
- **Tenant isolation** ensures a diagnosis for tenant A cannot read tenant B's runs, events, or findings; context retrieval is tenant-scoped at the storage boundary.
- **Per-attempt audit** records every model call and applied action for unattended operation.

## Multi-Tenancy And Isolation

- Every `Finding`, `RemediationAction`, and `ModelCall` carries `customer_id` and `connection_id`.
- Prompts are built only from tenant-scoped, redacted data. Raw record payloads and secrets are never sent to a model; redaction happens before prompt assembly, not inside it.
- Playbooks are public (connector-definition level) or tenant-private; a tenant's private playbook is never used in another tenant's prompt.
- Per-tenant model budgets (tokens/cost/day) cap spend so one tenant's flapping fleet cannot exhaust shared model quota.

## Guardrails For Automated Action

Automated remediation is the riskiest part and is governed explicitly (full rules in [OPERATIONS_AND_FAILURE_SEMANTICS.md](OPERATIONS_AND_FAILURE_SEMANTICS.md)):

- **Off by default**, opt-in per tenant and per control type.
- **Confidence floor** — below it, suggest only.
- **Blast-radius caps** — no automated action across more than N connections in a window, or touching a shared parser/destination, without escalation.
- **Reversible-first** — prefer `rate_limit_override` / `temporary_disable` over `quarantine`; severe actions require approval.
- **Cooldown + verification** — every action schedules a verify diagnosis; rollback or escalate if it did not help.

## Fail-Safe Toward The Fleet

If this composition is down, the Data Connector composition's own deterministic health loop keeps running. LLMOps augments operations; it is never on the ingestion critical path.

## Observability

Reuses the platform observability contract, adding:

- `niyanta_llmops_diagnosis_runs_total` (customer, connector, trigger, result)
- `niyanta_llmops_findings_total` (issue_category, confidence_bucket)
- `niyanta_llmops_remediations_total` (control_type, mode, outcome)
- `niyanta_llmops_model_calls_total`, `niyanta_llmops_model_tokens_total`, `niyanta_llmops_model_cost_usd_total` (customer, model_id)
- `niyanta_llmops_diagnosis_latency_seconds` (histogram)
- `niyanta_llmops_auto_action_blocked_total` (guardrail reason)

Trace hierarchy mirrors the composition tree (`sweep → diagnose → tool_lookup / model_call → remediate → apply_control → verify`). Payloads and secrets are redacted before any log or span attribute, identical to the Data Connector rule.

## Documentation Alignment

- [../../platform/ARCHITECTURE.md](../../platform/ARCHITECTURE.md): the activity runner and the composition model this is built on.
- [../dataconnector/INGESTION_ARCHITECTURE.md](../dataconnector/INGESTION_ARCHITECTURE.md) and [../dataconnector/DATA_CONNECTOR_SPEC.md](../dataconnector/DATA_CONNECTOR_SPEC.md): the composition this one operates and reads from.
- [LLMOPS_SPEC.md](LLMOPS_SPEC.md): domain model, storage, and APIs.
- [OPERATIONS_AND_FAILURE_SEMANTICS.md](OPERATIONS_AND_FAILURE_SEMANTICS.md): action safety and correctness rules.
- [impl/README.md](impl/README.md): phased build plan.
