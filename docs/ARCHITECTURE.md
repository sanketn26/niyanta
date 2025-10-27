# Niyanta System Architecture

**Version**: 1.0
**Last Updated**: 2025-10-27
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

Niyanta is a horizontally scalable distributed workload processing system designed for long-running, customer-specific jobs. The system provides:

- **Multi-tenant**: Isolated workload execution per customer with tiered SLA support
- **Fault-tolerant**: Checkpoint-based recovery and automatic work redistribution
- **Observable**: Comprehensive metrics, logging, and tracing per customer/workload
- **Flexible**: Support for compile-time and runtime workload registration
- **Deployable**: Runs on EC2/MicroVMs and Kubernetes with minimal external dependencies

---

## System Overview

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         External Clients                             │
│                    (Workload Submission APIs)                        │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             │ HTTPS/gRPC
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      Coordinator Cluster                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │ Coordinator  │  │ Coordinator  │  │ Coordinator  │              │
│  │   (Leader)   │  │  (Follower)  │  │  (Follower)  │              │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘              │
│         │                  │                  │                      │
│         └──────────────────┴──────────────────┘                      │
│                            │                                         │
└────────────────────────────┼─────────────────────────────────────────┘
                             │
                ┌────────────┼────────────┐
                │            │            │
                ▼            ▼            ▼
        ┌───────────┐ ┌───────────┐ ┌───────────┐
        │   Broker  │ │   State   │ │  Metrics  │
        │   (NATS/  │ │  Storage  │ │  (Prom)   │
        │   Redis)  │ │(Postgres/ │ │           │
        │           │ │   etcd)   │ │           │
        └─────┬─────┘ └─────┬─────┘ └───────────┘
              │             │
              │             │
    ┌─────────┴─────────────┴─────────────┐
    │         │             │              │
    ▼         ▼             ▼              ▼
┌────────┐┌────────┐┌────────┐┌────────┐
│Worker 1││Worker 2││Worker 3││Worker N│
│        ││        ││        ││        │
│Workload││Workload││Workload││Workload│
│  A, B  ││  C, D  ││  A, C  ││  B, E  │
└────────┘└────────┘└────────┘└────────┘
```

### Core Components

| Component | Count | Purpose | Stateful |
|-----------|-------|---------|----------|
| Coordinator | 3+ (HA) | Scheduling, lifecycle management, API gateway | No |
| Worker | N (horizontal) | Workload execution | No |
| State Storage | 1 cluster | Persistent state, checkpoints, config | Yes |
| Broker | 1 cluster | Control plane communication | No (ephemeral) |
| Metrics Store | 1 | Observability and monitoring | Yes (time-series) |

---

## Component Architecture

### 1. Coordinator

**Responsibilities**:
- Accept workload submissions via REST/gRPC API
- Schedule workloads to appropriate workers based on affinity, capacity, and SLA
- Monitor worker health via broker heartbeats
- Detect failures and trigger redistribution
- Manage workload lifecycle (start, pause, resume, cancel, complete)
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
│  │      Workload Manager                │          │
│  │  - Validation                        │          │
│  │  - Lifecycle State Machine           │          │
│  └──────────────┬───────────────────────┘          │
│                 │                                   │
│                 ▼                                   │
│  ┌──────────────────────────────────────┐          │
│  │      Scheduler                       │          │
│  │  - Affinity Rules Engine             │          │
│  │  - Capacity Planning                 │          │
│  │  - SLA Priority Queue                │          │
│  └──────────────┬───────────────────────┘          │
│                 │                                   │
│  ┌──────────────┴───────────────────────┐          │
│  │   Worker Health Monitor              │          │
│  │  - Heartbeat tracking                │          │
│  │  - Failure detection                 │          │
│  │  - Redistribution trigger            │          │
│  └──────────────┬───────────────────────┘          │
│                 │                                   │
│  ┌──────────────┴───────────────────────┐          │
│  │   State Manager (Client)             │          │
│  │  - Checkpoint coordination           │          │
│  │  - Audit logging                     │          │
│  └──────────────────────────────────────┘          │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Concurrency Model**:
- Request handler per API call (goroutine pool)
- Single scheduler loop (leader only in HA setup)
- Per-worker health monitor (one goroutine per worker)
- Event processor for broker messages (worker pool)

**Leader Election** (HA Mode):
- Use etcd or Postgres advisory locks for leader election
- Only leader runs scheduler and health monitors
- Followers handle API requests and forward to state storage
- Leader heartbeat timeout: 10 seconds
- Failover time: < 30 seconds

---

### 2. Worker

**Responsibilities**:
- Register with coordinator on boot (advertise capabilities)
- Listen for workload assignments on broker
- Execute workloads with resource isolation
- Report progress and checkpoints periodically
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
│  │   Workload Execution Engine          │          │
│  │  - Plugin loader                     │          │
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
│  │  - Capacity calculation              │          │
│  │  - Workload inventory snapshot       │          │
│  └──────────────────────────────────────┘          │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Workload Plugin Interface**:
```go
type Workload interface {
    // Initialize workload with config and optional checkpoint
    Init(ctx context.Context, config WorkloadConfig, checkpoint []byte) error

    // Execute workload (long-running)
    Execute(ctx context.Context, progressReporter ProgressReporter) error

    // Create checkpoint of current state
    Checkpoint(ctx context.Context) ([]byte, error)

    // Cleanup resources
    Close() error
}
```

**Resource Limits** (per workload):
- CPU: Configurable (default: 1 core)
- Memory: Configurable (default: 512MB)
- Disk I/O: Rate limited
- Network: Rate limited (optional)

---

### 3. State Storage

**Technology Decision Matrix**:

| Use Case | PostgreSQL | etcd |
|----------|-----------|------|
| Workload metadata | ✓ (Primary) | ✗ |
| Checkpoints (< 1MB) | ✓ | ✓ |
| Configuration | ✓ | ✓ |
| Audit logs | ✓ | ✗ |
| Leader election | ✓ (advisory locks) | ✓ (native) |
| Watch/notifications | ✗ (LISTEN/NOTIFY) | ✓ (native) |

**Recommended Approach**: **Hybrid**
- **PostgreSQL**: Primary data store (workloads, checkpoints, audit, config)
- **etcd**: Leader election and configuration watches (optional, if not using Postgres)

**Schema Design**: See [DATA_MODELS.md](DATA_MODELS.md)

**Consistency Model**:
- Strong consistency for workload state transitions (use transactions)
- Eventual consistency acceptable for metrics aggregation
- Checkpoint writes are atomic per workload

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
| `worker.{worker_id}.commands` | Coordinator → Specific Worker | Workload assignments, cancel |
| `worker.{worker_id}.status` | Worker → Coordinator | Heartbeats, progress |
| `coordinator.events` | Worker → Coordinator | Registration, completion |

**Message Durability**:
- No persistence required (ephemeral control plane)
- If worker misses message, coordinator will retry based on state storage
- Broker downtime handling: Coordinators and workers reconnect automatically

---

## Data Flow

### Workload Submission Flow

```
Client                Coordinator          State Store         Broker              Worker
  │                       │                     │                 │                   │
  │  POST /workload       │                     │                 │                   │
  ├──────────────────────>│                     │                 │                   │
  │                       │                     │                 │                   │
  │                       │ Validate & Create   │                 │                   │
  │                       │ (status: PENDING)   │                 │                   │
  │                       ├────────────────────>│                 │                   │
  │                       │                     │                 │                   │
  │                       │<────────────────────┤                 │                   │
  │  202 Accepted         │                     │                 │                   │
  │<──────────────────────┤                     │                 │                   │
  │  (workload_id)        │                     │                 │                   │
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
  │                       │                     │                 │  Start workload   │
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
  │ (workload runs)   │                    │                 │
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
  │                   │                 │ workloads        │                │
  │                   │                 │<─────────────────┤                │
  │                   │                 │                  │                │
  │                   │                 │ Update status:   │                │
  │                   │                 │ REDISTRIBUTING   │                │
  │                   │                 ├─────────────────>│                │
  │                   │                 │                  │                │
  │                   │                 │ Find new worker  │                │
  │                   │                 │ (Worker B)       │                │
  │                   │                 │                  │                │
  │                   │                 │ Assign workload  │                │
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

1. **Workload Assignment**: A workload in RUNNING state MUST have exactly one assigned worker
2. **Checkpoint Consistency**: Latest checkpoint MUST correspond to a valid execution point
3. **Lease Timeout**: Worker lease expires after 2x heartbeat interval (60s default)
4. **Idempotency**: State transitions MUST be idempotent (safe to replay)

---

## Communication Patterns

### 1. Request-Reply (Synchronous)
**Use Case**: Coordinator → Worker commands that need acknowledgment
- Assign workload
- Cancel workload
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
- Scheduler queries pending workloads from state storage
- Worker reads workload config from state storage
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
- Instance Type: c5.xlarge or larger (depends on workload)
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

**Workload Isolation**:
- Each workload runs in separate goroutine with context
- Resource limits enforced via cgroups (Linux) or job objects (Windows)
- Panic recovery to prevent worker crash

**Graceful Shutdown**:
1. Worker receives SIGTERM
2. Stop accepting new workloads
3. Send in-flight workload list to coordinator
4. Checkpoint and pause all workloads
5. Exit after drain timeout (default: 5 minutes)

**Ungraceful Failure**:
- Heartbeat timeout triggers coordinator detection (60s)
- Coordinator marks worker DEAD
- Workloads enter REDISTRIBUTING state
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
- Worker identity includes `worker_id` and supported `workload_types`

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
- Workload-specific secrets: Passed as encrypted blobs, decrypted in worker

**Encryption**:
- Data in transit: TLS 1.3 for all component communication
- Data at rest: PostgreSQL TDE or volume encryption

---

## Scalability Model

### Scaling Dimensions

| Component | Scale Trigger | Scale Limit | Notes |
|-----------|---------------|-------------|-------|
| Coordinator | CPU > 70% | 10 instances | Stateless, unlimited in theory |
| Worker | Queue depth > 100 | 500 instances | Depends on workload resource needs |
| State Storage | Connections, IOPS | 1 primary + replicas | Vertical scaling needed |
| Broker | Message throughput | 3-node cluster sufficient | Very lightweight |

### Capacity Planning

**Per-Customer Limits** (configurable):
- Max concurrent workloads: 100 (default), up to 1000 (premium)
- Max queue depth: 500
- Rate limits: 10 submissions/second

**System-Wide Limits** (initial):
- Total customers: 1000
- Total concurrent workloads: 10,000
- Worker pool size: 100-500 workers

### Performance Targets

| Metric | Target | Notes |
|--------|--------|-------|
| Workload submission latency | p95 < 100ms | API response time |
| Scheduling latency | p95 < 5s | Time from PENDING to SCHEDULED |
| Worker assignment latency | p95 < 10s | Time from SCHEDULED to RUNNING |
| Failure detection time | < 60s | Heartbeat timeout |
| Redistribution time | < 2 minutes | From failure to reassignment |
| Checkpoint frequency | 5-10 minutes | Configurable per workload |

---

## Monitoring & Observability

### Key Metrics

**Coordinator**:
- `niyanta_workload_submissions_total` (counter, by customer, status)
- `niyanta_workload_duration_seconds` (histogram, by customer, workload_type)
- `niyanta_scheduler_queue_depth` (gauge, by priority)
- `niyanta_worker_pool_size` (gauge, by status: healthy/dead)

**Worker**:
- `niyanta_worker_capacity_total` (gauge)
- `niyanta_worker_capacity_used` (gauge)
- `niyanta_workload_executions_total` (counter, by workload_type, result)
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
  "workload_id": "wl_789",
  "message": "Workload scheduled to worker-5",
  "worker_id": "worker-5"
}
```

**Log Levels**:
- **DEBUG**: Detailed internal state (disabled in production)
- **INFO**: Normal operations (workload state changes)
- **WARN**: Recoverable errors (retry attempts)
- **ERROR**: Failures requiring attention

### Alerting

**Critical Alerts** (page on-call):
- Coordinator leader election fails
- Database connection pool exhausted
- Worker failure rate > 10% in 5 minutes
- Workload redistribution failures

**Warning Alerts** (ticket):
- Queue depth > threshold for > 10 minutes
- Checkpoint failure rate > 5%
- API latency p95 > 500ms

---

## Decision Log

### ADR-001: State Storage Choice
**Decision**: PostgreSQL as primary, etcd optional for leader election
**Rationale**: PostgreSQL handles complex queries (audit logs, workload history), provides ACID guarantees, mature backup/restore. etcd better for watches but limited query capability.
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

1. **Workload Priority**: Should we support priority levels within a single customer? (e.g., critical vs. batch)
2. **Checkpoint Compression**: Should checkpoints be compressed automatically?
3. **Multi-Region**: How to handle cross-region deployments? (Future phase)
4. **Workload Versioning**: How to handle workload code updates without disrupting running instances?

---

## References

- [DATA_MODELS.md](DATA_MODELS.md) - Database schemas and data structures
- [API_SPEC.md](API_SPEC.md) - API contracts and message formats
- [IMPLEMENTATION_PLAN.md](IMPLEMENTATION_PLAN.md) - Phased delivery roadmap
