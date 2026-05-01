# Operations And Failure Semantics

**Version**: 0.1
**Last Updated**: 2026-05-02
**Status**: Draft

## Overview

This document defines the operational correctness rules for Niyanta ingestion. These rules apply to every connector kind, including HTTP polling, push ingestion, object storage, Kafka, Azure Event Hubs, GCP Pub/Sub, OCI Streaming, syslog/CEF, and custom worker plugins.

## Delivery Guarantee

Niyanta provides **at-least-once ingestion** by default.

This means:

- A record may be delivered more than once during retries, worker crashes, source rebalances, or destination uncertainty.
- A record must not be considered ingested until the destination has confirmed it.
- Exactly-once behavior is only possible when the destination supports idempotency keys, transactional writes, or deterministic upserts.

Connector definitions should declare:

- source event ID path, if available
- deterministic dedupe key when no source event ID exists
- destination idempotency key behavior
- retryable and non-retryable error classes

## Checkpoint Commit Rule

Checkpoints are committed only when Niyanta is sure ingestion has succeeded.

Required order:

```text
read from source
parse and transform
write to destination
receive destination success/ack
commit Niyanta checkpoint
commit source offset/cursor when applicable
```

For Kafka, Event Hubs, GCP Pub/Sub, and OCI Streaming:

- Do not commit source offset/cursor/ack before destination confirmation.
- Store the committed position in Niyanta even when also committing to the source system.
- If destination success is unknown, retry from the previous checkpoint and rely on dedupe/idempotency.

For file/object sources:

- Do not mark an object processed until all emitted records are delivered.
- If reading by byte offset, commit only the highest contiguous delivered offset.

For HTTP polling:

- Commit window/cursor checkpoint only after the response page has been delivered.
- If a page partially fails delivery, retry the page or use destination idempotency keys.

## Dead Letter Policy

Dead letters are supported through either file/object storage or a streaming sink.

Allowed dead-letter destinations:

- object storage path, such as S3/GCS/Azure Blob/OCI Object Storage
- Niyanta event stream
- Kafka/Event Hubs/Pub/Sub/OCI stream
- customer-provided webhook or sink adapter when explicitly configured

Dead-letter records must include:

- `customer_id`
- `connection_id`
- `run_id`
- `partition_id`, when applicable
- original source metadata
- redacted raw payload, when policy allows
- parser/validation error
- connector definition/version
- created timestamp

Policy options:

```yaml
dead_letter:
  mode: stream
  destination_ref: dlq_security_events
  include_raw_payload: redacted
  max_record_size_bytes: 1048576
  fail_run_on_dead_letter_failure: true
  quarantine_after:
    count: 1000
    window_seconds: 300
```

Dead-letter behavior:

- Parser or validation errors can be dead-lettered without failing the run when policy allows.
- Dead-letter sink failure fails the run by default, because otherwise records can disappear.
- Repeated dead-letter spikes can degrade or quarantine the connection.

## Connector Spec Versioning

Connector definitions are versioned YAML specs.

Version rules:

- Patch version: metadata, docs, sample queries, non-behavioral validation improvements.
- Minor version: backward-compatible fields, optional transforms, new auth methods, new parser options.
- Major version: breaking config schema, output schema, checkpoint shape, transform semantics, or destination contract.

Rules:

- Running connections pin `definition_id` and `definition_version`.
- New connector versions do not automatically change existing connections.
- Upgrades are explicit and auditable.
- Breaking checkpoint changes require a migration function or a new checkpoint namespace.
- Rollback must preserve the prior definition version and checkpoint state.

Upgrade flow:

```text
publish definition version
validate existing connection config
dry-run transform against sample/recent payloads
check checkpoint migration compatibility
apply upgrade
wake connection supervisor
audit upgrade event
```

## Secret Rotation

Secret rotation automatically wakes affected connection supervisors.

Rules:

- Connections store secret references, never raw secret values.
- A secret update emits a tenant-scoped `secret_updated` event.
- The connection manager finds all connections using that secret ref.
- Each affected connection supervisor receives a `secret_rotated` signal.
- The next run resolves the latest secret value before source access.
- In-flight runs may finish with the old secret unless the secret is revoked; revoked secrets should cancel or fail affected attempts.

Secret rotation flow:

```text
secret ref updated
audit event written
affected connections resolved
connection supervisors signaled
connection test optionally executed
next planned run uses new secret
```

## Backfill Isolation

Backfills never mutate the live checkpoint namespace.

Rules:

- Live ingestion uses namespace `live`.
- Backfills use namespace `backfill/{backfill_id}`.
- Backfills default to lower priority than live ingestion.
- Backfills obey tenant quotas, source limits, and destination backpressure.
- Backfill output must use idempotency keys that include source identity and event identity, not backfill run ID.

## Blast Radius Controls

A bad connection must not harm other tenants or unrelated connections.

Required controls:

- per-tenant concurrent run limits
- per-tenant queue depth limits
- per-connection parallel partition limits
- per-source QPS/burst limits
- per-destination flush concurrency limits
- worker-pool isolation modes: shared, pooled, dedicated
- automatic degradation/quarantine for repeated failures
- temporary rate-limit override and temporary disable controls

Failure containment:

- Auth failures affect only the connection using that credential.
- Parser/drop storms affect only the connection unless a shared parser version is proven bad.
- Destination failures affect only connections targeting that destination ref.
- Source throttling reduces planning only for that source/connection scope.
- A tenant cannot consume the entire shared worker pool unless explicitly configured.

## Audit Requirements

Every operator or automated control action must write an audit event:

- connector definition published
- connector connection created/updated/deleted
- connector version upgraded/rolled back
- secret reference changed
- rate-limit override applied/cleared
- temporary disable applied/expired/cleared
- pause/resume
- quarantine applied/cleared
- backfill requested/cancelled
- dead-letter policy changed

Audit events answer: who changed what, when, why, and from where.

