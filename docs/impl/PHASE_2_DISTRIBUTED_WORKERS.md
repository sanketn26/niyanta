# Phase 2: Distributed Coordinator And Workers

**Goal**: Split execution into a coordinator and many workers. Use NATS for control messages. Detect worker failure and redistribute activity attempts.

## Files To Add

```text
cmd/coordinator/main.go
cmd/worker/main.go
internal/broker/interfaces.go
internal/broker/nats/event_bus.go
internal/scheduler/scheduler.go
internal/scheduler/affinity.go
internal/worker/runtime.go
internal/coordinator/server.go
```

## Broker Interface

```go
// internal/broker/interfaces.go
package broker

import "context"

type Message struct {
    Subject string
    Data    []byte
}

type Handler func(context.Context, Message) error

type EventBus interface {
    Publish(ctx context.Context, subject string, data []byte) error
    Subscribe(ctx context.Context, subject string, handler Handler) error
    Request(ctx context.Context, subject string, data []byte) ([]byte, error)
}
```

## Worker Registration

```go
type Worker struct {
    ID             string
    Status         string
    CapacityTotal  int
    CapacityUsed   int
    Capabilities   []string
    IsolationPools []string
    LastHeartbeat  time.Time
}
```

Worker startup:

```go
func (w *Runtime) Start(ctx context.Context) error {
    if err := w.registry.Register(ctx, w.info()); err != nil {
        return err
    }

    go w.heartbeatLoop(ctx)

    subject := "worker." + w.id + ".commands"
    return w.bus.Subscribe(ctx, subject, w.handleCommand)
}
```

## Assignment Command

```go
type AssignAttemptCommand struct {
    AttemptID  string `json:"attempt_id"`
    ActivityID string `json:"activity_id"`
    Generation int64  `json:"generation"`
}
```

Workers must reject commands whose generation is stale.

```go
func (w *Runtime) handleAssign(ctx context.Context, cmd AssignAttemptCommand) error {
    activity, err := w.activities.Get(ctx, cmd.ActivityID)
    if err != nil {
        return err
    }
    if activity.Generation != cmd.Generation {
        return ErrStaleGeneration
    }
    go w.executor.RunAttempt(ctx, cmd.AttemptID)
    return nil
}
```

## Scheduler Loop

```go
func (s *Scheduler) runCycle(ctx context.Context) {
    pending, err := s.attempts.ListPending(ctx, s.cfg.BatchSize)
    if err != nil || len(pending) == 0 {
        return
    }

    workers, err := s.workers.ListAvailable(ctx)
    if err != nil || len(workers) == 0 {
        return
    }

    for _, att := range pending {
        activity, err := s.activities.Get(ctx, att.ActivityID)
        if err != nil {
            continue
        }
        worker := s.selectWorker(activity, workers)
        if worker == nil {
            continue
        }
        if err := s.dispatch(ctx, att, activity, worker); err != nil {
            s.logger.Warn("dispatch failed", "attempt_id", att.ID, "err", err)
        }
    }
}
```

## Worker Selection

Worker selection filters before scoring:

```go
func (s *Scheduler) selectWorker(a *models.Activity, workers []*models.Worker) *models.Worker {
    var best *models.Worker
    bestScore := -1

    for _, w := range workers {
        if w.CapacityUsed >= w.CapacityTotal {
            continue
        }
        if !supports(w.Capabilities, a.Type) {
            continue
        }
        if !allowedIsolationPool(w.IsolationPools, a.IsolationPool) {
            continue
        }

        score := w.CapacityTotal - w.CapacityUsed
        if score > bestScore {
            best = w
            bestScore = score
        }
    }
    return best
}
```

## Failure Detection

Coordinator loop:

```go
func (c *Coordinator) detectDeadWorkers(ctx context.Context) error {
    dead, err := c.workers.MarkDeadOlderThan(ctx, time.Now().Add(-60*time.Second))
    if err != nil {
        return err
    }
    for _, w := range dead {
        attempts, err := c.attempts.InterruptRunningByWorker(ctx, w.ID, "worker heartbeat timeout")
        if err != nil {
            return err
        }
        for _, att := range attempts {
            _ = c.retry.EnqueueReplacement(ctx, att)
        }
    }
    return nil
}
```

## Acceptance Tests

1. `docker compose up` starts Postgres, NATS, one coordinator, and two workers.
2. Kill a worker mid-activity; the attempt is interrupted and retried on another worker.
3. Stale generation commands are rejected.
4. A dedicated-isolation activity is only assigned to workers in its isolation pool.

## Done When

- Workers are stateless.
- Coordinator owns scheduling decisions.
- Activity progress survives worker death.

