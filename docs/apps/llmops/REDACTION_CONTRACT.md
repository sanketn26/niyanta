# LLMOps Redaction Contract

**Version**: 0.1
**Last Updated**: 2026-06-03
**Status**: Draft

## Purpose

LLMOps sends connector telemetry to an LLM to diagnose issues. This document is the **enforceable contract** for what may and may not leave the trust boundary in a prompt, where redaction happens, and how it is tested. It backs the safety rules in [OPERATIONS_AND_FAILURE_SEMANTICS.md](OPERATIONS_AND_FAILURE_SEMANTICS.md) and [ARCHITECTURE.md](ARCHITECTURE.md) §Multi-Tenancy.

Principle: **prompts contain only tenant-scoped, redacted metadata and references — never record payloads, never secrets.** Redaction is applied during prompt assembly, before any model call or log write, and is fail-closed: if a field cannot be classified, it is dropped.

## Allowed in a prompt

- Metadata and references: `customer_id`, `connection_id`, `connector_definition_id`, `connector_version`, `run_id`, `partition_id`, `source_kind`, `destination_kind`.
- Aggregates and counters: records read/emitted/dropped, lag seconds, checkpoint age, error rates, backoff counts.
- Health/control state: connection health, active controls, recent state transitions.
- Error **classes and messages with values stripped**: e.g. `HTTP 429 from source` ✔; the raw response body ✘.
- Event types and severities from `ingestion_events`; the `message` field only after value-redaction.
- Playbook text (knowledge), which is curated and contains no tenant data.

## Forbidden in a prompt (drop or mask)

| Category | Examples | Action |
|----------|----------|--------|
| Secrets / credentials | API tokens, passwords, `secret://` resolved values, auth headers, connection strings | **never present**; only refs exist, refs are dropped |
| Record payloads | any source record body, parsed event fields, sample rows | drop entirely |
| Raw responses/requests | HTTP bodies, response envelopes, query results | drop; keep status/shape only |
| PII | emails, names, IPs, user IDs inside payloads, tokens in URLs | mask |
| High-cardinality identifiers from data | cursors/offsets that embed data, signed URLs | mask query/signature parts |
| Free-text that may embed the above | error `message`, dead-letter reason | value-redact (see below) |

## Value redaction rules (for free-text fields that are allowed in principle)

Applied to any string field permitted above (e.g. `ingestion_events.message`, error messages):

- Mask anything matching secret patterns: `(?i)(bearer\s+[a-z0-9._\-]+)`, `(?i)(api[_-]?key|token|password|secret)\s*[=:]\s*\S+`, `secret://\S+`.
- Mask URLs to scheme+host+path; strip query string and userinfo: `https://api.vendor.com/events?token=...` → `https://api.vendor.com/events?<redacted>`.
- Mask emails, IPv4/IPv6 literals, and long base64/hex blobs (≥ 32 chars) as `<redacted>`.
- Truncate any single field to a configured max (default 512 chars) to bound accidental payload leakage.

## Where redaction runs

1. Context retrieval (`tool_lookup` children) returns **already-projected** fields — queries select metadata/aggregates, not payload columns. Payload columns are never selected.
2. The **redactor** runs over the assembled context bundle before prompt construction, applying the forbidden-category drops and value-redaction rules. Fail-closed on unknown fields.
3. Prompt assembly consumes only the redacted bundle.
4. Logging/tracing of prompts: the **raw prompt is never logged**. Only `prompt_hash` (sha256), token counts, and `redaction_applied: true` are persisted (`llmops_model_calls`). Span attributes follow the same rule as the rest of the platform (payloads/secrets redacted before emission).

## Prompt logging policy

- Store: `prompt_hash`, `model_id`, token counts, cost, latency, `redaction_applied`, `result`.
- Do **not** store: the prompt text, the model's raw completion, or any context bundle.
- For debugging, a tenant may opt in to **redacted** prompt capture (the post-redaction bundle, retained ≤ 7 days, tenant-scoped). Opt-out is the default.

## Model provider failure matrix

| Condition | Classification | Behavior |
|-----------|----------------|----------|
| HTTP 429 / rate limit | transient | retry with backoff (platform retry policy on `model_call`) |
| HTTP 5xx / provider outage | transient | retry; if exhausted → diagnosis `inconclusive` |
| Timeout | transient | retry; if exhausted → `inconclusive` |
| HTTP 4xx (bad request) | permanent | fail attempt; alert (likely a prompt/contract bug) |
| Auth/quota error to provider | permanent | fail; page on-call (LLMOps credential/billing issue) |
| Response fails `finding.schema.json` parse | retryable (bounded) | retry up to N; then `inconclusive` — never emit a malformed finding |
| Budget exhausted (tenant) | policy | stop calling model; fall back to suggest-only; emit `budget_exhausted` |

A `model_call` is a `RunChild`, so a completed call is recorded in the call log: a parent resume after a crash does not re-issue it and does not re-spend tokens.

## Test fixtures

Redaction is verified during tenant onboarding and in CI against fixtures in [fixtures/redaction/](fixtures/redaction/). Each fixture is a pair: an `input` context bundle (pre-redaction, deliberately seeded with secrets/payloads/PII) and the `expected` post-redaction bundle. A test asserts `redact(input) == expected` and additionally asserts no string from a known secret/payload set survives in the output.

Fixture format:

```json
{
  "name": "source_429_with_token_in_message",
  "input": {
    "connection_id": "conn_123",
    "events": [
      { "event_type": "source_rate_limited", "message": "429 from https://api.vendor.com/events?token=abc123secret; retry-after 60" }
    ],
    "secret_refs": { "api_token": "secret://cust_abc/github-prod-token" },
    "sample_record": { "user_email": "alice@example.com", "ssn": "123-45-6789" }
  },
  "expected": {
    "connection_id": "conn_123",
    "events": [
      { "event_type": "source_rate_limited", "message": "429 from https://api.vendor.com/events?<redacted>; retry-after 60" }
    ]
  },
  "must_not_contain": ["abc123secret", "secret://cust_abc/github-prod-token", "alice@example.com", "123-45-6789"]
}
```

Rules for fixtures:

- `sample_record` and `secret_refs` must be entirely absent from `expected` (whole-category drop).
- `must_not_contain` is an explicit deny-list checked against the serialized output, independent of `expected`.
- CI fails the build if any fixture's `redact(input)` diverges from `expected` or contains a `must_not_contain` string.
