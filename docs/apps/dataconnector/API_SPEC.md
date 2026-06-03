# Data Connector API Specification

**Version**: 0.1
**Last Updated**: 2026-06-03
**Status**: Draft

## Overview

This is the **Data Connector application's** API — connector definitions, connections, runs, catalogs, controls, the customer ingestion view, and significant-event filters. It is **not** part of the Niyanta engine API; the engine only exposes activities ([../../platform/API_SPEC.md](../../platform/API_SPEC.md)). These endpoints are the higher-level interface the Data Connector composition exposes over its activities, and they are owned by the composition.

Conventions inherited from the platform API:

- **Auth**: `Authorization: Bearer <api_key>`; `customer_id` is derived from the key and enforced at the storage boundary. Cross-tenant access fails closed (`404`).
- **Errors**: the shape and codes in [../../platform/API_SPEC.md](../../platform/API_SPEC.md) §Error Handling.
- **Pagination**: cursor-based (`limit` + `cursor`) preferred; offset alternative. See platform §Pagination.
- **Idempotency**: mutating creates accept an `Idempotency-Key` header.
- **Base URL**: `https://api.niyanta.example.com/v1`.

The domain model and storage for these resources are in [DATA_CONNECTOR_SPEC.md](DATA_CONNECTOR_SPEC.md); JSON-Schema validation for request bodies is in [schemas/](schemas/).

## Endpoint Index

### Connector definitions
- `POST /connector-definitions` — register a definition
- `GET /connector-definitions` — list
- `GET /connector-definitions/{id}/versions/{version}` — get one version
- `POST /connector-definitions:import` — import external (e.g. Airbyte) metadata

### Connections
- `POST /connector-connections` — create
- `GET /connector-connections` — list
- `GET /connector-connections/{id}` — get
- `PATCH /connector-connections/{id}` — update config/destination
- `POST /connector-connections/{id}:test` — connectivity test
- `POST /connector-connections/{id}:discover` — run discovery
- `GET /connector-connections/{id}/catalog` — get selected catalog
- `PATCH /connector-connections/{id}/catalog` — update stream selection
- `POST /connector-connections/{id}:run` — trigger a run / backfill
- `POST /connector-connections/{id}:pause` / `:resume`

### Controls (erring connections)
- `POST /connector-connections/{id}:rate-limit` — apply rate-limit override
- `POST /connector-connections/{id}:temporary-disable`
- `POST /connector-connections/{id}:clear-control`
- `GET /connector-connections/{id}/controls` — list active controls

### Runs
- `GET /connector-connections/{id}/runs` — list runs for a connection
- `GET /connector-runs/{id}` — get one run

### Customer ingestion operations
- `GET /customers/{cid}/ingestion-view`
- `GET /customers/{cid}/ingestion-events`
- `POST /customers/{cid}/ingestion-event-filters`
- `GET /customers/{cid}/ingestion-event-filters`
- `PATCH /customers/{cid}/ingestion-event-filters/{filter_id}`

---

## Connections

### Create Connection

`POST /connector-connections`

```json
{
  "definition_id": "github_audit",
  "definition_version": "1.0.0",
  "name": "prod-org-audit",
  "config": { "org": "example", "poll_interval_seconds": 300 },
  "secret_refs": { "api_token": "secret://cust_abc/github-prod-token" },
  "destination": { "type": "webhook", "endpoint_ref": "dest_security_lake" },
  "isolation": { "level": "shared", "worker_pool": "default" }
}
```

**Response** `201 Created`:
```json
{ "connection_id": "conn_123", "status": "configured", "created_at": "2026-06-03T00:00:00Z" }
```

Rules: secrets are references only (`400` if a raw secret is detected in `config`); requested `isolation.level` must be ≥ the definition's minimum (`400` otherwise). Creating an enabled connection launches its supervisor activity on the engine.

**Errors**: `400 INVALID_INPUT` (schema/secret/isolation), `404 NOT_FOUND` (unknown definition), `409 CONFLICT` (duplicate name for tenant).

### Get / List Connection

`GET /connector-connections/{id}` → the connection plus current `health` and active `controls`.
`GET /connector-connections?status=enabled&connector_id=github_audit&limit=50&cursor=...`

### Test Connection

`POST /connector-connections/{id}:test` → runs the connector's `Test` (validates credentials, reachability, permissions) as a short activity.
```json
{ "result": "ok", "checks": [ { "name": "auth", "ok": true }, { "name": "reachability", "ok": true } ], "latency_ms": 420 }
```
`result` ∈ `ok | failed`; `failed` returns `200` with per-check detail (not an HTTP error — it's a successful test that found a problem).

### Discover / Catalog

`POST /connector-connections/{id}:discover` → produces a versioned catalog snapshot.
`GET /connector-connections/{id}/catalog` → current selected catalog.
`PATCH /connector-connections/{id}/catalog` → change stream selection / sync modes; stored as a new catalog version. A run references a catalog version, not a mutable discovery result.

### Trigger Run / Backfill

`POST /connector-connections/{id}:run`
```json
{ "trigger": "manual", "window": { "start": "2026-05-01T00:00:00Z", "end": "2026-05-02T00:00:00Z" }, "backfill": true }
```
`202 Accepted` → `{ "run_id": "run_123", "status": "PENDING" }`. Backfills use an isolated checkpoint namespace and default to lower priority (see [OPERATIONS_AND_FAILURE_SEMANTICS.md](OPERATIONS_AND_FAILURE_SEMANTICS.md) §Backfill Isolation).

---

## Controls

### Apply Rate-Limit Override

`POST /connector-connections/{id}:rate-limit`
```json
{ "qps": 0.25, "burst": 1, "max_parallel_partitions": 1, "reason": "source 429 spike", "expires_at": "2026-06-03T18:00:00Z" }
```
`200 OK` → `{ "control_id": "ctrl_123", "control_type": "rate_limit_override", "expires_at": "..." }`. Writes an audit event. The planner reads active controls before creating runs (precedence in [INGESTION_ARCHITECTURE.md](INGESTION_ARCHITECTURE.md) §Erring Connection Controls).

### Temporarily Disable

`POST /connector-connections/{id}:temporary-disable`
```json
{ "expires_at": "2026-06-04T00:00:00Z", "resume_strategy": "auto_on_expiry", "reason": "vendor maintenance" }
```

### Clear Control

`POST /connector-connections/{id}:clear-control`
```json
{ "control_id": "ctrl_123", "reason": "source recovered" }
```

### List Active Controls

`GET /connector-connections/{id}/controls` → active (uncleared) controls with type, config, reason, expiry.

> These four endpoints are the only mutation path to a connection's effective control state. The LLMOps app's automated remediation calls exactly these (never the DB directly).

---

## Runs

`GET /connector-connections/{id}/runs?status=FAILED&limit=20` — recent runs with metrics.
`GET /connector-runs/{id}`:
```json
{
  "run_id": "run_123", "connection_id": "conn_123", "customer_id": "cust_abc",
  "activity_id": "act_789", "status": "COMPLETED", "trigger": "schedule",
  "window": { "start": "2026-05-01T10:00:00Z", "end": "2026-05-01T10:05:00Z" },
  "metrics": { "records_read": 1000, "records_emitted": 998, "records_dropped": 2 },
  "started_at": "...", "completed_at": "..."
}
```
`status` is the underlying engine `activity_status`; `health` of the connection is derived separately (see DATA_CONNECTOR_SPEC §Health).

---

## Customer Ingestion Operations

These are the primary operator surface for a customer (`cid`). All responses are tenant-scoped and derived from tenant-scoped tables (never a cross-tenant query filtered later). The caller authenticates as that customer or holds an internal support permission for the `cid`.

### Get Customer Ingestion View

`GET /customers/{cid}/ingestion-view`

**Query**: `window` (e.g. `15m`, `1h`, `24h`; default `1h`), `include_events` (default `true`), `status` (filter by effective status).

```json
{
  "customer_id": "cust_abc",
  "summary": { "connections_total": 42, "connected": 35, "degraded": 4, "failing": 2, "temporarily_disabled": 1, "quarantined": 0 },
  "connections": [
    {
      "connection_id": "conn_123", "name": "prod-github-audit", "connector_id": "github_audit",
      "health": "degraded", "effective_status": "rate_limited",
      "source_lag_seconds": 900, "checkpoint_age_seconds": 300,
      "records_read": 12000, "records_emitted": 11980, "records_dropped": 20,
      "active_controls": [ { "control_id": "ctrl_123", "control_type": "rate_limit_override", "expires_at": "2026-06-03T18:00:00Z", "reason": "source 429 spike" } ],
      "latest_significant_event": { "event_type": "source_rate_limited", "severity": "warning", "message": "Source returned repeated 429 responses" }
    }
  ]
}
```

### List Significant Ingestion Events

`GET /customers/{cid}/ingestion-events?significant=true&connection_id=conn_123&limit=50&cursor=...`
```json
{ "events": [ { "id": "evt_456", "connection_id": "conn_123", "severity": "warning", "event_type": "source_rate_limited", "significant": true, "message": "...", "created_at": "..." } ], "next_cursor": null }
```

### Significant Event Filters

`POST /customers/{cid}/ingestion-event-filters`
```json
{ "name": "auth failures", "enabled": true, "scope": { "connector_definition_id": "github_audit" }, "match": { "event_type": "auth_failure" }, "action": { "mark_significant": true } }
```
`GET /customers/{cid}/ingestion-event-filters` — list.
`PATCH /customers/{cid}/ingestion-event-filters/{filter_id}` — enable/disable or edit match. Filters classify events for the view and alerting; they do not change ingestion behavior (controls do).

---

## OpenAPI

The machine-readable contract for these endpoints is at [`api/openapi/dataconnector-v1.yaml`](../../../api/openapi/dataconnector-v1.yaml) (partial stub; see the engine contract at `api/openapi/niyanta-v1.yaml`). Request-body JSON Schemas are in [schemas/](schemas/).

## References

- [DATA_CONNECTOR_SPEC.md](DATA_CONNECTOR_SPEC.md) — domain model and storage
- [INGESTION_ARCHITECTURE.md](INGESTION_ARCHITECTURE.md) — control loops and controls precedence
- [OPERATIONS_AND_FAILURE_SEMANTICS.md](OPERATIONS_AND_FAILURE_SEMANTICS.md) — backfill isolation, audit
- [../../platform/API_SPEC.md](../../platform/API_SPEC.md) — engine API these build on
