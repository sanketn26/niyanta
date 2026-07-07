# Phase 4: Durable Timers, Deterministic Primitives, Signals, ContinueAsNew

**Goal**: Activities can sleep without occupying workers, wake from external events, replay deterministic values, and run for unbounded lifetimes with **bounded replay cost** (`ContinueAsNew`). Cancellation of suspended activities has defined semantics.

## Schema

```sql
CREATE TABLE pending_timers (
    id              TEXT PRIMARY KEY,
    activity_id     TEXT NOT NULL REFERENCES activities(id) ON DELETE CASCADE,
    call_id         TEXT NOT NULL REFERENCES activity_calls(id) ON DELETE CASCADE,
    wake_at         TIMESTAMPTZ NOT NULL,
    fired_at        TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_pending_timers_due
    ON pending_timers(wake_at)
    WHERE fired_at IS NULL;

CREATE TABLE signals (
    id              TEXT PRIMARY KEY,
    activity_id     TEXT NOT NULL REFERENCES activities(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    payload         JSONB NOT NULL DEFAULT '{}',
    dedupe_key      TEXT,                          -- sender-supplied Idempotency-Key
    delivered_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_signals_mailbox
    ON signals(activity_id, name, created_at)
    WHERE delivered_at IS NULL;

-- a retried external send with the same key is one mailbox entry
CREATE UNIQUE INDEX idx_signals_dedupe
    ON signals(activity_id, dedupe_key)
    WHERE dedupe_key IS NOT NULL;
```

## Deterministic `Now` And `NewID`

Record synthetic call-log entries.

```go
func (c *RuntimeContext) Now() time.Time {
    value, err := c.recordDeterministic("_now", nil, func() []byte {
        return []byte(time.Now().UTC().Format(time.RFC3339Nano))
    })
    if err != nil {
        panic(err)
    }
    t, _ := time.Parse(time.RFC3339Nano, string(value))
    return t
}

func (c *RuntimeContext) NewID() string {
    value, err := c.recordDeterministic("_id", nil, func() []byte {
        return []byte(c.ids.New("det"))
    })
    if err != nil {
        panic(err)
    }
    return string(value)
}
```

## Durable Sleep

```go
func (c *RuntimeContext) Sleep(d time.Duration) error {
    idx := c.callIndex
    c.callIndex++

    if logged := c.replayed(idx, "_sleep"); logged != nil {
        if logged.Status == models.CallCompleted {
            return nil
        }
        return SuspendError{Reason: "sleep still pending"}
    }

    wakeAt := c.Now().Add(d)
    call := models.ActivityCall{
        ID: c.ids.New("call"),
        ParentActivityID: c.activityID,
        ParentAttemptID: c.attemptID,
        CallIndex: idx,
        Operation: "_sleep",
        Input: mustJSON(map[string]any{"wake_at": wakeAt}),
        Status: models.CallRecorded,
    }

    if err := c.calls.RecordSleep(c.Context, call, wakeAt); err != nil {
        return err
    }
    return SuspendError{Reason: "sleep"}
}
```

Timer loop:

```go
func (c *Coordinator) timerLoop(ctx context.Context) {
    ticker := time.NewTicker(time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            timers, _ := c.timers.ClaimDue(ctx, time.Now(), 100)
            for _, timer := range timers {
                _ = c.calls.Complete(ctx, timer.CallID, []byte(`{}`))
                _ = c.attempts.Create(ctx, resumeAttempt(timer.ActivityID))
            }
        }
    }
}
```

## Signal Bus

```go
type SignalBus interface {
    Send(ctx context.Context, activityID, name string, payload []byte) error
    AwaitForActivity(ctx context.Context, activityID, name string) ([]byte, error)
}
```

Postgres send:

```go
func (b *PostgresSignalBus) Send(ctx context.Context, activityID, name string, payload []byte) error {
    _, err := b.db.Exec(ctx, `
        INSERT INTO signals(id, activity_id, name, payload)
        VALUES ($1, $2, $3, $4)
    `, b.ids.New("sig"), activityID, name, payload)
    if err != nil {
        return err
    }
    _, err = b.db.Exec(ctx, `SELECT pg_notify('signal_arrived', $1)`, activityID)
    return err
}
```

Await signal:

```go
func (c *RuntimeContext) AwaitSignal(name string, timeout time.Duration) ([]byte, error) {
    idx := c.callIndex
    c.callIndex++

    if logged := c.replayed(idx, "_await_signal"); logged != nil {
        if logged.Status == models.CallCompleted {
            return logged.Result, nil
        }
        return nil, SuspendError{Reason: "await_signal"}
    }

    if sig, ok := c.signals.TryClaim(c.Context, c.activityID, name); ok {
        return sig.Payload, c.calls.RecordCompleted(c.Context, idx, "_await_signal", sig.Payload)
    }

    callID := c.ids.New("call")
    if err := c.calls.RecordAwaitSignal(c.Context, callID, c.activityID, c.attemptID, idx, name, timeout); err != nil {
        return nil, err
    }
    return nil, SuspendError{Reason: "await_signal"}
}
```

## Synthetic-Entry Batching

`Now()`/`NewID()` record per call, but consecutive synthetic entries between real suspend points are buffered and flushed in one batch at the next durable write — a loop calling `Now()` must not issue one Postgres write per iteration.

## ContinueAsNew (G5)

The primitive that bounds replay history for unbounded-lifetime activities. Schema groundwork (`execution_seq`, `superseded_by`) landed in Phase 3.

```go
// Completes the current execution and atomically starts a successor with a
// fresh call log. Never returns on success (the attempt ends).
func (c *RuntimeContext) ContinueAsNew(input []byte) error
```

One transaction:

1. Mark current execution `COMPLETED` with `result = {"continued_as_new": true}`.
2. Increment `activities.execution_seq`; reset status to `PENDING` with the new input.
3. Carry over **undelivered** signals (mailbox survives the boundary).
4. Clear `pinned_version` — the successor may pin onto new code (the sanctioned upgrade point).

Preserved across the boundary: `activity_id` (signals keep routing), parent/root composition links, the execution chain for audit (`execution_seq` on every call/attempt row). Replay always loads only the **current** execution's call log.

## Cancellation Semantics

`:cancel` on an activity:

1. Cancel in-flight children depth-first (each child gets its own cancellation).
2. Re-dispatch a suspended parent once: its pending suspend-point call (`RunChild`/`Sleep`/`AwaitSignal`) returns `ErrCancelled`, giving the body one bounded chance to finish cleanup **using already-recorded state only** — new dispatches after cancellation are rejected.
3. Transition to `CANCELLED`; record child accounting in `result`.

## Acceptance Tests

1. `Sleep(48h)` creates a timer and frees the worker.
2. Fast-forward timer wakes the activity and resumes execution.
3. Signal sent before `AwaitSignal` is not lost.
4. Signal sent while no worker owns the activity wakes it.
5. `Now` and `NewID` replay the same values; a 1,000-iteration `Now()` loop produces O(suspend points) writes, not O(iterations).
6. Duplicate signal send with the same `Idempotency-Key` → one mailbox entry.
7. **ContinueAsNew**: an activity with 500 logged calls continues-as-new; the successor's first resume replays ~0 entries; signals to the same ID still arrive; the execution chain links both.
8. Cancel a parent suspended on `AwaitSignal` with one running child → child cancelled, parent's call site returns `ErrCancelled`, no new dispatches, terminal `CANCELLED`.

## Done When

- Long waits cost no worker capacity.
- Signals are durable and dedupe-able at the door.
- Replay is deterministic across suspend points.
- Replay cost is bounded by the current execution's call count, regardless of activity lifetime.
- Cancellation leaves no orphaned children.

