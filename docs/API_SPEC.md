# Niyanta API Specification

**Version**: 1.0
**Last Updated**: 2025-10-27
**Status**: Design Phase

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

---

## Overview

Niyanta exposes two API surfaces:

1. **Coordinator REST/gRPC API**: For clients to submit, query, and manage workloads
2. **Internal Broker Protocol**: For coordinator-worker communication (see [DATA_MODELS.md](DATA_MODELS.md))

This document focuses on the **Coordinator API** for external clients.

---

## Authentication

### API Key Authentication (Phase 1)

**Request Header**:
```
Authorization: Bearer <api_key>
```

**API Key Format**:
- Customer-specific: `nyt_<customer_id>_<random_32_chars>`
- Example: `nyt_cust_abc123_a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6`

**API Key Validation**:
1. Extract customer_id from key prefix
2. Verify key exists in database (table: `api_keys`)
3. Check key expiration and status
4. Inject `customer_id` into request context

### JWT Authentication (Phase 2)

**Request Header**:
```
Authorization: Bearer <jwt_token>
```

**JWT Claims**:
```json
{
  "sub": "user_123",
  "customer_id": "cust_abc123",
  "tier": "premium",
  "exp": 1735308896,
  "iat": 1735305296
}
```

---

## REST API Endpoints

**Base URL**: `https://api.niyanta.example.com/v1`

### 1. Workload Management

#### 1.1 Submit Workload

**Endpoint**: `POST /workloads`

**Description**: Submit a new workload for execution

**Request Headers**:
```
Authorization: Bearer <api_key>
Content-Type: application/json
Idempotency-Key: <uuid> (optional)
```

**Request Body**:
```json
{
  "workload_type": "video_transcoding",
  "input_params": {
    "video_url": "s3://input/video123.mp4",
    "output_format": "mp4",
    "resolution": "1080p"
  },
  "priority": 10,
  "affinity_rules": {
    "hard_affinity": {
      "worker_tags": {
        "hardware": "gpu"
      }
    }
  },
  "timeout_seconds": 3600
}
```

**Request Schema**:
```go
type WorkloadSubmitRequest struct {
    WorkloadType    string                 `json:"workload_type" binding:"required"`
    InputParams     map[string]interface{} `json:"input_params" binding:"required"`
    Priority        int                    `json:"priority,omitempty"` // Default: 10
    AffinityRules   *AffinityRules         `json:"affinity_rules,omitempty"`
    TimeoutSeconds  int                    `json:"timeout_seconds,omitempty"`
}
```

**Response**: `202 Accepted`
```json
{
  "workload_id": "wl_abc123",
  "status": "PENDING",
  "created_at": "2025-10-27T12:34:56Z",
  "estimated_start_time": "2025-10-27T12:35:30Z"
}
```

**Error Responses**:
- `400 Bad Request`: Invalid input (e.g., unknown workload_type, invalid input_params)
- `401 Unauthorized`: Invalid or missing API key
- `429 Too Many Requests`: Rate limit exceeded
- `503 Service Unavailable`: System at capacity (queue full)

---

#### 1.2 Get Workload Status

**Endpoint**: `GET /workloads/{workload_id}`

**Description**: Retrieve current status and details of a workload

**Request Headers**:
```
Authorization: Bearer <api_key>
```

**Response**: `200 OK`
```json
{
  "workload_id": "wl_abc123",
  "customer_id": "cust_abc123",
  "workload_type": "video_transcoding",
  "status": "RUNNING",
  "priority": 10,
  "progress_percent": 45,
  "worker_id": "worker-5",
  "input_params": {
    "video_url": "s3://input/video123.mp4"
  },
  "result_data": null,
  "error_message": null,
  "retry_count": 0,
  "created_at": "2025-10-27T12:34:56Z",
  "scheduled_at": "2025-10-27T12:35:00Z",
  "started_at": "2025-10-27T12:35:10Z",
  "completed_at": null,
  "estimated_completion_time": "2025-10-27T13:10:00Z"
}
```

**Error Responses**:
- `404 Not Found`: Workload does not exist or belongs to different customer
- `401 Unauthorized`: Invalid API key

---

#### 1.3 List Workloads

**Endpoint**: `GET /workloads`

**Description**: List workloads for the authenticated customer with filtering and pagination

**Query Parameters**:
- `status` (optional): Filter by status (e.g., `RUNNING`, `COMPLETED`)
- `workload_type` (optional): Filter by workload type
- `limit` (optional): Number of results per page (default: 50, max: 100)
- `offset` (optional): Offset for pagination (default: 0)
- `order_by` (optional): Sort field (default: `created_at`)
- `order` (optional): Sort direction (`asc` or `desc`, default: `desc`)

**Example Request**:
```
GET /workloads?status=RUNNING&limit=20&offset=0&order_by=priority&order=desc
```

**Response**: `200 OK`
```json
{
  "workloads": [
    {
      "workload_id": "wl_abc123",
      "workload_type": "video_transcoding",
      "status": "RUNNING",
      "priority": 50,
      "progress_percent": 45,
      "created_at": "2025-10-27T12:34:56Z",
      "started_at": "2025-10-27T12:35:10Z"
    },
    {
      "workload_id": "wl_def456",
      "workload_type": "report_generation",
      "status": "RUNNING",
      "priority": 30,
      "progress_percent": 80,
      "created_at": "2025-10-27T11:20:00Z",
      "started_at": "2025-10-27T11:21:00Z"
    }
  ],
  "pagination": {
    "total": 156,
    "limit": 20,
    "offset": 0,
    "has_more": true
  }
}
```

---

#### 1.4 Cancel Workload

**Endpoint**: `POST /workloads/{workload_id}/cancel`

**Description**: Cancel a pending or running workload

**Request Headers**:
```
Authorization: Bearer <api_key>
Content-Type: application/json
```

**Request Body** (optional):
```json
{
  "reason": "User requested cancellation"
}
```

**Response**: `200 OK`
```json
{
  "workload_id": "wl_abc123",
  "status": "CANCELLED",
  "cancelled_at": "2025-10-27T12:45:00Z"
}
```

**Error Responses**:
- `400 Bad Request`: Workload already completed or cancelled
- `404 Not Found`: Workload does not exist

---

#### 1.5 Pause Workload

**Endpoint**: `POST /workloads/{workload_id}/pause`

**Description**: Pause a running workload (creates checkpoint and stops execution)

**Request Headers**:
```
Authorization: Bearer <api_key>
```

**Response**: `200 OK`
```json
{
  "workload_id": "wl_abc123",
  "status": "PAUSED",
  "paused_at": "2025-10-27T12:50:00Z",
  "checkpoint_sequence": 5
}
```

**Error Responses**:
- `400 Bad Request`: Workload not in RUNNING state
- `404 Not Found`: Workload does not exist

---

#### 1.6 Resume Workload

**Endpoint**: `POST /workloads/{workload_id}/resume`

**Description**: Resume a paused workload

**Request Headers**:
```
Authorization: Bearer <api_key>
```

**Response**: `202 Accepted`
```json
{
  "workload_id": "wl_abc123",
  "status": "PENDING",
  "checkpoint_sequence": 5
}
```

**Error Responses**:
- `400 Bad Request`: Workload not in PAUSED state

---

#### 1.7 Get Workload Logs

**Endpoint**: `GET /workloads/{workload_id}/logs`

**Description**: Retrieve execution logs for a workload

**Query Parameters**:
- `limit` (optional): Number of log lines (default: 100, max: 1000)
- `since` (optional): ISO 8601 timestamp to get logs after
- `until` (optional): ISO 8601 timestamp to get logs before

**Response**: `200 OK`
```json
{
  "workload_id": "wl_abc123",
  "logs": [
    {
      "timestamp": "2025-10-27T12:35:15Z",
      "level": "INFO",
      "message": "Starting video transcoding",
      "details": {}
    },
    {
      "timestamp": "2025-10-27T12:36:00Z",
      "level": "INFO",
      "message": "Processed 1000/10000 frames",
      "details": {
        "progress_percent": 10
      }
    }
  ],
  "has_more": false
}
```

---

#### 1.8 Get Workload Audit Trail

**Endpoint**: `GET /workloads/{workload_id}/audit`

**Description**: Retrieve lifecycle events for a workload

**Response**: `200 OK`
```json
{
  "workload_id": "wl_abc123",
  "events": [
    {
      "event_id": 1234,
      "event_type": "workload.created",
      "timestamp": "2025-10-27T12:34:56Z",
      "actor_type": "user",
      "actor_id": "user_789",
      "details": {}
    },
    {
      "event_id": 1235,
      "event_type": "workload.scheduled",
      "timestamp": "2025-10-27T12:35:00Z",
      "actor_type": "coordinator",
      "actor_id": "coordinator-1",
      "details": {
        "worker_id": "worker-5"
      }
    },
    {
      "event_id": 1236,
      "event_type": "workload.started",
      "timestamp": "2025-10-27T12:35:10Z",
      "actor_type": "worker",
      "actor_id": "worker-5",
      "details": {}
    }
  ]
}
```

---

### 2. Workload Configuration Management

#### 2.1 Create Workload Config

**Endpoint**: `POST /workload-configs`

**Description**: Register a new workload type configuration

**Request Body**:
```json
{
  "workload_type": "video_transcoding",
  "description": "Transcode videos to various formats",
  "config_json": {
    "default_codec": "h264",
    "default_resolution": "1080p"
  },
  "resource_limits": {
    "cpu_cores": 2,
    "memory_mb": 2048,
    "disk_mb": 10240
  },
  "checkpoint_interval_sec": 300,
  "max_retries": 3,
  "timeout_seconds": 7200
}
```

**Response**: `201 Created`
```json
{
  "config_id": "wlcfg_xyz789",
  "workload_type": "video_transcoding",
  "created_at": "2025-10-27T12:00:00Z"
}
```

---

#### 2.2 List Workload Configs

**Endpoint**: `GET /workload-configs`

**Description**: List all workload type configurations for the customer

**Response**: `200 OK`
```json
{
  "configs": [
    {
      "config_id": "wlcfg_xyz789",
      "workload_type": "video_transcoding",
      "description": "Transcode videos to various formats",
      "resource_limits": {
        "cpu_cores": 2,
        "memory_mb": 2048
      },
      "created_at": "2025-10-27T12:00:00Z"
    }
  ]
}
```

---

#### 2.3 Update Workload Config

**Endpoint**: `PUT /workload-configs/{config_id}`

**Description**: Update an existing workload configuration (does not affect running workloads)

**Request Body**: (same as create, all fields optional)

**Response**: `200 OK`

---

#### 2.4 Delete Workload Config

**Endpoint**: `DELETE /workload-configs/{config_id}`

**Description**: Delete a workload configuration (fails if workloads exist)

**Response**: `204 No Content`

**Error Responses**:
- `409 Conflict`: Cannot delete config with associated workloads

---

### 3. Customer & Account Management

#### 3.1 Get Customer Info

**Endpoint**: `GET /customer`

**Description**: Retrieve authenticated customer's account information

**Response**: `200 OK`
```json
{
  "customer_id": "cust_abc123",
  "name": "Acme Corp",
  "tier": "premium",
  "limits": {
    "max_concurrent_workloads": 500,
    "max_queue_depth": 2000,
    "rate_limit_per_sec": 50
  },
  "usage": {
    "current_running_workloads": 45,
    "current_queue_depth": 12,
    "workloads_today": 234
  },
  "created_at": "2024-01-15T09:00:00Z"
}
```

---

#### 3.2 Get Usage Statistics

**Endpoint**: `GET /customer/stats`

**Description**: Retrieve usage metrics and statistics

**Query Parameters**:
- `start_date` (required): ISO 8601 date
- `end_date` (required): ISO 8601 date
- `group_by` (optional): `day`, `hour`, `workload_type`

**Response**: `200 OK`
```json
{
  "start_date": "2025-10-01",
  "end_date": "2025-10-27",
  "summary": {
    "total_workloads": 5432,
    "completed_workloads": 5123,
    "failed_workloads": 234,
    "cancelled_workloads": 75,
    "total_execution_hours": 1234.5
  },
  "by_workload_type": [
    {
      "workload_type": "video_transcoding",
      "count": 3210,
      "avg_duration_seconds": 1800,
      "failure_rate": 0.05
    },
    {
      "workload_type": "report_generation",
      "count": 2222,
      "avg_duration_seconds": 600,
      "failure_rate": 0.02
    }
  ]
}
```

---

### 4. Worker Management (Admin Only)

#### 4.1 List Workers

**Endpoint**: `GET /admin/workers`

**Description**: List all registered workers (requires admin privileges)

**Query Parameters**:
- `status` (optional): Filter by status (HEALTHY, DRAINING, DEAD)

**Response**: `200 OK`
```json
{
  "workers": [
    {
      "worker_id": "worker-5",
      "status": "HEALTHY",
      "capabilities": ["video_transcoding", "image_processing"],
      "capacity_total": 10,
      "capacity_used": 3,
      "tags": {
        "region": "us-west-2",
        "hardware": "gpu"
      },
      "last_heartbeat": "2025-10-27T12:55:30Z",
      "running_workloads": ["wl_abc123", "wl_def456", "wl_ghi789"]
    }
  ]
}
```

---

#### 4.2 Drain Worker

**Endpoint**: `POST /admin/workers/{worker_id}/drain`

**Description**: Mark worker as draining (no new workloads, finish current ones)

**Response**: `200 OK`

---

#### 4.3 Remove Worker

**Endpoint**: `DELETE /admin/workers/{worker_id}`

**Description**: Forcefully remove a worker (redistributes all workloads)

**Response**: `204 No Content`

---

### 5. Health & Monitoring

#### 5.1 Health Check

**Endpoint**: `GET /health`

**Description**: Basic liveness check

**Response**: `200 OK`
```json
{
  "status": "healthy",
  "timestamp": "2025-10-27T12:56:00Z"
}
```

---

#### 5.2 Readiness Check

**Endpoint**: `GET /ready`

**Description**: Readiness for traffic (checks dependencies)

**Response**: `200 OK` (if ready) or `503 Service Unavailable` (if not ready)
```json
{
  "status": "ready",
  "dependencies": {
    "database": "connected",
    "broker": "connected",
    "leader_elected": true
  }
}
```

---

#### 5.3 Metrics

**Endpoint**: `GET /metrics`

**Description**: Prometheus-compatible metrics

**Response**: `200 OK` (text/plain)
```
# HELP niyanta_workload_submissions_total Total number of workload submissions
# TYPE niyanta_workload_submissions_total counter
niyanta_workload_submissions_total{customer="cust_abc123",status="COMPLETED"} 1234
niyanta_workload_submissions_total{customer="cust_abc123",status="FAILED"} 56

# HELP niyanta_workload_duration_seconds Workload execution duration
# TYPE niyanta_workload_duration_seconds histogram
niyanta_workload_duration_seconds_bucket{customer="cust_abc123",workload_type="video_transcoding",le="60"} 100
niyanta_workload_duration_seconds_bucket{customer="cust_abc123",workload_type="video_transcoding",le="300"} 500
...
```

---

## gRPC API

### Service Definition (Protocol Buffers)

**File**: `api/niyanta/v1/workload_service.proto`

```protobuf
syntax = "proto3";

package niyanta.v1;

import "google/protobuf/timestamp.proto";
import "google/protobuf/struct.proto";

service WorkloadService {
  rpc SubmitWorkload(SubmitWorkloadRequest) returns (SubmitWorkloadResponse);
  rpc GetWorkload(GetWorkloadRequest) returns (GetWorkloadResponse);
  rpc ListWorkloads(ListWorkloadsRequest) returns (ListWorkloadsResponse);
  rpc CancelWorkload(CancelWorkloadRequest) returns (CancelWorkloadResponse);
  rpc PauseWorkload(PauseWorkloadRequest) returns (PauseWorkloadResponse);
  rpc ResumeWorkload(ResumeWorkloadRequest) returns (ResumeWorkloadResponse);
  rpc StreamWorkloadStatus(StreamWorkloadStatusRequest) returns (stream WorkloadStatusUpdate);
}

message SubmitWorkloadRequest {
  string workload_type = 1;
  google.protobuf.Struct input_params = 2;
  int32 priority = 3;
  AffinityRules affinity_rules = 4;
  int32 timeout_seconds = 5;
}

message SubmitWorkloadResponse {
  string workload_id = 1;
  WorkloadStatus status = 2;
  google.protobuf.Timestamp created_at = 3;
  google.protobuf.Timestamp estimated_start_time = 4;
}

message GetWorkloadRequest {
  string workload_id = 1;
}

message GetWorkloadResponse {
  Workload workload = 1;
}

message Workload {
  string workload_id = 1;
  string customer_id = 2;
  string workload_type = 3;
  WorkloadStatus status = 4;
  int32 priority = 5;
  int32 progress_percent = 6;
  string worker_id = 7;
  google.protobuf.Struct input_params = 8;
  google.protobuf.Struct result_data = 9;
  string error_message = 10;
  int32 retry_count = 11;
  google.protobuf.Timestamp created_at = 12;
  google.protobuf.Timestamp scheduled_at = 13;
  google.protobuf.Timestamp started_at = 14;
  google.protobuf.Timestamp completed_at = 15;
}

enum WorkloadStatus {
  WORKLOAD_STATUS_UNSPECIFIED = 0;
  WORKLOAD_STATUS_PENDING = 1;
  WORKLOAD_STATUS_SCHEDULED = 2;
  WORKLOAD_STATUS_RUNNING = 3;
  WORKLOAD_STATUS_PAUSED = 4;
  WORKLOAD_STATUS_REDISTRIBUTING = 5;
  WORKLOAD_STATUS_COMPLETED = 6;
  WORKLOAD_STATUS_FAILED = 7;
  WORKLOAD_STATUS_CANCELLED = 8;
}

message AffinityRules {
  AffinityConstraint hard_affinity = 1;
  AffinityConstraint soft_affinity = 2;
  AffinityConstraint anti_affinity = 3;
}

message AffinityConstraint {
  string worker_id = 1;
  map<string, string> worker_tags = 2;
  repeated string workload_types = 3;
}

// Streaming RPC for real-time status updates
message StreamWorkloadStatusRequest {
  string workload_id = 1;
}

message WorkloadStatusUpdate {
  string workload_id = 1;
  WorkloadStatus status = 2;
  int32 progress_percent = 3;
  google.protobuf.Timestamp timestamp = 4;
  string message = 5;
}

// Additional messages for List, Cancel, Pause, Resume...
```

### gRPC Authentication

**Metadata**:
```
authorization: Bearer <api_key>
```

**Example (Go Client)**:
```go
ctx := metadata.AppendToOutgoingContext(ctx, "authorization", "Bearer "+apiKey)
resp, err := client.SubmitWorkload(ctx, req)
```

---

## Error Handling

### Error Response Format

**REST API**:
```json
{
  "error": {
    "code": "INVALID_INPUT",
    "message": "Workload type 'invalid_type' is not supported",
    "details": {
      "supported_types": ["video_transcoding", "report_generation"]
    },
    "request_id": "req_abc123"
  }
}
```

**gRPC API**: Use standard gRPC status codes with details

### Error Codes

| HTTP Status | Error Code | Description |
|-------------|------------|-------------|
| 400 | `INVALID_INPUT` | Request validation failed |
| 400 | `INVALID_STATE` | Operation not allowed in current state |
| 401 | `UNAUTHORIZED` | Invalid or missing authentication |
| 403 | `FORBIDDEN` | Insufficient permissions |
| 404 | `NOT_FOUND` | Resource does not exist |
| 409 | `CONFLICT` | Resource conflict (e.g., duplicate) |
| 429 | `RATE_LIMITED` | Rate limit exceeded |
| 503 | `SERVICE_UNAVAILABLE` | System overloaded or dependency unavailable |
| 500 | `INTERNAL_ERROR` | Unexpected server error |

### Retry Guidelines

**Retryable Errors** (with exponential backoff):
- `503 Service Unavailable`
- `429 Too Many Requests` (respect `Retry-After` header)
- Network timeouts

**Non-Retryable Errors**:
- `400 Bad Request`
- `401 Unauthorized`
- `403 Forbidden`
- `404 Not Found`

---

## Rate Limiting

### Rate Limit Headers

**Response Headers**:
```
X-RateLimit-Limit: 50
X-RateLimit-Remaining: 42
X-RateLimit-Reset: 1735308900
```

### Rate Limit Response

**Status**: `429 Too Many Requests`
```json
{
  "error": {
    "code": "RATE_LIMITED",
    "message": "Rate limit exceeded. Retry after 30 seconds.",
    "details": {
      "limit": 50,
      "window_seconds": 60,
      "retry_after": 30
    }
  }
}
```

### Rate Limit Tiers

| Tier | Requests/Second | Burst |
|------|-----------------|-------|
| Free | 1 | 5 |
| Standard | 10 | 50 |
| Premium | 50 | 200 |
| Enterprise | 200 | 1000 |

---

## Pagination

### Cursor-Based Pagination (Preferred)

**Request**:
```
GET /workloads?limit=50&cursor=eyJpZCI6Ind...
```

**Response**:
```json
{
  "workloads": [...],
  "pagination": {
    "next_cursor": "eyJpZCI6Ind...",
    "has_more": true
  }
}
```

### Offset-Based Pagination (Alternative)

**Request**:
```
GET /workloads?limit=50&offset=100
```

**Response**:
```json
{
  "workloads": [...],
  "pagination": {
    "total": 500,
    "limit": 50,
    "offset": 100,
    "has_more": true
  }
}
```

---

## WebSocket API (Optional)

### Real-Time Workload Updates

**Endpoint**: `wss://api.niyanta.example.com/v1/ws/workloads/{workload_id}`

**Authentication**: Via query parameter or initial message
```
wss://api.niyanta.example.com/v1/ws/workloads/wl_abc123?token=<api_key>
```

**Message Format** (JSON):
```json
{
  "type": "status_update",
  "workload_id": "wl_abc123",
  "status": "RUNNING",
  "progress_percent": 65,
  "timestamp": "2025-10-27T13:05:00Z",
  "message": "Processing frame 6500/10000"
}
```

**Update Types**:
- `status_update`: Status or progress changed
- `log_line`: New log line available
- `completed`: Workload finished (includes result_data)
- `failed`: Workload failed (includes error_message)

---

## API Versioning

### URL Versioning

**Format**: `/v{major}/...`

**Example**:
- v1: `/v1/workloads`
- v2: `/v2/workloads` (future)

### Deprecation Policy

1. New version announced 6 months before old version deprecation
2. Old version maintained for 12 months after new version release
3. Deprecation warnings in response headers:
   ```
   Deprecation: Sun, 01 Jun 2025 00:00:00 GMT
   Sunset: Sun, 01 Dec 2025 00:00:00 GMT
   Link: <https://docs.niyanta.example.com/migration/v2>; rel="successor-version"
   ```

---

## Request/Response Examples

### Example 1: Complete Workload Lifecycle

**1. Submit Workload**:
```bash
curl -X POST https://api.niyanta.example.com/v1/workloads \
  -H "Authorization: Bearer nyt_cust_abc123_..." \
  -H "Content-Type: application/json" \
  -d '{
    "workload_type": "video_transcoding",
    "input_params": {
      "video_url": "s3://input/video123.mp4"
    }
  }'

# Response:
{
  "workload_id": "wl_abc123",
  "status": "PENDING",
  "created_at": "2025-10-27T12:34:56Z"
}
```

**2. Poll for Status**:
```bash
curl https://api.niyanta.example.com/v1/workloads/wl_abc123 \
  -H "Authorization: Bearer nyt_cust_abc123_..."

# Response:
{
  "workload_id": "wl_abc123",
  "status": "RUNNING",
  "progress_percent": 45,
  "worker_id": "worker-5",
  "started_at": "2025-10-27T12:35:10Z"
}
```

**3. Wait for Completion** (or use WebSocket):
```bash
curl https://api.niyanta.example.com/v1/workloads/wl_abc123 \
  -H "Authorization: Bearer nyt_cust_abc123_..."

# Response:
{
  "workload_id": "wl_abc123",
  "status": "COMPLETED",
  "progress_percent": 100,
  "result_data": {
    "output_url": "s3://output/video123_transcoded.mp4"
  },
  "completed_at": "2025-10-27T13:00:00Z"
}
```

---

### Example 2: Handle Failure with Retry

**1. Submit Workload** (transient failure scenario):
```bash
# Same as Example 1
```

**2. Worker Fails**:
```bash
curl https://api.niyanta.example.com/v1/workloads/wl_abc123 \
  -H "Authorization: Bearer nyt_cust_abc123_..."

# Response:
{
  "workload_id": "wl_abc123",
  "status": "REDISTRIBUTING",
  "retry_count": 1,
  "error_message": "Worker heartbeat timeout"
}
```

**3. Automatic Redistribution**:
```bash
# After coordinator redistributes...
curl https://api.niyanta.example.com/v1/workloads/wl_abc123 \
  -H "Authorization: Bearer nyt_cust_abc123_..."

# Response:
{
  "workload_id": "wl_abc123",
  "status": "RUNNING",
  "worker_id": "worker-7",
  "retry_count": 1,
  "progress_percent": 45  # Resumed from checkpoint
}
```

---

## OpenAPI Specification

Full OpenAPI 3.0 specification available at:
- **File**: `api/openapi/niyanta-v1.yaml`
- **Interactive Docs**: `https://api.niyanta.example.com/docs`

---

## SDK Examples

### Go SDK

```go
package main

import (
    "context"
    "github.com/yourusername/niyanta-go-sdk"
)

func main() {
    client := niyanta.NewClient("nyt_cust_abc123_...")

    // Submit workload
    workload, err := client.Workloads.Submit(context.Background(), &niyanta.SubmitWorkloadRequest{
        WorkloadType: "video_transcoding",
        InputParams: map[string]interface{}{
            "video_url": "s3://input/video123.mp4",
        },
    })
    if err != nil {
        panic(err)
    }

    // Wait for completion
    finalWorkload, err := client.Workloads.WaitForCompletion(context.Background(), workload.ID)
    if err != nil {
        panic(err)
    }

    fmt.Println("Result:", finalWorkload.ResultData)
}
```

### Python SDK

```python
from niyanta import Client

client = Client(api_key="nyt_cust_abc123_...")

# Submit workload
workload = client.workloads.submit(
    workload_type="video_transcoding",
    input_params={"video_url": "s3://input/video123.mp4"}
)

# Wait for completion
final_workload = client.workloads.wait_for_completion(workload.id)
print(f"Result: {final_workload.result_data}")
```

---

## References

- [ARCHITECTURE.md](ARCHITECTURE.md) - System architecture overview
- [DATA_MODELS.md](DATA_MODELS.md) - Data schemas and message formats
- [IMPLEMENTATION_PLAN.md](IMPLEMENTATION_PLAN.md) - Phased delivery plan
