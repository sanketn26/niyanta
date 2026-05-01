# Phase 3: Child Activities And Replay-Based Resume

**Goal**: Implement `ctx.RunChild`. Parent activities suspend durably while children run. On resume, completed child calls replay from the call log.

## Schema

```sql
CREATE TYPE call_status AS ENUM (
    'RECORDED',
    'CHILD_RUNNING',
    'COMPLETED',
    'FAILED'
);

CREATE TABLE activity_calls (
    id                  TEXT PRIMARY KEY,
    parent_activity_id  TEXT NOT NULL REFERENCES activities(id) ON DELETE CASCADE,
    parent_attempt_id   TEXT NOT NULL REFERENCES activity_attempts(id) ON DELETE CASCADE,
    call_index          INT NOT NULL,
    operation           TEXT NOT NULL,
    child_activity_id   TEXT,
    input               JSONB NOT NULL DEFAULT '{}',
    result              JSONB,
    error               TEXT,
    status              call_status NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at        TIMESTAMPTZ,
    UNIQUE(parent_activity_id, call_index)
);

CREATE INDEX idx_activity_calls_parent
    ON activity_calls(parent_activity_id, call_index);
```

## Context State

```go
type RuntimeContext struct {
    context.Context

    activityID string
    attemptID  string
    customerID string

    callIndex int
    replay    []*models.ActivityCall
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
        if logged.Operation != "RunChild" || logged.ActivityType != activityType {
            return nil, DeterminismError{
                Message: "RunChild call does not match recorded history",
            }
        }
        if logged.Status == models.CallCompleted {
            return logged.Result, nil
        }
        if logged.Status == models.CallFailed {
            return nil, errors.New(logged.Error)
        }
    }

    childID := c.ids.New("act")
    call := &models.ActivityCall{
        ID: c.ids.New("call"),
        ParentActivityID: c.activityID,
        ParentAttemptID: c.attemptID,
        CallIndex: idx,
        Operation: "RunChild",
        ActivityType: activityType,
        ChildActivityID: childID,
        Input: input,
        Status: models.CallChildRunning,
    }

    if err := c.calls.RecordAndCreateChild(c.Context, call); err != nil {
        return nil, err
    }

    return nil, SuspendError{Reason: "waiting for child activity"}
}
```

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

## Ingestion Relevance

This phase unlocks the ingestion supervisor shape:

```go
func Supervisor(ctx activity.Context, connID string) ([]byte, error) {
    planBytes, err := ctx.RunChild("plan_ingestion", []byte(connID), activity.ChildOptions{})
    if err != nil {
        return nil, err
    }

    var plan ingest.Plan
    _ = json.Unmarshal(planBytes, &plan)

    for _, p := range plan.Partitions {
        b, _ := json.Marshal(p)
        if _, err := ctx.RunChild("run_ingestion_partition", b, activity.ChildOptions{}); err != nil {
            return nil, err
        }
    }

    return []byte(`{"status":"planned"}`), nil
}
```

## Acceptance Tests

1. Parent with three children completes after four parent attempts.
2. Completed child calls replay without re-running the child.
3. A random branch that changes call order fails with `DeterminismError`.
4. Killing the worker during parent resume does not lose child results.

## Done When

- `RunChild` suspends parent attempts durably.
- Replayed calls return recorded results.
- Child activities run once for each recorded call.

