# LLMOps For Data Connectors — Specification

**Version**: 0.1
**Last Updated**: 2026-06-03
**Status**: Draft

## Overview

This document defines the domain model, storage, APIs, and activity contract for the LLMOps composition described in [ARCHITECTURE.md](ARCHITECTURE.md). It is the composition's own state and surface, built on unchanged Niyanta primitives. The platform stores arbitrary activity state; the tables below are this composition's domain state, not engine changes.

All entities are tenant-scoped. The composition reads Data Connector telemetry (see [../dataconnector/DATA_CONNECTOR_SPEC.md](../dataconnector/DATA_CONNECTOR_SPEC.md)) and acts only through the Data Connector control API.

## Domain Model

### DiagnosisRun

A triggered analysis of one connection. Backed by a `diagnose` activity attempt — the platform owns its execution; this record is the composition's view of it.

```yaml
id: diag_123
customer_id: cust_abc
connection_id: conn_123
activity_id: act_789          # the diagnose activity attempt
trigger: significant_event    # significant_event | health_transition | metric_threshold | periodic_sweep | verification
trigger_ref: evt_456          # the ingestion_event / metric / prior action that woke it
status: completed             # pending | running | completed | inconclusive | failed
finding_id: find_321
created_at: 2026-06-03T10:00:00Z
completed_at: 2026-06-03T10:00:12Z
```

Rules:

- Deduplicated per connection within a cooldown window so a flapping connection does not trigger a model call per event.
- A run that cannot reach a conclusion ends `inconclusive`, never silently dropped.

### Finding

Structured output of a diagnosis.

```yaml
id: find_321
customer_id: cust_abc
connection_id: conn_123
diagnosis_run_id: diag_123
issue_category: source_rate_limited   # auth_failure | source_rate_limited | source_outage |
                                      # parser_drop_storm | destination_failure | lag_growth |
                                      # checkpoint_stall | config_drift | unknown
root_cause: "Source returned sustained 429s; current QPS exceeds the vendor's documented limit."
confidence: 0.82                      # 0..1
evidence:                             # references only — never raw payloads
  - type: ingestion_event
    ref: evt_456
  - type: metric
    ref: niyanta_ingestion_rate_limit_backoffs_total
    window: 1h
suggested_action:
  control_type: rate_limit_override
  config:
    qps: 0.25
    max_parallel_partitions: 1
auto_actionable: true                 # passed guardrails for automated action
created_at: 2026-06-03T10:00:12Z
```

Rules:

- `evidence` carries references to tenant-scoped telemetry, not copied payloads.
- `confidence` and `auto_actionable` are inputs to the remediation guardrails; the model does not get to declare an action "safe" on its own — guardrails are evaluated separately (see [OPERATIONS_AND_FAILURE_SEMANTICS.md](OPERATIONS_AND_FAILURE_SEMANTICS.md)).

### RemediationAction

A proposed or applied connector control, linked to a finding. Lifecycle: `suggested → applied → verifying → (resolved | rolled_back | escalated)`.

```yaml
id: rem_654
customer_id: cust_abc
connection_id: conn_123
finding_id: find_321
control_type: rate_limit_override     # maps 1:1 to a Data Connector connection_controls type
config_json: { qps: 0.25, max_parallel_partitions: 1 }
mode: applied                         # suggested | applied
status: verifying                     # suggested | applied | verifying | resolved | rolled_back | escalated
applied_via_control_id: ctrl_999      # the dataconnector connection_controls row created
verify_diagnosis_run_id: diag_124
audit_event_ref: audit_111
created_by: llmops_policy:default     # the named policy, not a human
created_at: 2026-06-03T10:00:13Z
```

Rules:

- An action maps to exactly one Data Connector control type — no novel mutations.
- `applied` actions always schedule a verification diagnosis; if it does not improve, the action is `rolled_back` or `escalated` per policy.

### ModelCall (cost ledger)

One LLM invocation. Exists for observability, cost control, and replay accounting.

```yaml
id: mc_777
customer_id: cust_abc
connection_id: conn_123
diagnosis_run_id: diag_123
model_id: claude-opus-4-8
prompt_hash: sha256:...               # for dedup/audit; prompt itself is redacted, not stored raw
input_tokens: 4200
output_tokens: 600
cost_usd: 0.0123
latency_ms: 1850
redaction_applied: true
result: ok                            # ok | parse_error | provider_error | timeout
created_at: 2026-06-03T10:00:11Z
```

### Playbook (knowledge)

Curated connector failure modes and known-good remediations. Used as retrieval context and as a guardrail on which actions are allowed for a category.

```yaml
id: pb_429_default
scope: connector_definition           # public (connector_definition) | tenant_private
connector_definition_id: github_audit # null for generic
issue_category: source_rate_limited
description: "GitHub audit API enforces secondary rate limits; sustained 429 needs QPS reduction."
allowed_actions:
  - control_type: rate_limit_override
recommended_config:
  qps: 0.25
min_confidence_for_auto: 0.8
created_at: 2026-06-03T00:00:00Z
```

Rules:

- A tenant-private playbook is never used in another tenant's prompt.
- An action category absent from any applicable playbook is suggest-only by default.

## Storage Extensions

These tables are namespaced to the LLMOps composition. They reference Data Connector tables read-only (by id) and never modify them.

```sql
CREATE TABLE llmops_diagnosis_runs (
    id                  VARCHAR(64) PRIMARY KEY,
    customer_id         VARCHAR(64) NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    connection_id       VARCHAR(64) NOT NULL,           -- references connector_connections(id), read-only
    activity_id         VARCHAR(64) REFERENCES activities(id) ON DELETE SET NULL,
    trigger             VARCHAR(32) NOT NULL,
    trigger_ref         VARCHAR(64),
    status              VARCHAR(32) NOT NULL,
    finding_id          VARCHAR(64),
    created_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    completed_at        TIMESTAMP WITH TIME ZONE
);

CREATE INDEX idx_llmops_diag_customer_time
    ON llmops_diagnosis_runs(customer_id, created_at DESC);
CREATE INDEX idx_llmops_diag_connection
    ON llmops_diagnosis_runs(customer_id, connection_id, created_at DESC);

CREATE TABLE llmops_findings (
    id                  VARCHAR(64) PRIMARY KEY,
    customer_id         VARCHAR(64) NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    connection_id       VARCHAR(64) NOT NULL,
    diagnosis_run_id    VARCHAR(64) NOT NULL REFERENCES llmops_diagnosis_runs(id) ON DELETE CASCADE,
    issue_category      VARCHAR(64) NOT NULL,
    root_cause          TEXT NOT NULL,
    confidence          REAL NOT NULL,
    evidence_json       JSONB NOT NULL DEFAULT '[]',
    suggested_action_json JSONB,
    auto_actionable     BOOLEAN NOT NULL DEFAULT FALSE,
    created_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_llmops_findings_customer_cat
    ON llmops_findings(customer_id, issue_category, created_at DESC);

CREATE TABLE llmops_remediation_actions (
    id                      VARCHAR(64) PRIMARY KEY,
    customer_id             VARCHAR(64) NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    connection_id           VARCHAR(64) NOT NULL,
    finding_id              VARCHAR(64) NOT NULL REFERENCES llmops_findings(id) ON DELETE CASCADE,
    control_type            VARCHAR(64) NOT NULL,
    config_json             JSONB NOT NULL DEFAULT '{}',
    mode                    VARCHAR(16) NOT NULL,         -- suggested | applied
    status                  VARCHAR(16) NOT NULL,
    applied_via_control_id  VARCHAR(64),                  -- connection_controls(id) created via the DC API
    verify_diagnosis_run_id VARCHAR(64),
    audit_event_ref         VARCHAR(64),
    created_by              VARCHAR(128) NOT NULL,        -- named policy, e.g. llmops_policy:default
    created_at              TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_llmops_rem_active
    ON llmops_remediation_actions(customer_id, connection_id, status);

CREATE TABLE llmops_model_calls (
    id                  VARCHAR(64) PRIMARY KEY,
    customer_id         VARCHAR(64) NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    connection_id       VARCHAR(64),
    diagnosis_run_id    VARCHAR(64) REFERENCES llmops_diagnosis_runs(id) ON DELETE SET NULL,
    model_id            VARCHAR(128) NOT NULL,
    prompt_hash         VARCHAR(80) NOT NULL,
    input_tokens        INTEGER NOT NULL DEFAULT 0,
    output_tokens       INTEGER NOT NULL DEFAULT 0,
    cost_usd            NUMERIC(12,6) NOT NULL DEFAULT 0,
    latency_ms          INTEGER,
    redaction_applied   BOOLEAN NOT NULL DEFAULT TRUE,
    result              VARCHAR(32) NOT NULL,
    created_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_llmops_model_calls_cost
    ON llmops_model_calls(customer_id, created_at DESC);

CREATE TABLE llmops_playbooks (
    id                  VARCHAR(64) PRIMARY KEY,
    scope               VARCHAR(32) NOT NULL,             -- connector_definition | tenant_private
    customer_id         VARCHAR(64) REFERENCES customers(id) ON DELETE CASCADE,  -- null when public
    connector_definition_id VARCHAR(128),                -- null when generic
    issue_category      VARCHAR(64) NOT NULL,
    description         TEXT NOT NULL,
    allowed_actions_json JSONB NOT NULL DEFAULT '[]',
    recommended_config_json JSONB NOT NULL DEFAULT '{}',
    min_confidence_for_auto REAL NOT NULL DEFAULT 0.9,
    created_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE TABLE llmops_policies (
    id                  VARCHAR(64) PRIMARY KEY,
    customer_id         VARCHAR(64) NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    automation_json     JSONB NOT NULL DEFAULT '{}',      -- per control_type: suggest | auto, confidence floor
    budget_json         JSONB NOT NULL DEFAULT '{}',      -- tokens/cost per day
    enabled             BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);
```

## Runbooks (Config-Driven Behavior)

LLMOps is itself a **config-driven platform**: operators author **runbooks in YAML/JSON** that declare what to look for and how to respond, and the composition interprets them. A runbook is to LLMOps what a connector definition is to Data Connector — a declarative language owned entirely by this composition and built on unchanged Niyanta primitives. No one writes an activity to add a new diagnosis-or-remediation behavior; they write or edit a runbook.

A runbook binds a trigger → a diagnosis intent → guarded responses → verification. The `sweep`, `diagnose`, and `remediate` activities are generic interpreters of runbooks; the runbook supplies the behavior.

```yaml
id: rb_source_429
scope: connector_definition            # public (connector_definition) | tenant_private
connector_definition_id: github_audit  # null = applies to any connector
enabled: true
when:                                   # trigger predicate over Data Connector telemetry
  any_of:
    - significant_event: source_rate_limited
    - metric: niyanta_ingestion_rate_limit_backoffs_total
      over: 1h
      above: 5
diagnose:
  intent: "Determine whether sustained source 429s require a QPS reduction."
  context:                              # which tool_lookups to run (replay-safe, audited)
    - last_failed_runs: 5
    - lag_trend: 24h
    - active_controls: true
  min_confidence: 0.8
respond:
  - if: { issue_category: source_rate_limited, confidence_gte: 0.8 }
    action:
      control_type: rate_limit_override
      config: { qps: 0.25, max_parallel_partitions: 1 }
    mode: auto                          # auto | suggest  (still gated by tenant policy + guardrails)
    verify_after_seconds: 1800
    on_no_improvement: rollback         # rollback | escalate | retry_with: <runbook_id>
  - else:
    mode: suggest
```

Rules:

- A runbook is declarative and versioned; editing one changes behavior with no code change and no engine change.
- Runbook `when` predicates read only tenant-scoped Data Connector telemetry; `respond.action.control_type` must be a real Data Connector control.
- A runbook's `mode: auto` is necessary but not sufficient — the tenant **policy** and **guardrails** below still gate every automated action (defense in depth: a permissive runbook cannot override a conservative policy).
- Runbooks may be public (per connector definition) or tenant-private; precedence is tenant-private over public for the same trigger.
- Stored as YAML/JSON in a `llmops_runbooks` table (parsed to JSONB internally), exactly as connector definitions are stored.

```sql
CREATE TABLE llmops_runbooks (
    id                  VARCHAR(64) PRIMARY KEY,
    scope               VARCHAR(32) NOT NULL,             -- connector_definition | tenant_private
    customer_id         VARCHAR(64) REFERENCES customers(id) ON DELETE CASCADE,  -- null when public
    connector_definition_id VARCHAR(128),                -- null = any connector
    enabled             BOOLEAN NOT NULL DEFAULT TRUE,
    spec_json           JSONB NOT NULL,                   -- parsed runbook (when/diagnose/respond)
    version             INTEGER NOT NULL DEFAULT 1,
    created_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_llmops_runbooks_match
    ON llmops_runbooks(connector_definition_id, enabled);
```

## Policy Language

A separate, coarser tenant-level control from runbooks. **Runbooks decide behavior; the policy sets the outer bounds** every runbook must obey (automation opt-in, confidence floors, budgets, blast-radius). It is the higher-level safety interface this composition exposes, owned by LLMOps, not the platform. A tenant configures it without authoring activities.

```yaml
customer_id: cust_abc
automation:
  rate_limit_override: { mode: auto, min_confidence: 0.8 }
  temporary_disable:   { mode: auto, min_confidence: 0.85 }
  pause:               { mode: suggest }
  quarantine:          { mode: suggest }          # severe → human approval
guardrails:
  max_auto_actions_per_hour: 10
  max_connections_affected_per_window: 5
  block_shared_parser_or_destination: true
  cooldown_seconds: 1800
budget:
  model_tokens_per_day: 2000000
  model_cost_usd_per_day: 50
sweep:
  interval_seconds: 300
  also_wake_on_significant_event: true
```

## API Extensions

The LLMOps app's own API, layered on the engine activity API ([../../platform/API_SPEC.md](../../platform/API_SPEC.md)). It is not an engine concern.

Conventions inherited from the platform API:

- **Auth**: `Authorization: Bearer <api_key>`; `customer_id` derived from the key and enforced at the storage boundary. A caller may also hold an internal support permission for the `cid`. Cross-tenant access fails closed (`404`).
- **Errors**: shape and codes per platform §Error Handling.
- **Pagination**: cursor-based (`limit` + `cursor`), per platform §Pagination.
- **Idempotency**: mutating creates and `:apply` accept an `Idempotency-Key` header. `:apply` is additionally idempotent on (`connection_id`, `control_type`, `finding_id`) so a retry never double-applies a control.
- **Base URL**: `https://api.niyanta.example.com/v1`.

### Endpoint index

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/customers/{cid}/llmops/findings` | List findings (filter by `issue_category`, `connection_id`, `confidence_gte`) |
| GET | `/customers/{cid}/llmops/findings/{id}` | Get a finding |
| GET | `/customers/{cid}/llmops/connections/{connection_id}/findings` | Findings for one connection |
| POST | `/customers/{cid}/llmops/connections/{connection_id}:diagnose` | Manually trigger a diagnosis |
| GET | `/customers/{cid}/llmops/remediations` | List remediation actions |
| POST | `/customers/{cid}/llmops/remediations/{id}:apply` | Apply a suggested action |
| POST | `/customers/{cid}/llmops/remediations/{id}:rollback` | Roll back an applied action |
| GET/POST | `/customers/{cid}/llmops/runbooks` | List / create runbooks |
| GET/PUT | `/customers/{cid}/llmops/runbooks/{id}` | Get / update a runbook |
| GET/PUT | `/customers/{cid}/llmops/policy` | Get / set the tenant policy |
| GET/POST | `/customers/{cid}/llmops/playbooks` | List / create playbooks |
| GET | `/customers/{cid}/llmops/cost` | Model spend over a window |

### Trigger a diagnosis

`POST /customers/{cid}/llmops/connections/{connection_id}:diagnose`
```json
{ "reason": "operator investigating lag" }
```
`202 Accepted` → `{ "diagnosis_run_id": "diag_123", "status": "pending", "activity_id": "act_789" }`. The diagnosis runs as a `diagnose` activity on the engine; poll the finding or the engine activity for completion.

### Get a finding

`GET /customers/{cid}/llmops/findings/{id}` → `200 OK`
```json
{
  "id": "find_321", "connection_id": "conn_123", "diagnosis_run_id": "diag_123",
  "issue_category": "source_rate_limited", "confidence": 0.82,
  "root_cause": "Source returned sustained 429s; current QPS exceeds the documented limit.",
  "evidence": [ { "type": "ingestion_event", "ref": "evt_456" }, { "type": "metric", "ref": "niyanta_ingestion_rate_limit_backoffs_total", "window": "1h" } ],
  "suggested_action": { "control_type": "rate_limit_override", "config": { "qps": 0.25, "max_parallel_partitions": 1 } },
  "auto_actionable": true, "created_at": "2026-06-03T10:00:12Z"
}
```
`evidence` carries references only — never raw payloads.

### List findings

`GET /customers/{cid}/llmops/findings?issue_category=source_rate_limited&confidence_gte=0.8&limit=50&cursor=...`
```json
{ "findings": [ { "id": "find_321", "issue_category": "source_rate_limited", "confidence": 0.82, "...": "..." } ], "next_cursor": null }
```

### Apply / roll back a remediation

`POST /customers/{cid}/llmops/remediations/{id}:apply` (header `Idempotency-Key: <uuid>`)
```json
{ "approved_by": "user_789", "override_confidence_floor": false }
```
`200 OK` → the updated `RemediationAction` with `mode: applied`, `applied_via_control_id`, `verify_diagnosis_run_id`, `audit_event_ref`.

- Applying calls the **Data Connector control API** underneath (`POST /connector-connections/{id}:rate-limit`, etc. — see [../dataconnector/API_SPEC.md](../dataconnector/API_SPEC.md)). It never writes `connector_connections` directly.
- A manual `:apply` still respects guardrails unless an explicit operator override is supplied and permitted; `quarantine` always requires approval.
- `409 INVALID_STATE` if the action is not in `suggested`/`applied`; `403 FORBIDDEN` if automation/override is not permitted for that control type.

`POST /customers/{cid}/llmops/remediations/{id}:rollback`
```json
{ "reason": "no improvement at verification" }
```
Clears the control via the Data Connector `:clear-control` endpoint and sets status `rolled_back`.

### Runbooks / policy / playbooks

CRUD over the YAML/JSON artifacts in §Runbooks, §Policy Language, and §Playbook. `POST`/`PUT` validate the document (parse, predicate references, control-type validity) and return `400 INVALID_INPUT` with the offending path on failure. Bodies are the YAML/JSON shapes shown in those sections.

### Cost

`GET /customers/{cid}/llmops/cost?window=24h`
```json
{ "window": "24h", "model_tokens": 1_420_000, "model_cost_usd": 31.40, "budget_tokens_per_day": 2_000_000, "budget_cost_usd_per_day": 50, "throttled": false, "by_model": [ { "model_id": "claude-opus-4-8", "tokens": 1_420_000, "cost_usd": 31.40 } ] }
```

### Audit

Every `:apply`, `:rollback`, runbook/policy change, and budget auto-disable writes an audit event through the platform audit trail (see [OPERATIONS_AND_FAILURE_SEMANTICS.md](OPERATIONS_AND_FAILURE_SEMANTICS.md) §Audit Requirements). Audit events for a connection are retrievable via the Data Connector ingestion-events surface and the engine activity audit endpoint. Example applied-action audit payload:
```json
{
  "event_type": "llmops.remediation_applied",
  "details_json": {
    "remediation_id": "rem_654", "finding_id": "find_321", "connection_id": "conn_123",
    "control_type": "rate_limit_override", "confidence": 0.82,
    "created_by": "llmops_policy:default", "runbook_id": "rb_source_429",
    "model_id": "claude-opus-4-8", "prompt_hash": "sha256:...", "evidence": ["evt_456"]
  }
}
```

OpenAPI for these endpoints: [`api/openapi/llmops-v1.yaml`](../../../api/openapi/llmops-v1.yaml) (partial stub). Request-body JSON Schemas: [schemas/](schemas/).

## Activity Contract

This composition implements normal Niyanta activities. There is no special interface — `sweep`, `diagnose`, `tool_lookup`, `model_call`, `remediate`, and `apply_control` are registered activity types with the worker, advertised as capabilities like any other.

The composition supplies thin helpers on top of the platform runtime:

- a **context retriever** that issues tenant-scoped, read-only queries against Data Connector telemetry (these run as `tool_lookup` children so they are replay-safe and audited)
- a **redactor** that strips payloads/secrets before prompt assembly
- a **model client** wrapped so each call is a `model_call` child (retried, costed, recorded)
- a **structured-output parser/validator** that turns a model response into a `Finding` or fails the attempt
- a **guardrail evaluator** over the tenant policy and applicable playbooks
- a **control client** that calls the Data Connector control API as the `apply_control` child

Custom diagnosis logic is just the body of `diagnose`/`remediate`; it does not extend the engine.

## Validation Rules

Before enabling LLMOps for a tenant:

- A tenant policy exists and declares automation mode + confidence floor per control type (default all `suggest`).
- Budgets are set; the composition refuses to run model calls once a budget is exhausted (emits an event, falls back to suggest-only or pauses).
- Redaction is verified against the fixtures in [REDACTION_CONTRACT.md](REDACTION_CONTRACT.md) — no payloads or secrets reach a prompt.
- Every automatable issue category has at least one applicable playbook with `allowed_actions`.
- The control client is wired to the Data Connector API only; no direct DB write path to connector state exists.
