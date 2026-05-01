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

Niyanta is a self-orchestrated ingestion platform built on a durable distributed activity engine. Customers register connector connections and ingestion policies; the platform continuously plans, executes, observes, adapts, and recovers ingestion without requiring operators to submit individual jobs.

The system provides:

- **Self-orchestrated ingestion**: Connector supervisors, planners, timers, and signals decide what source work should run next.
- **Multi-tenant isolation**: Per-customer connector connections, quotas, rate limits, destinations, and SLA tiers.
- **High deployment density**: Shared regional control planes and worker pools host many tenants and connector connections efficiently.
- **Configurable isolation levels**: Shared, pooled, or dedicated execution depending on tenant, connector, network, and compliance requirements.
- **Fault-tolerant execution**: Activity replay, checkpoint-based recovery, leases, fencing, and automatic redistribution.
- **Highly observable operations**: Metrics, logs, traces, health state, source lag, checkpoint age, and destination delivery visibility per connection/run.
- **Scalable data plane**: Partitioned ingestion by tenant, connector, account, region, source partition, time window, object prefix, cursor, and destination.
- **Adaptable connectors**: Declarative connector definitions for common sources, with plugin escape hatches for custom protocols.

See [INGESTION_ARCHITECTURE.md](INGESTION_ARCHITECTURE.md) for the ingestion-specific control loops and runtime model.

---

## System Overview

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Customers / Operators / Sources                   │
│ definitions, connections, secrets, backfills, push events, signals    │
└────────────────────────────┬────────────────────────────────────────┘
                             │ HTTPS/gRPC
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                 Coordinator + Ingestion Control Plane                │
│ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ │
│ │ API Server   │ │ Connector    │ │ Ingestion    │ │ Health &     │ │
│ │              │ │ Registry     │ │ Planner      │ │ Policy Ctrl  │ │
│ └──────┬───────┘ └──────┬───────┘ └──────┬───────┘ └──────┬───────┘ │
│        │                │                │                │         │
│ ┌──────▼────────────────▼────────────────▼────────────────▼───────┐ │
│ │ Durable activity scheduler, timers, signals, leases, fencing      │ │
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
│                       Connector Runtime Workers                      │
│ ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌───────────────────┐ │
│ │ HTTP Poll  │ │ Push       │ │ Object     │ │ Stream / Plugin   │ │
│ │ Runtime    │ │ Gateway    │ │ Reader     │ │ Runtime           │ │
│ └─────┬──────┘ └─────┬──────┘ └─────┬──────┘ └─────────┬─────────┘ │
│       │              │              │                  │           │
│ ┌─────▼──────────────▼──────────────▼──────────────────▼─────────┐ │
│ │ auth, request, paging, parsing, transform, dedupe, sink, commit  │ │
│ └─────────────────────────────────────────────────────────────────┘ │
└──────────────┬──────────────────────────────────────┬───────────────┘
               │                                      │
               ▼                                      ▼
        External Sources                      Downstream Destinations
```

### Core Components

| Component | Count | Purpose | Stateful |
|-----------|-------|---------|----------|
| Coordinator | 3+ (HA) | API, ingestion planning, activity scheduling, lifecycle management | No |
| Connector Registry | 1 logical service | Versioned connector definitions, schemas, policies, capabilities | Yes |
| Connection Manager | 1 logical service | Customer connector instances, config validation, secret refs, destinations | Yes |
| Ingestion Planner | Leader-only loop, shardable later | Creates runs from schedules, lag, backfills, source state, and policies | No |
| Worker | N (horizontal) | Dense multi-tenant connector runtime execution and delivery | No |
| State Storage | 1 cluster | Persistent state, checkpoints, connector config, audit | Yes |
| Broker | 1 cluster | Control plane communication and optional durable signal substrate | Optional |
| Observability Store | 1+ | Metrics, logs, traces, health timelines | Yes |

---

## Component Architecture

### 1. Coordinator

**Responsibilities**:
- Accept connector definitions, connection configuration, backfill requests, signals, and operational API calls
- Reconcile connector connections and validate configuration, secret references, and destinations
- Plan ingestion runs from schedules, source lag, checkpoint state, backfills, and health policy
- Schedule activity attempts to appropriate workers based on connector capabilities, affinity, capacity, tenant quota, and SLA
- Monitor worker health via broker heartbeats
- Detect failures and trigger redistribution
- Manage connector, run, and activity lifecycle (start, pause, resume, cancel, complete, quarantine)
- Expose metrics and health endpoints

**Internal Modules**:

```
┌─────────────────────────────────────────────────────┐
│             Coordinator Process                      │
│                                                      │
│  ┌────────────────┐  ┌─────────────────┐           │
│  │   API Server   │  │  Authentication │           │
│  │  (REST/gRPC)   │  │   & Authorization│          │
│  └───────┬────────┘  └────────┬────────┘           │
│          │                     │                    │
│          ▼                     ▼                    │
│  ┌──────────────────────────────────────┐          │
│  │      Connection Manager              │          │
│  │  - Connector config validation       │          │
│  │  - Secret and destination refs       │          │
│  └──────────────┬───────────────────────┘          │
│                 │                                   │
│                 ▼                                   │
│  ┌──────────────────────────────────────┐          │
│  │      Ingestion Planner               │          │
│  │  - Schedules, lag, backfills         │          │
│  │  - Adaptive source policies          │          │
│  └──────────────┬───────────────────────┘          │
│                 │                                   │
│  ┌──────────────┴───────────────────────┐          │
│  │      Scheduler                       │          │
│  │  - Capability and affinity rules     │          │
│  │  - Capacity and quota planning       │          │
│  │  - SLA priority queue                │          │
│  └──────────────┬───────────────────────┘          │
│                 │                                   │
│  ┌──────────────┴───────────────────────┐          │
│  │   Health & State Manager             │          │
│  │  - Worker/source/destination health  │          │
│  │  - Checkpoints and audit logging     │          │
│  └──────────────────────────────────────┘          │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Concurrency Model**:
- Request handler per API call (goroutine pool)
- Connection reconciliation loop (leader only initially; shardable by customer/connection)
- Ingestion planning loop (leader only initially; shardable at high scale)
- Single scheduler loop for activity attempts (leader only in HA setup)
- Per-worker health monitor (one goroutine per worker)
- Source and destination health policy evaluators
- Event processor for broker messages (worker pool)

**Leader Election** (HA Mode):
- Use etcd or Postgres advisory locks for leader election
- Only leader runs planner, scheduler, timers, signal listener, and health controllers
- Followers handle API requests and forward to state storage
- Leader heartbeat timeout: 10 seconds
- Failover time: < 30 seconds

---

### 2. Worker

**Responsibilities**:
- Register with coordinator on boot and advertise connector runtime capabilities
- Listen for ingestion activity assignments on broker
- Execute connector runtime stages with resource isolation
- Resolve secrets through approved providers, never through activity input payloads
- Read from sources, parse, transform, dedupe, deliver, and commit checkpoints
- Report progress, source lag, delivery state, and checkpoints periodically
- Handle graceful shutdown (drain in-flight work)
- Send heartbeats to coordinator

**Internal Modules**:

```
┌─────────────────────────────────────────────────────┐
│              Worker Process                          │
│                                                      │
│  ┌────────────────┐  ┌─────────────────┐           │
│  │ Control Plane  │  │  Registration   │           │
│  │  Listener      │  │   Service       │           │
│  │  (Broker Sub)  │  │                 │           │
│  └───────┬────────┘  └────────┬────────┘           │
│          │                     │                    │
│          ▼                     ▼                    │
│  ┌──────────────────────────────────────┐          │
│  │   Connector Runtime Engine           │          │
│  │  - Declarative/runtime plugin loader │          │
│  │  - Resource isolation (cgroups)      │          │
│  │  - Context cancellation              │          │
│  └──────────────┬───────────────────────┘          │
│                 │                                   │
│                 ▼                                   │
│  ┌──────────────────────────────────────┐          │
│  │   Checkpoint Manager                 │          │
│  │  - Periodic checkpoint creation      │          │
│  │  - State serialization               │          │
│  │  - Recovery on startup               │          │
│  └──────────────┬───────────────────────┘          │
│                 │                                   │
│  ┌──────────────┴───────────────────────┐          │
│  │   Heartbeat & Health Reporter        │          │
│  │  - Capacity and source lag snapshot  │          │
│  │  - In-flight run inventory           │          │
│  └──────────────────────────────────────┘          │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Activity Plugin Interface**:
```go
type Activity interface {
    // Initialize activity with config and optional checkpoint
    Init(ctx context.Context, config ActivityConfig, checkpoint []byte) error

    // Execute activity (long-running)
    Execute(ctx context.Context, progressReporter ProgressReporter) error

    // Create checkpoint of current state
    Checkpoint(ctx context.Context) ([]byte, error)

    // Cleanup resources
    Close() error
}
```

**Resource Limits** (per activity/run):
- CPU: Configurable (default: 1 core)
- Memory: Configurable (default: 512MB)
- Disk I/O: Rate limited
- Network: Rate limited (optional)

---

### 3. State Storage

**Technology Decision Matrix**:

| Use Case | PostgreSQL | etcd |
|----------|-----------|------|
| Connector/activity metadata | ✓ (Primary) | ✗ |
| Checkpoints (< 1MB) | ✓ | ✓ |
| Configuration | ✓ | ✓ |
| Audit logs | ✓ | ✗ |
| Leader election | ✓ (advisory locks) | ✓ (native) |
| Watch/notifications | ✗ (LISTEN/NOTIFY) | ✓ (native) |

**Recommended Approach**: **Hybrid**
- **PostgreSQL**: Primary data store (connector config, activity state, checkpoints, audit)
- **etcd**: Leader election and configuration watches (optional, if not using Postgres)

**Schema Design**: See [DATA_MODELS.md](DATA_MODELS.md)

**Consistency Model**:
- Strong consistency for connector run and activity state transitions (use transactions)
- Eventual consistency acceptable for metrics aggregation
- Checkpoint writes are atomic per activity/run

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

### Self-Orchestrated Ingestion Flow

```
Operator/API        Coordinator          State Store       Planner/Scheduler       Worker          Destination
    │                   │                    │                    │                  │                  │
    │ Create connection │                    │                    │                  │                  │
    ├──────────────────>│                    │                    │                  │                  │
    │                   │ Validate config    │                    │                  │                  │
    │                   │ and secret refs    │                    │                  │                  │
    │                   ├───────────────────>│                    │                  │                  │
    │                   │ Reconcile enabled connection            │                  │                  │
    │                   ├────────────────────────────────────────>│                  │                  │
    │                   │                    │                    │ Plan windows,    │                  │
    │                   │                    │                    │ partitions, lag  │                  │
    │                   │                    │<───────────────────┤                  │                  │
    │                   │                    │ Create ConnectorRun│                  │                  │
    │                   │                    │ and activity       │                  │                  │
    │                   │                    │<───────────────────┤                  │                  │
    │                   │                    │                    │ Assign attempt   │                  │
    │                   │                    │                    ├─────────────────>│                  │
    │                   │                    │                    │                  │ Read source,     │
    │                   │                    │                    │                  │ parse, transform │
    │                   │                    │                    │                  │ dedupe, batch    │
    │                   │                    │                    │                  ├─────────────────>│
    │                   │                    │                    │                  │ Delivery ack     │
    │                   │                    │                    │                  │<─────────────────┤
    │                   │                    │ Commit checkpoint  │                  │                  │
    │                   │                    │<───────────────────────────────────────┤                  │
    │                   │                    │                    │ Health feedback  │                  │
    │                   │<───────────────────────────────────────────────────────────┤                  │
    │                   │ Adapt next poll/backfill/concurrency    │                  │                  │
```

This is the primary product flow. A customer creates a connector connection once; Niyanta owns ongoing run creation, partitioning, checkpointing, retries, backoff, and health transitions.

### Activity Submission Flow

This lower-level flow still exists for generic activity execution and internal ingestion activities. It is no longer the primary ingestion user experience.

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
- Worker reads activity/connector config from state storage
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
- Instance Type: c5.xlarge or larger (depends on connector runtime)
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
- Worker identity includes `worker_id` and supported activity and connector runtime types

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
| Worker | Queue depth > 100 | 500 instances | Depends on connector runtime resource needs |
| State Storage | Connections, IOPS | 1 primary + replicas | Vertical scaling needed |
| Broker | Message throughput | 3-node cluster sufficient | Very lightweight |

### Deployment Density And Isolation

Niyanta is designed for dense shared deployments by default. A regional deployment should host many tenants, connector definitions, and connector connections while enforcing tenant isolation at the storage, scheduling, secret, network, and observability layers.

Isolation modes:

| Mode | Compute | State | Secrets | Network | Default use |
|------|---------|-------|---------|---------|-------------|
| Shared | Shared worker pool | Shared DB with tenant keys/RLS | Tenant-scoped refs | Shared egress | Standard high-density tenants |
| Pooled | Worker pool per tier/region | Shared DB with tenant keys/RLS | Tenant-scoped refs | Segmented egress | High-volume tenants |
| Dedicated | Tenant-specific workers | Tenant schema/DB optional | Tenant vault namespace | Dedicated egress/VPC | Regulated or high-risk tenants |

Isolation requirements:

- All state rows include tenant scope.
- Storage APIs enforce tenant filters centrally.
- Secrets are referenced, not copied into activity inputs.
- Scheduler enforces per-tenant concurrency and queue caps.
- Metrics/logs/traces include tenant and connection dimensions without exposing payloads.
- Backfills and failing connections cannot consume the whole shared worker pool.

### Capacity Planning

**Per-Customer Limits** (configurable):
- Max enabled connector connections: tier-based
- Max concurrent ingestion runs: 100 (default), up to 1000 (premium)
- Max queue depth: 500
- API rate limits: 10 submissions/second
- Source and destination rate limits: connector-policy based
- Isolation mode: shared by default, pooled/dedicated by policy

**System-Wide Limits** (initial):
- Total customers: 1000
- Total enabled connector connections: 10,000
- Total planned runs per hour: 100,000
- Total concurrent activity attempts: 10,000
- Worker pool size: 100-500 workers

### Performance Targets

| Metric | Target | Notes |
|--------|--------|-------|
| Connection API latency | p95 < 100ms | API response time |
| Planner decision latency | p95 < 10s | Time from due connection to planned run |
| Scheduling latency | p95 < 5s | Time from PENDING attempt to assigned worker |
| Worker assignment latency | p95 < 10s | Time from planned run to RUNNING |
| Failure detection time | < 60s | Heartbeat timeout |
| Redistribution time | < 2 minutes | From failure to reassignment |
| Checkpoint commit latency | p95 < 1s | After destination delivery ack |
| Source lag | Connector SLO | Policy-defined by connector and tenant tier |

---

## Monitoring & Observability

### Key Metrics

**Coordinator**:
- `niyanta_ingestion_connections_total` (gauge, by customer, connector, status)
- `niyanta_ingestion_planner_cycle_duration_seconds` (histogram)
- `niyanta_ingestion_runs_total` (counter, by customer, connector, result)
- `niyanta_ingestion_run_duration_seconds` (histogram, by customer, connector)
- `niyanta_scheduler_queue_depth` (gauge, by priority)
- `niyanta_worker_pool_size` (gauge, by status: healthy/dead)

**Worker**:
- `niyanta_worker_capacity_total` (gauge)
- `niyanta_worker_capacity_used` (gauge)
- `niyanta_activity_attempts_total` (counter, by activity_type, result)
- `niyanta_ingestion_records_read_total` (counter, by connector)
- `niyanta_ingestion_records_emitted_total` (counter, by connector, destination)
- `niyanta_ingestion_records_dropped_total` (counter, by connector, reason)
- `niyanta_ingestion_source_lag_seconds` (gauge, by connection)
- `niyanta_ingestion_checkpoint_age_seconds` (gauge, by connection)
- `niyanta_ingestion_delivery_failures_total` (counter, by destination)
- `niyanta_ingestion_rate_limit_backoffs_total` (counter, by source)
- `niyanta_checkpoint_duration_seconds` (histogram)

**State Storage**:
- Standard PostgreSQL metrics (connections, query latency, replication lag)

### Logging Standards

**Structured Logs** (JSON):
```json
{
  "timestamp": "2025-10-27T12:34:56Z",
  "level": "INFO",
  "component": "coordinator",
  "correlation_id": "abc123",
  "customer_id": "customer_456",
  "connection_id": "conn_789",
  "connector_definition_id": "github_audit",
  "run_id": "run_123",
  "activity_id": "act_456",
  "message": "Ingestion run scheduled to worker-5",
  "worker_id": "worker-5"
}
```

**Log Levels**:
- **DEBUG**: Detailed internal state (disabled in production)
- **INFO**: Normal operations (connection/run/activity state changes)
- **WARN**: Recoverable errors (retry attempts)
- **ERROR**: Failures requiring attention

### Alerting

**Critical Alerts** (page on-call):
- Coordinator leader election fails
- Database connection pool exhausted
- Worker failure rate > 10% in 5 minutes
- Ingestion redistribution failures
- Critical connector lag exceeds SLO
- Destination delivery failures exceed policy

**Warning Alerts** (ticket):
- Queue depth > threshold for > 10 minutes
- Checkpoint failure rate > 5%
- API latency p95 > 500ms
- Connection enters degraded/failing/quarantined state

---

## Decision Log

### ADR-001: State Storage Choice
**Decision**: PostgreSQL as primary, etcd optional for leader election
**Rationale**: PostgreSQL handles complex queries (audit logs, connector/run/activity history), provides ACID guarantees, mature backup/restore. etcd better for watches but limited query capability.
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

1. **Ingestion Priority**: Should we support priority levels within a single customer? (e.g., critical vs. batch)
2. **Checkpoint Compression**: Should checkpoints be compressed automatically?
3. **Multi-Region**: How to handle cross-region deployments? (Future phase)
4. **Connector/Activity Versioning**: How to handle connector or activity code updates without disrupting running instances?

---

## References

- [DATA_MODELS.md](DATA_MODELS.md) - Database schemas and data structures
- [API_SPEC.md](API_SPEC.md) - API contracts and message formats
- [IMPLEMENTATION_PLAN.md](IMPLEMENTATION_PLAN.md) - Phased delivery roadmap
