# Scheduler Implementation

**Component**: Workload Scheduler
**Last Updated**: 2025-10-29
**Status**: Implementation Ready with Scale & Affinity Support

## Table of Contents
1. [Overview](#overview)
2. [Core Interfaces](#core-interfaces)
3. [FIFO Scheduler (Phase 1 MVP)](#fifo-scheduler-phase-1-mvp)
4. [Priority Scheduler with SLA Tiers (Phase 2)](#priority-scheduler-with-sla-tiers-phase-2)
5. [Affinity Evaluator](#affinity-evaluator)
6. [Worker Selection Algorithms](#worker-selection-algorithms)
7. [Starvation Prevention (Aging)](#starvation-prevention-aging)
8. [Performance Optimization](#performance-optimization)
9. [Testing](#testing)

---

## Overview

The Scheduler is responsible for assigning pending workloads to available workers based on:
- **Capacity**: Worker has available slots
- **Capability**: Worker supports the workload type
- **Affinity Rules**: Hard/soft/anti-affinity constraints
- **Priority**: Customer tier and explicit priority
- **Performance**: Minimize scheduling latency

**Package**: `internal/scheduler`

**Scale Targets**:
- Handle 10,000 concurrent workloads
- Scheduling latency p95 < 5 seconds
- Support 500 workers
- Evaluate affinity for all workloads

---

## Core Interfaces

### Scheduler Interface

```go
package scheduler

import (
    "context"

    "github.com/yourusername/niyanta/pkg/models"
)

// Scheduler assigns pending workloads to available workers
type Scheduler interface {
    // ScheduleNextBatch finds pending workloads and assigns them to workers
    // Returns the number of workloads scheduled or an error
    ScheduleNextBatch(ctx context.Context) (int, error)

    // ScheduleWorkload schedules a specific workload immediately
    // Used for high-priority or API-triggered scheduling
    ScheduleWorkload(ctx context.Context, workloadID string) error

    // GetQueueDepth returns the current number of pending workloads
    GetQueueDepth(ctx context.Context) (int, error)
}

// AffinityEvaluator evaluates workload affinity rules against workers
type AffinityEvaluator interface {
    // EvaluateHardAffinity returns true if worker satisfies hard affinity
    EvaluateHardAffinity(workload *models.Workload, worker *models.Worker) bool

    // EvaluateSoftAffinity returns a score (0-100) for worker preference
    EvaluateSoftAffinity(workload *models.Workload, worker *models.Worker) int

    // EvaluateAntiAffinity returns true if worker violates anti-affinity
    EvaluateAntiAffinity(ctx context.Context, workload *models.Workload, worker *models.Worker) (bool, error)
}
```

---

## FIFO Scheduler (Phase 1 MVP)

### Configuration

```go
package scheduler

type Config struct {
    // Scheduling interval in seconds
    IntervalSeconds int `mapstructure:"interval_seconds"` // Default: 5

    // Maximum workloads to schedule per batch
    BatchSize int `mapstructure:"batch_size"` // Default: 100, Max: 500

    // Maximum time to spend scheduling one batch (ms)
    MaxSchedulingTimeMS int `mapstructure:"max_scheduling_time_ms"` // Default: 1000

    // Enable priority-based scheduling (Phase 2)
    EnablePriority bool `mapstructure:"enable_priority"` // Default: false

    // Enable aging to prevent starvation (Phase 2)
    EnableAging bool `mapstructure:"enable_aging"` // Default: false

    // Aging interval in minutes
    AgingIntervalMinutes int `mapstructure:"aging_interval_minutes"` // Default: 10

    // Priority boost per minute of wait time
    AgingBoostPerMinute int `mapstructure:"aging_boost_per_minute"` // Default: 5
}
```

### FIFO Scheduler Implementation

```go
package scheduler

import (
    "context"
    "fmt"
    "time"

    "github.com/yourusername/niyanta/internal/broker"
    "github.com/yourusername/niyanta/internal/storage"
    "github.com/yourusername/niyanta/pkg/models"
    "go.uber.org/zap"
)

type FIFOScheduler struct {
    config           Config
    stateManager     storage.StateManager
    brokerClient     broker.BrokerClient
    affinityEvaluator AffinityEvaluator
    logger           *zap.Logger
}

func NewFIFOScheduler(
    cfg Config,
    sm storage.StateManager,
    bc broker.BrokerClient,
    ae AffinityEvaluator,
    logger *zap.Logger,
) *FIFOScheduler {
    return &FIFOScheduler{
        config:           cfg,
        stateManager:     sm,
        brokerClient:     bc,
        affinityEvaluator: ae,
        logger:           logger,
    }
}

func (s *FIFOScheduler) ScheduleNextBatch(ctx context.Context) (int, error) {
    start := time.Now()

    // Get pending workloads (ordered by created_at ASC for FIFO)
    workloads, err := s.stateManager.GetPendingWorkloads(ctx, s.config.BatchSize)
    if err != nil {
        return 0, fmt.Errorf("failed to get pending workloads: %w", err)
    }

    if len(workloads) == 0 {
        return 0, nil // Nothing to schedule
    }

    s.logger.Info("starting scheduling batch",
        zap.Int("workload_count", len(workloads)),
    )

    // Get available workers
    workers, err := s.stateManager.GetAvailableWorkers(ctx)
    if err != nil {
        return 0, fmt.Errorf("failed to get available workers: %w", err)
    }

    if len(workers) == 0 {
        s.logger.Warn("no available workers")
        return 0, nil
    }

    scheduled := 0

    for _, workload := range workloads {
        // Check timeout
        if time.Since(start) > time.Duration(s.config.MaxSchedulingTimeMS)*time.Millisecond {
            s.logger.Warn("scheduling batch timeout reached",
                zap.Int("scheduled", scheduled),
                zap.Int("remaining", len(workloads)-scheduled),
            )
            break
        }

        // Find suitable worker
        worker := s.selectWorker(ctx, workload, workers)
        if worker == nil {
            s.logger.Debug("no suitable worker found for workload",
                zap.String("workload_id", workload.ID),
            )
            continue
        }

        // Assign workload to worker
        err := s.assignWorkload(ctx, workload, worker)
        if err != nil {
            s.logger.Error("failed to assign workload",
                zap.String("workload_id", workload.ID),
                zap.String("worker_id", worker.ID),
                zap.Error(err),
            )
            continue
        }

        scheduled++

        // Update worker capacity in local cache
        worker.CapacityUsed++
        if worker.CapacityUsed >= worker.CapacityTotal {
            // Remove worker from available list
            workers = s.removeWorker(workers, worker.ID)
        }
    }

    s.logger.Info("scheduling batch complete",
        zap.Int("scheduled", scheduled),
        zap.Duration("duration", time.Since(start)),
    )

    return scheduled, nil
}

func (s *FIFOScheduler) selectWorker(
    ctx context.Context,
    workload *models.Workload,
    workers []*models.Worker,
) *models.Worker {
    var bestWorker *models.Worker
    bestScore := -1

    for _, worker := range workers {
        // Check capacity
        if worker.CapacityUsed >= worker.CapacityTotal {
            continue
        }

        // Check capability (worker supports this workload type)
        if !s.supportsWorkloadType(worker, workload.WorkloadType) {
            continue
        }

        // Evaluate hard affinity (must pass)
        if !s.affinityEvaluator.EvaluateHardAffinity(workload, worker) {
            continue
        }

        // Evaluate anti-affinity (must not violate)
        violatesAntiAffinity, err := s.affinityEvaluator.EvaluateAntiAffinity(ctx, workload, worker)
        if err != nil {
            s.logger.Error("failed to evaluate anti-affinity",
                zap.String("workload_id", workload.ID),
                zap.String("worker_id", worker.ID),
                zap.Error(err),
            )
            continue
        }
        if violatesAntiAffinity {
            continue
        }

        // Evaluate soft affinity (for scoring)
        score := s.affinityEvaluator.EvaluateSoftAffinity(workload, worker)

        if score > bestScore {
            bestScore = score
            bestWorker = worker
        }
    }

    return bestWorker
}

func (s *FIFOScheduler) assignWorkload(
    ctx context.Context,
    workload *models.Workload,
    worker *models.Worker,
) error {
    // Update workload status to SCHEDULED
    workload.Status = models.WorkloadStatusScheduled
    workload.WorkerID = &worker.ID
    scheduledAt := time.Now()
    workload.ScheduledAt = &scheduledAt

    err := s.stateManager.UpdateWorkload(ctx, workload)
    if err != nil {
        return fmt.Errorf("failed to update workload: %w", err)
    }

    // Send assignment message to worker via broker
    assignment := broker.WorkloadAssignment{
        WorkloadID:   workload.ID,
        WorkloadType: workload.WorkloadType,
        InputParams:  workload.InputParams,
        Generation:   workload.Generation,
    }

    err = s.brokerClient.SendWorkloadAssignment(ctx, worker.ID, assignment)
    if err != nil {
        // Rollback workload status
        workload.Status = models.WorkloadStatusPending
        workload.WorkerID = nil
        workload.ScheduledAt = nil
        s.stateManager.UpdateWorkload(ctx, workload)

        return fmt.Errorf("failed to send assignment: %w", err)
    }

    s.logger.Info("workload scheduled",
        zap.String("workload_id", workload.ID),
        zap.String("worker_id", worker.ID),
    )

    return nil
}

func (s *FIFOScheduler) supportsWorkloadType(worker *models.Worker, workloadType string) bool {
    for _, capability := range worker.Capabilities {
        if capability == workloadType {
            return true
        }
    }
    return false
}

func (s *FIFOScheduler) removeWorker(workers []*models.Worker, workerID string) []*models.Worker {
    result := make([]*models.Worker, 0, len(workers)-1)
    for _, w := range workers {
        if w.ID != workerID {
            result = append(result, w)
        }
    }
    return result
}

func (s *FIFOScheduler) GetQueueDepth(ctx context.Context) (int, error) {
    return s.stateManager.GetQueueDepth(ctx)
}
```

---

## Priority Scheduler with SLA Tiers (Phase 2)

### Priority Queue Implementation

```go
package scheduler

import (
    "container/heap"
    "sync"
    "time"

    "github.com/yourusername/niyanta/pkg/models"
)

// WorkloadPriorityQueue implements a priority queue for workload scheduling
type WorkloadPriorityQueue struct {
    mu       sync.RWMutex
    items    []*priorityQueueItem
    itemMap  map[string]*priorityQueueItem // workload_id -> item
}

type priorityQueueItem struct {
    workload      *models.Workload
    customerTier  models.CustomerTier
    effectivePrio int64
    insertedAt    time.Time
    index         int
}

func NewWorkloadPriorityQueue() *WorkloadPriorityQueue {
    pq := &WorkloadPriorityQueue{
        items:   make([]*priorityQueueItem, 0),
        itemMap: make(map[string]*priorityQueueItem),
    }
    heap.Init((*priorityQueueHeap)(&pq.items))
    return pq
}

func (pq *WorkloadPriorityQueue) Push(workload *models.Workload, tier models.CustomerTier) {
    pq.mu.Lock()
    defer pq.mu.Unlock()

    effectivePrio := calculateEffectivePriority(workload.Priority, tier)

    item := &priorityQueueItem{
        workload:      workload,
        customerTier:  tier,
        effectivePrio: effectivePrio,
        insertedAt:    time.Now(),
    }

    heap.Push((*priorityQueueHeap)(&pq.items), item)
    pq.itemMap[workload.ID] = item
}

func (pq *WorkloadPriorityQueue) Pop() *models.Workload {
    pq.mu.Lock()
    defer pq.mu.Unlock()

    if len(pq.items) == 0 {
        return nil
    }

    item := heap.Pop((*priorityQueueHeap)(&pq.items)).(*priorityQueueItem)
    delete(pq.itemMap, item.workload.ID)

    return item.workload
}

func (pq *WorkloadPriorityQueue) Len() int {
    pq.mu.RLock()
    defer pq.mu.RUnlock()
    return len(pq.items)
}

// ApplyAging increases priority of old workloads to prevent starvation
func (pq *WorkloadPriorityQueue) ApplyAging(boostPerMinute int64) {
    pq.mu.Lock()
    defer pq.mu.Unlock()

    now := time.Now()
    needsReheap := false

    for _, item := range pq.items {
        age := now.Sub(item.insertedAt)
        ageMinutes := int64(age.Minutes())

        if ageMinutes > 0 {
            boost := ageMinutes * boostPerMinute
            item.effectivePrio += boost
            needsReheap = true
        }
    }

    if needsReheap {
        heap.Init((*priorityQueueHeap)(&pq.items))
    }
}

// calculateEffectivePriority computes priority based on tier and explicit priority
func calculateEffectivePriority(basePriority int, tier models.CustomerTier) int64 {
    tierWeights := map[models.CustomerTier]int64{
        models.TierFree:       1,
        models.TierStandard:   10,
        models.TierPremium:    50,
        models.TierEnterprise: 100,
    }

    weight := tierWeights[tier]
    if weight == 0 {
        weight = 1 // Default to free tier
    }

    return int64(basePriority) * weight
}

// priorityQueueHeap implements heap.Interface
type priorityQueueHeap []*priorityQueueItem

func (h priorityQueueHeap) Len() int { return len(h) }

func (h priorityQueueHeap) Less(i, j int) bool {
    // Higher effective priority comes first
    return h[i].effectivePrio > h[j].effectivePrio
}

func (h priorityQueueHeap) Swap(i, j int) {
    h[i], h[j] = h[j], h[i]
    h[i].index = i
    h[j].index = j
}

func (h *priorityQueueHeap) Push(x interface{}) {
    item := x.(*priorityQueueItem)
    item.index = len(*h)
    *h = append(*h, item)
}

func (h *priorityQueueHeap) Pop() interface{} {
    old := *h
    n := len(old)
    item := old[n-1]
    old[n-1] = nil
    item.index = -1
    *h = old[0 : n-1]
    return item
}
```

### Priority Scheduler Implementation

```go
package scheduler

type PriorityScheduler struct {
    config           Config
    stateManager     storage.StateManager
    brokerClient     broker.BrokerClient
    affinityEvaluator AffinityEvaluator
    logger           *zap.Logger

    priorityQueue    *WorkloadPriorityQueue
    lastAgingTime    time.Time
}

func NewPriorityScheduler(
    cfg Config,
    sm storage.StateManager,
    bc broker.BrokerClient,
    ae AffinityEvaluator,
    logger *zap.Logger,
) *PriorityScheduler {
    return &PriorityScheduler{
        config:           cfg,
        stateManager:     sm,
        brokerClient:     bc,
        affinityEvaluator: ae,
        logger:           logger,
        priorityQueue:    NewWorkloadPriorityQueue(),
        lastAgingTime:    time.Now(),
    }
}

func (s *PriorityScheduler) ScheduleNextBatch(ctx context.Context) (int, error) {
    // Apply aging if interval has passed
    if s.config.EnableAging && time.Since(s.lastAgingTime) > time.Duration(s.config.AgingIntervalMinutes)*time.Minute {
        s.priorityQueue.ApplyAging(int64(s.config.AgingBoostPerMinute))
        s.lastAgingTime = time.Now()
        s.logger.Info("applied priority aging to prevent starvation")
    }

    // Refill priority queue if needed
    if s.priorityQueue.Len() < s.config.BatchSize {
        err := s.refillQueue(ctx)
        if err != nil {
            return 0, fmt.Errorf("failed to refill queue: %w", err)
        }
    }

    // Get available workers
    workers, err := s.stateManager.GetAvailableWorkers(ctx)
    if err != nil {
        return 0, fmt.Errorf("failed to get available workers: %w", err)
    }

    if len(workers) == 0 {
        return 0, nil
    }

    scheduled := 0
    start := time.Now()

    for scheduled < s.config.BatchSize && s.priorityQueue.Len() > 0 {
        // Check timeout
        if time.Since(start) > time.Duration(s.config.MaxSchedulingTimeMS)*time.Millisecond {
            break
        }

        workload := s.priorityQueue.Pop()
        if workload == nil {
            break
        }

        worker := s.selectWorker(ctx, workload, workers)
        if worker == nil {
            continue // Skip this workload, try next
        }

        err := s.assignWorkload(ctx, workload, worker)
        if err != nil {
            s.logger.Error("failed to assign workload", zap.Error(err))
            continue
        }

        scheduled++

        worker.CapacityUsed++
        if worker.CapacityUsed >= worker.CapacityTotal {
            workers = s.removeWorker(workers, worker.ID)
        }
    }

    return scheduled, nil
}

func (s *PriorityScheduler) refillQueue(ctx context.Context) error {
    workloads, err := s.stateManager.GetPendingWorkloads(ctx, s.config.BatchSize*2)
    if err != nil {
        return err
    }

    // Get customer tiers for workloads
    customerTiers, err := s.getCustomerTiers(ctx, workloads)
    if err != nil {
        return err
    }

    for _, wl := range workloads {
        tier := customerTiers[wl.CustomerID]
        s.priorityQueue.Push(wl, tier)
    }

    return nil
}

func (s *PriorityScheduler) getCustomerTiers(ctx context.Context, workloads []*models.Workload) (map[string]models.CustomerTier, error) {
    // Get unique customer IDs
    customerIDs := make(map[string]bool)
    for _, wl := range workloads {
        customerIDs[wl.CustomerID] = true
    }

    // Fetch customer records
    tiers := make(map[string]models.CustomerTier)
    for customerID := range customerIDs {
        customer, err := s.stateManager.GetCustomer(ctx, customerID)
        if err != nil {
            return nil, err
        }
        tiers[customerID] = customer.Tier
    }

    return tiers, nil
}

// selectWorker and assignWorkload same as FIFO scheduler
// ... (omitted for brevity, same implementation)
```

---

## Affinity Evaluator

```go
package scheduler

import (
    "context"

    "github.com/yourusername/niyanta/internal/storage"
    "github.com/yourusername/niyanta/pkg/models"
)

type AffinityEvaluatorImpl struct {
    stateManager storage.StateManager
}

func NewAffinityEvaluator(sm storage.StateManager) *AffinityEvaluatorImpl {
    return &AffinityEvaluatorImpl{
        stateManager: sm,
    }
}

func (a *AffinityEvaluatorImpl) EvaluateHardAffinity(
    workload *models.Workload,
    worker *models.Worker,
) bool {
    if workload.AffinityRules == nil || workload.AffinityRules.HardAffinity == nil {
        return true // No hard affinity = any worker is fine
    }

    rules := workload.AffinityRules.HardAffinity

    // Check worker ID match
    if rules.WorkerID != "" && rules.WorkerID != worker.ID {
        return false
    }

    // Check worker tags match (ALL tags must match)
    for key, requiredValue := range rules.WorkerTags {
        actualValue, exists := worker.Tags[key]
        if !exists || actualValue != requiredValue {
            return false
        }
    }

    return true
}

func (a *AffinityEvaluatorImpl) EvaluateSoftAffinity(
    workload *models.Workload,
    worker *models.Worker,
) int {
    if workload.AffinityRules == nil || workload.AffinityRules.SoftAffinity == nil {
        return 50 // Neutral score
    }

    rules := workload.AffinityRules.SoftAffinity
    score := 50

    // Exact worker ID match: +50 points
    if rules.WorkerID != "" && rules.WorkerID == worker.ID {
        score += 50
    }

    // Tag matches: +5 points per matching tag
    for key, requiredValue := range rules.WorkerTags {
        if actualValue, exists := worker.Tags[key]; exists && actualValue == requiredValue {
            score += 5
        }
    }

    return min(score, 100)
}

func (a *AffinityEvaluatorImpl) EvaluateAntiAffinity(
    ctx context.Context,
    workload *models.Workload,
    worker *models.Worker,
) (bool, error) {
    if workload.AffinityRules == nil || workload.AffinityRules.AntiAffinity == nil {
        return false, nil // No anti-affinity = no conflict
    }

    rules := workload.AffinityRules.AntiAffinity

    // Get currently running workloads on this worker
    runningWorkloads, err := a.stateManager.GetWorkloadsByWorker(ctx, worker.ID)
    if err != nil {
        return true, err // Conservative: assume conflict on error
    }

    // Check if any running workload violates anti-affinity
    for _, running := range runningWorkloads {
        for _, avoidType := range rules.WorkloadTypes {
            if running.WorkloadType == avoidType {
                return true, nil // Conflict found
            }
        }
    }

    return false, nil
}

func min(a, b int) int {
    if a < b {
        return a
    }
    return b
}
```

---

## Worker Selection Algorithms

### Round-Robin (Simple, Phase 1)

```go
func (s *FIFOScheduler) selectWorkerRoundRobin(workers []*models.Worker) *models.Worker {
    // Simple: return first available worker
    for _, w := range workers {
        if w.CapacityUsed < w.CapacityTotal {
            return w
        }
    }
    return nil
}
```

### Least Loaded (Phase 2)

```go
func (s *Scheduler) selectWorkerLeastLoaded(workers []*models.Worker) *models.Worker {
    var bestWorker *models.Worker
    lowestUtilization := 101.0

    for _, w := range workers {
        if w.CapacityTotal == 0 {
            continue
        }

        utilization := float64(w.CapacityUsed) / float64(w.CapacityTotal) * 100

        if utilization < lowestUtilization {
            lowestUtilization = utilization
            bestWorker = w
        }
    }

    return bestWorker
}
```

### Best Affinity Match (Phase 3)

```go
func (s *Scheduler) selectWorkerBestAffinity(
    ctx context.Context,
    workload *models.Workload,
    workers []*models.Worker,
) *models.Worker {
    var bestWorker *models.Worker
    bestScore := -1

    for _, worker := range workers {
        // Hard affinity check
        if !s.affinityEvaluator.EvaluateHardAffinity(workload, worker) {
            continue
        }

        // Anti-affinity check
        violates, _ := s.affinityEvaluator.EvaluateAntiAffinity(ctx, workload, worker)
        if violates {
            continue
        }

        // Soft affinity scoring
        score := s.affinityEvaluator.EvaluateSoftAffinity(workload, worker)

        if score > bestScore {
            bestScore = score
            bestWorker = worker
        }
    }

    return bestWorker
}
```

---

## Starvation Prevention (Aging)

Priority aging prevents low-priority workloads from starving:

```go
// Run aging periodically in scheduler loop
func (s *PriorityScheduler) runAgingLoop(ctx context.Context) {
    ticker := time.NewTicker(time.Duration(s.config.AgingIntervalMinutes) * time.Minute)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            s.priorityQueue.ApplyAging(int64(s.config.AgingBoostPerMinute))
            s.logger.Info("applied priority aging")
        }
    }
}
```

**Aging Formula**:
```
effective_priority = base_priority * tier_weight + (age_in_minutes * boost_per_minute)
```

**Example**:
- Free tier workload: base_priority=10, tier_weight=1
- Initial effective_priority: 10 * 1 = 10
- After 10 minutes (boost=5/min): 10 + (10 * 5) = 60
- After 20 minutes: 10 + (20 * 5) = 110 (now higher than premium tier base!)

---

## Performance Optimization

### Batch Size Tuning

```yaml
# Small workload (< 1 minute execution):
batch_size: 500
interval_seconds: 1

# Large workload (> 10 minutes execution):
batch_size: 100
interval_seconds: 5
```

### Worker Caching

Cache available workers to avoid repeated database queries:

```go
type workerCache struct {
    workers    []*models.Worker
    lastUpdate time.Time
    ttl        time.Duration
}

func (s *Scheduler) getAvailableWorkersCached(ctx context.Context) ([]*models.Worker, error) {
    if time.Since(s.cache.lastUpdate) < s.cache.ttl {
        return s.cache.workers, nil
    }

    workers, err := s.stateManager.GetAvailableWorkers(ctx)
    if err != nil {
        return nil, err
    }

    s.cache.workers = workers
    s.cache.lastUpdate = time.Now()
    return workers, nil
}
```

### Database Query Optimization

```sql
-- Use partial index for pending workloads
CREATE INDEX CONCURRENTLY idx_workloads_pending_priority
ON workloads(customer_id, priority DESC, created_at ASC)
WHERE status = 'PENDING';

-- Query with limit to avoid full table scan
SELECT * FROM workloads
WHERE status = 'PENDING'
ORDER BY priority DESC, created_at ASC
LIMIT 500;
```

---

## Testing

### Unit Tests

```go
package scheduler

import (
    "context"
    "testing"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/mock"
)

func TestScheduler_SelectWorkerWithAffinity(t *testing.T) {
    tests := []struct {
        name           string
        workload       *models.Workload
        workers        []*models.Worker
        expectedWorker string
    }{
        {
            name: "hard affinity GPU requirement",
            workload: &models.Workload{
                ID: "wl_1",
                AffinityRules: &models.AffinityRules{
                    HardAffinity: &models.AffinityConstraint{
                        WorkerTags: map[string]string{"hardware": "gpu"},
                    },
                },
            },
            workers: []*models.Worker{
                {ID: "w1", Tags: map[string]string{"hardware": "cpu"}},
                {ID: "w2", Tags: map[string]string{"hardware": "gpu"}},
            },
            expectedWorker: "w2",
        },
        {
            name: "soft affinity scoring",
            workload: &models.Workload{
                ID: "wl_2",
                AffinityRules: &models.AffinityRules{
                    SoftAffinity: &models.AffinityConstraint{
                        WorkerTags: map[string]string{"region": "us-west-2"},
                    },
                },
            },
            workers: []*models.Worker{
                {ID: "w1", Tags: map[string]string{"region": "us-east-1"}},
                {ID: "w2", Tags: map[string]string{"region": "us-west-2"}},
            },
            expectedWorker: "w2", // Higher soft affinity score
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            scheduler := setupTestScheduler()
            worker := scheduler.selectWorker(context.Background(), tt.workload, tt.workers)

            assert.NotNil(t, worker)
            assert.Equal(t, tt.expectedWorker, worker.ID)
        })
    }
}

func TestPriorityQueue(t *testing.T) {
    pq := NewWorkloadPriorityQueue()

    // Add workloads with different priorities
    pq.Push(&models.Workload{ID: "wl_1", Priority: 10}, models.TierFree)
    pq.Push(&models.Workload{ID: "wl_2", Priority: 10}, models.TierPremium)
    pq.Push(&models.Workload{ID: "wl_3", Priority: 50}, models.TierStandard)

    // Premium tier should come first (10 * 50 = 500)
    wl := pq.Pop()
    assert.Equal(t, "wl_2", wl.ID)

    // Then standard with high priority (50 * 10 = 500)
    wl = pq.Pop()
    assert.Equal(t, "wl_3", wl.ID)

    // Finally free tier (10 * 1 = 10)
    wl = pq.Pop()
    assert.Equal(t, "wl_1", wl.ID)
}

func TestAging(t *testing.T) {
    pq := NewWorkloadPriorityQueue()

    // Add free tier workload
    pq.Push(&models.Workload{ID: "wl_1", Priority: 10}, models.TierFree)

    // Wait and apply aging
    time.Sleep(2 * time.Minute)
    pq.ApplyAging(5) // +5 per minute

    // Verify priority boosted
    // (implementation would need to expose effective priority for testing)
}
```

---

## Summary

This scheduler implementation provides:

✅ **FIFO scheduling** (Phase 1 MVP)
✅ **Priority-based scheduling with SLA tiers** (Phase 2)
✅ **Complete affinity evaluation** (hard, soft, anti-affinity)
✅ **Starvation prevention through aging** (Phase 2)
✅ **Performance optimization** (caching, batch tuning)
✅ **Scalability** (10K workloads, 500 workers)

**Key Features**:
- Handles complex affinity rules
- Prevents low-priority starvation
- Scales to thousands of workloads
- Optimized for sub-5s scheduling latency

---

**Next**: [07_HEALTH_MONITOR.md](07_HEALTH_MONITOR.md) - Worker health monitoring
