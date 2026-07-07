# Niyanta Platform API Specification

**Version**: 2.0
**Last Updated**: 2026-06-03
**Status**: Design Phase

> **v2.0 is the engine API.** This document covers only the **platform** surface — activities, workers, health, and cross-cutting concerns (auth, errors, rate limiting, pagination, versioning). Anything a consumer builds on top exposes its own API elsewhere; the engine's only domain concept is the activity. The v1 `workloads` surface is renamed to `activities` per [adr/ADR-005-activity-execution-model.md](adr/ADR-005-activity-execution-model.md).

## Table of Contents
1. [Overview](#overview)
2. [Authentication](#authentication)
3. [REST API Endpoints](#rest-api-endpoints)
4. [gRPC API](#grpc-api)
5. [Error Handling](#error-handling)
6. [Rate Limiting](#rate-limiting)
7. [Pagination](#pagination)
8. [WebSocket API (Optional)](#websocket-api-optional)
9. [API Versioning](#api-versioning)
10. [OpenAPI Specification](#openapi-specification)
11. [SDK Examples](#sdk-examples)

---

## Overview

Niyanta exposes two API surfaces:

1. **Coordinator REST/gRPC API** — for clients to submit and manage **activities** and to administer workers and health. This is what consumers build on; the activity is the only domain concept the engine exposes.
2. **Internal Control-Plane Protocol** — coordinator↔worker communication (see [DATA_MODELS.md](DATA_MODELS.md) §Broker Message Formats).

This document focuses on the Coordinator API. Consumers layer their own higher-level APIs on top; those are not engine concerns.

---

## Authentication

### API Key Authentication (Phase 1)

```
Authorization: Bearer <api_key>
```

**API Key Format**: `nyt_<customer_id>_<random_32_chars>` — e.g. `nyt_cust_abc123_a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6`

**Validation**:
1. Extract `customer_id` from key prefix.
2. Verify key exists (table: `api_keys`), check expiration/status.
3. Inject `customer_id` into request context — **all** queries filter by it at the storage boundary.

### JWT Authentication (Phase 2)

```
Authorization: Bearer <jwt_token>
```

```json
{ "sub": "user_123", "customer_id": "cust_abc123", "tier": "premium", "exp": 1735308896, "iat": 1735305296 }
```

---

## REST API Endpoints

**Base URL**: `https://api.niyanta.example.com/v1`

### 1. Activity Management

#### 1.1 Submit Activity

**Endpoint**: `POST /activities`

**Description**: Submit a new top-level activity for execution. (Child activities are created by a parent's `RunChild`, not via this endpoint.)

**Request Headers**:
```
Authorization: Bearer <api_key>
Content-Type: application/json
Idempotency-Key: <uuid>   (optional; see Idempotency)
```

**Request Body**:
```json
{
  "activity_type": "sample_pipeline",
  "input_json": { "item_id": "item_123" },
  "priority": 10,
  "affinity_rules": { "hard_affinity": { "worker_tags": { "hardware": "gpu" } } },
  "retry_policy": { "max_attempts": 3, "backoff": "exponential" }
}
```

**Request Schema**:
```go
type ActivitySubmitRequest struct {
    ActivityType  string                 `json:"activity_type" binding:"required"`
    InputJSON     map[string]interface{} `json:"input_json" binding:"required"` // opaque to engine
    Priority      int                    `json:"priority,omitempty"`             // default 10
    AffinityRules *AffinityRules         `json:"affinity_rules,omitempty"`
    RetryPolicy   *RetryPolicy           `json:"retry_policy,omitempty"`
}
```

> `activity_type` must be a type advertised by at least one worker (`400 INVALID_INPUT` otherwise). `input_json` is opaque — the engine never inspects it. Secrets must be passed as references, never inline.

**Response**: `202 Accepted`
```json
{ "activity_id": "act_abc123", "status": "PENDING", "created_at": "2026-06-03T12:34:56Z", "estimated_start_time": "2026-06-03T12:35:30Z" }
```

**Errors**: `400 INVALID_INPUT` (unknown activity_type / bad input), `401 UNAUTHORIZED`, `429 RATE_LIMITED`, `503 SERVICE_UNAVAILABLE` (queue full).

---

#### 1.2 Get Activity

**Endpoint**: `GET /activities/{activity_id}`

**Query Parameters**: `wait` (optional, e.g. `30s`, max `60s`) — long-poll: the response returns early as soon as the activity reaches a terminal state (`COMPLETED`/`FAILED`/`CANCELLED`), otherwise after the wait window with the current state. This is the preferred completion-watch mechanism; clients should not poll in a tight loop.

**Response**: `200 OK`
```json
{
  "activity_id": "act_abc123",
  "customer_id": "cust_abc123",
  "activity_type": "sample_pipeline",
  "parent_activity_id": null,
  "root_activity_id": "act_abc123",
  "status": "SUSPENDED",
  "suspend_reason": "sleep",
  "wake_at": "2026-06-03T12:40:00Z",
  "priority": 10,
  "worker_id": null,
  "input_json": { "item_id": "item_123" },
  "result_json": null,
  "error_json": null,
  "current_attempt": 1,
  "generation": 1,
  "created_at": "2026-06-03T12:34:56Z",
  "scheduled_at": "2026-06-03T12:35:00Z",
  "started_at": "2026-06-03T12:35:10Z",
  "completed_at": null
}
```

**Errors**: `404 NOT_FOUND` (does not exist or belongs to another customer), `401 UNAUTHORIZED`.

---

#### 1.3 List Activities

**Endpoint**: `GET /activities`

**Query Parameters**: `status`, `activity_type`, `parent_activity_id`, `root_activity_id`, `limit` (default 50, max 100), `cursor` (preferred) or `offset`, `order_by` (default `created_at`), `order` (`asc`|`desc`).

**Example**: `GET /activities?status=RUNNING&activity_type=sample_pipeline&limit=20&order_by=priority&order=desc`

**Response**: `200 OK`
```json
{ "activities": [ { "activity_id": "act_abc123", "status": "RUNNING", "...": "..." } ], "next_cursor": "eyJpZCI6..." }
```

---

#### 1.4 Activity Lifecycle Operations

| Operation | Endpoint | Notes |
|-----------|----------|-------|
| Cancel | `POST /activities/{id}:cancel` | Cancels the activity and its in-flight children (depth-first; the suspended call site returns `ErrCancelled`). Body: `{ "reason": "..." }`. |
| Pause | `POST /activities/{id}:pause` | Pauses at the next safe suspend point. |
| Resume | `POST /activities/{id}:resume` | Resumes a paused activity. |
| Signal | `POST /activities/{id}:signal` | Deliver an external event. Body: `{ "name": "...", "payload": { } }`. Accepts `Idempotency-Key` header — a retried send with the same key is one mailbox entry. |

**Response**: `200 OK` with the updated activity, or `409 INVALID_STATE` if the transition is not allowed.

**Operator recovery operations** (Phase 5; RBAC `operator`+, every call audited — the escape hatches for nondeterminism poisoning per ADR-005 G6):

| Operation | Endpoint | Notes |
|-----------|----------|-------|
| Force-fail | `POST /activities/{id}:force-fail` | Terminal `FAILED` with operator-supplied reason. |
| Force-complete | `POST /activities/{id}:force-complete` | Terminal `COMPLETED` with operator-supplied `result_json`. |
| Truncate replay | `POST /activities/{id}:truncate-replay` | Drops the call-log suffix from a divergent index and re-enqueues. Body: `{ "from_call_index": N, "confirm": "<token>" }`. |

---

#### 1.5 Get Activity Attempts

**Endpoint**: `GET /activities/{activity_id}/attempts`

**Description**: The per-attempt audit trail (replaces the v1 `retry_count`). One row per physical attempt with worker, generation, error, and timing.

```json
{
  "attempts": [
    { "attempt_number": 1, "worker_id": "worker-3", "generation": 1, "status": "FAILED", "error": { "type": "transient", "message": "..." }, "started_at": "...", "ended_at": "..." },
    { "attempt_number": 2, "worker_id": "worker-7", "generation": 2, "status": "RUNNING", "started_at": "..." }
  ]
}
```

---

#### 1.6 Get Activity Logs

**Endpoint**: `GET /activities/{activity_id}/logs`

Streams or pages structured log lines for the activity and (optionally, `?include_children=true`) its composition tree. Standard pagination applies.

---

#### 1.7 Get Activity Audit Trail

**Endpoint**: `GET /activities/{activity_id}/audit`

Returns `audit_events` for the activity (created, scheduled, child_dispatched, suspended, resumed, signal_received, completed, redistributed, …).

---

#### 1.8 Inspection Reads (Phase 5)

The console's data sources; also `curl`-debuggable.

| Endpoint | Returns |
|----------|---------|
| `GET /activities/{id}/calls` | The call log for the current execution: `call_index`, `call_type`, `args_hash`, `child_activity_id`, completion state, recorded result. For a suspended activity, the last incomplete entry *is* "where it is parked." |
| `GET /activities/{id}/signals` | The signal mailbox: pending and delivered entries. |
| `GET /activities/{id}/executions` | The `ContinueAsNew` chain: one entry per execution (`execution_seq`, input, result, call count, timings). |

---

### 2. Customer & Account

#### 2.1 Get Customer Info

**Endpoint**: `GET /customers/{customer_id}`

```json
{ "customer_id": "cust_abc123", "name": "Acme", "tier": "premium", "limits": { "max_concurrent_activities": 500, "max_queue_depth": 2000, "rate_limit_per_sec": 50 } }
```

#### 2.2 Get Usage Statistics

**Endpoint**: `GET /customers/{customer_id}/usage`

**Query**: `window` (e.g. `1h`, `24h`, `30d`).

```json
{ "window": "24h", "activities_submitted": 1820, "activities_completed": 1790, "activities_failed": 30, "concurrent_peak": 240, "current_queue_depth": 12 }
```

---

### 3. Worker Management (Admin Only)

Requires an internal admin permission, not a customer key.

| Operation | Endpoint | Notes |
|-----------|----------|-------|
| List workers | `GET /workers` | Returns id, status, advertised `activity_types`, capacity, tags, last heartbeat. |
| Drain worker | `POST /workers/{id}:drain` | Stops new assignments; finishes in-flight. |
| Remove worker | `POST /workers/{id}:remove` | Marks DEAD; triggers redistribution of its activities. |

```json
// GET /workers
{ "workers": [ { "id": "worker-5", "status": "HEALTHY", "activity_types": {"sample_pipeline": "1.2.3", "sample_leaf": "1.2.3"}, "capacity_total": 10, "capacity_used": 3, "tags": { "region": "us-west-2" }, "last_heartbeat": "..." } ] }
```

---

### 4. Health & Monitoring

| Endpoint | Purpose | Response |
|----------|---------|----------|
| `GET /health` | Liveness | `200 {"status":"ok"}` |
| `GET /ready` | Readiness (DB reachable, leader known; broker included only if a Phase 7 substrate is configured) | `200`/`503` with dependency detail |
| `GET /metrics` | Prometheus exposition | `text/plain` metrics |

```
# GET /metrics (excerpt)
# HELP niyanta_activity_attempts_total Total activity attempts
# TYPE niyanta_activity_attempts_total counter
niyanta_activity_attempts_total{activity_type="sample_pipeline",result="completed"} 1024
# HELP niyanta_activity_duration_seconds Activity execution duration
# TYPE niyanta_activity_duration_seconds histogram
```

---

## gRPC API

### Service Definition (Protocol Buffers)

**File**: `api/niyanta/v1/activity_service.proto` (see [OpenAPI Specification](#openapi-specification) for artifact status).

```protobuf
syntax = "proto3";

package niyanta.v1;

import "google/protobuf/timestamp.proto";
import "google/protobuf/struct.proto";

service ActivityService {
  rpc SubmitActivity(SubmitActivityRequest) returns (SubmitActivityResponse);
  rpc GetActivity(GetActivityRequest) returns (GetActivityResponse);
  rpc ListActivities(ListActivitiesRequest) returns (ListActivitiesResponse);
  rpc CancelActivity(CancelActivityRequest) returns (CancelActivityResponse);
  rpc PauseActivity(PauseActivityRequest) returns (PauseActivityResponse);
  rpc ResumeActivity(ResumeActivityRequest) returns (ResumeActivityResponse);
  rpc SignalActivity(SignalActivityRequest) returns (SignalActivityResponse);
  rpc StreamActivityStatus(StreamActivityStatusRequest) returns (stream ActivityStatusUpdate);
}

message SubmitActivityRequest {
  string activity_type = 1;
  google.protobuf.Struct input_json = 2;
  int32 priority = 3;
  AffinityRules affinity_rules = 4;
  RetryPolicy retry_policy = 5;
}

message SubmitActivityResponse {
  string activity_id = 1;
  ActivityStatus status = 2;
  google.protobuf.Timestamp created_at = 3;
}

message Activity {
  string activity_id = 1;
  string customer_id = 2;
  string activity_type = 3;
  string parent_activity_id = 4;
  string root_activity_id = 5;
  ActivityStatus status = 6;
  int32 priority = 7;
  string worker_id = 8;
  google.protobuf.Struct input_json = 9;
  google.protobuf.Struct result_json = 10;
  google.protobuf.Struct error_json = 11;
  int32 current_attempt = 12;
  int64 generation = 13;
  google.protobuf.Timestamp created_at = 14;
  google.protobuf.Timestamp started_at = 15;
  google.protobuf.Timestamp completed_at = 16;
}

enum ActivityStatus {
  ACTIVITY_STATUS_UNSPECIFIED = 0;
  ACTIVITY_STATUS_PENDING = 1;
  ACTIVITY_STATUS_SCHEDULED = 2;
  ACTIVITY_STATUS_RUNNING = 3;
  ACTIVITY_STATUS_SUSPENDED = 4;
  ACTIVITY_STATUS_PAUSED = 5;
  ACTIVITY_STATUS_REDISTRIBUTING = 6;
  ACTIVITY_STATUS_COMPLETED = 7;
  ACTIVITY_STATUS_FAILED = 8;
  ACTIVITY_STATUS_CANCELLED = 9;
}

message AffinityRules {
  AffinityConstraint hard_affinity = 1;
  AffinityConstraint soft_affinity = 2;
  AffinityConstraint anti_affinity = 3;
}

message AffinityConstraint {
  string worker_id = 1;
  map<string, string> worker_tags = 2;
  repeated string activity_types = 3;
}

message RetryPolicy {
  int32 max_attempts = 1;
  string backoff = 2; // "exponential" | "linear" | "none"
}

message SignalActivityRequest {
  string activity_id = 1;
  string name = 2;
  google.protobuf.Struct payload = 3;
}

message StreamActivityStatusRequest { string activity_id = 1; }

message ActivityStatusUpdate {
  string activity_id = 1;
  ActivityStatus status = 2;
  google.protobuf.Timestamp timestamp = 3;
  string message = 4;
}
```

### gRPC Authentication

```
authorization: Bearer <api_key>
```

```go
ctx := metadata.AppendToOutgoingContext(ctx, "authorization", "Bearer "+apiKey)
resp, err := client.SubmitActivity(ctx, req)
```

---

## Error Handling

### Error Response Format

```json
{
  "error": {
    "code": "INVALID_INPUT",
    "message": "Activity type 'invalid_type' is not registered by any worker",
    "details": { "known_types": ["sample_pipeline", "sample_leaf"] },
    "request_id": "req_abc123"
  }
}
```

gRPC uses standard status codes with error details.

### Error Codes

| HTTP | Code | Description |
|------|------|-------------|
| 400 | `INVALID_INPUT` | Request validation failed |
| 409 | `INVALID_STATE` | Operation not allowed in current state |
| 401 | `UNAUTHORIZED` | Invalid/missing authentication |
| 403 | `FORBIDDEN` | Insufficient permissions |
| 404 | `NOT_FOUND` | Resource does not exist (or cross-tenant) |
| 409 | `CONFLICT` | Resource conflict (e.g. idempotency replay mismatch) |
| 429 | `RATE_LIMITED` | Rate limit exceeded |
| 503 | `SERVICE_UNAVAILABLE` | Overloaded or dependency unavailable |
| 500 | `INTERNAL_ERROR` | Unexpected server error |

### Retry Guidelines

Retryable with exponential backoff: `503`, `429` (respect `Retry-After`), network timeouts. Non-retryable: `400`, `401`, `403`, `404`, `409 INVALID_STATE`.

### Idempotency

`POST /activities` accepts an `Idempotency-Key` header (client UUID). The first request records the key → activity mapping (`idempotency_keys` table); a replay with the same key returns the original `activity_id` and `200`, or `409 CONFLICT` if the same key is reused with a different body.

---

## Rate Limiting

### Headers (on every response)

```
X-RateLimit-Limit: 50
X-RateLimit-Remaining: 47
X-RateLimit-Reset: 1735308896
```

### `429` Response

```json
{ "error": { "code": "RATE_LIMITED", "message": "Rate limit exceeded", "details": { "retry_after_seconds": 2 } } }
```

### Tiers

Per-customer token bucket sized by tier (`free` 1/s … `enterprise` 200/s), as in [DATA_MODELS.md](DATA_MODELS.md) §Customers.

---

## Pagination

### Cursor-Based (Preferred)

```
GET /activities?limit=50&cursor=eyJpZCI6ImFjdF8xMjMifQ
```
```json
{ "activities": [ ], "next_cursor": "eyJpZCI6ImFjdF8xNzMifQ", "has_more": true }
```
A null/absent `next_cursor` means the last page.

### Offset-Based (Alternative)

```
GET /activities?limit=50&offset=100
```
```json
{ "activities": [ ], "total": 1280, "limit": 50, "offset": 100 }
```

---

## WebSocket API (Optional)

### Real-Time Activity Updates

```
wss://api.niyanta.example.com/v1/activities/{activity_id}/stream
Authorization: Bearer <api_key>
```
Server pushes `ActivityStatusUpdate` frames on status change, child dispatch, suspend/resume, and completion.

---

## API Versioning

- **URL versioning**: `/v1`, `/v2`. Breaking changes increment the major version.
- **Deprecation policy**: a deprecated version is supported ≥ 6 months after its successor GA; `Deprecation` and `Sunset` response headers announce timelines.

---

## OpenAPI Specification

The machine-readable contract for this engine API is maintained at:

- **File**: [`api/openapi/niyanta-v1.yaml`](../../api/openapi/niyanta-v1.yaml)
- **Status**: Partial stub covering the engine endpoints above (activities, workers, health). It is the source of truth for engine request/response shapes; app endpoints are documented in their app specs and are **not** in this file. The gRPC `.proto` lives at `api/niyanta/v1/activity_service.proto` (to be generated from the same source as the build matures).

Regenerate SDKs/clients from the OpenAPI file; do not hand-edit generated clients.

---

## SDK Examples

### Go SDK

```go
client := niyanta.NewClient(niyanta.Config{
    BaseURL: "https://api.niyanta.example.com/v1",
    APIKey:  apiKey,
})

// Submit an activity
resp, err := client.SubmitActivity(ctx, &niyanta.ActivitySubmitRequest{
    ActivityType: "sample_pipeline",
    InputJSON:    map[string]any{"item_id": "item_123"},
})

// Poll, or stream status
act, err := client.GetActivity(ctx, resp.ActivityID)
```

### Python SDK

```python
client = niyanta.Client(base_url="https://api.niyanta.example.com/v1", api_key=api_key)

resp = client.submit_activity(activity_type="sample_pipeline",
                              input_json={"item_id": "item_123"})
act = client.get_activity(resp.activity_id)
```

---

## References

- [adr/ADR-005-activity-execution-model.md](adr/ADR-005-activity-execution-model.md) - activity execution model
- [ARCHITECTURE.md](ARCHITECTURE.md) - platform architecture overview
- [DATA_MODELS.md](DATA_MODELS.md) - schemas and message formats
- [../IMPLEMENTATION_PLAN.md](../IMPLEMENTATION_PLAN.md) - phased delivery plan
```
