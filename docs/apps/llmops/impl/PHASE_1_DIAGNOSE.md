# LLMOps Phase 1 — Read-Only Diagnosis

**Status**: Draft
**Last Updated**: 2026-06-03

**Goal**: Stand up the composition end-to-end in read-only mode. A per-tenant supervisor wakes, picks at-risk connections, asks a model what is wrong, and records a structured `Finding` as a suggestion. No action is taken on the fleet.

**Depends on**: platform phases 0–4 (activities, `RunChild`, `Sleep`, `AwaitSignal`, retries, replay) and data-connector phase 5 (connections, runs, `ingestion_events`, health state, metrics).

## Scope

In:

- `sweep` activity (one per tenant or per shard): `Sleep` between sweeps; `AwaitSignal` for significant events; deterministic candidate selection from retrieved state; `RunChild("diagnose", connectionID)` per candidate, with per-connection cooldown dedup.
- `diagnose` activity: gather a fixed, redacted context bundle from Data Connector telemetry; one `RunChild("model_call", prompt)`; structured-output parse + validation into a `Finding`.
- `model_call` child: wrapped model client (retry/backoff on 429/timeout); writes a `llmops_model_calls` row (tokens, cost, latency, result).
- Redactor: strips payloads/secrets before prompt assembly; verified against sample telemetry.
- Storage: `llmops_diagnosis_runs`, `llmops_findings`, `llmops_model_calls`.
- API: `GET findings`, `GET findings/{id}`, `GET connections/{id}/findings`, `POST connections/{id}:diagnose` (manual trigger).
- Outcome states: `completed`, `inconclusive`, `failed`. Inconclusive is first-class.

Out (later phases):

- Tool-lookup child activities (Phase 2) — Phase 1 uses one fixed context query inline within `diagnose`'s context build (still no payloads).
- Runbooks/playbooks (Phase 2) — Phase 1 uses a single built-in diagnosis prompt template.
- Any remediation or control API call (Phase 3).

## Key Deliverables

- Activity types `sweep`, `diagnose`, `model_call` registered and advertised as worker capabilities.
- Candidate selection is deterministic (replay-safe): derived only from retrieved, recorded state — no `time.Now()`/`rand` in parent code, per the platform determinism rule.
- Prompt assembly is redaction-first; a test asserts no payload/secret string can reach a prompt.
- Structured output: model returns JSON matching the `Finding` schema (`issue_category`, `root_cause`, `confidence`, `evidence` refs, optional `suggested_action`); parse failure → retry → `inconclusive`.
- Tenant isolation: every query for context is tenant-scoped at the storage boundary; a diagnosis for tenant A cannot read tenant B.
- Metrics: `niyanta_llmops_diagnosis_runs_total`, `niyanta_llmops_findings_total`, `niyanta_llmops_model_calls_total`, `niyanta_llmops_model_tokens_total`, `niyanta_llmops_diagnosis_latency_seconds`.

## Done When

- A significant `source_rate_limited` event on a test connection wakes `sweep`, which runs `diagnose`, which produces a `Finding` with a non-trivial root cause and a `suggested_action`, visible via the findings API — and no control was applied.
- Killing the `diagnose` worker after the `model_call` completes resumes the parent from the call log without a second model call (verified by token count unchanged).
- A malformed model response yields an `inconclusive` run, not a bad finding.
