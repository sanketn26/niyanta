# Niyanta System Architecture

**Version**: 1.1
**Last Updated**: 2026-05-01
**Status**: Design Phase

## Table of Contents
1. [Executive Summary](#executive-summary)
2. [System Overview](#system-overview)
3. [Component Architecture](#component-architecture)
4. [Data Flow](#data-flow)
5. [State Management Architecture](#state-management-architecture)
6. [Communication Patterns](#communication-patterns)
7. [Deployment Architecture](#deployment-architecture)
8. [High Availability & Fault Tolerance](#high-availability--fault-tolerance)
9. [Security Architecture](#security-architecture)
10. [Scalability Model](#scalability-model)

---

## Executive Summary

Niyanta is an **idiomatic, composable activity runner that provides execution guarantees**. That is the product. An *activity* is a unit of work; activities compose (a parent activity can invoke optional child activities); and Niyanta guarantees how that composition runs: it survives crashes, resumes where it left off, retries on failure, runs activities in parallel across a worker fleet, coordinates them, and isolates them by tenant.

Everything else — connectors, ingestion planners, diagnosis loops, remediation — is **not the platform**. Those are *compositions of activities* that people build on Niyanta. The platform does not know what an activity "means"; it knows how to run it durably.

Niyanta provides:

- **A runtime**: Executes activities and their composed child activities to completion across a distributed worker fleet.
- **Execution guarantees**: Durability, replay-based resume across `RunChild` boundaries, framework-enforced retries, optional checkpointing, leases, and fencing — applied to an activity *and* its sub-activities. Authors do not write retry loops or serialize state.
- **State**: Durable storage of activity state, attempts, call logs, signal mailboxes, and checkpoints.
- **Parallel execution**: Many independent activities run concurrently across workers; the engine handles dispatch, redistribution, and scale-out. (Within a single parent, `RunChild` calls are sequential — see [adr/ADR-005-activity-execution-model.md](adr/ADR-005-activity-execution-model.md).)
- **Coordination**: Scheduling by capability, capacity, affinity, and priority; leader election; durable timers (`Sleep`) and external-event signals (`AwaitSignal`).
- **Isolation**: Per-tenant boundaries over every activity, its state, and its observability dimensions, with shared / pooled / dedicated execution modes.

The rest is plumbing.

### Activity Compositions Built On Niyanta

The platform is application-agnostic. Specific products are described as **activity compositions** ("cookbooks") that wire activities together and lean on the guarantees above. They are not a special platform layer — there is no app SPI or app runtime, only activities. The current compositions:

- **Data Connector** — self-orchestrated, multi-tenant data ingestion. Supervisor → plan → partition-run → deliver, expressed as composed activities. See [../apps/dataconnector/INGESTION_ARCHITECTURE.md](../apps/dataconnector/INGESTION_ARCHITECTURE.md).
- **LLMOps for Data Connectors** — uses LLMs to detect issues, diagnose lag/failures, and operationalize connectors. Sweep → diagnose → (tool lookups) → remediate → verify, expressed as composed activities that call the Data Connector control API. See [../apps/llmops/ARCHITECTURE.md](../apps/llmops/ARCHITECTURE.md).

New compositions add their own activity types and their own state without changing the engine. The platform sections below describe the runner; the app docs describe what gets composed on it.

**A composition can itself be a platform — and the engine does not change to allow it.** A layer may expose a higher-level interface that hides the activities beneath it, but doing so is entirely that layer's own responsibility: its own config language, its own parser, its own state, its own API, all built on the unchanged platform primitives. Niyanta does not grow features to "enable" apps-as-platforms; it stays a plain activity runner. The Data Connector composition offers a declarative **connector config language** (YAML connector definitions and connections) — that language, its validation, and its storage live in the Data Connector composition, not in the engine. A connector author writes YAML and never sees an activity, even though every connector run *is* an activity composition underneath. LLMOps may similarly expose a playbook/policy language over its diagnosis and remediation activities, again owned by LLMOps, not the platform. So the stack is recursive:

```
Niyanta              composable activity runner + execution guarantees
  └─ Data Connector  activity composition that also exposes a YAML connector language
       └─ connector authors write declarative YAML, never touch activities
  └─ LLMOps          activity composition that may expose a playbook/policy language
```

Each layer is "just activities composed beneath a chosen interface." Niyanta does not need to know a layer above it exists; a layer above does not need to know it is activities all the way down. Crucially, the platform carries none of this layering itself — owning the higher-level interface is the app's burden, by design. This is what keeps the engine small and every composition independent.

### What Is And Is Not Niyanta's Concern

Niyanta's concern is exactly one thing: **run an activity, give it execution guarantees, and hold enough state to resume it if it gets stuck or its worker crashes.** That is the whole contract — runtime, guarantees, state for resume, coordination, isolation. Niyanta is agnostic to *what* an activity does and to *what config drove it*.

In particular, the **connector YAML language and the LLMOps runbook/policy languages are not Niyanta configuration.** Niyanta neither defines, parses, validates, nor stores them as platform config. They are each composition's own config, interpreted by that composition's own activities. Niyanta only ever sees "an activity is running; here is its durable state; resume it if needed" — it does not see a connector definition or a runbook. A connector definition is meaningful to the Data Connector composition's activities; a runbook is meaningful to the LLMOps composition's activities; both are opaque payloads to the engine.

This separation is what lets compositions evolve their config languages freely (add a connector kind, add a runbook field) without any platform change — because the platform was never looking.

---

## System Overview

### High-Level Architecture

This is the **engine**. It runs activities; apps register activity types and submit work. Nothing below is connector- or LLM-specific.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Clients / Apps / Operators                        │
│   submit activities, signal activities, query state, admin workers    │
└────────────────────────────┬────────────────────────────────────────┘
                             │ HTTPS/gRPC
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                            Coordinator                               │
│ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ │
│ │ API Server   │ │ Scheduler    │ │ Timer /      │ │ Health &     │ │
│ │ (REST/gRPC)  │ │ (capability, │ │ Signal       │ │ Lifecycle    │ │
│ │              │ │  affinity,   │ │ Engine       │ │ Manager      │ │
│ │              │ │  quota, SLA) │ │              │ │              │ │
│ └──────┬───────┘ └──────┬───────┘ └──────┬───────┘ └──────┬───────┘ │
│        │                │                │                │         │
│ ┌──────▼────────────────▼────────────────▼────────────────▼───────┐ │
│ │ Durable activity scheduler, replay, timers, signals, leases,      │ │
│ │ fencing, redistribution                                           │ │
│ └────────────────────────────┬─────────────────────────────────────┘ │
└──────────────────────────────┼───────────────────────────────────────┘
                               │
               ┌───────────────┼───────────────┐
               │               │               │
               ▼               ▼               ▼
       ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
       │ Broker       │ │ State Store  │ │ Observability│
       │ NATS/        │ │ Postgres     │ │ Metrics/Logs │
       │ JetStream    │ │ + optional   │ │ Traces       │
       │              │ │ etcd         │ │              │
       └──────┬───────┘ └──────┬───────┘ └──────────────┘
              │                │
              ▼                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                              Workers                                 │
│  Execute registered activity types (any app's). Per-activity:        │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │ activity runtime · RunChild dispatch · Heartbeat · context       │ │
│  │ cancellation · resource isolation (cgroups)                      │ │
│  └─────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

> What runs *inside* an activity — connector source reads, transforms, destination delivery, LLM calls — is the app's concern. The engine sees only "activity of type X with opaque input, running on worker W." The Data Connector worker runtime (HTTP poll, object reader, stream listener, sinks) is documented in [../apps/dataconnector/INGESTION_ARCHITECTURE.md](../apps/dataconnector/INGESTION_ARCHITECTURE.md).

### Core Components

| Component | Count | Purpose | Stateful |
|-----------|-------|---------|----------|
| Coordinator | 3+ (HA) | API, activity scheduling, timers/signals, lifecycle management | No |
| Worker | N (horizontal) | Execute any registered activity type; dispatch children; heartbeat | No |
| State Storage | 1 cluster | Activities, attempts, call log, signals, checkpoints, audit | Yes |
| Broker | 1 cluster | Coordinator↔worker control plane; optional durable signal substrate | Optional |
| Observability Store | 1+ | Metrics, logs, traces | Yes |

App-specific control-plane components (the Data Connector's connector registry, connection manager, and ingestion planner; the LLMOps sweep/diagnosis loops) are **not** engine components — they are activity compositions and their own state, layered on the above. See [../apps/dataconnector/INGESTION_ARCHITECTURE.md](../apps/dataconnector/INGESTION_ARCHITECTURE.md) and [../apps/llmops/ARCHITECTURE.md](../apps/llmops/ARCHITECTURE.md).

---

## Component Architecture

### 1. Coordinator

**Responsibilities** (engine-only):
- Accept activity submissions, signals, lifecycle operations, and worker-admin API calls
- Schedule activity attempts to workers by activity-type capability, affinity, capacity, tenant quota, and SLA priority
- Drive durable timers (`Sleep`) and deliver signals (`AwaitSignal`) to suspended activities
- Record the call log and re-dispatch parents for replay-based resume after a child completes
- Monitor worker health via broker heartbeats; detect failure and trigger redistribution with fencing
- Manage activity lifecycle (schedule, suspend, resume, pause, cancel, complete)
- Expose metrics and health endpoints

The coordinator does **not** validate connector config, plan ingestion runs, or evaluate source/destination health — those are app activities. It schedules and resumes activities; the activities decide what to do.

**Internal Modules**:

```
┌─────────────────────────────────────────────────────┐
│             Coordinator Process                      │
│                                                      │
│  ┌────────────────┐  ┌─────────────────┐           │
│  │   API Server   │  │  Authentication │           │
│  │  (REST/gRPC)   │  │  & Authorization│           │
│  └───────┬────────┘  └────────┬────────┘           │
│          │                     │                    │
│          ▼                     ▼                    │
│  ┌──────────────────────────────────────┐          │
│  │      Scheduler                       │          │
│  │  - Activity-type capability matching  │          │
│  │  - Affinity (hard/soft/anti)          │          │
│  │  - Capacity, tenant quota, SLA queue  │          │
│  └──────────────┬───────────────────────┘          │
│                 │                                   │
│                 ▼                                   │
│  ┌──────────────────────────────────────┐          │
│  │   Timer / Signal Engine              │          │
│  │  - Durable Sleep wakeups              │          │
│  │  - Signal mailbox delivery            │          │
│  │  - Call-log record + parent re-dispatch│         │
│  └──────────────┬───────────────────────┘          │
│                 │                                   │
│  ┌──────────────┴───────────────────────┐          │
│  │   Health & Lifecycle Manager         │          │
│  │  - Worker heartbeat monitoring        │          │
│  │  - Lease expiry, fencing, redistribute│          │
│  │  - Attempt records + audit logging    │          │
│  └──────────────────────────────────────┘          │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Concurrency Model**:
- Request handler per API call (goroutine pool)
- Single scheduler loop for activity attempts (leader only in HA setup)
- Timer wheel + signal listener (leader only)
- Per-worker health monitor (one goroutine per worker)
- Event processor for broker messages (worker pool)

**Leader Election** (HA Mode):
- Use etcd or Postgres advisory locks for leader election
- Only leader runs scheduler, timers, signal listener, and health controllers
- Followers handle API requests and forward to state storage
- Leader heartbeat timeout: 10 seconds; failover time: < 30 seconds

---

### 2. Worker

**Responsibilities** (engine-only):
- Register with coordinator on boot and advertise the **activity types** it can execute
- Listen for activity assignments on the broker
- Execute the activity body with resource isolation and context cancellation
- Dispatch children (`RunChild`), persist heartbeat state (`Heartbeat`), and surface deterministic primitives (`Now`, `NewID`, `Sleep`, `AwaitSignal`) via `ActivityContext`
- Report attempt progress and send heartbeats to the coordinator
- Handle graceful shutdown (drain in-flight work)

What an activity does internally (resolve secrets, read a source, call an LLM) is the app's code running on the worker; the engine provides the runtime and the context, not the app logic.

**Internal Modules**:

```
┌─────────────────────────────────────────────────────┐
│              Worker Process                          │
│                                                      │
│  ┌────────────────┐  ┌─────────────────┐           │
│  │ Control Plane  │  │  Registration   │           │
│  │  Listener      │  │   Service       │           │
│  │  (Broker Sub)  │  │  (activity types)│          │
│  └───────┬────────┘  └────────┬────────┘           │
│          │                     │                    │
│          ▼                     ▼                    │
│  ┌──────────────────────────────────────┐          │
│  │   Activity Runtime                   │          │
│  │  - Activity-type registry / dispatch  │          │
│  │  - ActivityContext (RunChild, Sleep,  │          │
│  │    AwaitSignal, Now, NewID, Heartbeat)│          │
│  │  - Resource isolation, cancellation   │          │
│  └──────────────┬───────────────────────┘          │
│                 │                                   │
│  ┌──────────────┴───────────────────────┐          │
│  │   Heartbeat & Health Reporter        │          │
│  │  - Capacity snapshot                  │          │
│  │  - In-flight activity inventory       │          │
│  └──────────────────────────────────────┘          │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Activity Interface** (per [adr/ADR-005-activity-execution-model.md](adr/ADR-005-activity-execution-model.md) — supersedes the v1 `Workload` interface):

```go
// One Activity type. Activities that call ctx.RunChild are de-facto orchestrators;
// those that don't are leaves. The framework treats them uniformly.
type Activity interface {
    Execute(ctx ActivityContext, input []byte) (result []byte, err error)
}

// ActivityContext exposes the durable primitives. Parent code (anything calling
// RunChild/Sleep/AwaitSignal) must be deterministic; all side effects go through RunChild.
type ActivityContext interface {
    RunChild(activityType string, input any) ([]byte, error) // durable suspend point
    Sleep(d time.Duration) error                             // durable timer
    AwaitSignal(name string, timeout time.Duration) ([]byte, error)
    Now() time.Time
    NewID() string
    Heartbeat(state []byte) error // optional intra-attempt checkpoint
    LastHeartbeat() []byte
}
```

There is no `Init`/`Checkpoint`/`Close`: framework-managed retries replace manual retry loops, and replay across `RunChild` (plus optional `Heartbeat`) replaces author-written checkpoint serialization.

**Resource Limits** (per activity attempt):
- CPU: Configurable (default: 1 core)
- Memory: Configurable (default: 512MB)
- Disk I/O / Network: Rate limited (optional)

---

### 3. State Storage

**Technology Decision Matrix**:

| Use Case | PostgreSQL | etcd |
|----------|-----------|------|
| Activity / attempt / call-log / signal state | ✓ (Primary) | ✗ |
| Checkpoints (< 1MB) | ✓ | ✓ |
| Audit logs | ✓ | ✗ |
| Leader election | ✓ (advisory locks) | ✓ (native) |
| Watch/notifications | ✗ (LISTEN/NOTIFY) | ✓ (native) |

**Recommended Approach**: **Hybrid**
- **PostgreSQL**: Primary data store (activity state, attempts, call log, signals, checkpoints, audit). Apps add their own tables here.
- **etcd**: Leader election and watches (optional, if not using Postgres)

**Schema Design**: See [DATA_MODELS.md](DATA_MODELS.md). App tables (connector_*, llmops_*) live in the same Postgres but are owned by their apps and never alter core tables.

**Consistency Model**:
- Strong consistency for activity state transitions and call-log writes (transactions)
- Eventual consistency acceptable for metrics aggregation
- Checkpoint writes are atomic per activity attempt

**Backup Strategy**:
- PostgreSQL: Continuous WAL archiving + daily base backups
- Retention: 30 days
- Point-in-time recovery (PITR) capability

---

### 4. Broker

**Technology Selection**: **NATS**

**Rationale**:
- Lightweight, minimal dependencies
- Native request-reply pattern (for bidirectional communication)
- Subject-based routing (per-worker channels)
- At-least-once delivery with acknowledgments
- Clustering for HA

**Alternative**: Redis Streams (if Redis already in infrastructure)

**Message Channels**:

| Channel Pattern | Direction | Purpose |
|----------------|-----------|---------|
| `coordinator.broadcast` | Coordinator → All Workers | System-wide announcements |
| `worker.{worker_id}.commands` | Coordinator → Specific Worker | Activity assignments, cancel |
| `worker.{worker_id}.status` | Worker → Coordinator | Heartbeats, progress |
| `coordinator.events` | Worker → Coordinator | Registration, completion |

**Message Durability**:
- No persistence required (ephemeral control plane)
- If worker misses message, coordinator will retry based on state storage
- Broker downtime handling: Coordinators and workers reconnect automatically

---

## Data Flow

These are **engine** flows — submission, composition/replay, checkpoint, and redistribution of activities. App-level flows (e.g. the Data Connector's self-orchestrated ingestion loop, where a connection produces ongoing runs) are built from these and documented in [../apps/dataconnector/INGESTION_ARCHITECTURE.md](../apps/dataconnector/INGESTION_ARCHITECTURE.md) §Data Flow.

### Activity Submission Flow

```
Client                Coordinator          State Store         Broker              Worker
  │                       │                     │                 │                   │
  │  POST /activity       │                     │                 │                   │
  ├──────────────────────>│                     │                 │                   │
  │                       │                     │                 │                   │
  │                       │ Validate & Create   │                 │                   │
  │                       │ (status: PENDING)   │                 │                   │
  │                       ├────────────────────>│                 │                   │
  │                       │                     │                 │                   │
  │                       │<────────────────────┤                 │                   │
  │  202 Accepted         │                     │                 │                   │
  │<──────────────────────┤                     │                 │                   │
  │  (activity_id)        │                     │                 │                   │
  │                       │                     │                 │                   │
  │                       │ Scheduler selects   │                 │                   │
  │                       │ target worker       │                 │                   │
  │                       │                     │                 │                   │
  │                       │ Update state        │                 │                   │
  │                       │ (status: SCHEDULED) │                 │                   │
  │                       ├────────────────────>│                 │                   │
  │                       │                     │                 │                   │
  │                       │ Send assignment     │                 │                   │
  │                       ├─────────────────────┼────────────────>│                   │
  │                       │                     │                 │                   │
  │                       │                     │                 │  Start activity   │
  │                       │                     │                 ├──────────────────>│
  │                       │                     │                 │                   │
  │                       │                     │                 │  Ack              │
  │                       │<────────────────────┼─────────────────┤                   │
  │                       │                     │                 │                   │
  │                       │ Update state        │                 │                   │
  │                       │ (status: RUNNING)   │                 │                   │
  │                       ├────────────────────>│                 │                   │
```

### Checkpoint & Progress Flow

```
Worker              Broker            Coordinator       State Store
  │                   │                    │                 │
  │ (activity runs)   │                    │                 │
  │                   │                    │                 │
  │ Create checkpoint │                    │                 │
  │                   │                    │                 │
  │ Publish progress  │                    │                 │
  ├──────────────────>│                    │                 │
  │                   │  Forward           │                 │
  │                   ├───────────────────>│                 │
  │                   │                    │                 │
  │                   │                    │ Store checkpoint│
  │                   │                    ├────────────────>│
  │                   │                    │                 │
  │                   │                    │ Update metrics  │
  │                   │                    ├────────────────>│
```

### Failure & Redistribution Flow

```
Worker A            Broker         Coordinator        State Store       Worker B
  │                   │                 │                  │                │
  │ (stops sending    │                 │                  │                │
  │  heartbeats)      │                 │                  │                │
  │                   │                 │ Detect timeout   │                │
  │                   │                 │ (30s missed)     │                │
  │                   │                 │                  │                │
  │                   │                 │ Mark worker DEAD │                │
  │                   │                 ├─────────────────>│                │
  │                   │                 │                  │                │
  │                   │                 │ Get assigned     │                │
  │                   │                 │ activities       │                │
  │                   │                 │<─────────────────┤                │
  │                   │                 │                  │                │
  │                   │                 │ Update status:   │                │
  │                   │                 │ REDISTRIBUTING   │                │
  │                   │                 ├─────────────────>│                │
  │                   │                 │                  │                │
  │                   │                 │ Find new worker  │                │
  │                   │                 │ (Worker B)       │                │
  │                   │                 │                  │                │
  │                   │                 │ Assign activity  │                │
  │                   │                 ├─────────────────────────────────>│
  │                   │                 │                  │                │
  │                   │                 │                  │   Load last    │
  │                   │                 │                  │<───checkpoint──┤
  │                   │                 │                  │                │
  │                   │                 │                  │   Resume exec  │
  │                   │                 │<─────────────────┼────────────────┤
  │                   │                 │ Update: RUNNING  │                │
  │                   │                 ├─────────────────>│                │
```

---

## State Management Architecture

### State Transition Diagram

```
                           ┌─────────────┐
                           │   PENDING   │ (Initial state)
                           └──────┬──────┘
                                  │
                                  │ Scheduler assigns
                                  ▼
                           ┌─────────────┐
                    ┌─────>│  SCHEDULED  │
                    │      └──────┬──────┘
                    │             │
                    │             │ Worker accepts
                    │             ▼
                    │      ┌─────────────┐
                    │      │   RUNNING   │<────┐
                    │      └──────┬──────┘     │
                    │             │            │ Resume
                    │             │            │
       Cancel       │       ┌─────┴──────┬─────────────┐
       ┌────────────┘       │            │             │
       │              Complete      Pause/Error    Failure
       │                    │            │             │
       ▼                    ▼            ▼             ▼
┌─────────────┐      ┌─────────────┐ ┌──────────┐ ┌──────────────┐
│  CANCELLED  │      │  COMPLETED  │ │  PAUSED  │ │REDISTRIBUTING│
└─────────────┘      └─────────────┘ └────┬─────┘ └──────┬───────┘
                                           │               │
                                           └───────────────┘
                                                   │
                                                Resume on
                                               another worker
```

### State Invariants

1. **Activity Assignment**: An activity attempt in RUNNING state MUST have exactly one assigned worker
2. **Checkpoint Consistency**: Latest checkpoint MUST correspond to a valid execution point
3. **Lease Timeout**: Worker lease expires after 2x heartbeat interval (60s default)
4. **Idempotency**: State transitions MUST be idempotent (safe to replay)

---

## Communication Patterns

### 1. Request-Reply (Synchronous)
**Use Case**: Coordinator → Worker commands that need acknowledgment
- Assign activity
- Cancel activity
- Request capacity report

**Timeout**: 10 seconds
**Retry**: 3 attempts with exponential backoff

### 2. Publish-Subscribe (Asynchronous)
**Use Case**: Worker → Coordinator status updates
- Heartbeats (every 30s)
- Progress reports (on checkpoint, or every 5 minutes)
- Completion notifications

**Delivery Guarantee**: At-least-once

### 3. State-Based (Via State Storage)
**Use Case**: Cross-component coordination
- Scheduler queries pending activity attempts from state storage
- Worker reads activity input and call log from state storage
- Audit trail writes

---

## Deployment Architecture

### 1. Kubernetes Deployment

```yaml
# Coordinator Deployment (Replicas: 3 for HA)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: niyanta-coordinator
spec:
  replicas: 3
  selector:
    matchLabels:
      app: coordinator
  template:
    spec:
      containers:
      - name: coordinator
        image: niyanta-coordinator:latest
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
          limits:
            cpu: 2000m
            memory: 2Gi
        env:
        - name: POSTGRES_URL
          valueFrom:
            secretKeyRef:
              name: niyanta-secrets
              key: postgres-url
        - name: NATS_URL
          value: "nats://nats-service:4222"

---
# Worker Deployment (Horizontal autoscaling)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: niyanta-worker
spec:
  replicas: 5  # Scaled by HPA
  selector:
    matchLabels:
      app: worker
  template:
    spec:
      containers:
      - name: worker
        image: niyanta-worker:latest
        resources:
          requests:
            cpu: 2000m
            memory: 4Gi
          limits:
            cpu: 4000m
            memory: 8Gi

---
# Horizontal Pod Autoscaler for Workers
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: niyanta-worker-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: niyanta-worker
  minReplicas: 3
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

### 2. EC2 / MicroVM Deployment

**Coordinator**:
- Instance Type: t3.medium (2 vCPU, 4GB RAM)
- Count: 3 instances across AZs
- Load Balancer: ALB in front for API traffic
- Auto-recovery: CloudWatch alarms + Auto Scaling Group

**Worker**:
- Instance Type: c5.xlarge or larger (depends on the activity types it runs)
- Count: Auto-scaling group (min: 3, max: 100)
- Scaling Metric: Custom CloudWatch metric from coordinator (queue depth)

**State Storage (PostgreSQL)**:
- RDS Multi-AZ deployment
- Instance: db.m5.large (start), scale up as needed
- Read replicas for query offloading (optional)

**Broker (NATS)**:
- Clustered deployment (3 nodes)
- Instance Type: t3.small (sufficient for control plane traffic)

---

## High Availability & Fault Tolerance

### Coordinator HA

**Leader Election**:
- Use etcd lease or PostgreSQL advisory locks
- Leader acquires lock with TTL (10s)
- Leader renews lease every 5s
- Followers attempt to acquire lock on leader failure

**Split-Brain Prevention**:
- Fencing via generation numbers (stored in state)
- New leader increments generation before scheduling
- Workers reject commands from old generation

**API Availability**:
- All coordinator instances can serve API requests
- Leader-only: Scheduler, health monitor
- Follower failover time: < 30 seconds

### Worker Fault Tolerance

**Activity Isolation**:
- Each activity attempt runs in a separate goroutine with context
- Resource limits enforced via cgroups (Linux) or job objects (Windows)
- Panic recovery to prevent worker crash

**Graceful Shutdown**:
1. Worker receives SIGTERM
2. Stop accepting new activity attempts
3. Send in-flight activity/run list to coordinator
4. Checkpoint and pause all activities
5. Exit after drain timeout (default: 5 minutes)

**Ungraceful Failure**:
- Heartbeat timeout triggers coordinator detection (60s)
- Coordinator marks worker DEAD
- Activity attempts enter REDISTRIBUTING state
- Scheduler finds new workers for reassignment

### State Storage HA

**PostgreSQL**:
- Multi-AZ with synchronous replication
- Automatic failover via RDS (AWS) or Patroni (self-hosted)
- RPO: 0 (synchronous replication)
- RTO: < 2 minutes

**etcd** (if used):
- 3 or 5 node cluster (odd number)
- Raft consensus for consistency
- Survives minority node failures

---

## Security Architecture

### Authentication & Authorization

**API Authentication**:
- Option 1: API keys (for simple deployment)
- Option 2: JWT tokens (for SSO integration)
- Option 3: mTLS (for service-to-service)

**Customer Isolation**:
- Each API call includes `customer_id` (extracted from auth token)
- All database queries filtered by `customer_id`
- Workers do not have direct customer context (prevents cross-customer access)

**Worker Authentication**:
- Workers authenticate to coordinator using pre-shared secrets or mTLS
- Worker identity includes `worker_id` and the activity types it can execute

### Network Security

**Kubernetes**:
- NetworkPolicies to isolate:
  - Coordinators can reach: State storage, broker, workers
  - Workers can reach: Broker, state storage (read-only)
  - External clients can reach: Coordinator API only

**EC2**:
- Security Groups:
  - Coordinator: Inbound 443/8080 (API), Outbound all
  - Worker: Inbound from coordinator only, Outbound all
  - State storage: Inbound 5432/2379 from coordinator/worker, no public access
  - Broker: Inbound 4222 from coordinator/worker

### Secrets Management

**Configuration**:
- Database credentials: Kubernetes Secrets / AWS Secrets Manager
- Broker credentials: Same as above
- Connector secrets: Stored as secret references and resolved inside approved worker runtimes

**Encryption**:
- Data in transit: TLS 1.3 for all component communication
- Data at rest: PostgreSQL TDE or volume encryption

---

## Scalability Model

### Scaling Dimensions

| Component | Scale Trigger | Scale Limit | Notes |
|-----------|---------------|-------------|-------|
| Coordinator | CPU > 70% | 10 instances | Stateless, unlimited in theory |
| Worker | Queue depth > 100 | 500 instances | Depends on activity-type resource needs |
| State Storage | Connections, IOPS | 1 primary + replicas | Vertical scaling needed |
| Broker | Message throughput | 3-node cluster sufficient | Very lightweight |

### Deployment Density And Isolation

Niyanta is designed for dense shared deployments by default. A regional deployment should host many tenants and many activity types while enforcing tenant isolation at the storage, scheduling, secret, network, and observability layers. (Apps add their own density dimensions — the Data Connector counts connector connections; see [../apps/dataconnector/INGESTION_ARCHITECTURE.md](../apps/dataconnector/INGESTION_ARCHITECTURE.md) §Multi-Tenancy.)

Isolation modes:

| Mode | Compute | State | Secrets | Network | Default use |
|------|---------|-------|---------|---------|-------------|
| Shared | Shared worker pool | Shared DB with tenant keys/RLS | Tenant-scoped refs | Shared egress | Standard high-density tenants |
| Pooled | Worker pool per tier/region | Shared DB with tenant keys/RLS | Tenant-scoped refs | Segmented egress | High-volume tenants |
| Dedicated | Tenant-specific workers | Tenant schema/DB optional | Tenant vault namespace | Dedicated egress/VPC | Regulated or high-risk tenants |

Isolation requirements:

- All state rows include tenant scope (`customer_id`).
- Storage APIs enforce tenant filters centrally, at the boundary.
- Secrets are referenced, not copied into activity inputs.
- Scheduler enforces per-tenant concurrency and queue caps.
- Metrics/logs/traces include the tenant dimension without exposing activity payloads.
- A single tenant's activities cannot consume the whole shared worker pool.

### Capacity Planning

**Per-Customer Limits** (engine-level, configurable):
- Max concurrent activities: 100 (default), up to 1000 (premium)
- Max queue depth: 500
- API rate limits: 10 submissions/second
- Isolation mode: shared by default, pooled/dedicated by policy

Apps layer their own limits on top (e.g. max connector connections per tier).

**System-Wide Limits** (initial):
- Total customers: 1000
- Total concurrent activity attempts: 10,000
- Total activity submissions per hour: 100,000
- Worker pool size: 100-500 workers

### Performance Targets

| Metric | Target | Notes |
|--------|--------|-------|
| Activity submission API latency | p95 < 100ms | API response time |
| Scheduling latency | p95 < 5s | Time from PENDING attempt to assigned worker |
| Worker assignment latency | p95 < 10s | Time from scheduled to RUNNING |
| Replay/resume latency | p95 < 2s | Re-dispatch of a parent after a child completes (bounded by call count) |
| Failure detection time | < 60s | Heartbeat timeout |
| Redistribution time | < 2 minutes | From failure to reassignment |
| Checkpoint commit latency | p95 < 1s | Heartbeat persist |

App-level SLOs (connection planner latency, source lag) are defined in the app docs.

---

## Monitoring & Observability

### Key Metrics

These are **engine** metrics. Apps emit their own (e.g. the Data Connector's `niyanta_ingestion_*` series in [../apps/dataconnector/INGESTION_ARCHITECTURE.md](../apps/dataconnector/INGESTION_ARCHITECTURE.md) §Core Metrics; LLMOps' `niyanta_llmops_*` in [../apps/llmops/ARCHITECTURE.md](../apps/llmops/ARCHITECTURE.md) §Observability).

**Coordinator**:
- `niyanta_activities_total` (gauge, by customer, activity_type, status)
- `niyanta_activity_duration_seconds` (histogram, by activity_type)
- `niyanta_scheduler_queue_depth` (gauge, by priority)
- `niyanta_scheduler_dispatch_latency_seconds` (histogram)
- `niyanta_signals_delivered_total` (counter, by name)
- `niyanta_timers_fired_total` (counter)
- `niyanta_worker_pool_size` (gauge, by status: healthy/dead)

**Worker**:
- `niyanta_worker_capacity_total` (gauge)
- `niyanta_worker_capacity_used` (gauge)
- `niyanta_activity_attempts_total` (counter, by activity_type, result)
- `niyanta_run_child_calls_total` (counter, by activity_type)
- `niyanta_activity_replay_duration_seconds` (histogram)
- `niyanta_checkpoint_duration_seconds` (histogram)

**State Storage**:
- Standard PostgreSQL metrics (connections, query latency, replication lag)

### Logging Standards

**Structured Logs** (JSON). The engine emits core fields; apps add their own dimensions (e.g. `connection_id`, `connector_definition_id`) in their activity logs.

```json
{
  "timestamp": "2026-06-03T12:34:56Z",
  "level": "INFO",
  "component": "coordinator",
  "correlation_id": "abc123",
  "customer_id": "customer_456",
  "activity_id": "act_456",
  "activity_type": "ingestion_supervisor",
  "attempt_number": 1,
  "message": "Activity scheduled to worker-5",
  "worker_id": "worker-5"
}
```

**Log Levels**:
- **DEBUG**: Detailed internal state (disabled in production)
- **INFO**: Normal operations (connection/run/activity state changes)
- **WARN**: Recoverable errors (retry attempts)
- **ERROR**: Failures requiring attention

### Alerting

**Critical Alerts** (page on-call) — engine-level:
- Coordinator leader election fails
- Database connection pool exhausted
- Worker failure rate > 10% in 5 minutes
- Activity redistribution failures
- Scheduler dispatch latency far exceeds SLO

**Warning Alerts** (ticket):
- Queue depth > threshold for > 10 minutes
- Replay/resume failure rate > threshold
- API latency p95 > 500ms

Apps define their own domain alerts (connector lag, destination delivery failures, etc.) on their own metrics.

---

## Decision Log

### ADR-001: State Storage Choice
**Decision**: PostgreSQL as primary, etcd optional for leader election
**Rationale**: PostgreSQL handles complex queries (audit logs, activity/attempt/call-log history), provides ACID guarantees, mature backup/restore. etcd better for watches but limited query capability.
**Alternatives Considered**: MongoDB (schemaless but weak consistency), Cassandra (eventual consistency issues)

### ADR-002: Broker Choice
**Decision**: NATS
**Rationale**: Lightweight, native request-reply, subject-based routing, HA clustering. Redis Streams considered but NATS has better operational simplicity.
**Alternatives Considered**: RabbitMQ (heavier), Kafka (overkill for control plane)

### ADR-003: Language Choice
**Decision**: Go
**Rationale**: Excellent concurrency primitives, low memory footprint, fast startup, strong ecosystem for distributed systems.
**Alternatives Considered**: Rust (steeper learning curve), Java (higher memory overhead)

### ADR-004: Checkpoint Storage
**Decision**: Store checkpoints in PostgreSQL (bytea column), limit 10MB per checkpoint
**Rationale**: Simplifies architecture (one storage system), PostgreSQL handles blobs well up to 1GB. For very large checkpoints, offload to S3/blob storage in Phase 2.
**Alternatives Considered**: Separate blob storage (S3) - adds complexity

---

## Open Questions

1. **Intra-tenant priority**: Should activities support priority levels within a single customer? (e.g., critical vs. batch)
2. **Checkpoint compression**: Should checkpoint/heartbeat blobs be compressed automatically?
3. **Multi-region**: How to handle cross-region deployments? (Future phase)
4. **Activity-type versioning**: How to handle activity code updates without disrupting running instances? (Activities run to completion on the version they started; see ADR-005.)

---

## References

- [DATA_MODELS.md](DATA_MODELS.md) - Database schemas and data structures
- [API_SPEC.md](API_SPEC.md) - API contracts and message formats
- [IMPLEMENTATION_PLAN.md](../IMPLEMENTATION_PLAN.md) - Phased delivery roadmap
