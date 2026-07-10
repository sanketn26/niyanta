# Phase 2: Distributed Coordinator And Workers

**Goal**: Split execution into a coordinator and many workers. Dispatch is **Postgres-native** — workers pull with `FOR UPDATE SKIP LOCKED`, woken by `LISTEN/NOTIFY`. Detect worker failure, redistribute attempts, and prove that a stale worker **cannot write** after redistribution.

No broker. Postgres remains the only external dependency. The `TaskQueue`/`EventBus` interfaces stay so an alternative can be evaluated after the Phase 9 capacity envelope if measured scale demands it.

## Files To Add

```text
cmd/coordinator/main.go
cmd/worker/main.go
internal/broker/interfaces.go
internal/broker/postgres/notify_bus.go
internal/scheduler/scheduler.go
internal/scheduler/affinity.go
internal/worker/runtime.go
internal/coordinator/server.go
```

## Dispatch Model: Pull, Not Push

Workers pull runnable attempts directly from Postgres, filtered by the activity types they advertise:

```go
const claimSQL = `
WITH next AS (
    SELECT a.id
    FROM activity_attempts a
    JOIN activities act ON act.id = a.activity_id
    WHERE a.status = 'PENDING'
      AND a.not_before <= NOW()
      AND act.type = ANY($1)          -- worker's advertised activity types
    ORDER BY a.not_before, a.id
    LIMIT $2
    FOR UPDATE SKIP LOCKED
)
UPDATE activity_attempts a
SET status = 'RUNNING', worker_id = $3, started_at = NOW(),
    lease_expires_at = NOW() + $4::interval
FROM next WHERE a.id = next.id
RETURNING a.id, a.activity_id, a.number
`
```

Latency: the coordinator (and submit path) issues `NOTIFY task_ready, '<activity_type>'` after enqueueing; idle workers hold a `LISTEN` connection and claim immediately. The poll loop (every 2s) remains as the fallback, so a dropped notification only costs latency, never correctness.

```go
// internal/broker/postgres/notify_bus.go — EventBus over LISTEN/NOTIFY.
// Payloads are hints only; Postgres rows are the source of truth.
```

## Worker Registration And Heartbeats

```go
type Worker struct {
    ID            string
    Status        string            // HEALTHY | DRAINING | DEAD
    CapacityTotal int
    CapacityUsed  int
    Capabilities  map[string]string // activity type -> version
    Tags          map[string]string
    LastHeartbeat time.Time
}
```

Worker startup: register (upsert `workers` row), start the heartbeat loop (every 30s: renew `last_heartbeat`, renew leases for in-flight attempts, report capacity + in-flight inventory), start the claim loop.

## Leases And Fencing

The lease lives on the **attempt** (`activity_attempts.lease_expires_at`), renewed by heartbeat. The fence token lives on the **activity** (`activities.generation`), bumped on every redistribution.

The Phase 1 write-path fencing invariant now gets its distributed acceptance test: after redistribution bumps the generation, **every** write from the old worker — completion, failure, heartbeat state, lease renewal — fails with `ErrFenced` at the storage layer:

```sql
-- every mutating statement carries the caller's generation
UPDATE activity_attempts SET status = 'COMPLETED', ...
WHERE id = $1
  AND EXISTS (SELECT 1 FROM activities
              WHERE id = $2 AND generation = $3);
-- zero rows updated => ErrFenced => worker stops
```

Command-path rejection (a worker ignoring a stale instruction) is necessary but not sufficient; the write path is where split-brain corrupts state.

## Scheduler Responsibilities

With pull-based dispatch, the coordinator's scheduler does **admission and placement policy**, not delivery:

```go
func (s *Scheduler) runCycle(ctx context.Context) {
    // 1. Admission: move PENDING activities to runnable attempts
    //    (quota checks land in Phase 5; Phase 2 admits everything).
    // 2. Affinity: stamp placement constraints onto the attempt row
    //    (hard = SQL filter on claim; soft = score hint; anti = exclusion set).
    // 3. Notify: NOTIFY task_ready for newly runnable work.
}
```

Hard affinity is enforced in the claim query (workers claim only what they are allowed to run); soft affinity is a scoring hint column workers order by; anti-affinity adds an exclusion predicate (needs `GetActivitiesByWorker`).

## Failure Detection

```go
func (c *Coordinator) detectDeadWorkers(ctx context.Context) error {
    dead, err := c.workers.MarkDeadOlderThan(ctx, time.Now().Add(-60*time.Second))
    if err != nil {
        return err
    }
    for _, w := range dead {
        // Interrupt in-flight attempts, bump each activity's generation,
        // enqueue replacement attempts — one transaction per activity.
        attempts, err := c.attempts.InterruptRunningByWorker(ctx, w.ID, "worker heartbeat timeout")
        if err != nil {
            return err
        }
        for _, att := range attempts {
            _ = c.retry.EnqueueReplacement(ctx, att) // bumps generation
        }
    }
    return nil
}
```

A separate sweep redistributes attempts whose lease expired even though the worker still heartbeats (stuck attempt, live worker).

## Graceful Shutdown

SIGTERM → worker sets itself `DRAINING` (stops claiming), finishes or checkpoints in-flight attempts, deregisters. Drain timeout default 5 minutes, then in-flight attempts are released for redistribution.

## Acceptance Tests

1. `docker compose up` starts Postgres, one coordinator, and two workers — nothing else.
2. Kill a worker mid-activity; the attempt is interrupted, generation bumped, and retried on another worker within 60s.
3. **Stale-writer test**: `SIGSTOP` a worker mid-attempt; lease expires; redistribution happens; `SIGCONT` the worker — its completion, heartbeat, and lease-renewal writes all return `ErrFenced`; the redistributed attempt's result is the recorded one.
4. Dispatch latency p95 < 1s submit→RUNNING on an idle system (NOTIFY path, not poll-bound).
5. A hard-affinity activity (e.g. tag `gpu`) is only ever claimed by matching workers.

## Done When

- Workers are stateless and pull their own work.
- Coordinator owns admission, placement policy, and failure detection — not message delivery.
- Activity progress survives worker death.
- No write from a fenced worker can ever land.
