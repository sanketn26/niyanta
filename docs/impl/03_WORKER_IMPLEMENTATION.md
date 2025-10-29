# Worker Implementation

**Component**: Worker (Execution Plane)
**Last Updated**: 2025-10-29
**Status**: Implementation Ready

## Table of Contents
1. [Overview](#overview)
2. [Core Interfaces](#core-interfaces)
3. [Worker Struct](#worker-struct)
4. [Workload Execution Engine](#workload-execution-engine)
5. [Heartbeat Sender](#heartbeat-sender)
6. [Control Plane Listener](#control-plane-listener)
7. [Registration Service](#registration-service)
8. [Resource Isolation](#resource-isolation)
9. [Complete Implementation Example](#complete-implementation-example)
10. [Testing](#testing)

---

## Overview

The Worker is the execution plane of Niyanta, responsible for:
- Executing workloads assigned by the coordinator
- Managing workload lifecycle and resource isolation
- Reporting progress and heartbeats to the coordinator
- Creating checkpoints for fault tolerance
- Handling graceful shutdown

**Package**: `internal/worker`

**Key Design Principles**:
- Concurrent workload execution with goroutine pools
- Resource isolation per workload
- Automatic checkpoint creation
- Graceful shutdown with workload draining

---

## Core Interfaces

### Worker Interface

```go
package worker

import (
    "context"
)

// Worker defines the execution plane interface
type Worker interface {
    // Start initializes and starts the worker
    // This is a blocking call that runs until context is cancelled
    Start(ctx context.Context) error

    // Shutdown gracefully stops the worker
    Shutdown(ctx context.Context) error

    // ID returns the unique worker ID
    ID() string

    // Status returns the current worker status
    Status() WorkerStatus

    // Capacity returns the worker's capacity information
    Capacity() CapacityInfo
}

// WorkerStatus represents the worker's operational state
type WorkerStatus string

const (
    WorkerStatusHealthy  WorkerStatus = "HEALTHY"
    WorkerStatusDraining WorkerStatus = "DRAINING"
    WorkerStatusDead     WorkerStatus = "DEAD"
)

// CapacityInfo holds capacity metrics
type CapacityInfo struct {
    Total int `json:"total"`
    Used  int `json:"used"`
    Free  int `json:"free"`
}
```

---

## Worker Struct

### Main Worker Structure

```go
package worker

import (
    "context"
    "fmt"
    "sync"
    "time"

    "github.com/yourusername/niyanta/internal/broker"
    "github.com/yourusername/niyanta/internal/checkpoint"
    "github.com/yourusername/niyanta/internal/storage"
    "github.com/yourusername/niyanta/internal/workload"
    "github.com/yourusername/niyanta/pkg/models"
    "go.uber.org/zap"
)

// WorkerImpl is the concrete implementation of Worker
type WorkerImpl struct {
    // Worker identity
    id           string
    capabilities []string
    tags         map[string]string

    // Configuration
    config Config

    // Core dependencies
    stateManager      storage.StateManager
    brokerClient      broker.BrokerClient
    checkpointManager checkpoint.Manager
    workloadRegistry  workload.Registry
    logger            *zap.Logger

    // Execution state
    status             WorkerStatus
    statusMu           sync.RWMutex
    runningWorkloads   map[string]*WorkloadExecution
    workloadsMu        sync.RWMutex
    capacityTotal      int
    capacitySemaphore  chan struct{} // Semaphore for capacity control

    // Lifecycle management
    startTime    time.Time
    shutdownOnce sync.Once
    wg           sync.WaitGroup
}

// Config holds worker configuration
type Config struct {
    // Worker identity
    WorkerID     string            `yaml:"worker_id" env:"WORKER_ID"`
    Capabilities []string          `yaml:"capabilities" env:"CAPABILITIES"`
    Tags         map[string]string `yaml:"tags" env:"TAGS"`

    // Capacity
    MaxConcurrentWorkloads int `yaml:"max_concurrent_workloads" env:"MAX_CONCURRENT_WORKLOADS" default:"10"`

    // Heartbeat configuration
    HeartbeatInterval time.Duration `yaml:"heartbeat_interval" env:"HEARTBEAT_INTERVAL" default:"30s"`

    // Checkpoint configuration
    CheckpointInterval time.Duration `yaml:"checkpoint_interval" env:"CHECKPOINT_INTERVAL" default:"5m"`

    // Shutdown configuration
    DrainTimeout time.Duration `yaml:"drain_timeout" env:"DRAIN_TIMEOUT" default:"5m"`

    // Resource limits per workload
    DefaultCPUCores int `yaml:"default_cpu_cores" env:"DEFAULT_CPU_CORES" default:"1"`
    DefaultMemoryMB int `yaml:"default_memory_mb" env:"DEFAULT_MEMORY_MB" default:"512"`
}

// WorkloadExecution tracks a running workload
type WorkloadExecution struct {
    Workload      *models.Workload
    Context       context.Context
    Cancel        context.CancelFunc
    WorkloadImpl  workload.Workload
    StartTime     time.Time
    LastCheckpoint time.Time
}

// NewWorker creates a new worker instance
func NewWorker(
    config Config,
    stateManager storage.StateManager,
    brokerClient broker.BrokerClient,
    checkpointManager checkpoint.Manager,
    workloadRegistry workload.Registry,
    logger *zap.Logger,
) (*WorkerImpl, error) {
    // Validate configuration
    if err := config.Validate(); err != nil {
        return nil, fmt.Errorf("invalid configuration: %w", err)
    }

    // Generate worker ID if not provided
    if config.WorkerID == "" {
        config.WorkerID = fmt.Sprintf("worker-%s", generateID())
    }

    w := &WorkerImpl{
        id:                config.WorkerID,
        capabilities:      config.Capabilities,
        tags:              config.Tags,
        config:            config,
        stateManager:      stateManager,
        brokerClient:      brokerClient,
        checkpointManager: checkpointManager,
        workloadRegistry:  workloadRegistry,
        logger:            logger.With(zap.String("worker_id", config.WorkerID)),
        status:            WorkerStatusHealthy,
        runningWorkloads:  make(map[string]*WorkloadExecution),
        capacityTotal:     config.MaxConcurrentWorkloads,
        capacitySemaphore: make(chan struct{}, config.MaxConcurrentWorkloads),
        startTime:         time.Now(),
    }

    return w, nil
}

// Validate validates the configuration
func (c *Config) Validate() error {
    if c.MaxConcurrentWorkloads < 1 {
        return fmt.Errorf("invalid max_concurrent_workloads: %d", c.MaxConcurrentWorkloads)
    }
    if c.HeartbeatInterval < time.Second {
        return fmt.Errorf("heartbeat interval too short: %v", c.HeartbeatInterval)
    }
    if len(c.Capabilities) == 0 {
        return fmt.Errorf("at least one capability required")
    }
    return nil
}
```

---

## Workload Execution Engine

### Execution Manager

```go
// ExecuteWorkload executes a workload
func (w *WorkerImpl) ExecuteWorkload(ctx context.Context, assignment *models.Workload) error {
    // Acquire capacity
    select {
    case w.capacitySemaphore <- struct{}{}:
        defer func() { <-w.capacitySemaphore }()
    case <-ctx.Done():
        return ctx.Err()
    }

    w.logger.Info("starting workload execution",
        zap.String("workload_id", assignment.ID),
        zap.String("workload_type", assignment.WorkloadConfigID),
    )

    // Create workload execution context
    execCtx, cancel := context.WithCancel(ctx)
    defer cancel()

    // Get workload implementation from registry
    workloadImpl, err := w.workloadRegistry.Get(assignment.WorkloadConfigID)
    if err != nil {
        return fmt.Errorf("failed to get workload implementation: %w", err)
    }

    // Load checkpoint if exists
    checkpointData, err := w.loadLatestCheckpoint(ctx, assignment.ID)
    if err != nil {
        w.logger.Warn("failed to load checkpoint", zap.Error(err))
    }

    // Initialize workload
    config := workload.Config{
        WorkloadID:   assignment.ID,
        CustomerID:   assignment.CustomerID,
        InputParams:  assignment.InputParams,
        ResourceLimits: workload.ResourceLimits{
            CPUCores: w.config.DefaultCPUCores,
            MemoryMB: w.config.DefaultMemoryMB,
        },
    }

    if err := workloadImpl.Init(execCtx, config, checkpointData); err != nil {
        return fmt.Errorf("workload initialization failed: %w", err)
    }
    defer workloadImpl.Close()

    // Track execution
    execution := &WorkloadExecution{
        Workload:       assignment,
        Context:        execCtx,
        Cancel:         cancel,
        WorkloadImpl:   workloadImpl,
        StartTime:      time.Now(),
        LastCheckpoint: time.Now(),
    }

    w.workloadsMu.Lock()
    w.runningWorkloads[assignment.ID] = execution
    w.workloadsMu.Unlock()

    defer func() {
        w.workloadsMu.Lock()
        delete(w.runningWorkloads, assignment.ID)
        w.workloadsMu.Unlock()
    }()

    // Update workload status to RUNNING
    if err := w.updateWorkloadStatus(ctx, assignment.ID, models.WorkloadStatusRunning); err != nil {
        return fmt.Errorf("failed to update status: %w", err)
    }

    // Start automatic checkpointing
    w.wg.Add(1)
    go func() {
        defer w.wg.Done()
        w.automaticCheckpointing(execCtx, execution)
    }()

    // Create progress reporter
    progressReporter := &ProgressReporter{
        worker:     w,
        workloadID: assignment.ID,
        logger:     w.logger,
    }

    // Execute workload
    result, execErr := w.executeWithTimeout(execCtx, workloadImpl, progressReporter, assignment)

    // Handle execution result
    if execErr != nil {
        w.logger.Error("workload execution failed",
            zap.String("workload_id", assignment.ID),
            zap.Error(execErr),
        )

        // Report failure
        w.reportWorkloadCompletion(ctx, assignment.ID, models.WorkloadStatusFailed, nil, execErr)
        return execErr
    }

    // Report success
    w.logger.Info("workload completed successfully",
        zap.String("workload_id", assignment.ID),
    )

    w.reportWorkloadCompletion(ctx, assignment.ID, models.WorkloadStatusCompleted, result, nil)
    return nil
}

// executeWithTimeout executes the workload with a timeout
func (w *WorkerImpl) executeWithTimeout(
    ctx context.Context,
    workloadImpl workload.Workload,
    progressReporter workload.ProgressReporter,
    assignment *models.Workload,
) (map[string]interface{}, error) {
    // Create timeout context if workload has timeout
    execCtx := ctx
    var cancel context.CancelFunc

    if assignment.TimeoutSeconds > 0 {
        execCtx, cancel = context.WithTimeout(ctx, time.Duration(assignment.TimeoutSeconds)*time.Second)
        defer cancel()
    }

    // Execute workload
    resultChan := make(chan error, 1)
    go func() {
        resultChan <- workloadImpl.Execute(execCtx, progressReporter)
    }()

    // Wait for completion or timeout
    select {
    case err := <-resultChan:
        if err != nil {
            return nil, err
        }
        // Get result data
        return workloadImpl.GetResult()

    case <-execCtx.Done():
        if execCtx.Err() == context.DeadlineExceeded {
            return nil, fmt.Errorf("workload execution timeout exceeded")
        }
        return nil, execCtx.Err()
    }
}

// automaticCheckpointing periodically creates checkpoints
func (w *WorkerImpl) automaticCheckpointing(ctx context.Context, execution *WorkloadExecution) {
    ticker := time.NewTicker(w.config.CheckpointInterval)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return

        case <-ticker.C:
            if err := w.createCheckpoint(ctx, execution); err != nil {
                w.logger.Error("automatic checkpoint failed",
                    zap.String("workload_id", execution.Workload.ID),
                    zap.Error(err),
                )
            }
        }
    }
}

// createCheckpoint creates a checkpoint for a workload
func (w *WorkerImpl) createCheckpoint(ctx context.Context, execution *WorkloadExecution) error {
    w.logger.Debug("creating checkpoint", zap.String("workload_id", execution.Workload.ID))

    // Get checkpoint data from workload
    checkpointData, err := execution.WorkloadImpl.Checkpoint(ctx)
    if err != nil {
        return fmt.Errorf("workload checkpoint failed: %w", err)
    }

    // Save checkpoint
    if err := w.checkpointManager.Save(ctx, execution.Workload.ID, checkpointData); err != nil {
        return fmt.Errorf("failed to save checkpoint: %w", err)
    }

    execution.LastCheckpoint = time.Now()

    w.logger.Info("checkpoint created",
        zap.String("workload_id", execution.Workload.ID),
        zap.Int("size_bytes", len(checkpointData)),
    )

    return nil
}

// loadLatestCheckpoint loads the latest checkpoint for a workload
func (w *WorkerImpl) loadLatestCheckpoint(ctx context.Context, workloadID string) ([]byte, error) {
    checkpoint, err := w.checkpointManager.GetLatest(ctx, workloadID)
    if err != nil {
        return nil, err
    }

    if checkpoint == nil {
        return nil, nil
    }

    return checkpoint.CheckpointData, nil
}

// ProgressReporter implements workload.ProgressReporter
type ProgressReporter struct {
    worker     *WorkerImpl
    workloadID string
    logger     *zap.Logger
}

// ReportProgress reports workload progress
func (pr *ProgressReporter) ReportProgress(ctx context.Context, percent int, message string) error {
    pr.logger.Debug("reporting progress",
        zap.String("workload_id", pr.workloadID),
        zap.Int("percent", percent),
        zap.String("message", message),
    )

    // Update database
    if err := pr.worker.stateManager.UpdateWorkloadProgress(ctx, pr.workloadID, percent); err != nil {
        return fmt.Errorf("failed to update progress: %w", err)
    }

    // Send progress message to coordinator via broker
    msg := broker.Message{
        MessageID:     generateMessageID(),
        Type:          "workload.progress",
        Timestamp:     time.Now(),
        SenderID:      pr.worker.id,
        SenderType:    "worker",
        CorrelationID: pr.workloadID,
        Payload: map[string]interface{}{
            "workload_id":      pr.workloadID,
            "progress_percent": percent,
            "message":          message,
        },
    }

    if err := pr.worker.brokerClient.Publish(ctx, fmt.Sprintf("worker.%s.status", pr.worker.id), msg); err != nil {
        pr.logger.Warn("failed to publish progress message", zap.Error(err))
    }

    return nil
}
```

---

## Heartbeat Sender

```go
// startHeartbeat starts the heartbeat sender
func (w *WorkerImpl) startHeartbeat(ctx context.Context) {
    ticker := time.NewTicker(w.config.HeartbeatInterval)
    defer ticker.Stop()

    w.logger.Info("heartbeat sender started", zap.Duration("interval", w.config.HeartbeatInterval))

    for {
        select {
        case <-ctx.Done():
            w.logger.Info("heartbeat sender stopping")
            return

        case <-ticker.C:
            if err := w.sendHeartbeat(ctx); err != nil {
                w.logger.Error("failed to send heartbeat", zap.Error(err))
            }
        }
    }
}

// sendHeartbeat sends a heartbeat message to the coordinator
func (w *WorkerImpl) sendHeartbeat(ctx context.Context) error {
    capacity := w.Capacity()

    // Get list of running workload IDs
    w.workloadsMu.RLock()
    runningWorkloadIDs := make([]string, 0, len(w.runningWorkloads))
    for id := range w.runningWorkloads {
        runningWorkloadIDs = append(runningWorkloadIDs, id)
    }
    w.workloadsMu.RUnlock()

    // Create heartbeat message
    msg := broker.Message{
        MessageID:  generateMessageID(),
        Type:       "worker.heartbeat",
        Timestamp:  time.Now(),
        SenderID:   w.id,
        SenderType: "worker",
        Payload: map[string]interface{}{
            "worker_id":         w.id,
            "status":            string(w.Status()),
            "capacity_total":    capacity.Total,
            "capacity_used":     capacity.Used,
            "running_workloads": runningWorkloadIDs,
        },
    }

    // Publish to worker status channel
    channel := fmt.Sprintf("worker.%s.status", w.id)
    if err := w.brokerClient.Publish(ctx, channel, msg); err != nil {
        return fmt.Errorf("failed to publish heartbeat: %w", err)
    }

    // Update database
    if err := w.stateManager.UpdateWorkerHeartbeat(ctx, w.id); err != nil {
        w.logger.Warn("failed to update heartbeat in database", zap.Error(err))
    }

    w.logger.Debug("heartbeat sent",
        zap.Int("capacity_used", capacity.Used),
        zap.Int("capacity_total", capacity.Total),
    )

    return nil
}
```

---

## Control Plane Listener

```go
// startControlPlaneListener listens for control messages from coordinator
func (w *WorkerImpl) startControlPlaneListener(ctx context.Context) {
    w.logger.Info("control plane listener started")

    // Subscribe to worker-specific command channel
    commandChannel := fmt.Sprintf("worker.%s.commands", w.id)
    msgChan, err := w.brokerClient.Subscribe(ctx, commandChannel)
    if err != nil {
        w.logger.Error("failed to subscribe to command channel", zap.Error(err))
        return
    }

    for {
        select {
        case <-ctx.Done():
            w.logger.Info("control plane listener stopping")
            return

        case msg := <-msgChan:
            w.handleControlMessage(ctx, msg)
        }
    }
}

// handleControlMessage handles a control message from the coordinator
func (w *WorkerImpl) handleControlMessage(ctx context.Context, msg broker.Message) {
    w.logger.Info("received control message",
        zap.String("type", msg.Type),
        zap.String("message_id", msg.MessageID),
    )

    switch msg.Type {
    case "workload.assign":
        w.handleWorkloadAssignment(ctx, msg)

    case "workload.cancel":
        w.handleWorkloadCancellation(ctx, msg)

    case "worker.drain":
        w.handleDrainRequest(ctx, msg)

    default:
        w.logger.Warn("unknown control message type", zap.String("type", msg.Type))
    }
}

// handleWorkloadAssignment handles a workload assignment
func (w *WorkerImpl) handleWorkloadAssignment(ctx context.Context, msg broker.Message) {
    workloadID, ok := msg.Payload["workload_id"].(string)
    if !ok {
        w.logger.Error("invalid workload assignment: missing workload_id")
        return
    }

    w.logger.Info("received workload assignment", zap.String("workload_id", workloadID))

    // Fetch full workload details from state manager
    workload, err := w.stateManager.GetWorkload(ctx, workloadID)
    if err != nil {
        w.logger.Error("failed to fetch workload", zap.String("workload_id", workloadID), zap.Error(err))
        return
    }

    // Execute workload in goroutine
    w.wg.Add(1)
    go func() {
        defer w.wg.Done()

        if err := w.ExecuteWorkload(ctx, workload); err != nil {
            w.logger.Error("workload execution failed",
                zap.String("workload_id", workloadID),
                zap.Error(err),
            )
        }
    }()

    // Send acknowledgment
    ack := broker.Message{
        MessageID:     generateMessageID(),
        Type:          "workload.ack",
        Timestamp:     time.Now(),
        SenderID:      w.id,
        SenderType:    "worker",
        CorrelationID: msg.MessageID,
        Payload: map[string]interface{}{
            "workload_id": workloadID,
            "accepted":    true,
        },
    }

    if err := w.brokerClient.Publish(ctx, "coordinator.events", ack); err != nil {
        w.logger.Warn("failed to send acknowledgment", zap.Error(err))
    }
}

// handleWorkloadCancellation handles a workload cancellation request
func (w *WorkerImpl) handleWorkloadCancellation(ctx context.Context, msg broker.Message) {
    workloadID, ok := msg.Payload["workload_id"].(string)
    if !ok {
        w.logger.Error("invalid cancellation: missing workload_id")
        return
    }

    w.logger.Info("received cancellation request", zap.String("workload_id", workloadID))

    // Find and cancel the workload
    w.workloadsMu.Lock()
    execution, exists := w.runningWorkloads[workloadID]
    w.workloadsMu.Unlock()

    if !exists {
        w.logger.Warn("workload not running", zap.String("workload_id", workloadID))
        return
    }

    // Cancel the execution context
    execution.Cancel()

    // Create final checkpoint
    if err := w.createCheckpoint(ctx, execution); err != nil {
        w.logger.Warn("failed to create final checkpoint", zap.Error(err))
    }

    w.logger.Info("workload cancelled", zap.String("workload_id", workloadID))
}

// handleDrainRequest handles a drain request from coordinator
func (w *WorkerImpl) handleDrainRequest(ctx context.Context, msg broker.Message) {
    w.logger.Info("received drain request")

    w.statusMu.Lock()
    w.status = WorkerStatusDraining
    w.statusMu.Unlock()

    // Coordinator will stop assigning new workloads
    // Existing workloads will complete naturally
}
```

---

## Registration Service

```go
// registerWithCoordinator registers this worker with the coordinator
func (w *WorkerImpl) registerWithCoordinator(ctx context.Context) error {
    w.logger.Info("registering with coordinator",
        zap.Strings("capabilities", w.capabilities),
        zap.Int("capacity", w.capacityTotal),
    )

    // Create worker record in database
    worker := &models.Worker{
        ID:            w.id,
        Status:        models.WorkerStatusHealthy,
        Capabilities:  w.capabilities,
        CapacityTotal: w.capacityTotal,
        CapacityUsed:  0,
        Tags:          w.tags,
        LastHeartbeat: time.Now(),
        RegisteredAt:  time.Now(),
    }

    if err := w.stateManager.RegisterWorker(ctx, worker); err != nil {
        return fmt.Errorf("failed to register worker in database: %w", err)
    }

    // Send registration message via broker
    msg := broker.Message{
        MessageID:  generateMessageID(),
        Type:       "worker.register",
        Timestamp:  time.Now(),
        SenderID:   w.id,
        SenderType: "worker",
        Payload: map[string]interface{}{
            "worker_id":      w.id,
            "capabilities":   w.capabilities,
            "capacity_total": w.capacityTotal,
            "tags":           w.tags,
            "version":        "1.0.0",
        },
    }

    if err := w.brokerClient.Publish(ctx, "coordinator.events", msg); err != nil {
        return fmt.Errorf("failed to publish registration message: %w", err)
    }

    w.logger.Info("successfully registered with coordinator")
    return nil
}
```

---

## Resource Isolation

### CPU and Memory Limits (Linux)

```go
package worker

import (
    "fmt"
    "os"
    "path/filepath"
    "strconv"
    "strings"
)

// applyCGroupLimits applies cgroup-based resource limits
func (w *WorkerImpl) applyCGroupLimits(workloadID string, limits workload.ResourceLimits) error {
    // This is a simplified example for Linux cgroups v2
    // In production, use a library like github.com/containerd/cgroups

    cgroupPath := filepath.Join("/sys/fs/cgroup", "niyanta", workloadID)

    // Create cgroup
    if err := os.MkdirAll(cgroupPath, 0755); err != nil {
        return fmt.Errorf("failed to create cgroup: %w", err)
    }

    // Set CPU limit (in microseconds per 100ms period)
    cpuMax := fmt.Sprintf("%d 100000", limits.CPUCores*100000)
    if err := os.WriteFile(filepath.Join(cgroupPath, "cpu.max"), []byte(cpuMax), 0644); err != nil {
        return fmt.Errorf("failed to set CPU limit: %w", err)
    }

    // Set memory limit (in bytes)
    memoryMax := fmt.Sprintf("%d", limits.MemoryMB*1024*1024)
    if err := os.WriteFile(filepath.Join(cgroupPath, "memory.max"), []byte(memoryMax), 0644); err != nil {
        return fmt.Errorf("failed to set memory limit: %w", err)
    }

    // Add current process to cgroup
    pid := os.Getpid()
    if err := os.WriteFile(filepath.Join(cgroupPath, "cgroup.procs"), []byte(strconv.Itoa(pid)), 0644); err != nil {
        return fmt.Errorf("failed to add process to cgroup: %w", err)
    }

    w.logger.Info("applied resource limits",
        zap.String("workload_id", workloadID),
        zap.Int("cpu_cores", limits.CPUCores),
        zap.Int("memory_mb", limits.MemoryMB),
    )

    return nil
}

// removeCGroupLimits removes cgroup limits after workload completion
func (w *WorkerImpl) removeCGroupLimits(workloadID string) error {
    cgroupPath := filepath.Join("/sys/fs/cgroup", "niyanta", workloadID)
    return os.RemoveAll(cgroupPath)
}
```

---

## Complete Implementation Example

### Worker Start Method

```go
// Start starts the worker
func (w *WorkerImpl) Start(ctx context.Context) error {
    w.logger.Info("starting worker", zap.String("worker_id", w.id))

    // Register with coordinator
    if err := w.registerWithCoordinator(ctx); err != nil {
        return fmt.Errorf("registration failed: %w", err)
    }

    // Start heartbeat sender
    w.wg.Add(1)
    go func() {
        defer w.wg.Done()
        w.startHeartbeat(ctx)
    }()

    // Start control plane listener
    w.wg.Add(1)
    go func() {
        defer w.wg.Done()
        w.startControlPlaneListener(ctx)
    }()

    w.logger.Info("worker started successfully")

    // Wait for context cancellation
    <-ctx.Done()

    w.logger.Info("worker stopping")
    return nil
}

// Shutdown gracefully shuts down the worker
func (w *WorkerImpl) Shutdown(ctx context.Context) error {
    var shutdownErr error

    w.shutdownOnce.Do(func() {
        w.logger.Info("initiating graceful shutdown")

        // Set status to draining
        w.statusMu.Lock()
        w.status = WorkerStatusDraining
        w.statusMu.Unlock()

        // Cancel all running workloads
        w.workloadsMu.Lock()
        for _, execution := range w.runningWorkloads {
            w.logger.Info("cancelling workload", zap.String("workload_id", execution.Workload.ID))
            execution.Cancel()

            // Create final checkpoint
            if err := w.createCheckpoint(context.Background(), execution); err != nil {
                w.logger.Warn("failed to create final checkpoint",
                    zap.String("workload_id", execution.Workload.ID),
                    zap.Error(err),
                )
            }
        }
        w.workloadsMu.Unlock()

        // Wait for workloads to finish with timeout
        done := make(chan struct{})
        go func() {
            w.wg.Wait()
            close(done)
        }()

        select {
        case <-done:
            w.logger.Info("all workloads stopped")
        case <-ctx.Done():
            w.logger.Warn("shutdown timeout exceeded, forcing stop")
            shutdownErr = ctx.Err()
        }

        // Unregister from coordinator
        if err := w.stateManager.UnregisterWorker(context.Background(), w.id); err != nil {
            w.logger.Error("failed to unregister worker", zap.Error(err))
        }

        w.logger.Info("worker shutdown complete")
    })

    return shutdownErr
}

// ID returns the worker ID
func (w *WorkerImpl) ID() string {
    return w.id
}

// Status returns the current status
func (w *WorkerImpl) Status() WorkerStatus {
    w.statusMu.RLock()
    defer w.statusMu.RUnlock()
    return w.status
}

// Capacity returns capacity information
func (w *WorkerImpl) Capacity() CapacityInfo {
    used := len(w.capacitySemaphore)
    return CapacityInfo{
        Total: w.capacityTotal,
        Used:  used,
        Free:  w.capacityTotal - used,
    }
}
```

### Main Entry Point (cmd/worker/main.go)

```go
package main

import (
    "context"
    "fmt"
    "os"
    "os/signal"
    "syscall"
    "time"

    "github.com/yourusername/niyanta/internal/broker/nats"
    "github.com/yourusername/niyanta/internal/checkpoint"
    "github.com/yourusername/niyanta/internal/config"
    "github.com/yourusername/niyanta/internal/observability/logger"
    "github.com/yourusername/niyanta/internal/storage/postgres"
    "github.com/yourusername/niyanta/internal/worker"
    "github.com/yourusername/niyanta/internal/workload"
    "go.uber.org/zap"
)

func main() {
    // Load configuration
    cfg, err := config.LoadWorkerConfig()
    if err != nil {
        fmt.Fprintf(os.Stderr, "Failed to load configuration: %v\n", err)
        os.Exit(1)
    }

    // Initialize logger
    log, err := logger.NewLogger(cfg.Logging)
    if err != nil {
        fmt.Fprintf(os.Stderr, "Failed to initialize logger: %v\n", err)
        os.Exit(1)
    }
    defer log.Sync()

    log.Info("starting niyanta worker",
        zap.String("version", "1.0.0"),
        zap.String("worker_id", cfg.Worker.WorkerID),
    )

    // Initialize state manager
    stateManager, err := postgres.NewStateManager(cfg.Database, log)
    if err != nil {
        log.Fatal("failed to initialize state manager", zap.Error(err))
    }
    defer stateManager.Close()

    // Initialize broker client
    brokerClient, err := nats.NewBrokerClient(cfg.Broker, log)
    if err != nil {
        log.Fatal("failed to initialize broker client", zap.Error(err))
    }
    defer brokerClient.Close()

    // Initialize checkpoint manager
    checkpointMgr := checkpoint.NewManager(stateManager, log)

    // Initialize workload registry and register workload types
    registry := workload.NewRegistry(log)

    // Register built-in workload types
    if err := registerWorkloadTypes(registry); err != nil {
        log.Fatal("failed to register workload types", zap.Error(err))
    }

    // Create worker
    w, err := worker.NewWorker(
        cfg.Worker,
        stateManager,
        brokerClient,
        checkpointMgr,
        registry,
        log,
    )
    if err != nil {
        log.Fatal("failed to create worker", zap.Error(err))
    }

    // Set up context with cancellation
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // Handle signals for graceful shutdown
    sigChan := make(chan os.Signal, 1)
    signal.Notify(sigChan, syscall.SIGINT, syscall.SIGTERM)

    // Start worker in goroutine
    errChan := make(chan error, 1)
    go func() {
        if err := w.Start(ctx); err != nil {
            errChan <- err
        }
    }()

    // Wait for shutdown signal or error
    select {
    case sig := <-sigChan:
        log.Info("received shutdown signal", zap.String("signal", sig.String()))
        cancel()

    case err := <-errChan:
        log.Error("worker error", zap.Error(err))
        cancel()
    }

    // Graceful shutdown with timeout
    shutdownCtx, shutdownCancel := context.WithTimeout(context.Background(), cfg.Worker.DrainTimeout)
    defer shutdownCancel()

    if err := w.Shutdown(shutdownCtx); err != nil {
        log.Error("shutdown failed", zap.Error(err))
        os.Exit(1)
    }

    log.Info("worker stopped successfully")
}

func registerWorkloadTypes(registry workload.Registry) error {
    // Register built-in workload types
    // See 09_WORKLOAD_INTERFACE.md for workload implementations
    return nil
}
```

---

## Testing

### Unit Test Example

```go
package worker

import (
    "context"
    "testing"
    "time"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/mock"
    "go.uber.org/zap/zaptest"
)

func TestWorker_ExecuteWorkload(t *testing.T) {
    // Create mocks
    mockStateManager := mock_storage.NewMockStateManager()
    mockBrokerClient := mock_broker.NewMockBrokerClient()
    mockCheckpointMgr := mock_checkpoint.NewMockManager()
    mockRegistry := mock_workload.NewMockRegistry()
    mockWorkload := mock_workload.NewMockWorkload()

    logger := zaptest.NewLogger(t)

    config := Config{
        WorkerID:               "test-worker",
        Capabilities:           []string{"test_workload"},
        MaxConcurrentWorkloads: 5,
        HeartbeatInterval:      10 * time.Second,
    }

    // Configure mocks
    mockRegistry.On("Get", "test_workload").Return(mockWorkload, nil)
    mockCheckpointMgr.On("GetLatest", mock.Anything, "wl_123").Return(nil, nil)
    mockWorkload.On("Init", mock.Anything, mock.Anything, mock.Anything).Return(nil)
    mockWorkload.On("Execute", mock.Anything, mock.Anything).Return(nil)
    mockWorkload.On("GetResult").Return(map[string]interface{}{"status": "success"}, nil)
    mockWorkload.On("Close").Return(nil)
    mockStateManager.On("UpdateWorkloadStatus", mock.Anything, "wl_123", mock.Anything).Return(nil)
    mockBrokerClient.On("Publish", mock.Anything, mock.Anything, mock.Anything).Return(nil)

    // Create worker
    w, err := NewWorker(config, mockStateManager, mockBrokerClient, mockCheckpointMgr, mockRegistry, logger)
    assert.NoError(t, err)

    // Create test workload
    testWorkload := &models.Workload{
        ID:               "wl_123",
        CustomerID:       "cust_1",
        WorkloadConfigID: "test_workload",
        Status:           models.WorkloadStatusScheduled,
    }

    // Execute workload
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    err = w.ExecuteWorkload(ctx, testWorkload)
    assert.NoError(t, err)

    // Verify mocks
    mockRegistry.AssertExpectations(t)
    mockWorkload.AssertExpectations(t)
}
```

---

## Phase Implementation

### Phase 1: MVP
- Basic workload execution
- Heartbeat sender
- Simple resource limits
- Manual checkpointing

### Phase 2: Production
- Automatic checkpointing
- Enhanced resource isolation (cgroups)
- Graceful shutdown with draining
- Comprehensive error handling

### Phase 3: Advanced
- Dynamic capacity adjustment
- Advanced resource metrics
- Workload prioritization
- GPU support

---

**Next**: [04_STATE_MANAGER.md](04_STATE_MANAGER.md) - State storage implementation
