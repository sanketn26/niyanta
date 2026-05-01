# Phase 6: HA, Scale, And Hardening

**Goal**: Make the Phase 5 product resilient at production scale: coordinator HA, durable signals via JetStream, autoscaling, determinism lint, chaos testing, and scale tuning.

## Leader Election

```go
type LeaderElector interface {
    Campaign(ctx context.Context, candidateID string) (<-chan Leadership, error)
}

type Leadership struct {
    IsLeader   bool
    Generation int64
}
```

Coordinator main loop:

```go
for leadership := range leadershipCh {
    if leadership.IsLeader {
        runtime.StartLeaderLoops(leadership.Generation)
    } else {
        runtime.StopLeaderLoops()
    }
}
```

Only the leader runs:

- ingestion planner
- scheduler
- timer loop
- signal delivery loop
- worker failure monitor
- adaptive health controller

Followers serve read APIs and can accept write APIs that persist state, but they do not schedule work.

## JetStream Signal Bus

```go
type JetStreamSignalBus struct {
    js nats.JetStreamContext
}

func (b *JetStreamSignalBus) Send(ctx context.Context, activityID, name string, payload []byte) error {
    subject := "signals." + activityID + "." + name
    _, err := b.js.Publish(subject, payload, nats.Context(ctx))
    return err
}

func (b *JetStreamSignalBus) AwaitForActivity(ctx context.Context, activityID, name string) ([]byte, error) {
    subject := "signals." + activityID + "." + name
    sub, err := b.js.PullSubscribe(subject, "activity-"+activityID)
    if err != nil {
        return nil, err
    }
    msgs, err := sub.Fetch(1, nats.Context(ctx))
    if err != nil {
        return nil, err
    }
    if len(msgs) == 0 {
        return nil, ErrNoSignal
    }
    _ = msgs[0].Ack()
    return msgs[0].Data, nil
}
```

## Planner Sharding

At high scale, shard planner work by tenant or connection ID.

```go
func Owns(shardCount, shardIndex int, key string) bool {
    h := fnv.New32a()
    _, _ = h.Write([]byte(key))
    return int(h.Sum32()%uint32(shardCount)) == shardIndex
}
```

Each leader loop can run multiple shard workers:

```go
for shard := 0; shard < cfg.PlannerShards; shard++ {
    go planner.RunShard(ctx, cfg.PlannerShards, shard)
}
```

## Autoscaling Metrics

Expose:

- `niyanta_scheduler_queue_depth`
- `niyanta_ingestion_source_lag_seconds`
- `niyanta_worker_capacity_used`
- `niyanta_worker_capacity_total`
- `niyanta_ingestion_run_duration_seconds`

Kubernetes HPA target:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: niyanta-worker
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: niyanta-worker
  minReplicas: 3
  maxReplicas: 500
  metrics:
    - type: Pods
      pods:
        metric:
          name: niyanta_scheduler_queue_depth_per_worker
        target:
          type: AverageValue
          averageValue: "50"
```

## Determinism Linter

Reject direct use of nondeterministic APIs inside activity packages:

```go
var banned = map[string]string{
    "time.Now": "use ctx.Now()",
    "rand.Int": "use ctx.NewID() or recorded randomness",
    "uuid.New": "use ctx.NewID()",
}
```

AST check shape:

```go
func inspectCall(pass *analysis.Pass, call *ast.CallExpr) {
    sel, ok := call.Fun.(*ast.SelectorExpr)
    if !ok {
        return
    }
    pkg, ok := sel.X.(*ast.Ident)
    if !ok {
        return
    }
    key := pkg.Name + "." + sel.Sel.Name
    if hint, banned := banned[key]; banned {
        pass.Reportf(call.Pos(), "%s is nondeterministic in activities; %s", key, hint)
    }
}
```

## Chaos Tests

Run a 1-hour scenario:

1. Randomly kill workers.
2. Kill the leader coordinator.
3. Pause Postgres for 5 seconds.
4. Drop NATS connection briefly.
5. Force source 429 responses.
6. Force destination 503 responses.

Assertions:

- no committed checkpoint without delivered records
- no cross-tenant data reads
- all interrupted attempts are retried or failed according to policy
- coordinator leadership recovers under 30 seconds
- source lag returns under SLO after the fault window

## Done When

- Coordinator failover is under 30 seconds.
- Sustained scale target passes for 24 hours.
- HPA scales workers up and down from real metrics.
- Determinism linter runs in CI.
- Chaos tests validate recovery behavior.

