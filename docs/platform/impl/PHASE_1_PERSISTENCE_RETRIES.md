# Phase 1: Persistence, Framework-Managed Retries, Write-Path Fencing

**Goal**: Move Phase 0 to Postgres and add framework-managed retries. Activity code does not implement retry loops. Establish the **write-path fencing invariant** now, while there is one process, so it is load-bearing before there are two.

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

Two enums from day one — the logical (activity) and physical (attempt) lifecycles are different state machines (see [../DATA_MODELS.md](../DATA_MODELS.md)):

```sql
CREATE TYPE activity_status AS ENUM (
    'PENDING',
    'RUNNING',
    'COMPLETED',
    'FAILED'
    -- SUSPENDED/PAUSED/REDISTRIBUTING/CANCELLED join in later phases
);

CREATE TYPE attempt_status AS ENUM (
    'PENDING',       -- enqueued, claimable
    'RUNNING',
    'COMPLETED',
    'FAILED',
    'INTERRUPTED',   -- crash/restart recovery
    'FENCED'         -- write rejected: stale generation
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
    status          attempt_status NOT NULL,
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
    activity_id     TEXT NOT NULL REFERENCES activities(id) ON DELETE CASCADE,
    attempt_id      TEXT NOT NULL REFERENCES activity_attempts(id) ON DELETE CASCADE,
    sequence_number INT NOT NULL,
    state           JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(activity_id, attempt_id, sequence_number)
);

-- reads are activity-scoped (a retry sees the prior attempt's last heartbeat);
-- writes are attempt-scoped, hence the three-part key
CREATE INDEX idx_heartbeats_latest
    ON activity_heartbeats(activity_id, id DESC);
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

## Write-Path Fencing (G3)

Every state-mutating store method takes the caller's `generation` and issues a conditional write. A failed condition returns a typed `ErrFenced`; the executor treats it as "I am stale — stop."

```go
// internal/storage/errors.go
var ErrFenced = errors.New("write rejected: stale generation")
```

```sql
-- pattern for every mutating statement
UPDATE activity_attempts SET status = 'COMPLETED', ...
WHERE id = $1
  AND EXISTS (SELECT 1 FROM activities WHERE id = $2 AND generation = $3);
-- zero rows => ErrFenced
```

This is a **shared contract test** (same suite runs against the in-memory and Postgres impls, and any future impl): a handle holding a stale generation cannot complete an activity, fail an attempt, write heartbeat state, or renew a lease. No store method is exempt.

## Heartbeat Scope And The Replay Rule (G2)

Heartbeat state is **cross-attempt**: a retry reads the prior attempt's last heartbeat (key the read by `activity_id`, not `attempt_id`; the `UNIQUE(activity_id, attempt_id, sequence_number)` key keeps writes attempt-scoped).

Stated rule, enforced at runtime in Phase 3 when suspend points exist: **`Heartbeat`/`LastHeartbeat` are only legal in activities that never suspend** (`RunChild`/`Sleep`/`AwaitSignal`). Branching on heartbeat state in replayed code is nondeterministic by construction. Document the rule in the `ActivityContext` godoc and ADR-005 now.

## Payload Caps

`input` and `result` are capped (default 256KB, configurable); heartbeat state capped at 10MB. Oversize is a typed error at submit/completion time — never silent truncation. (Child results are re-read on every replay from Phase 3 onward; the cap is what keeps replay cheap.)

## Acceptance Tests

1. `flaky_sleep` fails attempts 1 and 2, then succeeds attempt 3; attempt 3 reads attempt 2's heartbeat state.
2. Kill the process mid-run, restart, and verify a fresh attempt is created.
3. Migration up/down works.
4. Two executor goroutines cannot claim the same attempt.
5. **Fencing contract test**: every mutating store method rejects a stale generation with `ErrFenced` and changes nothing.
6. Oversize input/result rejected with a clear typed error.

## Done When

- Postgres is the primary state store.
- Retry behavior is visible as multiple `activity_attempts` rows.
- No activity implementation contains retry loops.
- No state-mutating write commits without a generation match.

