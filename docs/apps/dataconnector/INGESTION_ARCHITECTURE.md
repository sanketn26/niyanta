# Self-Orchestrated Ingestion Architecture

**Version**: 0.1
**Last Updated**: 2026-05-01
**Status**: Draft

## Overview

Niyanta's primary product shape is a self-orchestrated ingestion platform. The system owns the full ingestion lifecycle: discover enabled connector connections, plan work, execute source reads or receive pushes, normalize records, deliver them to downstream destinations, observe health, adapt scheduling, and recover from failures without operator-driven job submission.

The activity execution engine remains the durability substrate, but ingestion is not modeled as "customers submit arbitrary workloads." Instead, customers register connector connections and ingestion policies. Niyanta continuously decides what should run next.

Core properties:

- **Self-orchestrated**: The platform creates and manages ingestion runs from connector state, source lag, schedules, signals, and backpressure.
- **Multi-tenant by default**: Every connection, run, checkpoint, secret, metric, and destination is scoped to a tenant.
- **High deployment density**: Many tenants and connector connections share the same control plane and worker fleet without requiring one deployment per customer.
- **Isolated execution**: Tenant data, credentials, checkpoints, rate limits, and failure domains are isolated even when compute is shared.
- **Highly observable**: Every source, connection, run, page, transform, destination write, checkpoint, and failure produces metrics, logs, traces, and health state.
- **Scalable**: Work is partitioned by tenant, connector, source account, object prefix, stream partition, time window, and destination capacity.
- **Adaptable**: Connector behavior is mostly declarative, with plugin escape hatches for sources that need custom code.
- **Correct by default**: Checkpoints, leases, dedupe keys, retries, and idempotent delivery are framework-managed where possible.
- **At-least-once ingestion**: Records are checkpointed only after confirmed destination delivery; duplicates are handled through dedupe/idempotency.

## Architectural Shift

The earlier architecture is a generic activity scheduler:

```
external client -> coordinator -> worker -> workload complete
```

The ingestion architecture is a continuous control loop:

```
connector definition + customer connection + source state + destination health
        -> ingestion planner
        -> partitioned ingestion runs
        -> checkpointed delivery
        -> health/lag feedback
        -> adaptive re-planning
```

Activities are still useful, but as internal execution units:

- An ingestion run is an activity attempt.
- A page fetch, object claim, transform batch, or destination flush may be a child activity.
- Durable sleeps and signals drive polling intervals, retries, backfill, and source-triggered wakeups.
- Replay and call logs make long-running ingestion recoverable without connector authors writing checkpoint code for every step.

## High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                         Customer / Operator API                       │
│       definitions, connections, secrets, destinations, backfills       │
└─────────────────────────────┬────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────────┐
│                        Ingestion Control Plane                        │
│ ┌─────────────────┐ ┌─────────────────┐ ┌──────────────────────────┐ │
│ │ Connector        │ │ Ingestion       │ │ Health & Policy          │ │
│ │ Registry         │ │ Planner         │ │ Controller               │ │
│ │ definitions      │ │ lag/schedule    │ │ backoff/quarantine/SLO   │ │
│ └────────┬────────┘ └────────┬────────┘ └────────────┬─────────────┘ │
│          │                   │                       │               │
│ ┌────────▼────────┐ ┌────────▼────────┐ ┌────────────▼─────────────┐ │
│ │ Connection       │ │ Partition       │ │ Backfill & Replay        │ │
│ │ Manager          │ │ Planner         │ │ Manager                  │ │
│ │ config/secrets   │ │ windows/cursors │ │ isolated checkpoints     │ │
│ └────────┬────────┘ └────────┬────────┘ └────────────┬─────────────┘ │
└──────────┼───────────────────┼───────────────────────┼───────────────┘
           │                   │                       │
           ▼                   ▼                       ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    Durable Activity Execution Engine                  │
│ coordinator, scheduler, worker leases, retries, RunChild, Sleep,       │
│ AwaitSignal, call log replay, checkpoints, audit                       │
└─────────────────────────────┬────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────────┐
│                         Connector Runtime Workers                     │
│ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌───────────────┐ │
│ │ HTTP Poller  │ │ Push Gateway │ │ Object Reader │ │ Stream Listener│ │
│ └──────┬───────┘ └──────┬───────┘ └──────┬───────┘ └───────┬───────┘ │
│        │                │                │                 │         │
│ ┌──────▼────────────────▼────────────────▼─────────────────▼───────┐ │
│ │ auth, rate limits, parsing, transforms, dedupe, batching, sinks    │ │
│ └───────────────────────────────────────────────────────────────────┘ │
└──────────────┬──────────────────────────────────────────┬────────────┘
               │                                          │
               ▼                                          ▼
       External Sources                           Downstream Destinations
       APIs, blobs, streams                       streams, webhooks, DBs,
       syslog, agents                             object storage, SIEMs
```

## Control Loops

### 1. Connection Reconciliation Loop

Watches connector definitions, customer connections, secrets, and destination config.

Responsibilities:

- Validate connection config against the connector definition.
- Resolve secret references and detect missing or expired credentials.
- Materialize schedules and source partitions.
- Mark connections `configured`, `connected`, `degraded`, `paused`, `disabled`, or `quarantined`.
- Emit configuration drift and validation errors.

### 2. Ingestion Planning Loop

Continuously decides which ingestion runs should exist.

Inputs:

- Last committed checkpoint.
- Source lag and expected event cadence.
- Poll interval and lookback window.
- Source rate limits.
- Destination backpressure.
- Tenant quotas and priority.
- Backfill requests.
- Health policy, including backoff and quarantine state.

Outputs:

- `ConnectorRun` records.
- Activity attempts for each runnable partition/window.
- Durable timers for the next planned poll.
- Signals to wake blocked ingestion supervisors.

### 3. Adaptive Health Loop

Turns telemetry into scheduling decisions.

Examples:

- Increase poll interval when source returns repeated empty windows.
- Reduce concurrency when a source emits rate-limit headers.
- Split a large backfill into smaller windows when run duration exceeds target.
- Quarantine a connection after repeated auth failures.
- Pause delivery when destination errors exceed policy.
- Prioritize high-lag connections within a customer's quota.
- Apply customer or connection-specific rate-limit overrides for erring connections.
- Respect temporary disables by skipping run planning until the disable expires.

## Runtime Execution Model

Each connector connection has a long-lived supervisor activity:

```go
func IngestionSupervisor(ctx activity.Context, conn ConnectorConnection) error {
    for {
        plan := ctx.RunChild("plan_ingestion", conn.ID)
        for _, partition := range plan.Partitions {
            ctx.RunChild("run_ingestion_partition", partition)
        }
        ctx.Sleep(plan.NextWakeAfter)
    }
}
```

The actual implementation can optimize this shape, but the semantic model is important:

- The supervisor owns durable orchestration.
- Partition activities own source reads and destination delivery.
- Child results and checkpoints allow resume without re-fetching completed work.
- Signals can wake the supervisor for manual backfills, config changes, credential rotation, or push-triggered work.

## Partitioning Model

Partitioning is how the platform scales safely.

Supported partition dimensions:

- Tenant/customer.
- Connector definition and version.
- Connection/account/region.
- Source endpoint.
- Time window.
- Pagination cursor.
- Blob prefix or object key range.
- Stream topic/partition/shard.
- Destination sink.

Partitioning rules:

- A mutable checkpoint namespace has a single writer unless the connector declares partition-safe execution.
- Backfills use isolated checkpoint namespaces so they do not corrupt live ingestion.
- Partition size should be adjusted from observed run duration and record volume.
- Partition concurrency is capped by tenant quota, source rate limit, worker capacity, and destination capacity.

### Stream Subscription Partition Ownership

Kafka, Azure Event Hubs, GCP Pub/Sub, and OCI Streaming are handled as partitioned stream subscriptions. A single connector connection may span many worker processes.

Ownership rules:

- The coordinator is the source of truth for partition ownership.
- Each topic partition, Event Hubs partition, Pub/Sub subscription shard, or OCI stream partition has a lease row.
- A lease has `connection_id`, `partition_id`, `lease_owner`, `generation`, `lease_expires_at`, and the last committed checkpoint.
- Workers consume only partitions assigned by the scheduler.
- Workers heartbeat owned partitions while consuming.
- Checkpoint and source offset/cursor commit happen only after destination delivery succeeds.
- If a worker dies, the coordinator expires its leases and reassigns partitions to other workers.
- If a stale worker resumes, generation fencing prevents it from committing checkpoints or offsets.

This lets one high-volume connection scale across many processes while preserving one active writer per partition.

## Multi-Tenancy, Density, And Isolation

Niyanta should optimize for high deployment density: a small number of shared regional deployments should host many tenants, many connector definitions, and many connector connections. Isolation is enforced by the platform, not by requiring a separate deployment per tenant.

### Tenant Boundaries

Every persisted and emitted object must carry `customer_id` or an equivalent tenant scope:

- connector definitions when private to a tenant
- connector connections
- secret references
- destination definitions
- connector runs
- activity attempts
- checkpoints
- dedupe state
- audit events
- logs, metrics, and traces

Cross-tenant access must fail closed. Query APIs must include tenant filters at the storage boundary, not only in handler code.

### Deployment Density Model

Dense shared deployments are the default:

- One regional control plane can host many tenants.
- One worker pool can execute many tenants' ingestion runs.
- Workers advertise capabilities; the scheduler places runs by capability, tenant quota, source limits, and isolation policy.
- Dedicated worker pools are an exception for regulated tenants, custom network access, noisy sources, or strict data residency.

Density controls:

- Per-tenant concurrency caps.
- Per-tenant queue depth caps.
- Per-source and per-destination rate limits.
- Per-tenant CPU/memory/network budgets.
- Fair scheduling with priority aging.
- Automatic quarantine for failing connections.
- Backpressure before work is admitted.

### Isolation Levels

| Level | Compute | State | Secrets | Network | Use case |
|-------|---------|-------|---------|---------|----------|
| Shared | Shared worker pool | Shared DB with tenant keys/RLS | Tenant-scoped secret refs | Shared egress pools | Default high-density SaaS |
| Pooled | Dedicated worker pool per tenant tier/region | Shared DB with tenant keys/RLS | Tenant-scoped secret refs | Segmented egress | Higher-volume tenants |
| Dedicated | Dedicated workers and optional dedicated DB/schema | Tenant-specific | Tenant-specific vault namespace | Tenant-specific egress/VPC | Regulated or high-risk tenants |

The connector definition should declare the minimum isolation level it supports. A connection may request a stronger isolation level if the customer or source requires it.

### Noisy-Neighbor Protection

The planner and scheduler must prevent one tenant, source, or destination from exhausting shared capacity:

- Tenant quotas cap concurrent runs and queued runs.
- Source rate-limit observations reduce future planning for that connection.
- Destination failures stop checkpoint advancement and reduce flush concurrency.
- Parser/drop storms can quarantine a connection.
- Backfills run in separate checkpoint namespaces and lower priority by default.
- Worker assignment accounts for tenant spread so a single tenant does not occupy an entire pool unless explicitly allowed.

### Customer Ingestion Operations

Operators need a tenant-scoped view and controls for ingestion. For a given `customer_id` (`cid`), Niyanta must expose:

- all connector connections for that customer
- connection health and current control state
- recent significant events
- active rate-limit overrides
- temporary disables and expiry times
- source lag, checkpoint age, records read/emitted/dropped, and latest error
- recent runs and backfills

This view is read-optimized and must be derived from tenant-scoped tables. It must not query across tenants and filter later.

### Significant Event Filters

Significant event filters define which ingestion events should be highlighted, alerted, or retained for operator review. They are scoped by tenant and can optionally target a connector definition or connection.

Examples:

- auth failure for any connection in a customer
- source 429 rate limiting for a specific connector
- parser drop rate above threshold
- destination failures for a connection
- checkpoint age above SLO
- connection enters `failing`, `disabled`, or `quarantined`

Filters do not change ingestion behavior by themselves. They classify events for the ingestion view and alerting. Control policies change behavior.

### Erring Connection Controls

Operators and automated health policies can apply temporary controls to an erring connection:

- **rate-limit override**: lower QPS, burst, or parallel partition count for a connection
- **temporary disable**: stop planning new runs until `expires_at`
- **pause**: operator pause with no automatic expiry
- **resume**: clear pause/temporary disable when safe
- **quarantine**: platform-enforced stop after repeated severe failures

Control precedence:

1. `quarantined`
2. `disabled`
3. `temporary_disable`
4. `paused`
5. `rate_limit_override`
6. connector default policy

The planner must read active controls before creating runs. Workers must still enforce per-request rate limits defensively, but global decisions belong to the planner.

## Data Flow

This is the primary product flow, built on the engine's activity primitives (submit, `RunChild`, `Sleep`, checkpoint, redistribute — see [../../platform/ARCHITECTURE.md](../../platform/ARCHITECTURE.md) §Data Flow). A customer creates a connector connection once; the Data Connector composition owns ongoing run creation, partitioning, checkpointing, retries, backoff, and health transitions.

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

The connection reconciliation, planner, and scheduler shown here are **app activities and app state**, not engine components — the engine sees a supervisor activity that sleeps, wakes, and runs child partition activities.

## Data Plane Stages

Every ingestion run follows the same staged pipeline:

1. **Acquire**: Claim a source partition, load checkpoint, resolve secrets, and acquire a lease.
2. **Read**: Fetch a page, object, stream batch, pushed payload, or plugin-produced batch.
3. **Parse**: Convert JSON, CSV, XML, syslog, CEF, text, or custom formats into records.
4. **Normalize**: Map to a declared output schema and preserve raw source fields when policy allows.
5. **Validate**: Apply schema, required field, timestamp, size, and tenant isolation checks.
6. **Dedupe**: Use source event IDs or deterministic keys with bounded retention.
7. **Deliver**: Write batches to destination with retry and idempotency keys.
8. **Commit**: Persist checkpoint only after delivery succeeds.
9. **Observe**: Emit metrics, logs, traces, health transitions, and audit events.

## Observability Contract

Observability is part of connector correctness, not a Phase 5 add-on.

### Required Dimensions

Every metric/log/span should include, where available:

- `customer_id`
- `connector_definition_id`
- `connector_version`
- `connection_id`
- `run_id`
- `partition_id`
- `source_kind`
- `destination_kind`
- `worker_id`
- `attempt_number`

### Core Metrics

- `niyanta_ingestion_connections_total`
- `niyanta_ingestion_runs_total`
- `niyanta_ingestion_run_duration_seconds`
- `niyanta_ingestion_records_read_total`
- `niyanta_ingestion_records_emitted_total`
- `niyanta_ingestion_records_dropped_total`
- `niyanta_ingestion_bytes_read_total`
- `niyanta_ingestion_source_lag_seconds`
- `niyanta_ingestion_checkpoint_age_seconds`
- `niyanta_ingestion_destination_latency_seconds`
- `niyanta_ingestion_delivery_failures_total`
- `niyanta_ingestion_rate_limit_backoffs_total`
- `niyanta_ingestion_quarantined_connections_total`
- `niyanta_ingestion_planner_cycle_duration_seconds`

### Health State

Connection health is derived from telemetry:

| State | Meaning |
|-------|---------|
| `configured` | Config is valid, no successful run yet |
| `connected` | Recent successful run and checkpoint commit |
| `idle` | Source is healthy but no records are expected or available |
| `lagging` | Successful runs, but lag exceeds policy |
| `degraded` | Partial failures, throttling, parser drops, or destination retries |
| `failing` | Repeated run failures |
| `paused` | Operator or policy pause |
| `disabled` | Config disabled |
| `quarantined` | Automatically stopped to protect the platform or source |

### Traces

Trace hierarchy:

```
connection supervisor
  planner cycle
    connector run
      source request / object read / stream receive
      parse batch
      transform batch
      destination flush
      checkpoint commit
```

Payloads and secrets must be redacted before logs or span attributes are emitted.

## Adaptation Policies

Policies are declarative YAML documents attached to connector definitions or connections.

```yaml
scheduling:
  min_interval_seconds: 60
  max_interval_seconds: 3600
  target_run_duration_seconds: 30
  max_parallel_partitions: 8
source_limits:
  qps: 5
  burst: 10
  respect_retry_after: true
backoff:
  on_empty_window: increase_interval
  on_429: exponential
  on_auth_failure: quarantine_after_threshold
lag:
  warning_seconds: 900
  critical_seconds: 3600
  priority_boost: true
```

The planner reads these policies and changes future run creation. Workers should not make global scheduling decisions; they report observations and enforce local request/delivery limits.

## Scalability Model

Scale comes from independent control and data-plane scaling.

Control plane:

- Coordinator leaders run planner, scheduler, health, and timer loops.
- Followers serve read APIs and can take over with generation fencing.
- Planning loops should shard by customer or connection at high scale.

Data plane:

- Workers advertise connector capabilities, source adapters, CPU/memory, and network budgets.
- Runs are scheduled by capability and quota.
- Polling and object ingestion scale horizontally by partition.
- Push and stream ingestion scale by listener partition, broker subject, or shard ownership.

Initial targets:

- 10,000 enabled connections.
- 100,000 planned runs per hour.
- 10,000 concurrent activity attempts.
- p95 planner decision latency under 10 seconds.
- p95 checkpoint commit latency under 1 second.
- Lag SLO defined per connector and tenant tier.

## Failure Handling

Failure handling is state-driven:

- Worker death interrupts attempts and re-enqueues from last committed checkpoint.
- Source 429 or 503 triggers adaptive backoff and records source health.
- Auth failures stop new source calls and move the connection toward quarantine.
- Parser failures can drop, dead-letter, or fail the run according to policy.
- Destination failures prevent checkpoint commit unless delivery is idempotently confirmed.
- Duplicate execution is tolerated through dedupe and destination idempotency keys.

See [OPERATIONS_AND_FAILURE_SEMANTICS.md](OPERATIONS_AND_FAILURE_SEMANTICS.md) for the exact at-least-once delivery, checkpoint, dead-letter, versioning, secret rotation, backfill, and blast-radius rules.

## Documentation Alignment

Use this document as the north star for ingestion-specific design. The lower-level activity docs remain valid, but should be read as implementation mechanics:

- [ARCHITECTURE.md](../../platform/ARCHITECTURE.md): platform architecture and deployment shape.
- [DATA_CONNECTOR_SPEC.md](DATA_CONNECTOR_SPEC.md): connector definition schema and runtime contract.
- [IMPLEMENTATION_PLAN.md](../../IMPLEMENTATION_PLAN.md): phased build plan.
- [adr/ADR-005-activity-execution-model.md](../../platform/adr/ADR-005-activity-execution-model.md): durable activity substrate.
