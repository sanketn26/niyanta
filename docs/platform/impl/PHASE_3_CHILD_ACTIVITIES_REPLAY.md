# Phase 3: Child Activities And Replay-Based Resume

**Goal**: Implement `ctx.RunChild`. Parent activities suspend durably while children run. On resume, completed child calls replay from the call log. Dispatch is **exactly-once**, replay is **version-safe**, and a determinism violation is a recoverable operational event.

## Schema

The call log is keyed by **activity** (+ execution), not by attempt — replay spans attempts. The row records which attempt first made the call (`origin_attempt_id`) for audit, but identity and uniqueness are activity-scoped.

```sql
CREATE TYPE call_status AS ENUM (
    'RECORDED',
    'CHILD_RUNNING',
    'COMPLETED',
    'FAILED'
);

CREATE TABLE activity_calls (
    id                  TEXT PRIMARY KEY,
    activity_id         TEXT NOT NULL REFERENCES activities(id) ON DELETE CASCADE,
    execution_seq       INT NOT NULL DEFAULT 1,   -- ContinueAsNew groundwork (G5); Phase 4 uses it
    call_index          INT NOT NULL,
    operation           TEXT NOT NULL,
    args_hash           TEXT,                     -- detects nondeterministic divergence
    child_activity_id   TEXT,
    origin_attempt_id   TEXT REFERENCES activity_attempts(id) ON DELETE SET NULL,
    input               JSONB NOT NULL DEFAULT '{}',
    result              JSONB,                    -- capped (see Phase 1 payload caps)
    error               TEXT,
    status              call_status NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at        TIMESTAMPTZ,
    UNIQUE(activity_id, execution_seq, call_index)
);

CREATE INDEX idx_activity_calls_replay
    ON activity_calls(activity_id, execution_seq, call_index);
```

Also added to `activities` in this phase's migration: `pinned_version TEXT` (version pinning, below), `execution_seq INT NOT NULL DEFAULT 1`, `superseded_by TEXT` (`ContinueAsNew` linkage, used in Phase 4).

## Context State

```go
type RuntimeContext struct {
    context.Context

    activityID   string
    attemptID    string
    customerID   string
    executionSeq int // current execution (ContinueAsNew increments; Phase 4)

    callIndex int
    replay    []*models.ActivityCall // this execution's log only
    calls     storage.ActivityCallStore
    queue     storage.AttemptStore
}
```

## RunChild Algorithm

```go
func (c *RuntimeContext) RunChild(activityType string, input []byte, opts activity.ChildOptions) ([]byte, error) {
    idx := c.callIndex
    c.callIndex++

    if idx < len(c.replay) {
        logged := c.replay[idx]
        if logged.Operation != "RunChild" || logged.ActivityType != activityType ||
            logged.ArgsHash != hash(input) {
            return nil, DeterminismError{
                CallIndex: idx,
                Message:   "RunChild call does not match recorded history",
            }
        }
        switch logged.Status {
        case models.CallCompleted:
            return logged.Result, nil
        case models.CallFailed:
            return nil, errors.New(logged.Error)
        default:
            // Child dispatched but not finished: re-suspend and WAIT.
            // A logged call is NEVER re-dispatched — this is what makes
            // child dispatch exactly-once (G4).
            return nil, SuspendError{Reason: "waiting for child activity"}
        }
    }

    childID := c.ids.New("act")
    call := &models.ActivityCall{
        ID:              c.ids.New("call"),
        ActivityID:      c.activityID,
        ExecutionSeq:    c.executionSeq,
        CallIndex:       idx,
        Operation:       "RunChild",
        ActivityType:    activityType,
        ArgsHash:        hash(input),
        ChildActivityID: childID,
        OriginAttemptID: c.attemptID,
        Input:           input,
        Status:          models.CallChildRunning,
    }

    // One transaction: append call-log row + create child + suspend parent.
    // If it commits, the child exists exactly once; if not, nothing exists.
    if err := c.calls.RecordAndCreateChild(c.Context, call); err != nil {
        return nil, err
    }

    return nil, SuspendError{Reason: "waiting for child activity"}
}
```

**Contract (G4)**: child *dispatch* is exactly-once — the dispatch transaction is atomic with the call-log append, and replay never re-dispatches a logged call. Side effects *inside* a child remain at-least-once, bounded by the child's own retry policy. (This tightens ADR-005's earlier "children may be invoked twice" wording; update the ADR with this phase.)

## Version-Pinned Replay (G1)

Replay is only meaningful against the code that produced the log. Enforce it at scheduling time:

- Workers advertise `{activity_type: version}` (Phase 2 capabilities).
- The first suspend point stamps `activities.pinned_version` with the executing worker's version for that type.
- Resume attempts are claimable **only** by workers advertising that exact version (extra predicate in the Phase 2 claim query).
- No compatible worker → the activity parks in `SUSPENDED` with condition `no_compatible_worker`, visible in the API and console, and a gauge/alert fires. It resumes automatically when a compatible worker appears. It is never replayed against different code.
- Activities with no suspend history are unpinned and run on any version. `ContinueAsNew` (Phase 4) is the sanctioned point to re-pin onto new code.

## Determinism Violations Are Not Retried (G6)

A replay mismatch (operation, type, or `args_hash` at an index) fails the activity with `error_json.type = "nondeterminism"` and the divergent `call_index`. This error **suppresses automatic retry** — replaying the same code against the same log fails identically, so retrying is wasted work by construction. The state must be fully inspectable (call log + divergence index in the error payload). Operator recovery endpoints (`:force-fail`, `:force-complete`, `:truncate-replay`) land in Phase 5.

## Heartbeat Guard (G2)

The Phase 1 rule becomes enforced: calling `Heartbeat()` or `LastHeartbeat()` in an activity that has any call-log entries returns `ErrHeartbeatWithReplay` immediately. The two resume mechanisms are mutually exclusive per activity.

## Parent Resume

When a child completes:

```go
func (c *Coordinator) onActivityCompleted(ctx context.Context, childID string, result []byte) error {
    call, err := c.calls.GetByChildActivity(ctx, childID)
    if err != nil {
        return err
    }

    return c.tx(ctx, func(ctx context.Context) error {
        if err := c.calls.Complete(ctx, call.ID, result); err != nil {
            return err
        }
        return c.attempts.Create(ctx, &models.Attempt{
            ID: c.ids.New("att"),
            ActivityID: call.ParentActivityID,
            Status: models.ActivityPending,
            Reason: "resume_after_child",
        })
    })
}
```

## Executor Handling For Suspend

```go
result, err := impl.Execute(actx, activity.Input)
switch {
case errors.As(err, &SuspendError{}):
    return e.attempts.MarkWaiting(ctx, att.ID, err.Error())
case err != nil:
    return e.finishFailed(ctx, att, err)
default:
    return e.finishCompleted(ctx, att, result)
}
```

## Sample Composite Activities

Engine samples only — no domain semantics:

- `sequential_pipeline`: parent runs three generic children and combines results.
- `flaky_pipeline`: children fail randomly; the resume mechanism carries the parent through.
- `nondeterministic_bad`: deliberately skips a `RunChild` on replay; used by the violation tests.

## Acceptance Tests

1. Parent with three children completes after four parent attempts.
2. Completed child calls replay without re-running the child.
3. **Exactly-once**: across arbitrary parent worker kills, each recorded call has exactly one child activity row (asserted by counting, not by observing success).
4. A random branch that changes call order fails with `DeterminismError` naming the divergent index — and is **not** retried automatically.
5. Killing the worker during parent resume does not lose child results.
6. **Version pin**: suspend a parent on version `1.0.0`; offer only `1.1.0` workers → parent parks with `no_compatible_worker`, no false violation; a returning `1.0.0` worker completes it.
7. `Heartbeat()` after a `RunChild` returns `ErrHeartbeatWithReplay`.

## Done When

- `RunChild` suspends parent attempts durably.
- Replayed calls return recorded results; logged calls are never re-dispatched.
- Child activities run exactly once per recorded call.
- Replay never runs against a different code version than the one pinned.
- A determinism violation is inspectable and non-retried, awaiting Phase 5 recovery endpoints.

