# Phase 1: Persistence And Framework-Managed Retries

**Goal**: Move Phase 0 to Postgres and add framework-managed retries. Activity code does not implement retry loops.

## Files To Add

```text
internal/storage/postgres/store.go
internal/storage/postgres/activity_store.go
internal/storage/postgres/attempt_store.go
internal/retry/policy.go
internal/retry/errors.go
migrations/000001_activity_core.up.sql
migrations/000001_activity_core.down.sql
```

## Schema

```sql
CREATE TYPE activity_status AS ENUM (
    'PENDING',
    'RUNNING',
    'COMPLETED',
    'FAILED',
    'INTERRUPTED'
);

CREATE TABLE activities (
    id              TEXT PRIMARY KEY,
    customer_id     TEXT NOT NULL,
    type            TEXT NOT NULL,
    status          activity_status NOT NULL,
    input           JSONB NOT NULL DEFAULT '{}',
    result          JSONB,
    error           TEXT,
    generation      BIGINT NOT NULL DEFAULT 1,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE activity_attempts (
    id              TEXT PRIMARY KEY,
    activity_id     TEXT NOT NULL REFERENCES activities(id) ON DELETE CASCADE,
    number          INT NOT NULL,
    status          activity_status NOT NULL,
    worker_id       TEXT,
    not_before      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    started_at      TIMESTAMPTZ,
    ended_at        TIMESTAMPTZ,
    error           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(activity_id, number)
);

CREATE INDEX idx_attempts_pending
    ON activity_attempts(not_before, id)
    WHERE status = 'PENDING';

CREATE TABLE activity_heartbeats (
    id              BIGSERIAL PRIMARY KEY,
    attempt_id      TEXT NOT NULL REFERENCES activity_attempts(id) ON DELETE CASCADE,
    state           JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

## Retry Policy

```go
// internal/retry/policy.go
package retry

import "time"

type Policy struct {
    MaxAttempts        int
    InitialInterval    time.Duration
    BackoffCoefficient float64
    MaxInterval        time.Duration
}

func (p Policy) NextDelay(attemptNumber int) time.Duration {
    if attemptNumber <= 1 {
        return p.InitialInterval
    }
    d := float64(p.InitialInterval)
    for i := 1; i < attemptNumber; i++ {
        d *= p.BackoffCoefficient
    }
    if max := float64(p.MaxInterval); d > max {
        d = max
    }
    return time.Duration(d)
}
```

```go
// internal/retry/errors.go
package retry

type NonRetryableError struct{ Err error }

func (e NonRetryableError) Error() string { return e.Err.Error() }
func (e NonRetryableError) Unwrap() error { return e.Err }

func IsNonRetryable(err error) bool {
    _, ok := err.(NonRetryableError)
    return ok
}
```

## Claim Query

Use `FOR UPDATE SKIP LOCKED` so multiple executors can poll safely later.

```go
const claimNextSQL = `
WITH next AS (
    SELECT id
    FROM activity_attempts
    WHERE status = 'PENDING' AND not_before <= NOW()
    ORDER BY not_before, id
    LIMIT 1
    FOR UPDATE SKIP LOCKED
)
UPDATE activity_attempts a
SET status = 'RUNNING', started_at = NOW()
FROM next
WHERE a.id = next.id
RETURNING a.id, a.activity_id, a.number, a.status, a.worker_id, a.not_before, a.started_at, a.ended_at, a.error
`
```

## Attempt Completion Logic

```go
func (e *Executor) finishFailed(ctx context.Context, att *models.Attempt, err error) error {
    policy := e.registry.RetryPolicy(att.ActivityType)
    if retry.IsNonRetryable(err) || att.Number >= policy.MaxAttempts {
        if ferr := e.attempts.Fail(ctx, att.ID, err.Error()); ferr != nil {
            return ferr
        }
        return e.activities.UpdateStatus(ctx, att.ActivityID, models.ActivityFailed, nil, err.Error())
    }

    next := &models.Attempt{
        ID:         e.ids.New("att"),
        ActivityID: att.ActivityID,
        Number:     att.Number + 1,
        Status:     models.ActivityPending,
        NotBefore:  time.Now().Add(policy.NextDelay(att.Number)),
    }

    return e.tx(ctx, func(ctx context.Context) error {
        if err := e.attempts.Fail(ctx, att.ID, err.Error()); err != nil {
            return err
        }
        return e.attempts.Create(ctx, next)
    })
}
```

## Crash Recovery

On executor startup:

```sql
UPDATE activity_attempts
SET status = 'INTERRUPTED', ended_at = NOW(), error = 'executor restarted'
WHERE status = 'RUNNING' AND worker_id = $1
RETURNING activity_id, number;
```

For each interrupted attempt, enqueue a new attempt unless the retry policy is exhausted.

## Acceptance Tests

1. `flaky_sleep` fails attempts 1 and 2, then succeeds attempt 3.
2. Kill the process mid-run, restart, and verify a fresh attempt is created.
3. Migration up/down works.
4. Two executor goroutines cannot claim the same attempt.

## Done When

- Postgres is the primary state store.
- Retry behavior is visible as multiple `activity_attempts` rows.
- No activity implementation contains retry loops.

