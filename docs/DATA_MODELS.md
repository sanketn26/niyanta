# Niyanta Data Models

**Version**: 1.0
**Last Updated**: 2025-10-27
**Status**: Design Phase

## Table of Contents
1. [Overview](#overview)
2. [Database Schema](#database-schema)
3. [State Storage Models](#state-storage-models)
4. [Broker Message Formats](#broker-message-formats)
5. [Go Struct Definitions](#go-struct-definitions)
6. [Data Validation Rules](#data-validation-rules)
7. [Indexing Strategy](#indexing-strategy)
8. [Data Retention Policies](#data-retention-policies)

---

## Overview

Niyanta uses PostgreSQL as the primary state storage. This document defines:
- Database table schemas
- Data types and constraints
- Message formats for broker communication
- Go struct representations
- Validation rules and invariants

---

## Database Schema

### Entity Relationship Diagram

```
┌─────────────────┐
│    customers    │
│─────────────────│
│ id (PK)         │
│ name            │
│ tier            │
│ created_at      │
└────────┬────────┘
         │
         │ 1:N
         │
┌────────┴────────────┐         ┌──────────────────┐
│    workload_configs │         │   workers        │
│─────────────────────│         │──────────────────│
│ id (PK)             │         │ id (PK)          │
│ customer_id (FK)    │         │ status           │
│ workload_type       │         │ last_heartbeat   │
│ config_json         │         │ capacity_total   │
│ created_at          │         │ capacity_used    │
└────────┬────────────┘         │ capabilities     │
         │                      └────────┬─────────┘
         │ 1:N                           │
         │                               │ 1:N
┌────────┴─────────────┐                │
│     workloads        │<───────────────┘
│──────────────────────│
│ id (PK)              │
│ customer_id (FK)     │
│ workload_config_id   │
│ worker_id (FK, null) │
│ status               │
│ priority             │
│ affinity_rules       │
│ created_at           │
│ scheduled_at         │
│ started_at           │
│ completed_at         │
└──────────┬───────────┘
           │
           │ 1:N
           │
┌──────────┴───────────┐       ┌──────────────────┐
│   checkpoints        │       │   audit_events   │
│──────────────────────│       │──────────────────│
│ id (PK)              │       │ id (PK)          │
│ workload_id (FK)     │       │ workload_id (FK) │
│ sequence_number      │       │ event_type       │
│ checkpoint_data      │       │ timestamp        │
│ created_at           │       │ details_json     │
└──────────────────────┘       └──────────────────┘
```

---

## State Storage Models

### 1. Customers Table

**Purpose**: Store customer (tenant) information and SLA tier

```sql
CREATE TABLE customers (
    id                  VARCHAR(64) PRIMARY KEY,
    name                VARCHAR(255) NOT NULL,
    tier                VARCHAR(32) NOT NULL CHECK (tier IN ('free', 'standard', 'premium', 'enterprise')),
    max_concurrent_workloads INT NOT NULL DEFAULT 100,
    max_queue_depth     INT NOT NULL DEFAULT 500,
    rate_limit_per_sec  INT NOT NULL DEFAULT 10,
    created_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_customers_tier ON customers(tier);
```

**Columns**:
- `id`: Unique customer identifier (e.g., `cust_abc123`)
- `name`: Customer display name
- `tier`: Service tier (determines SLA)
- `max_concurrent_workloads`: Maximum number of simultaneously running workloads
- `max_queue_depth`: Maximum pending workloads allowed
- `rate_limit_per_sec`: API submission rate limit
- `created_at`, `updated_at`: Audit timestamps

**Tier Characteristics**:
| Tier | Max Concurrent | Max Queue | Rate Limit | Priority Weight |
|------|----------------|-----------|------------|-----------------|
| free | 10 | 50 | 1/sec | 1 |
| standard | 100 | 500 | 10/sec | 10 |
| premium | 500 | 2000 | 50/sec | 50 |
| enterprise | 2000 | 10000 | 200/sec | 100 |

---

### 2. Workload Configs Table

**Purpose**: Store reusable workload type configurations per customer

```sql
CREATE TABLE workload_configs (
    id                  VARCHAR(64) PRIMARY KEY,
    customer_id         VARCHAR(64) NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    workload_type       VARCHAR(128) NOT NULL,
    description         TEXT,
    config_json         JSONB NOT NULL,
    resource_limits     JSONB NOT NULL DEFAULT '{"cpu_cores": 1, "memory_mb": 512, "disk_mb": 1024}',
    checkpoint_interval_sec INT NOT NULL DEFAULT 300,
    max_retries         INT NOT NULL DEFAULT 3,
    timeout_seconds     INT NOT NULL DEFAULT 3600,
    created_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    UNIQUE(customer_id, workload_type)
);

CREATE INDEX idx_workload_configs_customer ON workload_configs(customer_id);
CREATE INDEX idx_workload_configs_type ON workload_configs(workload_type);
```

**Columns**:
- `id`: Unique config identifier (e.g., `wlcfg_xyz789`)
- `customer_id`: Owner of this configuration
- `workload_type`: Type name (e.g., `video_transcoding`, `report_generation`)
- `config_json`: Workload-specific configuration (arbitrary JSON)
- `resource_limits`: CPU, memory, disk limits for this workload type
- `checkpoint_interval_sec`: How often to checkpoint (0 = manual only)
- `max_retries`: Retry attempts on transient failures
- `timeout_seconds`: Maximum execution time before forced termination

**Example `config_json`**:
```json
{
  "input_bucket": "s3://videos-input",
  "output_bucket": "s3://videos-output",
  "codec": "h264",
  "resolution": "1080p"
}
```

---

### 3. Workloads Table

**Purpose**: Store individual workload instances and their lifecycle state

```sql
CREATE TYPE workload_status AS ENUM (
    'PENDING',
    'SCHEDULED',
    'RUNNING',
    'PAUSED',
    'REDISTRIBUTING',
    'COMPLETED',
    'FAILED',
    'CANCELLED'
);

CREATE TABLE workloads (
    id                  VARCHAR(64) PRIMARY KEY,
    customer_id         VARCHAR(64) NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    workload_config_id  VARCHAR(64) NOT NULL REFERENCES workload_configs(id),
    worker_id           VARCHAR(64) REFERENCES workers(id) ON DELETE SET NULL,
    status              workload_status NOT NULL DEFAULT 'PENDING',
    priority            INT NOT NULL DEFAULT 10,
    affinity_rules      JSONB,
    input_params        JSONB NOT NULL,
    result_data         JSONB,
    error_message       TEXT,
    retry_count         INT NOT NULL DEFAULT 0,
    progress_percent    INT NOT NULL DEFAULT 0 CHECK (progress_percent >= 0 AND progress_percent <= 100),
    lease_expires_at    TIMESTAMP WITH TIME ZONE,
    generation          BIGINT NOT NULL DEFAULT 1,
    created_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    scheduled_at        TIMESTAMP WITH TIME ZONE,
    started_at          TIMESTAMP WITH TIME ZONE,
    completed_at        TIMESTAMP WITH TIME ZONE,
    updated_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_workloads_customer ON workloads(customer_id);
CREATE INDEX idx_workloads_status ON workloads(status);
CREATE INDEX idx_workloads_worker ON workloads(worker_id) WHERE worker_id IS NOT NULL;
CREATE INDEX idx_workloads_priority ON workloads(customer_id, status, priority DESC) WHERE status = 'PENDING';
CREATE INDEX idx_workloads_lease ON workloads(lease_expires_at) WHERE lease_expires_at IS NOT NULL;
```

**Columns**:
- `id`: Unique workload identifier (e.g., `wl_abc123`)
- `customer_id`: Owner of this workload
- `workload_config_id`: Reference to configuration template
- `worker_id`: Currently assigned worker (NULL if not assigned)
- `status`: Current lifecycle state (see state machine in ARCHITECTURE.md)
- `priority`: Scheduling priority (higher = more urgent); derived from customer tier + explicit priority
- `affinity_rules`: JSON describing placement constraints (see below)
- `input_params`: Workload-specific input data (e.g., file paths, parameters)
- `result_data`: Output data on completion
- `error_message`: Error details if FAILED
- `retry_count`: Number of retry attempts so far
- `progress_percent`: 0-100, updated by worker
- `lease_expires_at`: Worker must renew before this time or workload is redistributed
- `generation`: Incremented on each redistribution to prevent split-brain
- Timestamps for lifecycle events

**Example `affinity_rules`**:
```json
{
  "hard_affinity": {
    "worker_tags": ["gpu", "us-west-2"]
  },
  "soft_affinity": {
    "worker_id": "worker-5"
  },
  "anti_affinity": {
    "workload_types": ["cpu_intensive"]
  }
}
```

**Example `input_params`**:
```json
{
  "video_url": "s3://input/video123.mp4",
  "customer_metadata": {"user_id": "user_456"}
}
```

---

### 4. Workers Table

**Purpose**: Track registered workers and their health status

```sql
CREATE TYPE worker_status AS ENUM ('HEALTHY', 'DRAINING', 'DEAD');

CREATE TABLE workers (
    id                  VARCHAR(64) PRIMARY KEY,
    status              worker_status NOT NULL DEFAULT 'HEALTHY',
    capabilities        JSONB NOT NULL,
    capacity_total      INT NOT NULL,
    capacity_used       INT NOT NULL DEFAULT 0,
    tags                JSONB,
    last_heartbeat      TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    registered_at       TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_workers_status ON workers(status);
CREATE INDEX idx_workers_heartbeat ON workers(last_heartbeat) WHERE status = 'HEALTHY';
```

**Columns**:
- `id`: Unique worker identifier (e.g., `worker-abc123`)
- `status`: Current health status
  - `HEALTHY`: Accepting new workloads
  - `DRAINING`: Finishing current workloads, no new assignments
  - `DEAD`: Not responding, workloads should be redistributed
- `capabilities`: List of supported workload types
- `capacity_total`: Max concurrent workloads this worker can handle
- `capacity_used`: Current number of running workloads
- `tags`: Key-value pairs for affinity matching (e.g., region, hardware)
- `last_heartbeat`: Timestamp of last heartbeat message
- Audit timestamps

**Example `capabilities`**:
```json
{
  "supported_workload_types": [
    "video_transcoding",
    "image_processing",
    "report_generation"
  ],
  "version": "1.2.3"
}
```

**Example `tags`**:
```json
{
  "region": "us-west-2",
  "hardware": "gpu",
  "instance_type": "c5.2xlarge"
}
```

---

### 5. Checkpoints Table

**Purpose**: Store workload checkpoints for recovery

```sql
CREATE TABLE checkpoints (
    id                  BIGSERIAL PRIMARY KEY,
    workload_id         VARCHAR(64) NOT NULL REFERENCES workloads(id) ON DELETE CASCADE,
    sequence_number     INT NOT NULL,
    checkpoint_data     BYTEA NOT NULL,
    data_size_bytes     INT NOT NULL,
    created_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    UNIQUE(workload_id, sequence_number)
);

CREATE INDEX idx_checkpoints_workload ON checkpoints(workload_id, sequence_number DESC);
```

**Columns**:
- `id`: Auto-incrementing primary key
- `workload_id`: Which workload this checkpoint belongs to
- `sequence_number`: Monotonically increasing checkpoint number (1, 2, 3, ...)
- `checkpoint_data`: Binary checkpoint data (workload-specific format)
- `data_size_bytes`: Size of checkpoint_data (for metrics)
- `created_at`: When this checkpoint was created

**Constraints**:
- Max checkpoint size: 10MB (enforced at application layer)
- Keep last 10 checkpoints per workload (older ones deleted)

---

### 6. Audit Events Table

**Purpose**: Immutable log of all workload lifecycle events

```sql
CREATE TABLE audit_events (
    id                  BIGSERIAL PRIMARY KEY,
    workload_id         VARCHAR(64) NOT NULL REFERENCES workloads(id) ON DELETE CASCADE,
    customer_id         VARCHAR(64) NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    event_type          VARCHAR(64) NOT NULL,
    actor_type          VARCHAR(32) NOT NULL CHECK (actor_type IN ('coordinator', 'worker', 'system', 'user')),
    actor_id            VARCHAR(64),
    details_json        JSONB,
    timestamp           TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_audit_events_workload ON audit_events(workload_id, timestamp DESC);
CREATE INDEX idx_audit_events_customer ON audit_events(customer_id, timestamp DESC);
CREATE INDEX idx_audit_events_type ON audit_events(event_type, timestamp DESC);
```

**Columns**:
- `id`: Auto-incrementing primary key
- `workload_id`: Which workload this event relates to
- `customer_id`: For cross-customer queries and billing
- `event_type`: Event name (e.g., `workload.created`, `workload.scheduled`, `workload.completed`)
- `actor_type`: Who triggered this event
- `actor_id`: Specific actor (e.g., `worker-5`, `user_789`)
- `details_json`: Additional context (arbitrary JSON)
- `timestamp`: When event occurred

**Common Event Types**:
- `workload.created`
- `workload.scheduled`
- `workload.started`
- `workload.checkpoint_created`
- `workload.progress_updated`
- `workload.paused`
- `workload.resumed`
- `workload.failed`
- `workload.completed`
- `workload.cancelled`
- `workload.redistributed`
- `worker.registered`
- `worker.heartbeat_missed`
- `worker.marked_dead`

**Example Event**:
```json
{
  "event_type": "workload.redistributed",
  "details_json": {
    "from_worker": "worker-3",
    "to_worker": "worker-7",
    "reason": "heartbeat_timeout",
    "checkpoint_sequence": 5
  }
}
```

---

### 7. Coordinator State Table (Leader Election)

**Purpose**: Manage coordinator leader election (if not using etcd)

```sql
CREATE TABLE coordinator_state (
    key                 VARCHAR(64) PRIMARY KEY,
    value               TEXT NOT NULL,
    expires_at          TIMESTAMP WITH TIME ZONE NOT NULL,
    updated_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

-- Pre-insert leader lock key
INSERT INTO coordinator_state (key, value, expires_at)
VALUES ('leader_lock', '', NOW())
ON CONFLICT (key) DO NOTHING;
```

**Usage**:
```sql
-- Try to acquire leader lock
UPDATE coordinator_state
SET value = 'coordinator-abc123',
    expires_at = NOW() + INTERVAL '10 seconds',
    updated_at = NOW()
WHERE key = 'leader_lock'
  AND (expires_at < NOW() OR value = 'coordinator-abc123')
RETURNING value;
```

---

## Broker Message Formats

### Message Envelope

All broker messages use this common structure:

```go
type Message struct {
    MessageID     string                 `json:"message_id"`
    Type          string                 `json:"type"`
    Timestamp     time.Time              `json:"timestamp"`
    SenderID      string                 `json:"sender_id"`
    SenderType    string                 `json:"sender_type"` // "coordinator" or "worker"
    CorrelationID string                 `json:"correlation_id,omitempty"`
    Payload       map[string]interface{} `json:"payload"`
}
```

---

### 1. Worker Registration Message

**Channel**: `coordinator.events`
**Type**: `worker.register`
**Direction**: Worker → Coordinator

```go
type WorkerRegistrationPayload struct {
    WorkerID     string   `json:"worker_id"`
    Capabilities []string `json:"capabilities"`
    CapacityTotal int     `json:"capacity_total"`
    Tags         map[string]string `json:"tags"`
    Version      string   `json:"version"`
}
```

**Example**:
```json
{
  "message_id": "msg_abc123",
  "type": "worker.register",
  "timestamp": "2025-10-27T12:34:56Z",
  "sender_id": "worker-5",
  "sender_type": "worker",
  "payload": {
    "worker_id": "worker-5",
    "capabilities": ["video_transcoding", "image_processing"],
    "capacity_total": 10,
    "tags": {
      "region": "us-west-2",
      "hardware": "gpu"
    },
    "version": "1.0.0"
  }
}
```

---

### 2. Workload Assignment Message

**Channel**: `worker.{worker_id}.commands`
**Type**: `workload.assign`
**Direction**: Coordinator → Worker

```go
type WorkloadAssignmentPayload struct {
    WorkloadID       string                 `json:"workload_id"`
    WorkloadType     string                 `json:"workload_type"`
    ConfigJSON       map[string]interface{} `json:"config_json"`
    InputParams      map[string]interface{} `json:"input_params"`
    CheckpointData   []byte                 `json:"checkpoint_data,omitempty"` // base64 encoded
    Generation       int64                  `json:"generation"`
    LeaseSeconds     int                    `json:"lease_seconds"`
}
```

**Example**:
```json
{
  "message_id": "msg_xyz789",
  "type": "workload.assign",
  "timestamp": "2025-10-27T12:35:00Z",
  "sender_id": "coordinator-1",
  "sender_type": "coordinator",
  "payload": {
    "workload_id": "wl_abc123",
    "workload_type": "video_transcoding",
    "config_json": {
      "codec": "h264",
      "resolution": "1080p"
    },
    "input_params": {
      "video_url": "s3://input/video123.mp4"
    },
    "generation": 1,
    "lease_seconds": 60
  }
}
```

---

### 3. Heartbeat Message

**Channel**: `worker.{worker_id}.status`
**Type**: `worker.heartbeat`
**Direction**: Worker → Coordinator

```go
type HeartbeatPayload struct {
    WorkerID        string   `json:"worker_id"`
    Status          string   `json:"status"` // "HEALTHY", "DRAINING"
    CapacityUsed    int      `json:"capacity_used"`
    CapacityTotal   int      `json:"capacity_total"`
    RunningWorkloads []string `json:"running_workloads"`
}
```

**Example**:
```json
{
  "message_id": "msg_hb001",
  "type": "worker.heartbeat",
  "timestamp": "2025-10-27T12:35:30Z",
  "sender_id": "worker-5",
  "sender_type": "worker",
  "payload": {
    "worker_id": "worker-5",
    "status": "HEALTHY",
    "capacity_used": 3,
    "capacity_total": 10,
    "running_workloads": ["wl_abc123", "wl_def456", "wl_ghi789"]
  }
}
```

---

### 4. Progress Update Message

**Channel**: `worker.{worker_id}.status`
**Type**: `workload.progress`
**Direction**: Worker → Coordinator

```go
type ProgressUpdatePayload struct {
    WorkloadID       string `json:"workload_id"`
    ProgressPercent  int    `json:"progress_percent"`
    CheckpointSeq    int    `json:"checkpoint_sequence,omitempty"`
    Message          string `json:"message,omitempty"`
}
```

**Example**:
```json
{
  "message_id": "msg_prog001",
  "type": "workload.progress",
  "timestamp": "2025-10-27T12:40:00Z",
  "sender_id": "worker-5",
  "sender_type": "worker",
  "payload": {
    "workload_id": "wl_abc123",
    "progress_percent": 45,
    "checkpoint_sequence": 3,
    "message": "Processed 4500/10000 frames"
  }
}
```

---

### 5. Workload Completion Message

**Channel**: `coordinator.events`
**Type**: `workload.completed` or `workload.failed`
**Direction**: Worker → Coordinator

```go
type WorkloadCompletionPayload struct {
    WorkloadID   string                 `json:"workload_id"`
    Status       string                 `json:"status"` // "COMPLETED" or "FAILED"
    ResultData   map[string]interface{} `json:"result_data,omitempty"`
    ErrorMessage string                 `json:"error_message,omitempty"`
    ErrorType    string                 `json:"error_type,omitempty"` // "transient", "permanent"
}
```

**Example (Success)**:
```json
{
  "message_id": "msg_comp001",
  "type": "workload.completed",
  "timestamp": "2025-10-27T13:00:00Z",
  "sender_id": "worker-5",
  "sender_type": "worker",
  "payload": {
    "workload_id": "wl_abc123",
    "status": "COMPLETED",
    "result_data": {
      "output_url": "s3://output/video123_transcoded.mp4",
      "duration_seconds": 1500
    }
  }
}
```

**Example (Failure)**:
```json
{
  "message_id": "msg_fail001",
  "type": "workload.failed",
  "timestamp": "2025-10-27T13:00:00Z",
  "sender_id": "worker-5",
  "sender_type": "worker",
  "payload": {
    "workload_id": "wl_abc123",
    "status": "FAILED",
    "error_message": "Input file not found: s3://input/video123.mp4",
    "error_type": "permanent"
  }
}
```

---

### 6. Workload Cancellation Message

**Channel**: `worker.{worker_id}.commands`
**Type**: `workload.cancel`
**Direction**: Coordinator → Worker

```go
type WorkloadCancellationPayload struct {
    WorkloadID string `json:"workload_id"`
    Reason     string `json:"reason"`
}
```

**Example**:
```json
{
  "message_id": "msg_cancel001",
  "type": "workload.cancel",
  "timestamp": "2025-10-27T12:45:00Z",
  "sender_id": "coordinator-1",
  "sender_type": "coordinator",
  "payload": {
    "workload_id": "wl_abc123",
    "reason": "User requested cancellation"
  }
}
```

---

## Go Struct Definitions

### Core Domain Models

```go
package models

import (
    "time"
    "encoding/json"
)

// WorkloadStatus represents the lifecycle state of a workload
type WorkloadStatus string

const (
    WorkloadStatusPending        WorkloadStatus = "PENDING"
    WorkloadStatusScheduled      WorkloadStatus = "SCHEDULED"
    WorkloadStatusRunning        WorkloadStatus = "RUNNING"
    WorkloadStatusPaused         WorkloadStatus = "PAUSED"
    WorkloadStatusRedistributing WorkloadStatus = "REDISTRIBUTING"
    WorkloadStatusCompleted      WorkloadStatus = "COMPLETED"
    WorkloadStatusFailed         WorkloadStatus = "FAILED"
    WorkloadStatusCancelled      WorkloadStatus = "CANCELLED"
)

// WorkerStatus represents the health state of a worker
type WorkerStatus string

const (
    WorkerStatusHealthy  WorkerStatus = "HEALTHY"
    WorkerStatusDraining WorkerStatus = "DRAINING"
    WorkerStatusDead     WorkerStatus = "DEAD"
)

// CustomerTier represents the service level tier
type CustomerTier string

const (
    TierFree       CustomerTier = "free"
    TierStandard   CustomerTier = "standard"
    TierPremium    CustomerTier = "premium"
    TierEnterprise CustomerTier = "enterprise"
)

// Customer represents a tenant in the system
type Customer struct {
    ID                     string       `json:"id" db:"id"`
    Name                   string       `json:"name" db:"name"`
    Tier                   CustomerTier `json:"tier" db:"tier"`
    MaxConcurrentWorkloads int          `json:"max_concurrent_workloads" db:"max_concurrent_workloads"`
    MaxQueueDepth          int          `json:"max_queue_depth" db:"max_queue_depth"`
    RateLimitPerSec        int          `json:"rate_limit_per_sec" db:"rate_limit_per_sec"`
    CreatedAt              time.Time    `json:"created_at" db:"created_at"`
    UpdatedAt              time.Time    `json:"updated_at" db:"updated_at"`
}

// WorkloadConfig represents a reusable workload type configuration
type WorkloadConfig struct {
    ID                    string          `json:"id" db:"id"`
    CustomerID            string          `json:"customer_id" db:"customer_id"`
    WorkloadType          string          `json:"workload_type" db:"workload_type"`
    Description           string          `json:"description" db:"description"`
    ConfigJSON            json.RawMessage `json:"config_json" db:"config_json"`
    ResourceLimits        ResourceLimits  `json:"resource_limits" db:"resource_limits"`
    CheckpointIntervalSec int             `json:"checkpoint_interval_sec" db:"checkpoint_interval_sec"`
    MaxRetries            int             `json:"max_retries" db:"max_retries"`
    TimeoutSeconds        int             `json:"timeout_seconds" db:"timeout_seconds"`
    CreatedAt             time.Time       `json:"created_at" db:"created_at"`
    UpdatedAt             time.Time       `json:"updated_at" db:"updated_at"`
}

// ResourceLimits defines resource constraints for a workload
type ResourceLimits struct {
    CPUCores int `json:"cpu_cores"`
    MemoryMB int `json:"memory_mb"`
    DiskMB   int `json:"disk_mb"`
}

// Workload represents an individual workload instance
type Workload struct {
    ID               string          `json:"id" db:"id"`
    CustomerID       string          `json:"customer_id" db:"customer_id"`
    WorkloadConfigID string          `json:"workload_config_id" db:"workload_config_id"`
    WorkerID         *string         `json:"worker_id,omitempty" db:"worker_id"`
    Status           WorkloadStatus  `json:"status" db:"status"`
    Priority         int             `json:"priority" db:"priority"`
    AffinityRules    *AffinityRules  `json:"affinity_rules,omitempty" db:"affinity_rules"`
    InputParams      json.RawMessage `json:"input_params" db:"input_params"`
    ResultData       json.RawMessage `json:"result_data,omitempty" db:"result_data"`
    ErrorMessage     string          `json:"error_message,omitempty" db:"error_message"`
    RetryCount       int             `json:"retry_count" db:"retry_count"`
    ProgressPercent  int             `json:"progress_percent" db:"progress_percent"`
    LeaseExpiresAt   *time.Time      `json:"lease_expires_at,omitempty" db:"lease_expires_at"`
    Generation       int64           `json:"generation" db:"generation"`
    CreatedAt        time.Time       `json:"created_at" db:"created_at"`
    ScheduledAt      *time.Time      `json:"scheduled_at,omitempty" db:"scheduled_at"`
    StartedAt        *time.Time      `json:"started_at,omitempty" db:"started_at"`
    CompletedAt      *time.Time      `json:"completed_at,omitempty" db:"completed_at"`
    UpdatedAt        time.Time       `json:"updated_at" db:"updated_at"`
}

// AffinityRules defines workload placement constraints
type AffinityRules struct {
    HardAffinity *AffinityConstraint `json:"hard_affinity,omitempty"`
    SoftAffinity *AffinityConstraint `json:"soft_affinity,omitempty"`
    AntiAffinity *AffinityConstraint `json:"anti_affinity,omitempty"`
}

// AffinityConstraint specifies placement rules
type AffinityConstraint struct {
    WorkerID       string            `json:"worker_id,omitempty"`
    WorkerTags     map[string]string `json:"worker_tags,omitempty"`
    WorkloadTypes  []string          `json:"workload_types,omitempty"`
}

// Worker represents a workload execution node
type Worker struct {
    ID             string         `json:"id" db:"id"`
    Status         WorkerStatus   `json:"status" db:"status"`
    Capabilities   []string       `json:"capabilities" db:"capabilities"`
    CapacityTotal  int            `json:"capacity_total" db:"capacity_total"`
    CapacityUsed   int            `json:"capacity_used" db:"capacity_used"`
    Tags           map[string]string `json:"tags" db:"tags"`
    LastHeartbeat  time.Time      `json:"last_heartbeat" db:"last_heartbeat"`
    RegisteredAt   time.Time      `json:"registered_at" db:"registered_at"`
    UpdatedAt      time.Time      `json:"updated_at" db:"updated_at"`
}

// Checkpoint represents a saved workload state
type Checkpoint struct {
    ID              int64     `json:"id" db:"id"`
    WorkloadID      string    `json:"workload_id" db:"workload_id"`
    SequenceNumber  int       `json:"sequence_number" db:"sequence_number"`
    CheckpointData  []byte    `json:"checkpoint_data" db:"checkpoint_data"`
    DataSizeBytes   int       `json:"data_size_bytes" db:"data_size_bytes"`
    CreatedAt       time.Time `json:"created_at" db:"created_at"`
}

// AuditEvent represents an immutable lifecycle event
type AuditEvent struct {
    ID          int64           `json:"id" db:"id"`
    WorkloadID  string          `json:"workload_id" db:"workload_id"`
    CustomerID  string          `json:"customer_id" db:"customer_id"`
    EventType   string          `json:"event_type" db:"event_type"`
    ActorType   string          `json:"actor_type" db:"actor_type"`
    ActorID     string          `json:"actor_id" db:"actor_id"`
    DetailsJSON json.RawMessage `json:"details_json" db:"details_json"`
    Timestamp   time.Time       `json:"timestamp" db:"timestamp"`
}
```

---

## Data Validation Rules

### Workload Validation

```go
func (w *Workload) Validate() error {
    if w.ID == "" || !strings.HasPrefix(w.ID, "wl_") {
        return errors.New("invalid workload ID")
    }
    if w.CustomerID == "" {
        return errors.New("customer_id required")
    }
    if w.WorkloadConfigID == "" {
        return errors.New("workload_config_id required")
    }
    if w.Priority < 0 || w.Priority > 1000 {
        return errors.New("priority must be between 0 and 1000")
    }
    if w.ProgressPercent < 0 || w.ProgressPercent > 100 {
        return errors.New("progress_percent must be between 0 and 100")
    }
    return nil
}
```

### Checkpoint Validation

```go
const MaxCheckpointSize = 10 * 1024 * 1024 // 10MB

func (c *Checkpoint) Validate() error {
    if c.WorkloadID == "" {
        return errors.New("workload_id required")
    }
    if c.SequenceNumber < 1 {
        return errors.New("sequence_number must be positive")
    }
    if len(c.CheckpointData) == 0 {
        return errors.New("checkpoint_data required")
    }
    if len(c.CheckpointData) > MaxCheckpointSize {
        return fmt.Errorf("checkpoint_data exceeds max size of %d bytes", MaxCheckpointSize)
    }
    return nil
}
```

---

## Indexing Strategy

### Critical Indexes for Performance

1. **Workload Scheduling Query**:
```sql
-- Coordinator queries for workloads to schedule
EXPLAIN SELECT * FROM workloads
WHERE customer_id = $1
  AND status = 'PENDING'
ORDER BY priority DESC, created_at ASC
LIMIT 100;
```
**Index**: `idx_workloads_priority` (composite on customer_id, status, priority DESC)

2. **Worker Lease Expiration Check**:
```sql
-- Coordinator finds expired leases
SELECT * FROM workloads
WHERE status = 'RUNNING'
  AND lease_expires_at < NOW();
```
**Index**: `idx_workloads_lease` (on lease_expires_at with partial index)

3. **Audit Log Queries**:
```sql
-- User queries workload history
SELECT * FROM audit_events
WHERE workload_id = $1
ORDER BY timestamp DESC
LIMIT 50;
```
**Index**: `idx_audit_events_workload` (composite on workload_id, timestamp DESC)

---

## Data Retention Policies

### Cleanup Rules

| Table | Retention Policy | Cleanup Method |
|-------|------------------|----------------|
| `workloads` | Keep COMPLETED/FAILED for 90 days | Nightly job archives to cold storage |
| `checkpoints` | Keep last 10 per workload | On new checkpoint, delete old ones |
| `audit_events` | Keep for 1 year | Monthly partition drops |
| `workers` | Delete DEAD workers after 7 days | Nightly cleanup job |

### Archival Strategy

**Completed Workloads**:
- After 90 days, move to `workloads_archive` table (same schema)
- Archive table uses table partitioning by completion month
- Audit events remain in primary table (longer retention)

**Example Cleanup Job** (run daily):
```sql
-- Archive old workloads
INSERT INTO workloads_archive
SELECT * FROM workloads
WHERE status IN ('COMPLETED', 'FAILED', 'CANCELLED')
  AND completed_at < NOW() - INTERVAL '90 days';

DELETE FROM workloads
WHERE status IN ('COMPLETED', 'FAILED', 'CANCELLED')
  AND completed_at < NOW() - INTERVAL '90 days';

-- Delete old checkpoints (keep last 10)
DELETE FROM checkpoints
WHERE id IN (
  SELECT id FROM checkpoints c1
  WHERE (
    SELECT COUNT(*) FROM checkpoints c2
    WHERE c2.workload_id = c1.workload_id
      AND c2.sequence_number >= c1.sequence_number
  ) > 10
);

-- Delete old dead workers
DELETE FROM workers
WHERE status = 'DEAD'
  AND updated_at < NOW() - INTERVAL '7 days';
```

---

## State Consistency Guarantees

### Transaction Boundaries

**Workload State Transitions** (atomic):
```go
tx.Begin()
  - Update workload status
  - Insert audit event
  - Update worker capacity_used (if applicable)
tx.Commit()
```

**Checkpoint Creation** (atomic):
```go
tx.Begin()
  - Insert checkpoint
  - Update workload progress_percent
  - Insert audit event
tx.Commit()
```

### Idempotency Keys

For API requests that mutate state, use idempotency keys to prevent duplicate submissions:

```go
type WorkloadSubmission struct {
    IdempotencyKey string `json:"idempotency_key"` // UUID generated by client
    // ... other fields
}

// In database, track processed idempotency keys
CREATE TABLE idempotency_keys (
    key         VARCHAR(64) PRIMARY KEY,
    workload_id VARCHAR(64) NOT NULL,
    created_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_idempotency_created ON idempotency_keys(created_at);
```

---

## Schema Migration Strategy

Use **golang-migrate** or similar tool for versioned migrations.

**Example Migration (001_initial_schema.up.sql)**:
```sql
BEGIN;

CREATE TABLE customers ( ... );
CREATE TABLE workload_configs ( ... );
-- ... all tables

CREATE INDEX idx_workloads_priority ON workloads(...);
-- ... all indexes

COMMIT;
```

**Rollback (001_initial_schema.down.sql)**:
```sql
BEGIN;

DROP TABLE IF EXISTS audit_events CASCADE;
DROP TABLE IF EXISTS checkpoints CASCADE;
DROP TABLE IF EXISTS workloads CASCADE;
DROP TABLE IF EXISTS workers CASCADE;
DROP TABLE IF EXISTS workload_configs CASCADE;
DROP TABLE IF EXISTS customers CASCADE;

COMMIT;
```

---

## Future Considerations

1. **Sharding**: If single PostgreSQL instance becomes bottleneck, shard by `customer_id`
2. **Large Checkpoints**: Store checkpoints > 10MB in S3, keep reference in database
3. **Metrics Aggregation**: Pre-aggregate common queries (per-customer stats) in materialized views
4. **Read Replicas**: Offload audit log queries and dashboard queries to read replicas

---

## References

- [ARCHITECTURE.md](ARCHITECTURE.md) - System architecture overview
- [API_SPEC.md](API_SPEC.md) - API contracts
- [IMPLEMENTATION_PLAN.md](IMPLEMENTATION_PLAN.md) - Phased delivery plan
