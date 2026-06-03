# Niyanta Platform Data Models

**Version**: 2.0
**Last Updated**: 2026-06-03
**Status**: Design Phase

> **v2.0 supersedes the v1 workload model.** Niyanta is a composable activity runner; its canonical persisted state is the **activity** and its execution record (attempts, call log, signals), per [adr/ADR-005-activity-execution-model.md](adr/ADR-005-activity-execution-model.md). The earlier `workloads` / `workload_configs` / `retry_count` schema is replaced. A migration map from the old vocabulary is in the appendix.

This document defines the **platform** schema only — the engine's state for running activities and resuming them after failure. It is application-agnostic: it knows about activities, attempts, the call log, signals, checkpoints, workers, tenants, and audit. It does **not** know about connectors, runbooks, or any app's domain. Apps add their own tables (see [../apps/dataconnector/DATA_CONNECTOR_SPEC.md](../apps/dataconnector/DATA_CONNECTOR_SPEC.md) §Storage Extensions and [../apps/llmops/LLMOPS_SPEC.md](../apps/llmops/LLMOPS_SPEC.md) §Storage Extensions).

## Table of Contents
1. [Overview](#overview)
2. [Database Schema](#database-schema)
3. [State Storage Models](#state-storage-models)
4. [Broker Message Formats](#broker-message-formats)
5. [Go Struct Definitions](#go-struct-definitions)
6. [Data Validation Rules](#data-validation-rules)
7. [Indexing Strategy](#indexing-strategy)
8. [Data Retention Policies](#data-retention-policies)
9. [Schema Migration Strategy](#schema-migration-strategy)
10. [Appendix: v1 → v2 Migration Map](#appendix-v1--v2-migration-map)

---

## Overview

Niyanta uses PostgreSQL as the primary state storage. The model is built around four core platform tables:

- **`activities`** — the logical unit of work and its current lifecycle state.
- **`activity_attempts`** — one row per physical execution attempt (replaces the `retry_count` integer); the audit trail of retries.
- **`activity_calls`** — the durable **call log** of `RunChild` / `Sleep` / `AwaitSignal` invocations, used for replay-based resume.
- **`signals`** — the durable per-activity mailbox for external events.

Supporting tables: `customers` (tenants), `workers`, `checkpoints` (intra-attempt heartbeat state), `audit_events`, `coordinator_state`, `idempotency_keys`.

This document defines table schemas, broker message formats, Go structs, validation rules, and invariants.

---

## Database Schema

### Entity Relationship Diagram

```
┌─────────────────┐
│    customers    │  (tenant scope on every activity)
│─────────────────│
│ id (PK)         │
│ tier            │
└────────┬────────┘
         │ 1:N
         ▼
┌──────────────────────┐        ┌──────────────────┐
│     activities       │        │   workers        │
│──────────────────────│        │──────────────────│
│ id (PK)              │        │ id (PK)          │
│ customer_id (FK)     │        │ status           │
│ activity_type        │        │ capabilities     │
│ parent_activity_id   │◄─┐     │ capacity_*       │
│ status               │  │self │ last_heartbeat   │
│ priority             │  │1:N  └──────────────────┘
│ generation           │  │ (parent → child via RunChild)
└───┬───────┬──────┬───┘──┘
    │1:N    │1:N   │1:N
    ▼       ▼      ▼
┌─────────┐ ┌──────────────┐ ┌──────────────┐
│activity_│ │activity_calls│ │  signals     │
│attempts │ │ (call log)   │ │ (mailbox)    │
│─────────│ │──────────────│ │──────────────│
│id (PK)  │ │id (PK)       │ │id (PK)       │
│activity │ │activity_id   │ │activity_id   │
│ _id     │ │call_index    │ │name          │
│attempt# │ │call_type     │ │payload       │
│worker_id│ │child_act_id  │ │delivered     │
│error    │ │result_json   │ │created_at    │
└────┬────┘ └──────────────┘ └──────────────┘
     │1:N
     ▼
┌──────────────┐      ┌──────────────────┐
│  checkpoints │      │   audit_events   │
│ (heartbeat)  │      │──────────────────│
│──────────────│      │ id (PK)          │
│ id (PK)      │      │ activity_id (FK) │
│ attempt_id   │      │ event_type       │
│ seq_number   │      │ actor_type       │
│ data         │      │ details_json     │
└──────────────┘      └──────────────────┘
```

---

## State Storage Models

### 1. Customers Table

**Purpose**: Store tenant information and SLA tier. Unchanged in v2 except for terminology (concurrency limits now count activities).

```sql
CREATE TABLE customers (
    id                  VARCHAR(64) PRIMARY KEY,
    name                VARCHAR(255) NOT NULL,
    tier                VARCHAR(32) NOT NULL CHECK (tier IN ('free', 'standard', 'premium', 'enterprise')),
    max_concurrent_activities INT NOT NULL DEFAULT 100,
    max_queue_depth     INT NOT NULL DEFAULT 500,
    rate_limit_per_sec  INT NOT NULL DEFAULT 10,
    created_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_customers_tier ON customers(tier);
```

**Tier Characteristics**:

| Tier | Max Concurrent Activities | Max Queue | Rate Limit | Priority Weight |
|------|---------------------------|-----------|------------|-----------------|
| free | 10 | 50 | 1/sec | 1 |
| standard | 100 | 500 | 10/sec | 10 |
| premium | 500 | 2000 | 50/sec | 50 |
| enterprise | 2000 | 10000 | 200/sec | 100 |

---

### 2. Activities Table

**Purpose**: Store each logical unit of work and its lifecycle state. An activity may be a top-level activity or a child dispatched by a parent's `RunChild` call.

```sql
CREATE TYPE activity_status AS ENUM (
    'PENDING',
    'SCHEDULED',
    'RUNNING',
    'SUSPENDED',     -- parked at a RunChild / Sleep / AwaitSignal durable suspend point
    'PAUSED',
    'REDISTRIBUTING',
    'COMPLETED',
    'FAILED',
    'CANCELLED'
);

CREATE TABLE activities (
    id                  VARCHAR(64) PRIMARY KEY,
    customer_id         VARCHAR(64) NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    activity_type       VARCHAR(128) NOT NULL,
    parent_activity_id  VARCHAR(64) REFERENCES activities(id) ON DELETE CASCADE,
    root_activity_id    VARCHAR(64),                 -- top of the composition tree (for tracing/queries)
    worker_id           VARCHAR(64) REFERENCES workers(id) ON DELETE SET NULL,
    status              activity_status NOT NULL DEFAULT 'PENDING',
    priority            INT NOT NULL DEFAULT 10,
    affinity_rules      JSONB,
    input_json          JSONB NOT NULL,              -- opaque to the engine; meaningful to the activity's app
    result_json         JSONB,
    error_json          JSONB,                       -- {message, type: transient|permanent, ...}
    retry_policy_json   JSONB NOT NULL DEFAULT '{"max_attempts": 3, "backoff": "exponential"}',
    lease_expires_at    TIMESTAMP WITH TIME ZONE,
    generation          BIGINT NOT NULL DEFAULT 1,   -- fencing token, bumped on redistribution
    suspend_reason      VARCHAR(32),                 -- run_child | sleep | await_signal (when SUSPENDED)
    wake_at             TIMESTAMP WITH TIME ZONE,     -- for Sleep / AwaitSignal timeout
    created_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    scheduled_at        TIMESTAMP WITH TIME ZONE,
    started_at          TIMESTAMP WITH TIME ZONE,
    completed_at        TIMESTAMP WITH TIME ZONE,
    updated_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_activities_customer       ON activities(customer_id);
CREATE INDEX idx_activities_status         ON activities(status);
CREATE INDEX idx_activities_worker         ON activities(worker_id) WHERE worker_id IS NOT NULL;
CREATE INDEX idx_activities_parent         ON activities(parent_activity_id) WHERE parent_activity_id IS NOT NULL;
CREATE INDEX idx_activities_root           ON activities(root_activity_id);
CREATE INDEX idx_activities_priority       ON activities(customer_id, status, priority DESC) WHERE status = 'PENDING';
CREATE INDEX idx_activities_lease          ON activities(lease_expires_at) WHERE lease_expires_at IS NOT NULL;
CREATE INDEX idx_activities_wake           ON activities(wake_at) WHERE status = 'SUSPENDED' AND wake_at IS NOT NULL;
```

**Notes**:

- `activity_type` is registered by an app and advertised by workers as a capability. The engine does not interpret it.
- `input_json` / `result_json` are opaque payloads — a connector definition or an LLMOps runbook lives here as far as the engine is concerned.
- `parent_activity_id` makes the composition tree explicit; a `SUSPENDED` parent waiting on a child holds no worker slot.
- `generation` is the fence token; a stale worker resuming with an old generation is rejected (split-brain prevention).
- There is no `workload_config_id`. Reusable configuration is an app concern (e.g. the connector definition); the engine stores only the per-activity `input_json`.

**Example `affinity_rules`** (unchanged from v1 semantics, applies to dispatch of any activity, parent or child):

```json
{
  "hard_affinity": { "worker_tags": ["gpu", "us-west-2"] },
  "soft_affinity": { "worker_id": "worker-5" },
  "anti_affinity": { "activity_types": ["cpu_intensive"] }
}
```

---

### 3. Activity Attempts Table

**Purpose**: One row per physical execution attempt. Replaces the v1 `retry_count` integer with a full per-attempt audit trail — essential for unattended retries.

```sql
CREATE TABLE activity_attempts (
    id                  BIGSERIAL PRIMARY KEY,
    activity_id         VARCHAR(64) NOT NULL REFERENCES activities(id) ON DELETE CASCADE,
    attempt_number      INT NOT NULL,
    worker_id           VARCHAR(64) REFERENCES workers(id) ON DELETE SET NULL,
    generation          BIGINT NOT NULL,
    status              activity_status NOT NULL,
    error_json          JSONB,
    started_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    ended_at            TIMESTAMP WITH TIME ZONE,
    UNIQUE(activity_id, attempt_number)
);

CREATE INDEX idx_attempts_activity ON activity_attempts(activity_id, attempt_number DESC);
CREATE INDEX idx_attempts_worker   ON activity_attempts(worker_id) WHERE worker_id IS NOT NULL;
```

**Invariant**: the `activities` row records the *logical* execution; `activity_attempts` rows record the *physical* retries. The current attempt is `MAX(attempt_number)`.

---

### 4. Activity Calls Table (Call Log)

**Purpose**: The durable record that makes replay-based resume work. Every `RunChild`, `Sleep`, and `AwaitSignal` made by a parent activity is appended here with a deterministic `call_index`. On parent resume, the activity body re-executes from the top; each call checks this log — calls with a recorded result return immediately without re-dispatching (no re-query, no re-spend).

```sql
CREATE TYPE activity_call_type AS ENUM ('run_child', 'sleep', 'await_signal');

CREATE TABLE activity_calls (
    id                  BIGSERIAL PRIMARY KEY,
    activity_id         VARCHAR(64) NOT NULL REFERENCES activities(id) ON DELETE CASCADE,
    call_index          INT NOT NULL,               -- deterministic position in the parent body
    call_type           activity_call_type NOT NULL,
    child_activity_id   VARCHAR(64) REFERENCES activities(id) ON DELETE SET NULL,  -- for run_child
    args_hash           VARCHAR(80),                -- detects nondeterministic divergence on replay
    result_json         JSONB,                      -- recorded child result / sleep completion / signal payload
    completed           BOOLEAN NOT NULL DEFAULT FALSE,
    created_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    completed_at        TIMESTAMP WITH TIME ZONE,
    UNIQUE(activity_id, call_index)
);

CREATE INDEX idx_calls_activity ON activity_calls(activity_id, call_index);
CREATE INDEX idx_calls_child    ON activity_calls(child_activity_id) WHERE child_activity_id IS NOT NULL;
```

**Invariants**:

- `call_index` is assigned in deterministic body order; a replay that produces a different `args_hash` at a given index indicates a determinism violation and fails fast.
- Replay cost is bounded by call count, not wall-clock duration (a 30-day monitor with 50 calls replays 50 lookups).

---

### 5. Signals Table (Per-Activity Mailbox)

**Purpose**: Durable delivery of external events to a running (or suspended) activity. `ctx.AwaitSignal(name, timeout)` consumes from here. A signal arriving while no worker holds the parent is persisted until the parent resumes.

```sql
CREATE TABLE signals (
    id                  BIGSERIAL PRIMARY KEY,
    activity_id         VARCHAR(64) NOT NULL REFERENCES activities(id) ON DELETE CASCADE,
    name                VARCHAR(128) NOT NULL,
    payload_json        JSONB,
    delivered           BOOLEAN NOT NULL DEFAULT FALSE,
    created_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    delivered_at        TIMESTAMP WITH TIME ZONE
);

CREATE INDEX idx_signals_pending ON signals(activity_id, name) WHERE delivered = FALSE;
```

**Delivery semantics**: at-least-once with idempotent handlers (see ADR-005 open question). The `SignalBus` interface abstracts the substrate (Postgres LISTEN/NOTIFY or a dedicated table early; NATS JetStream later) without changing activity code.

---

### 6. Workers Table

**Purpose**: Track registered workers and their health. Capabilities now advertise the **activity types** a worker can execute (any app's types).

```sql
CREATE TYPE worker_status AS ENUM ('HEALTHY', 'DRAINING', 'DEAD');

CREATE TABLE workers (
    id                  VARCHAR(64) PRIMARY KEY,
    status              worker_status NOT NULL DEFAULT 'HEALTHY',
    capabilities        JSONB NOT NULL,             -- {"activity_types": [...], "version": "..."}
    capacity_total      INT NOT NULL,
    capacity_used       INT NOT NULL DEFAULT 0,
    tags                JSONB,
    last_heartbeat      TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    registered_at       TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_workers_status    ON workers(status);
CREATE INDEX idx_workers_heartbeat ON workers(last_heartbeat) WHERE status = 'HEALTHY';
```

**Example `capabilities`**:

```json
{ "activity_types": ["run_ingestion_partition", "diagnose", "model_call"], "version": "1.2.3" }
```

---

### 7. Checkpoints Table (Intra-Attempt Heartbeat State)

**Purpose**: Optional fine-grained progress *within* a single attempt, persisted via `ctx.Heartbeat(state)`. On retry of the same attempt, `ctx.LastHeartbeat()` returns the last persisted state. This is distinct from replay (which handles cross-`RunChild` resume); checkpoints handle long in-process loops between calls.

```sql
CREATE TABLE checkpoints (
    id                  BIGSERIAL PRIMARY KEY,
    activity_id         VARCHAR(64) NOT NULL REFERENCES activities(id) ON DELETE CASCADE,
    attempt_id          BIGINT REFERENCES activity_attempts(id) ON DELETE CASCADE,
    sequence_number     INT NOT NULL,
    checkpoint_data     BYTEA NOT NULL,
    data_size_bytes     INT NOT NULL,
    created_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    UNIQUE(activity_id, sequence_number)
);

CREATE INDEX idx_checkpoints_activity ON checkpoints(activity_id, sequence_number DESC);
```

**Constraints**: max checkpoint size 10MB (app layer; see [adr/ADR-005-activity-execution-model.md](adr/ADR-005-activity-execution-model.md) ADR-004 in ARCHITECTURE); keep last 10 per activity.

---

### 8. Audit Events Table

**Purpose**: Immutable log of activity lifecycle events.

```sql
CREATE TABLE audit_events (
    id                  BIGSERIAL PRIMARY KEY,
    activity_id         VARCHAR(64) REFERENCES activities(id) ON DELETE CASCADE,
    customer_id         VARCHAR(64) NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    event_type          VARCHAR(64) NOT NULL,
    actor_type          VARCHAR(32) NOT NULL CHECK (actor_type IN ('coordinator', 'worker', 'system', 'user')),
    actor_id            VARCHAR(64),
    details_json        JSONB,
    timestamp           TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_audit_events_activity ON audit_events(activity_id, timestamp DESC);
CREATE INDEX idx_audit_events_customer ON audit_events(customer_id, timestamp DESC);
CREATE INDEX idx_audit_events_type     ON audit_events(event_type, timestamp DESC);
```

**Common Event Types**:

- `activity.created`, `activity.scheduled`, `activity.started`
- `activity.child_dispatched`, `activity.suspended`, `activity.resumed`
- `activity.signal_received`, `activity.heartbeat`
- `activity.paused`, `activity.completed`, `activity.failed`, `activity.cancelled`, `activity.redistributed`
- `worker.registered`, `worker.heartbeat_missed`, `worker.marked_dead`

App-specific audit events (connector upgraded, remediation applied) are written by apps via the same table.

---

### 9. Coordinator State Table (Leader Election)

**Purpose**: Coordinator leader election when not using etcd. Unchanged from v1.

```sql
CREATE TABLE coordinator_state (
    key                 VARCHAR(64) PRIMARY KEY,
    value               TEXT NOT NULL,
    expires_at          TIMESTAMP WITH TIME ZONE NOT NULL,
    updated_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

INSERT INTO coordinator_state (key, value, expires_at)
VALUES ('leader_lock', '', NOW())
ON CONFLICT (key) DO NOTHING;
```

```sql
-- Acquire / renew leader lock
UPDATE coordinator_state
SET value = 'coordinator-abc123', expires_at = NOW() + INTERVAL '10 seconds', updated_at = NOW()
WHERE key = 'leader_lock'
  AND (expires_at < NOW() OR value = 'coordinator-abc123')
RETURNING value;
```

---

## Broker Message Formats

### Message Envelope

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

### 1. Worker Registration — `worker.register` (Worker → Coordinator, `coordinator.events`)

```go
type WorkerRegistrationPayload struct {
    WorkerID      string            `json:"worker_id"`
    ActivityTypes []string          `json:"activity_types"`
    CapacityTotal int               `json:"capacity_total"`
    Tags          map[string]string `json:"tags"`
    Version       string            `json:"version"`
}
```

### 2. Activity Assignment — `activity.assign` (Coordinator → Worker, `worker.{id}.commands`)

```go
type ActivityAssignmentPayload struct {
    ActivityID    string                 `json:"activity_id"`
    ActivityType  string                 `json:"activity_type"`
    InputJSON     map[string]interface{} `json:"input_json"`
    Generation    int64                  `json:"generation"`
    LeaseSeconds  int                    `json:"lease_seconds"`
    AttemptNumber int                    `json:"attempt_number"`
}
```

> Secrets are never embedded in `input_json`; workers resolve secret references through approved providers.

### 3. Heartbeat — `worker.heartbeat` (Worker → Coordinator, `worker.{id}.status`)

```go
type HeartbeatPayload struct {
    WorkerID         string   `json:"worker_id"`
    Status           string   `json:"status"`            // "HEALTHY" | "DRAINING"
    CapacityUsed     int      `json:"capacity_used"`
    CapacityTotal    int      `json:"capacity_total"`
    RunningActivities []string `json:"running_activities"`
}
```

### 4. Progress / Heartbeat State — `activity.progress` (Worker → Coordinator, `worker.{id}.status`)

```go
type ProgressUpdatePayload struct {
    ActivityID    string `json:"activity_id"`
    AttemptNumber int    `json:"attempt_number"`
    CheckpointSeq int    `json:"checkpoint_sequence,omitempty"`
    Message       string `json:"message,omitempty"`
}
```

### 5. Child Dispatch / Completion — `activity.child_dispatch`, `activity.completed`, `activity.failed`

```go
type ActivityCompletionPayload struct {
    ActivityID   string                 `json:"activity_id"`
    AttemptNumber int                   `json:"attempt_number"`
    Status       string                 `json:"status"`        // "COMPLETED" | "FAILED"
    ResultJSON   map[string]interface{} `json:"result_json,omitempty"`
    ErrorMessage string                 `json:"error_message,omitempty"`
    ErrorType    string                 `json:"error_type,omitempty"` // "transient" | "permanent"
}
```

### 6. Activity Cancellation — `activity.cancel` (Coordinator → Worker, `worker.{id}.commands`)

```go
type ActivityCancellationPayload struct {
    ActivityID string `json:"activity_id"`
    Reason     string `json:"reason"`
}
```

### 7. Signal Delivery — `activity.signal` (Coordinator → Worker, or internal via SignalBus)

```go
type SignalPayload struct {
    ActivityID string                 `json:"activity_id"`
    Name       string                 `json:"name"`
    Payload    map[string]interface{} `json:"payload,omitempty"`
}
```

---

## Go Struct Definitions

```go
package models

import (
    "encoding/json"
    "time"
)

type ActivityStatus string

const (
    ActivityStatusPending        ActivityStatus = "PENDING"
    ActivityStatusScheduled      ActivityStatus = "SCHEDULED"
    ActivityStatusRunning         ActivityStatus = "RUNNING"
    ActivityStatusSuspended      ActivityStatus = "SUSPENDED"
    ActivityStatusPaused         ActivityStatus = "PAUSED"
    ActivityStatusRedistributing ActivityStatus = "REDISTRIBUTING"
    ActivityStatusCompleted      ActivityStatus = "COMPLETED"
    ActivityStatusFailed         ActivityStatus = "FAILED"
    ActivityStatusCancelled      ActivityStatus = "CANCELLED"
)

type WorkerStatus string

const (
    WorkerStatusHealthy  WorkerStatus = "HEALTHY"
    WorkerStatusDraining WorkerStatus = "DRAINING"
    WorkerStatusDead     WorkerStatus = "DEAD"
)

type CustomerTier string

const (
    TierFree       CustomerTier = "free"
    TierStandard   CustomerTier = "standard"
    TierPremium    CustomerTier = "premium"
    TierEnterprise CustomerTier = "enterprise"
)

// Customer is a tenant.
type Customer struct {
    ID                      string       `json:"id" db:"id"`
    Name                    string       `json:"name" db:"name"`
    Tier                    CustomerTier `json:"tier" db:"tier"`
    MaxConcurrentActivities int          `json:"max_concurrent_activities" db:"max_concurrent_activities"`
    MaxQueueDepth           int          `json:"max_queue_depth" db:"max_queue_depth"`
    RateLimitPerSec         int          `json:"rate_limit_per_sec" db:"rate_limit_per_sec"`
    CreatedAt               time.Time    `json:"created_at" db:"created_at"`
    UpdatedAt               time.Time    `json:"updated_at" db:"updated_at"`
}

// Activity is the logical unit of work.
type Activity struct {
    ID               string          `json:"id" db:"id"`
    CustomerID       string          `json:"customer_id" db:"customer_id"`
    ActivityType     string          `json:"activity_type" db:"activity_type"`
    ParentActivityID *string         `json:"parent_activity_id,omitempty" db:"parent_activity_id"`
    RootActivityID   *string         `json:"root_activity_id,omitempty" db:"root_activity_id"`
    WorkerID         *string         `json:"worker_id,omitempty" db:"worker_id"`
    Status           ActivityStatus  `json:"status" db:"status"`
    Priority         int             `json:"priority" db:"priority"`
    AffinityRules    *AffinityRules  `json:"affinity_rules,omitempty" db:"affinity_rules"`
    InputJSON        json.RawMessage `json:"input_json" db:"input_json"`
    ResultJSON       json.RawMessage `json:"result_json,omitempty" db:"result_json"`
    ErrorJSON        json.RawMessage `json:"error_json,omitempty" db:"error_json"`
    RetryPolicyJSON  json.RawMessage `json:"retry_policy_json" db:"retry_policy_json"`
    LeaseExpiresAt   *time.Time      `json:"lease_expires_at,omitempty" db:"lease_expires_at"`
    Generation       int64           `json:"generation" db:"generation"`
    SuspendReason    *string         `json:"suspend_reason,omitempty" db:"suspend_reason"`
    WakeAt           *time.Time      `json:"wake_at,omitempty" db:"wake_at"`
    CreatedAt        time.Time       `json:"created_at" db:"created_at"`
    ScheduledAt      *time.Time      `json:"scheduled_at,omitempty" db:"scheduled_at"`
    StartedAt        *time.Time      `json:"started_at,omitempty" db:"started_at"`
    CompletedAt      *time.Time      `json:"completed_at,omitempty" db:"completed_at"`
    UpdatedAt        time.Time       `json:"updated_at" db:"updated_at"`
}

// ActivityAttempt is one physical execution attempt.
type ActivityAttempt struct {
    ID            int64           `json:"id" db:"id"`
    ActivityID    string          `json:"activity_id" db:"activity_id"`
    AttemptNumber int             `json:"attempt_number" db:"attempt_number"`
    WorkerID      *string         `json:"worker_id,omitempty" db:"worker_id"`
    Generation    int64           `json:"generation" db:"generation"`
    Status        ActivityStatus  `json:"status" db:"status"`
    ErrorJSON     json.RawMessage `json:"error_json,omitempty" db:"error_json"`
    StartedAt     time.Time       `json:"started_at" db:"started_at"`
    EndedAt       *time.Time      `json:"ended_at,omitempty" db:"ended_at"`
}

// ActivityCall is one row in the durable call log used for replay.
type ActivityCall struct {
    ID              int64           `json:"id" db:"id"`
    ActivityID      string          `json:"activity_id" db:"activity_id"`
    CallIndex       int             `json:"call_index" db:"call_index"`
    CallType        string          `json:"call_type" db:"call_type"` // run_child | sleep | await_signal
    ChildActivityID *string         `json:"child_activity_id,omitempty" db:"child_activity_id"`
    ArgsHash        string          `json:"args_hash,omitempty" db:"args_hash"`
    ResultJSON      json.RawMessage `json:"result_json,omitempty" db:"result_json"`
    Completed       bool            `json:"completed" db:"completed"`
    CreatedAt       time.Time       `json:"created_at" db:"created_at"`
    CompletedAt     *time.Time      `json:"completed_at,omitempty" db:"completed_at"`
}

// Signal is one entry in an activity's durable mailbox.
type Signal struct {
    ID          int64           `json:"id" db:"id"`
    ActivityID  string          `json:"activity_id" db:"activity_id"`
    Name        string          `json:"name" db:"name"`
    PayloadJSON json.RawMessage `json:"payload_json,omitempty" db:"payload_json"`
    Delivered   bool            `json:"delivered" db:"delivered"`
    CreatedAt   time.Time       `json:"created_at" db:"created_at"`
    DeliveredAt *time.Time      `json:"delivered_at,omitempty" db:"delivered_at"`
}

type AffinityRules struct {
    HardAffinity *AffinityConstraint `json:"hard_affinity,omitempty"`
    SoftAffinity *AffinityConstraint `json:"soft_affinity,omitempty"`
    AntiAffinity *AffinityConstraint `json:"anti_affinity,omitempty"`
}

type AffinityConstraint struct {
    WorkerID      string            `json:"worker_id,omitempty"`
    WorkerTags    map[string]string `json:"worker_tags,omitempty"`
    ActivityTypes []string          `json:"activity_types,omitempty"`
}

type Worker struct {
    ID            string            `json:"id" db:"id"`
    Status        WorkerStatus      `json:"status" db:"status"`
    ActivityTypes []string          `json:"activity_types" db:"-"`
    CapacityTotal int               `json:"capacity_total" db:"capacity_total"`
    CapacityUsed  int               `json:"capacity_used" db:"capacity_used"`
    Tags          map[string]string `json:"tags" db:"tags"`
    LastHeartbeat time.Time         `json:"last_heartbeat" db:"last_heartbeat"`
    RegisteredAt  time.Time         `json:"registered_at" db:"registered_at"`
    UpdatedAt     time.Time         `json:"updated_at" db:"updated_at"`
}

type Checkpoint struct {
    ID             int64     `json:"id" db:"id"`
    ActivityID     string    `json:"activity_id" db:"activity_id"`
    AttemptID      *int64    `json:"attempt_id,omitempty" db:"attempt_id"`
    SequenceNumber int       `json:"sequence_number" db:"sequence_number"`
    CheckpointData []byte    `json:"checkpoint_data" db:"checkpoint_data"`
    DataSizeBytes  int       `json:"data_size_bytes" db:"data_size_bytes"`
    CreatedAt      time.Time `json:"created_at" db:"created_at"`
}

type AuditEvent struct {
    ID          int64           `json:"id" db:"id"`
    ActivityID  *string         `json:"activity_id,omitempty" db:"activity_id"`
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

### Activity Validation

```go
func (a *Activity) Validate() error {
    if a.ID == "" || !strings.HasPrefix(a.ID, "act_") {
        return errors.New("invalid activity ID")
    }
    if a.CustomerID == "" {
        return errors.New("customer_id required")
    }
    if a.ActivityType == "" {
        return errors.New("activity_type required")
    }
    if a.Priority < 0 || a.Priority > 1000 {
        return errors.New("priority must be between 0 and 1000")
    }
    return nil
}
```

### Call Log Determinism Check (on replay)

```go
// During replay, the re-executed body must produce the same call sequence.
func (c *ActivityCall) AssertReplayMatch(observedType string, observedArgsHash string) error {
    if c.CallType != observedType {
        return fmt.Errorf("determinism violation at call_index %d: logged %s, replayed %s",
            c.CallIndex, c.CallType, observedType)
    }
    if c.ArgsHash != "" && c.ArgsHash != observedArgsHash {
        return fmt.Errorf("determinism violation at call_index %d: args changed on replay", c.CallIndex)
    }
    return nil
}
```

### Checkpoint Validation

```go
const MaxCheckpointSize = 10 * 1024 * 1024 // 10MB

func (c *Checkpoint) Validate() error {
    if c.ActivityID == "" {
        return errors.New("activity_id required")
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

1. **Activity scheduling query** — `idx_activities_priority` (customer_id, status, priority DESC) for `PENDING` selection.
2. **Lease expiration sweep** — `idx_activities_lease` finds expired `RUNNING` leases for redistribution.
3. **Wake sweep** — `idx_activities_wake` finds `SUSPENDED` activities whose `Sleep`/`AwaitSignal` timeout is due.
4. **Replay lookup** — `idx_calls_activity` (activity_id, call_index) for the per-call log scan on resume.
5. **Pending signal lookup** — `idx_signals_pending` for `AwaitSignal` delivery.
6. **Audit queries** — `idx_audit_events_activity`, `idx_audit_events_customer`.

---

## Data Retention Policies

| Table | Retention | Cleanup Method |
|-------|-----------|----------------|
| `activities` | Keep terminal (COMPLETED/FAILED/CANCELLED) 90 days | Nightly archive to cold storage |
| `activity_attempts` | Follows parent activity | Cascade on archive |
| `activity_calls` | Follows parent activity | Cascade on archive |
| `signals` | Delete delivered after 7 days | Nightly cleanup |
| `checkpoints` | Keep last 10 per activity | On new checkpoint, prune older |
| `audit_events` | Keep 1 year | Monthly partition drop |
| `workers` | Delete DEAD after 7 days | Nightly cleanup |

```sql
-- Archive terminal activities (cascades attempts/calls)
INSERT INTO activities_archive
SELECT * FROM activities
WHERE status IN ('COMPLETED', 'FAILED', 'CANCELLED')
  AND completed_at < NOW() - INTERVAL '90 days';

DELETE FROM activities
WHERE status IN ('COMPLETED', 'FAILED', 'CANCELLED')
  AND completed_at < NOW() - INTERVAL '90 days';

-- Prune delivered signals
DELETE FROM signals WHERE delivered = TRUE AND delivered_at < NOW() - INTERVAL '7 days';

-- Delete old dead workers
DELETE FROM workers WHERE status = 'DEAD' AND updated_at < NOW() - INTERVAL '7 days';
```

---

## State Consistency Guarantees

### Transaction Boundaries

**Activity state transition** (atomic):

```
tx.Begin()
  - Update activities.status (+ generation on redistribution)
  - Insert activity_attempts row (on new attempt)
  - Insert audit_event
  - Update workers.capacity_used (if applicable)
tx.Commit()
```

**RunChild dispatch** (atomic — the durable suspend point):

```
tx.Begin()
  - Insert activity_calls row (call_index, run_child, child_activity_id, completed=false)
  - Insert child activities row (status=PENDING, parent_activity_id set)
  - Update parent activities.status = SUSPENDED, suspend_reason = run_child
  - Free parent worker slot
tx.Commit()
```

**RunChild result record** (atomic — parent re-dispatch):

```
tx.Begin()
  - Update activity_calls.result_json, completed=true
  - Update parent activities.status = SCHEDULED (re-dispatch for replay)
tx.Commit()
```

**Idempotency**: only `RunChild` dispatch must be idempotent — a parent crash between dispatch and result-record may re-invoke a child. Children carry their own attempt boundary.

### Idempotency Keys

```sql
CREATE TABLE idempotency_keys (
    key         VARCHAR(64) PRIMARY KEY,
    activity_id VARCHAR(64) NOT NULL,
    created_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_idempotency_created ON idempotency_keys(created_at);
```

---

## Schema Migration Strategy

Use **golang-migrate** for versioned migrations. Conventions:

- One numbered pair per change: `NNN_description.up.sql` / `NNN_description.down.sql`.
- Each migration wrapped in `BEGIN; … COMMIT;`. Down migration must reverse the up.
- **Phase map** (which migration introduces what; mirrors [../IMPLEMENTATION_PLAN.md](../IMPLEMENTATION_PLAN.md) and the platform impl phases):

| Migration | Phase | Introduces |
|-----------|-------|-----------|
| `001_core` | Platform Phase 1 | `customers`, `activities`, `activity_attempts`, `workers`, `checkpoints`, `audit_events` |
| `002_call_log` | Platform Phase 3 | `activity_calls` + replay indexes |
| `003_signals` | Platform Phase 4 | `signals` mailbox + pending index |
| `004_coordinator_state` | Platform Phase 6 | `coordinator_state` (if Postgres-based leader election) |
| app migrations | dataconnector P5 / llmops P1+ | app tables (`connector_*`, `llmops_*`) — owned by the app, never altering core tables |

```sql
-- 001_core.up.sql
BEGIN;
CREATE TYPE activity_status AS ENUM (...);
CREATE TABLE customers ( ... );
CREATE TABLE activities ( ... );
CREATE TABLE activity_attempts ( ... );
-- ... indexes
COMMIT;
```

```sql
-- 001_core.down.sql
BEGIN;
DROP TABLE IF EXISTS audit_events CASCADE;
DROP TABLE IF EXISTS checkpoints CASCADE;
DROP TABLE IF EXISTS activity_attempts CASCADE;
DROP TABLE IF EXISTS activities CASCADE;
DROP TABLE IF EXISTS workers CASCADE;
DROP TABLE IF EXISTS customers CASCADE;
DROP TYPE IF EXISTS activity_status;
COMMIT;
```

**Rollback policy**: app tables are dropped before platform tables; never drop a core table while an app table references it. Breaking changes to the call-log or signal shape require a new migration plus a replay-compatibility note (see ADR-005 on checkpoint/state migration).

---

## Appendix: v1 → v2 Migration Map

For anyone holding the old workload vocabulary:

| v1 (workload model) | v2 (activity model, ADR-005) | Note |
|---------------------|------------------------------|------|
| `workloads` | `activities` | Adds `parent_activity_id`, `root_activity_id`, `suspend_reason`, `wake_at` |
| `workload_configs` | *(removed)* | Reusable config is an app concern (e.g. connector definition); engine stores per-activity `input_json` only |
| `workload_status` enum | `activity_status` enum | Adds `SUSPENDED` (durable suspend point) |
| `workloads.retry_count` (int) | `activity_attempts` (table) | One row per physical attempt |
| `workloads.input_params` | `activities.input_json` | Opaque to engine |
| `workloads.result_data` | `activities.result_json` | |
| `workloads.error_message` | `activities.error_json` | Structured `{message, type}` |
| *(none)* | `activity_calls` | New: durable call log for replay |
| *(none)* | `signals` | New: per-activity mailbox |
| `checkpoints.workload_id` | `checkpoints.activity_id` + `attempt_id` | Now scoped to attempt (heartbeat semantics) |
| `workers.capabilities.supported_workload_types` | `workers.capabilities.activity_types` | Advertises any app's activity types |
| `customers.max_concurrent_workloads` | `customers.max_concurrent_activities` | |

---

## Future Considerations

1. **Sharding**: shard by `customer_id` if a single PostgreSQL instance becomes the bottleneck.
2. **Large checkpoints**: store > 10MB checkpoints in S3, keep a reference in `checkpoints` (ADR-004, Phase 2).
3. **Replay performance**: for activities with very high call counts, batch `RunChild` calls or tune per-call-site replay keys (ADR-005 engine Phase 4).
4. **Read replicas**: offload audit and dashboard queries.

---

## References

- [adr/ADR-005-activity-execution-model.md](adr/ADR-005-activity-execution-model.md) - the canonical activity execution model
- [ARCHITECTURE.md](ARCHITECTURE.md) - platform architecture overview
- [API_SPEC.md](API_SPEC.md) - engine API contracts
- [../IMPLEMENTATION_PLAN.md](../IMPLEMENTATION_PLAN.md) - phased delivery plan
