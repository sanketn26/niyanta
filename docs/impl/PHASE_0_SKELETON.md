# Phase 0: End-to-End Skeleton, In-Memory Only

**Goal**: One binary, one process, one activity, in-memory state. The system can submit an activity, execute it, and report completion.

**End result**:

```bash
go run ./cmd/niyanta demo sleep 2s
```

prints an activity ID, runs the activity, and exits with `COMPLETED`.

## Files To Create

```text
go.mod
cmd/niyanta/main.go
pkg/models/activity.go
pkg/activity/activity.go
pkg/activity/context.go
internal/storage/interfaces.go
internal/storage/memory/store.go
internal/registry/registry.go
internal/executor/executor.go
internal/samples/sleep.go
```

## Domain Models

```go
// pkg/models/activity.go
package models

import "time"

type ActivityStatus string

const (
    ActivityPending   ActivityStatus = "PENDING"
    ActivityRunning   ActivityStatus = "RUNNING"
    ActivityCompleted ActivityStatus = "COMPLETED"
    ActivityFailed    ActivityStatus = "FAILED"
)

type Activity struct {
    ID         string
    Type       string
    CustomerID string
    Input      []byte
    Result     []byte
    Error      string
    Status     ActivityStatus
    CreatedAt  time.Time
    UpdatedAt  time.Time
}

type Attempt struct {
    ID         string
    ActivityID string
    Number     int
    Status     ActivityStatus
    StartedAt  *time.Time
    EndedAt    *time.Time
}
```

## Activity API

```go
// pkg/activity/activity.go
package activity

import (
    "context"
    "time"
)

type Activity interface {
    Execute(ctx Context, input []byte) ([]byte, error)
}

type Factory func() Activity

type Context interface {
    context.Context
    ActivityID() string
    CustomerID() string

    Heartbeat(state []byte) error

    RunChild(activityType string, input []byte, opts ChildOptions) ([]byte, error)
    Sleep(d time.Duration) error
    Now() time.Time
    NewID() string
    AwaitSignal(name string, timeout time.Duration) ([]byte, error)
}

type ChildOptions struct{}
```

Phase 0 stubs `RunChild`, `Sleep`, `Now`, `NewID`, and `AwaitSignal`. They return `ErrNotImplemented` until later phases.

```go
// pkg/activity/context.go
package activity

import (
    "context"
    "errors"
    "time"
)

var ErrNotImplemented = errors.New("activity primitive not implemented in this phase")

type RuntimeContext struct {
    context.Context
    activityID string
    customerID string
}

func NewContext(ctx context.Context, activityID, customerID string) *RuntimeContext {
    return &RuntimeContext{Context: ctx, activityID: activityID, customerID: customerID}
}

func (c *RuntimeContext) ActivityID() string { return c.activityID }
func (c *RuntimeContext) CustomerID() string { return c.customerID }
func (c *RuntimeContext) Heartbeat(state []byte) error { return nil }
func (c *RuntimeContext) RunChild(string, []byte, ChildOptions) ([]byte, error) { return nil, ErrNotImplemented }
func (c *RuntimeContext) Sleep(time.Duration) error { return ErrNotImplemented }
func (c *RuntimeContext) Now() time.Time { return time.Now().UTC() }
func (c *RuntimeContext) NewID() string { return "" }
func (c *RuntimeContext) AwaitSignal(string, time.Duration) ([]byte, error) { return nil, ErrNotImplemented }
```

## Storage Interfaces

```go
// internal/storage/interfaces.go
package storage

import (
    "context"
    "github.com/your-org/niyanta/pkg/models"
)

type ActivityStore interface {
    CreateActivity(ctx context.Context, activity *models.Activity) error
    Get(ctx context.Context, id string) (*models.Activity, error)
    UpdateStatus(ctx context.Context, id string, status models.ActivityStatus, result []byte, errText string) error
}

type AttemptStore interface {
    CreateAttempt(ctx context.Context, attempt *models.Attempt) error
    ClaimNext(ctx context.Context) (*models.Attempt, error)
    Complete(ctx context.Context, attemptID string, result []byte) error
    Fail(ctx context.Context, attemptID string, errText string) error
}
```

## In-Memory Store

```go
// internal/storage/memory/store.go
package memory

import (
    "context"
    "errors"
    "sync"

    "github.com/your-org/niyanta/pkg/models"
)

type Store struct {
    mu         sync.Mutex
    activities map[string]*models.Activity
    attempts   map[string]*models.Attempt
    queue      []string
}

func NewStore() *Store {
    return &Store{
        activities: map[string]*models.Activity{},
        attempts:   map[string]*models.Attempt{},
    }
}

func (s *Store) CreateActivity(ctx context.Context, a *models.Activity) error {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.activities[a.ID] = a
    return nil
}

func (s *Store) Get(ctx context.Context, id string) (*models.Activity, error) {
    s.mu.Lock()
    defer s.mu.Unlock()
    a, ok := s.activities[id]
    if !ok {
        return nil, errors.New("activity not found")
    }
    cp := *a
    return &cp, nil
}

func (s *Store) CreateAttempt(ctx context.Context, att *models.Attempt) error {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.attempts[att.ID] = att
    s.queue = append(s.queue, att.ID)
    return nil
}

func (s *Store) ClaimNext(ctx context.Context) (*models.Attempt, error) {
    s.mu.Lock()
    defer s.mu.Unlock()
    if len(s.queue) == 0 {
        return nil, errors.New("no pending attempts")
    }
    id := s.queue[0]
    s.queue = s.queue[1:]
    att := s.attempts[id]
    att.Status = models.ActivityRunning
    cp := *att
    return &cp, nil
}

func (s *Store) UpdateStatus(ctx context.Context, id string, status models.ActivityStatus, result []byte, errText string) error {
    s.mu.Lock()
    defer s.mu.Unlock()
    a, ok := s.activities[id]
    if !ok {
        return errors.New("activity not found")
    }
    a.Status = status
    a.Result = result
    a.Error = errText
    return nil
}

func (s *Store) Complete(ctx context.Context, attemptID string, result []byte) error {
    s.mu.Lock()
    defer s.mu.Unlock()
    att, ok := s.attempts[attemptID]
    if !ok {
        return errors.New("attempt not found")
    }
    att.Status = models.ActivityCompleted
    return nil
}

func (s *Store) Fail(ctx context.Context, attemptID string, errText string) error {
    s.mu.Lock()
    defer s.mu.Unlock()
    att, ok := s.attempts[attemptID]
    if !ok {
        return errors.New("attempt not found")
    }
    att.Status = models.ActivityFailed
    return nil
}
```

Use one concrete store that satisfies both interfaces. Keep locking simple; correctness is more important than cleverness in Phase 0.

## Registry

```go
// internal/registry/registry.go
package registry

import (
    "fmt"
    "sync"

    "github.com/your-org/niyanta/pkg/activity"
)

type Registry struct {
    mu        sync.RWMutex
    factories map[string]activity.Factory
}

func New() *Registry {
    return &Registry{factories: map[string]activity.Factory{}}
}

func (r *Registry) Register(name string, f activity.Factory) {
    r.mu.Lock()
    defer r.mu.Unlock()
    r.factories[name] = f
}

func (r *Registry) Create(name string) (activity.Activity, error) {
    r.mu.RLock()
    defer r.mu.RUnlock()
    f, ok := r.factories[name]
    if !ok {
        return nil, fmt.Errorf("unknown activity type %q", name)
    }
    return f(), nil
}
```

## Executor

```go
// internal/executor/executor.go
package executor

import (
    "context"

    "github.com/your-org/niyanta/internal/registry"
    "github.com/your-org/niyanta/internal/storage"
    "github.com/your-org/niyanta/pkg/activity"
    "github.com/your-org/niyanta/pkg/models"
)

type Executor struct {
    activities storage.ActivityStore
    attempts   storage.AttemptStore
    registry   *registry.Registry
}

func (e *Executor) RunOnce(ctx context.Context) error {
    att, err := e.attempts.ClaimNext(ctx)
    if err != nil {
        return err
    }

    a, err := e.activities.Get(ctx, att.ActivityID)
    if err != nil {
        return err
    }

    impl, err := e.registry.Create(a.Type)
    if err != nil {
        _ = e.activities.UpdateStatus(ctx, a.ID, models.ActivityFailed, nil, err.Error())
        return err
    }

    actx := activity.NewContext(ctx, a.ID, a.CustomerID)
    result, err := impl.Execute(actx, a.Input)
    if err != nil {
        _ = e.activities.UpdateStatus(ctx, a.ID, models.ActivityFailed, nil, err.Error())
        return err
    }

    if err := e.attempts.Complete(ctx, att.ID, result); err != nil {
        return err
    }
    return e.activities.UpdateStatus(ctx, a.ID, models.ActivityCompleted, result, "")
}
```

## Acceptance Test

```go
func TestSleepActivityCompletes(t *testing.T) {
    store := memory.NewStore()
    reg := registry.New()
    reg.Register("sleep", func() activity.Activity { return samples.Sleep{} })

    id := "act_1"
    require.NoError(t, store.CreateActivity(ctx, &models.Activity{
        ID: id, CustomerID: "cust_1", Type: "sleep", Status: models.ActivityPending,
    }))
    require.NoError(t, store.CreateAttempt(ctx, &models.Attempt{
        ID: "att_1", ActivityID: id, Number: 1, Status: models.ActivityPending,
    }))

    ex := executor.New(store, store, reg)
    require.NoError(t, ex.RunOnce(ctx))

    got, err := store.Get(ctx, id)
    require.NoError(t, err)
    require.Equal(t, models.ActivityCompleted, got.Status)
}
```

## Done When

- `go test ./...` passes.
- `go run ./cmd/niyanta demo sleep 2s` completes.
- No external services are required.
