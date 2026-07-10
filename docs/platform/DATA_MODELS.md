# Niyanta Platform Data Models

**Version**: 2.0
**Last Updated**: 2026-06-03
**Status**: Design Phase

> **v2.0 supersedes the v1 workload model.** Niyanta is a composable activity runner; its canonical persisted state is the **activity** and its execution record (attempts, call log, signals), per [adr/ADR-005-activity-execution-model.md](adr/ADR-005-activity-execution-model.md). The earlier `workloads` / `workload_configs` / `retry_count` schema is replaced. A migration map from the old vocabulary is in the appendix.

This document defines the **platform** schema only — the engine's state for running activities and resuming them after failure. It is use-case agnostic: it knows about activities, attempts, the call log, signals, checkpoints, workers, tenants, and audit. It does **not** know about any consumer's domain. Consumers may add their own tables in the same Postgres; they never alter core tables.

## Table of Contents
1. [Overview](#overview)
2. [Database Schema](#database-schema)
3. [State Storage Models](#state-storage-models)
4. [Control-Plane Message Formats](#control-plane-message-formats)
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

This document defines table schemas, control-plane message formats, Go structs, validation rules, and invariants.

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
    input_json          JSONB NOT NULL,              -- opaque to the engine; size-capped (default 256KB)
    result_json         JSONB,                       -- size-capped (default 256KB)
    error_json          JSONB,                       -- {message, type: transient|permanent|nondeterminism, ...}
    retry_policy_json   JSONB NOT NULL DEFAULT '{"max_attempts": 3, "backoff": "exponential"}',
    generation          BIGINT NOT NULL DEFAULT 1,   -- fencing token, bumped on redistribution
    pinned_version      VARCHAR(64),                 -- G1: code version stamped at first suspend; resume only on matching workers
    execution_seq       INT NOT NULL DEFAULT 1,      -- G5: current execution in the ContinueAsNew chain
    superseded_by       VARCHAR(64),                 -- G5: set on an execution completed via ContinueAsNew (audit chain)
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
CREATE INDEX idx_activities_pinned         ON activities(activity_type, pinned_version) WHERE pinned_version IS NOT NULL;
CREATE INDEX idx_activities_wake           ON activities(wake_at) WHERE status = 'SUSPENDED' AND wake_at IS NOT NULL;
```

> The execution **lease lives on the attempt** (see `activity_attempts`), not here — a lease belongs to a physical execution. The activity carries only the `generation` fence token.

**Notes**:

- `activity_type` is registered by consumer code and advertised by workers as a capability (with a version). The engine does not interpret it.
- `input_json` / `result_json` are opaque payloads — whatever domain config or result a consumer puts here, the engine never parses it. Both are size-capped (default 256KB); child results are re-read on every replay, and the cap is what keeps replay cheap.
- `parent_activity_id` makes the composition tree explicit; a `SUSPENDED` parent waiting on a child holds no worker slot.
- `generation` is the fence token; a stale worker resuming with an old generation is rejected (split-brain prevention).
- There is no `workload_config_id`. Reusable configuration is a consumer concern; the engine stores only the per-activity `input_json`.

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

Attempts have their **own status enum** — the physical lifecycle differs from the logical one (`INTERRUPTED` and `FENCED` exist only here; `SUSPENDED`/`PAUSED` never do).

```sql
CREATE TYPE attempt_status AS ENUM (
    'PENDING',       -- enqueued, claimable by a worker
    'RUNNING',
    'COMPLETED',
    'FAILED',
    'INTERRUPTED',   -- worker died / lease expired / restart recovery
    'FENCED'         -- terminal write rejected: stale generation
);

CREATE TABLE activity_attempts (
    id                  BIGSERIAL PRIMARY KEY,
    activity_id         VARCHAR(64) NOT NULL REFERENCES activities(id) ON DELETE CASCADE,
    attempt_number      INT NOT NULL,
    execution_seq       INT NOT NULL DEFAULT 1,      -- which ContinueAsNew execution this attempt belongs to
    worker_id           VARCHAR(64) REFERENCES workers(id) ON DELETE SET NULL,
    generation          BIGINT NOT NULL,
    status              attempt_status NOT NULL DEFAULT 'PENDING',
    not_before          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),  -- retry backoff gate
    lease_expires_at    TIMESTAMP WITH TIME ZONE,    -- renewed by worker heartbeat; expiry => redistribution
    error_json          JSONB,
    started_at          TIMESTAMP WITH TIME ZONE,
    ended_at            TIMESTAMP WITH TIME ZONE,
    UNIQUE(activity_id, execution_seq, attempt_number)
);

CREATE INDEX idx_attempts_activity ON activity_attempts(activity_id, attempt_number DESC);
CREATE INDEX idx_attempts_pending  ON activity_attempts(not_before, id) WHERE status = 'PENDING';
CREATE INDEX idx_attempts_lease    ON activity_attempts(lease_expires_at) WHERE status = 'RUNNING';
CREATE INDEX idx_attempts_worker   ON activity_attempts(worker_id) WHERE worker_id IS NOT NULL;
```

**Invariants**:

- The `activities` row records the *logical* execution; `activity_attempts` rows record the *physical* retries. The current attempt is `MAX(attempt_number)` within the current `execution_seq`.
- The dispatch queue **is** `idx_attempts_pending` — workers claim with `FOR UPDATE SKIP LOCKED`.
- **Write-path fencing (G3)**: every mutating write on an attempt (complete, fail, heartbeat, lease renewal) is conditional on the activity's `generation` matching the writer's; a miss returns `ErrFenced` and the attempt is marked `FENCED`.

---

### 4. Activity Calls Table (Call Log)

**Purpose**: The durable record that makes replay-based resume work. Every `RunChild`, `Sleep`, and `AwaitSignal` made by a parent activity is appended here with a deterministic `call_index`. On parent resume, the activity body re-executes from the top; each call checks this log — calls with a recorded result return immediately without re-dispatching (no re-query, no re-spend).

```sql
CREATE TYPE activity_call_type AS ENUM ('run_child', 'sleep', 'await_signal');

CREATE TABLE activity_calls (
    id                  BIGSERIAL PRIMARY KEY,
    activity_id         VARCHAR(64) NOT NULL REFERENCES activities(id) ON DELETE CASCADE,
    execution_seq       INT NOT NULL DEFAULT 1,     -- G5: scoped to the current ContinueAsNew execution
    call_index          INT NOT NULL,               -- deterministic position in the parent body
    call_type           activity_call_type NOT NULL,
    child_activity_id   VARCHAR(64) REFERENCES activities(id) ON DELETE SET NULL,  -- for run_child
    args_hash           VARCHAR(80),                -- detects nondeterministic divergence on replay
    result_json         JSONB,                      -- recorded child result / sleep completion / signal payload (size-capped)
    completed           BOOLEAN NOT NULL DEFAULT FALSE,
    created_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    completed_at        TIMESTAMP WITH TIME ZONE,
    UNIQUE(activity_id, execution_seq, call_index)
);

CREATE INDEX idx_calls_activity ON activity_calls(activity_id, execution_seq, call_index);
CREATE INDEX idx_calls_child    ON activity_calls(child_activity_id) WHERE child_activity_id IS NOT NULL;
```

**Invariants**:

- `call_index` is assigned in deterministic body order; a replay that produces a different `args_hash` at a given index is a determinism violation — the activity fails with `error_json.type = "nondeterminism"` and is **not retried automatically** (G6; operator recovery via `:truncate-replay` / `:force-*`).
- **Exactly-once dispatch (G4)**: the call-log append and child creation commit in one transaction; replay never re-dispatches a logged call — an incomplete entry means re-suspend and wait.
- Replay loads only the **current** `execution_seq`; `ContinueAsNew` starts a fresh one, bounding replay cost regardless of activity lifetime (G5).

---

### 5. Signals Table (Per-Activity Mailbox)

**Purpose**: Durable delivery of external events to a running (or suspended) activity. `ctx.AwaitSignal(name, timeout)` consumes from here. A signal arriving while no worker holds the parent is persisted until the parent resumes.

```sql
CREATE TABLE signals (
    id                  BIGSERIAL PRIMARY KEY,
    activity_id         VARCHAR(64) NOT NULL REFERENCES activities(id) ON DELETE CASCADE,
    name                VARCHAR(128) NOT NULL,
    payload_json        JSONB,
    dedupe_key          VARCHAR(128),               -- sender-supplied Idempotency-Key
    delivered           BOOLEAN NOT NULL DEFAULT FALSE,
    created_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    delivered_at        TIMESTAMP WITH TIME ZONE
);

CREATE INDEX idx_signals_pending ON signals(activity_id, name) WHERE delivered = FALSE;

-- a retried external send with the same key is one mailbox entry
CREATE UNIQUE INDEX idx_signals_dedupe ON signals(activity_id, dedupe_key) WHERE dedupe_key IS NOT NULL;
```

**Delivery semantics**: at-least-once with idempotent handlers (see ADR-005 open question), with sender-side dedupe at the door via `dedupe_key`. The `SignalBus` interface abstracts the Postgres-backed substrate; alternatives require post-Phase-9 performance evidence.

---

### 5a. Pending Timers Table (Durable Sleep)

**Purpose**: the coordinator's timer sweep. `ctx.Sleep(d)` records a call-log entry and a timer row; the sweep (poll every second on `wake_at`) completes the call and re-enqueues the activity.

```sql
CREATE TABLE pending_timers (
    id                  BIGSERIAL PRIMARY KEY,
    activity_id         VARCHAR(64) NOT NULL REFERENCES activities(id) ON DELETE CASCADE,
    call_id             BIGINT NOT NULL REFERENCES activity_calls(id) ON DELETE CASCADE,
    wake_at             TIMESTAMP WITH TIME ZONE NOT NULL,
    fired_at            TIMESTAMP WITH TIME ZONE,
    created_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_pending_timers_due ON pending_timers(wake_at) WHERE fired_at IS NULL;
```

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
{ "activity_types": {"sample_pipeline": "1.2.3", "sample_leaf": "1.2.3"} }
```

Capabilities map each activity type to the code **version** the worker runs — the basis for version-pinned replay (a suspended activity resumes only on a worker advertising its pinned version).

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

Consumers may write their own domain audit events via the same table under their own `event_type` namespace. Operator recovery actions (`force-complete`, `truncate-replay`, drain) are always audited here.

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

## Control-Plane Message Formats

**There is no broker on the critical path** (ADR-002, revised). Coordinator↔worker communication is state-based: **the tables above are the messages**.

| v1 "message" | v2 mechanism |
|--------------|--------------|
| Worker registration | Upsert into `workers` (capabilities = `{activity_type: version}`) |
| Activity assignment | Worker **claims** a `PENDING` attempt via `FOR UPDATE SKIP LOCKED` — claiming the row is the assignment; there is no assign/ack protocol |
| Worker heartbeat | Update `workers.last_heartbeat` + renew `activity_attempts.lease_expires_at` (generation-fenced) |
| Progress / checkpoint | Insert into `checkpoints` (generation-fenced) |
| Completion / failure | Conditional update on the attempt + activity rows (generation-fenced) |
| Cancellation / pause | Coordinator flags the activity row; workers observe on heartbeat or notification |
| Signal delivery | Insert into `signals`; coordinator delivers to the waiting call |

### Notification Hints (LISTEN/NOTIFY)

Notifications carry **hints, never state** — payloads are identifiers only, and a dropped notification costs at most one poll interval:

| Topic | Payload | Woken party | Fallback |
|-------|---------|-------------|----------|
| `task_ready` | `activity_type` | Idle workers (claim loop) | 2s claim poll |
| `signal_arrived` | `activity_id` | Coordinator signal delivery | wake sweep |
| `control_changed` | `activity_id` | Worker running that activity | 30s heartbeat |

### Optional Post-Phase-9 Substrate Envelope

If the JetStream `TaskQueue`/`SignalBus` substrates are adopted at scale, their wire format wraps the same identifiers-only philosophy; Postgres rows remain the source of truth:

```go
// Used ONLY by optional broker substrates adopted after production profiling.
type Message struct {
    MessageID     string    `json:"message_id"`
    Type          string    `json:"type"`      // task_ready | signal_arrived | control_changed
    Timestamp     time.Time `json:"timestamp"`
    SenderID      string    `json:"sender_id"`
    CorrelationID string    `json:"correlation_id,omitempty"`
    RefID         string    `json:"ref_id"`    // activity_id / activity_type — a pointer, never a payload
}
```

> Secrets are never embedded in `input_json`; workers resolve secret references through approved providers.

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

// AttemptStatus is the physical-attempt lifecycle — distinct from ActivityStatus.
type AttemptStatus string

const (
    AttemptStatusPending     AttemptStatus = "PENDING"
    AttemptStatusRunning     AttemptStatus = "RUNNING"
    AttemptStatusCompleted   AttemptStatus = "COMPLETED"
    AttemptStatusFailed      AttemptStatus = "FAILED"
    AttemptStatusInterrupted AttemptStatus = "INTERRUPTED"
    AttemptStatusFenced      AttemptStatus = "FENCED"
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
    Generation       int64           `json:"generation" db:"generation"`
    PinnedVersion    *string         `json:"pinned_version,omitempty" db:"pinned_version"`
    ExecutionSeq     int             `json:"execution_seq" db:"execution_seq"`
    SupersededBy     *string         `json:"superseded_by,omitempty" db:"superseded_by"`
    SuspendReason    *string         `json:"suspend_reason,omitempty" db:"suspend_reason"`
    WakeAt           *time.Time      `json:"wake_at,omitempty" db:"wake_at"`
    CreatedAt        time.Time       `json:"created_at" db:"created_at"`
    ScheduledAt      *time.Time      `json:"scheduled_at,omitempty" db:"scheduled_at"`
    StartedAt        *time.Time      `json:"started_at,omitempty" db:"started_at"`
    CompletedAt      *time.Time      `json:"completed_at,omitempty" db:"completed_at"`
    UpdatedAt        time.Time       `json:"updated_at" db:"updated_at"`
}

// ActivityAttempt is one physical execution attempt. The lease lives here.
type ActivityAttempt struct {
    ID             int64           `json:"id" db:"id"`
    ActivityID     string          `json:"activity_id" db:"activity_id"`
    AttemptNumber  int             `json:"attempt_number" db:"attempt_number"`
    ExecutionSeq   int             `json:"execution_seq" db:"execution_seq"`
    WorkerID       *string         `json:"worker_id,omitempty" db:"worker_id"`
    Generation     int64           `json:"generation" db:"generation"`
    Status         AttemptStatus   `json:"status" db:"status"`
    NotBefore      time.Time       `json:"not_before" db:"not_before"`
    LeaseExpiresAt *time.Time      `json:"lease_expires_at,omitempty" db:"lease_expires_at"`
    ErrorJSON      json.RawMessage `json:"error_json,omitempty" db:"error_json"`
    StartedAt      *time.Time      `json:"started_at,omitempty" db:"started_at"`
    EndedAt        *time.Time      `json:"ended_at,omitempty" db:"ended_at"`
}

// ActivityCall is one row in the durable call log used for replay.
type ActivityCall struct {
    ID              int64           `json:"id" db:"id"`
    ActivityID      string          `json:"activity_id" db:"activity_id"`
    ExecutionSeq    int             `json:"execution_seq" db:"execution_seq"`
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
    DedupeKey   *string         `json:"dedupe_key,omitempty" db:"dedupe_key"`
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
2. **Lease expiration sweep** — `idx_attempts_lease` finds expired `RUNNING` attempt leases for redistribution (the lease lives on the attempt).
2a. **Dispatch claim** — `idx_attempts_pending` backs the workers' `FOR UPDATE SKIP LOCKED` claim query.
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

**Fencing invariant (G3), applies to every transaction below**: no state-mutating write commits without a generation match — every worker-originated statement carries `WHERE … generation = $expected`; zero rows updated returns `ErrFenced` and the writer stops. This is enforced in the storage layer and covered by the shared contract-test suite.

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
| `001_core` | Phase 1 | `customers`, `activities`, `activity_attempts` (own attempt-status enum), `workers`, `checkpoints` (attempt-scoped key), `audit_events` |
| `002_call_log` | Phase 3 | `activity_calls` + replay indexes; `activities.pinned_version`, `execution_seq`, `superseded_by` |
| `003_signals` | Phase 4 | `signals` mailbox + pending index + dedupe key; `pending_timers` |
| `004_api_keys` | Phase 5 | `api_keys`, `idempotency_keys` retention |
| `005_console_auth` | Phase 7 | browser principals/sessions/preferences in a separate console schema; no core-state access |
| `006_coordinator_state` | Phase 9 | `coordinator_state` for Postgres-based leader election |

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

**Rollback policy**: consumer-owned tables are dropped before platform tables; never drop a core table while an external table references it. Breaking changes to the call-log or signal shape require a new migration plus a replay-compatibility note (see ADR-005 on checkpoint/state migration).

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
