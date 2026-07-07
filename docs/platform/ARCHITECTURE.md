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

Everything else — any domain product, pipeline, or control loop — is **not the platform**. Those are *compositions of activities* that consumers build on Niyanta. The platform does not know what an activity "means"; it knows how to run it durably.

Niyanta provides:

- **A runtime**: Executes activities and their composed child activities to completion across a distributed worker fleet.
- **Execution guarantees**: Durability, replay-based resume across `RunChild` boundaries, framework-enforced retries, optional checkpointing, leases, and fencing — applied to an activity *and* its sub-activities. Authors do not write retry loops or serialize state.
- **State**: Durable storage of activity state, attempts, call logs, signal mailboxes, and checkpoints.
- **Parallel execution**: Many independent activities run concurrently across workers; the engine handles dispatch, redistribution, and scale-out. (Within a single parent, `RunChild` calls are sequential — see [adr/ADR-005-activity-execution-model.md](adr/ADR-005-activity-execution-model.md).)
- **Coordination**: Scheduling by capability, capacity, affinity, and priority; leader election; durable timers (`Sleep`) and external-event signals (`AwaitSignal`).
- **Isolation**: Per-tenant boundaries over every activity, its state, and its observability dimensions, with shared / pooled / dedicated execution modes.

The rest is plumbing.

### What Is And Is Not Niyanta's Concern

Niyanta's concern is exactly one thing: **run an activity, give it execution guarantees, and hold enough state to resume it if it gets stuck or its worker crashes.** That is the whole contract — runtime, guarantees, state for resume, coordination, isolation. Niyanta is agnostic to *what* an activity does and to *what config drove it*.

Anything built on the engine — any product that wires activity types together, possibly under its own higher-level interface or config language — is a **consumer** of this contract, not part of it. Such a layer owns its own config, parser, state, and API, all built on unchanged platform primitives; to the engine, that layer's config is an opaque payload inside an activity's `input_json`. Niyanta does not grow features to "enable" layers above it, does not know they exist, and never parses their configuration. This is what keeps the engine small and lets consumers evolve freely without any platform change — the platform was never looking.

---

## System Overview

### High-Level Architecture

This is the **engine**. It runs activities; consumers register activity types and submit work. Nothing below is specific to any domain.

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
       │ Dispatch     │ │ State Store  │ │ Observability│
       │ Postgres     │ │ Postgres     │ │ Metrics/Logs │
       │ SKIP LOCKED  │ │ + optional   │ │ Traces       │
       │ + NOTIFY     │ │ etcd         │ │ (console)    │
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

> What runs *inside* an activity is the consumer's code. The engine sees only "activity of type X with opaque input, running on worker W."

### Core Components

| Component | Count | Purpose | Stateful |
|-----------|-------|---------|----------|
| Coordinator | 3+ (HA) | API, activity scheduling, timers/signals, lifecycle management | No |
| Worker | N (horizontal) | Execute any registered activity type; dispatch children; heartbeat | No |
| State Storage | 1 cluster | Activities, attempts, call log, signals, checkpoints, audit | Yes |
| Broker (optional, Phase 7) | 1 cluster | Drop-in `TaskQueue`/`SignalBus` substrate at scale; not on the critical path | Optional |
| Observability Store | 1+ | Metrics, logs, traces | Yes |

Domain-specific control planes built by consumers are **not** engine components — they are activity compositions plus their own state, layered on the above and documented wherever those products live.

---

## Component Architecture

### 1. Coordinator

**Responsibilities** (engine-only):
- Accept activity submissions, signals, lifecycle operations, and worker-admin API calls
- Schedule activity attempts to workers by activity-type capability, affinity, capacity, tenant quota, and SLA priority
- Drive durable timers (`Sleep`) and deliver signals (`AwaitSignal`) to suspended activities
- Record the call log and re-dispatch parents for replay-based resume after a child completes
- Monitor worker health via heartbeat rows in the state store; detect failure and trigger redistribution with fencing
- Manage activity lifecycle (schedule, suspend, resume, pause, cancel, complete)
- Expose metrics and health endpoints

The coordinator does **not** interpret activity payloads or make domain decisions — it schedules and resumes activities; the activities decide what to do.

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
- `LISTEN` connection + event processor for control-plane notifications (worker pool)

**Leader Election** (HA Mode):
- Use etcd or Postgres advisory locks for leader election
- Only leader runs scheduler, timers, signal listener, and health controllers
- Followers handle API requests and forward to state storage
- Leader heartbeat timeout: 10 seconds; failover time: < 30 seconds

---

### 2. Worker

**Responsibilities** (engine-only):
- Register in the state store on boot and advertise the **activity types** (with versions) it can execute
- Claim activity attempts from the state store (`FOR UPDATE SKIP LOCKED` pull, woken by `LISTEN/NOTIFY`)
- Execute the activity body with resource isolation and context cancellation
- Dispatch children (`RunChild`), persist heartbeat state (`Heartbeat`), and surface deterministic primitives (`Now`, `NewID`, `Sleep`, `AwaitSignal`) via `ActivityContext`
- Report attempt progress and send heartbeats to the coordinator
- Handle graceful shutdown (drain in-flight work)

What an activity does internally is the consumer's code running on the worker; the engine provides the runtime and the context, not the business logic.

**Internal Modules**:

```
┌─────────────────────────────────────────────────────┐
│              Worker Process                          │
│                                                      │
│  ┌────────────────┐  ┌─────────────────┐           │
│  │ Claim Loop     │  │  Registration   │           │
│  │ (SKIP LOCKED + │  │   Service       │           │
│  │  LISTEN wakeup)│  │ (types+versions)│           │
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

**Schema Design**: See [DATA_MODELS.md](DATA_MODELS.md). Consumers may add their own tables in the same Postgres, but they never alter core engine tables.

**Consistency Model**:
- Strong consistency for activity state transitions and call-log writes (transactions)
- Eventual consistency acceptable for metrics aggregation
- Checkpoint writes are atomic per activity attempt

**Backup Strategy**:
- PostgreSQL: Continuous WAL archiving + daily base backups
- Retention: 30 days
- Point-in-time recovery (PITR) capability

---

### 4. Control-Plane Communication (Dispatch)

**Technology Selection**: **Postgres-native dispatch** — no broker on the critical path.

**Rationale** (minimal external dependencies):
- Workers **pull** runnable attempts with `SELECT … FOR UPDATE SKIP LOCKED`, filtered by their advertised activity types — safe concurrent claiming with no extra infrastructure.
- `LISTEN/NOTIFY` wakes idle workers so dispatch latency is not bounded by the poll interval; the poll loop remains as fallback, so a dropped notification costs latency, never correctness.
- Registration, heartbeats, and completions are rows in the state store — one source of truth, no state/broker reconciliation problem.

**Optional broker (Phase 7)**: NATS JetStream implementations of the `TaskQueue` and `SignalBus` interfaces exist as drop-in substitutes if measured Postgres dispatch throughput demands them. Adopting them is a configuration change, not an activity-code change.

**Durability model**:
- Notifications are hints only; Postgres rows are the source of truth.
- A worker that misses a notification claims the work on its next poll tick.

---

## Data Flow

These are **engine** flows — submission, composition/replay, checkpoint, and redistribution of activities. Anything domain-shaped is built from these primitives by consumers.

### Activity Submission Flow (pull-based dispatch)

```
Client                Coordinator             State Store (Postgres)        Worker
  │                       │                          │                        │
  │  POST /activities     │                          │                        │
  ├──────────────────────>│                          │                        │
  │                       │ Validate & create        │                        │
  │                       │ activity + attempt       │                        │
  │                       │ (status: PENDING)        │                        │
  │                       ├─────────────────────────>│                        │
  │  202 Accepted         │                          │                        │
  │<──────────────────────┤   NOTIFY task_ready      │                        │
  │  (activity_id)        ├─────────────────────────>│ ── notification ──────>│
  │                       │                          │                        │
  │                       │                          │  Claim attempt         │
  │                       │                          │  (FOR UPDATE SKIP      │
  │                       │                          │   LOCKED, capability + │
  │                       │                          │   affinity + version   │
  │                       │                          │   predicates)          │
  │                       │                          │<───────────────────────┤
  │                       │                          │  status: RUNNING,      │
  │                       │                          │  lease set, worker_id  │
  │                       │                          │───────────────────────>│
  │                       │                          │                        │
  │                       │                          │      (executes)        │
```

> No push, no ack protocol: claiming the row *is* the assignment. A worker that misses the `NOTIFY` claims on its next poll tick (2s fallback).

### Checkpoint & Progress Flow

```
Worker                      State Store (Postgres)          Coordinator
  │                                │                             │
  │ (activity runs)                │                             │
  │                                │                             │
  │ Heartbeat: renew lease,        │                             │
  │ write checkpoint row           │                             │
  │ (WHERE generation = $expected) │                             │
  ├───────────────────────────────>│                             │
  │                                │   Health monitor reads      │
  │                                │   worker/lease freshness    │
  │                                │<────────────────────────────┤
```

### Failure & Redistribution Flow

```
Worker A               Coordinator          State Store (Postgres)       Worker B
  │                        │                       │                        │
  │ (stops heartbeating)   │                       │                        │
  │                        │ Detect stale          │                        │
  │                        │ heartbeat/lease (60s) │                        │
  │                        ├──────────────────────>│                        │
  │                        │ Mark worker DEAD;     │                        │
  │                        │ attempts INTERRUPTED; │                        │
  │                        │ bump generation;      │                        │
  │                        │ enqueue new attempts; │                        │
  │                        │ NOTIFY task_ready     │                        │
  │                        ├──────────────────────>│ ── notification ──────>│
  │                        │                       │   Claim attempt        │
  │                        │                       │<───────────────────────┤
  │                        │                       │   Load checkpoint /    │
  │                        │                       │   call log, resume     │
  │                        │                       │───────────────────────>│
  │ (wakes up later)       │                       │                        │
  │ write with old         │                       │                        │
  │ generation ──ErrFenced─┼──────────────────────>│ ✗ zero rows updated    │
```

---

## State Management Architecture

### State Transition Diagram

```
                           ┌─────────────┐
                           │   PENDING   │ (initial; also after resume/wake)
                           └──────┬──────┘
                                  │ Scheduler admits
                                  ▼
                           ┌─────────────┐
                    ┌─────>│  SCHEDULED  │<───────────────────────┐
                    │      └──────┬──────┘                        │
                    │             │ Worker claims                 │
                    │             ▼                               │
                    │      ┌─────────────┐   RunChild/Sleep/      │
                    │      │   RUNNING   │──AwaitSignal──┐        │
                    │      └──┬───┬───┬──┘               ▼        │
                    │         │   │   │           ┌───────────┐   │
       Cancel       │         │   │   │           │ SUSPENDED │───┘
   (any non-        │         │   │   │           └───────────┘ child done /
    terminal state) │         │   │   │            holds no      timer fired /
       ┌────────────┘         │   │   │            worker slot   signal arrived
       │             Complete │   │   │ Retries exhausted or
       │              ┌───────┘   │   │ nondeterminism (no auto-retry)
       │              │     Pause │   └──────────────┐
       │              │           │                  │
       │              │           │      Worker died / lease expired
       │              │           │                  │
       ▼              ▼           ▼                  ▼
┌─────────────┐ ┌───────────┐ ┌────────┐   ┌──────────────┐   ┌────────┐
│  CANCELLED  │ │ COMPLETED │ │ PAUSED │   │REDISTRIBUTING│   │ FAILED │
└─────────────┘ └───────────┘ └───┬────┘   └──────┬───────┘   └────────┘
                                  │ Resume        │ New attempt,
                                  └───────────────┴─> bumped generation,
                                                      back to SCHEDULED
```

### State Invariants

1. **Activity Assignment**: An activity attempt in RUNNING state MUST have exactly one assigned worker
2. **Checkpoint Consistency**: Latest checkpoint MUST correspond to a valid execution point
3. **Lease Timeout**: Worker lease expires after 2x heartbeat interval (60s default)
4. **Idempotency**: State transitions MUST be idempotent (safe to replay)

---

## Communication Patterns

All coordinator↔worker communication is **state-based with notification hints** — Postgres rows are the messages; `LISTEN/NOTIFY` only reduces latency.

### 1. State-Based (Via State Storage) — the primary pattern
**Use Case**: Everything that must be correct
- Dispatch: workers claim `PENDING` attempts (`FOR UPDATE SKIP LOCKED`, filtered by capability/version/affinity) — claiming the row *is* the assignment
- Completion/failure: workers write results with generation-fenced conditional updates
- Heartbeats (every 30s): workers renew `last_heartbeat` and attempt leases
- Cancellation/pause: coordinator flags the row; workers observe on heartbeat or via notification
- Audit trail writes

**Delivery Guarantee**: transactional — the write either committed or it didn't; no message/state reconciliation problem.

### 2. Notification Hints (LISTEN/NOTIFY)
**Use Case**: Latency only — never correctness
- `task_ready` wakes idle workers to claim immediately
- `signal_arrived` wakes the coordinator's signal delivery
- `control_changed` prompts workers to re-read cancellation/pause flags

A dropped notification costs at most one poll interval (2s dispatch, 30s control), never a lost operation.

### 3. Optional Broker Substrate (Phase 7)
If JetStream `TaskQueue`/`SignalBus` implementations are adopted at scale, they replace pattern 2's transport behind the same interfaces; pattern 1 remains the source of truth.

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

**Broker (optional, Phase 7 only)**:
- Only deployed if JetStream substrates are adopted at scale; not part of the baseline footprint
- Clustered deployment (3 nodes), t3.small sufficient for control-plane traffic

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
  - Coordinators can reach: State storage
  - Workers can reach: State storage (their scoped role), plus whatever egress their activity types need
  - Console can reach: Coordinator API, plus coordinator/worker `/metrics` (scrape)
  - External clients can reach: Coordinator API and console only
  - (Phase 7 only, if adopted) broker reachable from coordinator/worker

**EC2**:
- Security Groups:
  - Coordinator: Inbound 443/8080 (API) + `/metrics` from console, Outbound all
  - Worker: Inbound `/metrics` from console only, Outbound all
  - State storage: Inbound 5432/2379 from coordinator/worker, no public access
  - Console: Inbound 443 (UI), Outbound to coordinator/workers/state
  - (Phase 7 only, if adopted) Broker: Inbound 4222 from coordinator/worker

### Secrets Management

**Configuration**:
- Database credentials: Kubernetes Secrets / AWS Secrets Manager
- Broker credentials (Phase 7 only, if adopted): Same as above
- Activity secrets: passed as secret references in `input_json` (never values) and resolved inside approved worker runtimes

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
| State Storage | Transitions/sec, connections, IOPS | 1 primary + replicas | The engine's scaling unit; also carries dispatch |
| Broker (optional, Phase 7) | Message throughput | 3-node cluster sufficient | Only if JetStream substrates adopted |

### Deployment Density And Isolation

Niyanta is designed for dense shared deployments by default. A regional deployment should host many tenants and many activity types while enforcing tenant isolation at the storage, scheduling, secret, network, and observability layers.

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

> The meaningful load unit for capacity planning is **state transitions/sec** (each suspend point is several transactions), not concurrent-activity count. The engine's transition-throughput envelope is published as part of its contract (Phase 7 load tests).

---

## Monitoring & Observability

### Key Metrics

These are **engine** metrics, exposed as Prometheus `/metrics` on every component and consumed by the Activity Manager console's integrated metrics store (Phase 6) or any external Prometheus. Consumers emit their own domain metrics under their own prefixes.

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
- `niyanta_fenced_writes_total` (counter — the audit trail of redistribution races)
- `niyanta_parked_no_compatible_worker` (gauge — activities pinned to a version no worker advertises)
- `niyanta_nondeterminism_failures_total` (counter)

**State Storage**:
- Standard PostgreSQL metrics (connections, query latency, replication lag)

### Logging Standards

**Structured Logs** (JSON). The engine emits core fields; consumers add their own dimensions in their activity logs.

```json
{
  "timestamp": "2026-06-03T12:34:56Z",
  "level": "INFO",
  "component": "coordinator",
  "correlation_id": "abc123",
  "customer_id": "customer_456",
  "activity_id": "act_456",
  "activity_type": "sample_pipeline",
  "attempt_number": 1,
  "message": "Activity scheduled to worker-5",
  "worker_id": "worker-5"
}
```

**Log Levels**:
- **DEBUG**: Detailed internal state (disabled in production)
- **INFO**: Normal operations (activity/attempt state changes)
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
- Parked `no_compatible_worker` activities > 0 for > 15 minutes
- Any `niyanta_nondeterminism_failures_total` increase

Alert rules ship as code (`deploy/prometheus/alerts.yaml`) and are evaluated by the console's integrated store out of the box (Phase 6). Consumers define their own domain alerts on their own metrics.

---

## Decision Log

### ADR-001: State Storage Choice
**Decision**: PostgreSQL as primary, etcd optional for leader election
**Rationale**: PostgreSQL handles complex queries (audit logs, activity/attempt/call-log history), provides ACID guarantees, mature backup/restore. etcd better for watches but limited query capability.
**Alternatives Considered**: MongoDB (schemaless but weak consistency), Cassandra (eventual consistency issues)

### ADR-002: Dispatch / Broker Choice (revised 2026-07-06)
**Decision**: Postgres-native dispatch (`FOR UPDATE SKIP LOCKED` pull + `LISTEN/NOTIFY` wakeups); no broker on the critical path. NATS JetStream remains an optional Phase 7 substrate behind the `TaskQueue`/`SignalBus` interfaces.
**Rationale**: The state store is the source of truth either way; a broker added a stateful dependency and a second dispatch path without a correctness gain. Minimal-external-dependencies principle wins until measured throughput says otherwise.
**Alternatives Considered**: NATS (original v1 choice — deferred, not rejected), RabbitMQ (heavier), Kafka (overkill for control plane), Redis Streams

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
4. ~~**Activity-type versioning**~~ *Resolved*: suspended activities are **version-pinned** — they resume only on workers advertising the version recorded at first suspend; `ContinueAsNew` is the sanctioned re-pin point; drain tooling retires old versions (see IMPLEMENTATION_PLAN G1).

---

## References

- [DATA_MODELS.md](DATA_MODELS.md) - Database schemas and data structures
- [API_SPEC.md](API_SPEC.md) - API contracts and message formats
- [IMPLEMENTATION_PLAN.md](../IMPLEMENTATION_PLAN.md) - Phased delivery roadmap
