# Coordinator Implementation

**Component**: Coordinator (Control Plane)
**Last Updated**: 2025-10-29
**Status**: Implementation Ready

## Table of Contents
1. [Overview](#overview)
2. [Core Interfaces](#core-interfaces)
3. [Coordinator Struct](#coordinator-struct)
4. [API Server Implementation](#api-server-implementation)
5. [Leader Election](#leader-election)
6. [Component Integration](#component-integration)
7. [Graceful Shutdown](#graceful-shutdown)
8. [Complete Implementation Example](#complete-implementation-example)
9. [Testing](#testing)

---

## Overview

The Coordinator is the control plane of Niyanta, responsible for:
- Exposing REST/gRPC APIs for workload management
- Coordinating scheduling, health monitoring, and state management
- Supporting High Availability (HA) through leader election
- Managing workload lifecycle state transitions

**Package**: `internal/coordinator`

**Key Design Principles**:
- Interface-driven design for testability
- Dependency injection for all components
- Graceful shutdown with context cancellation
- Leader-only operations (scheduler, health monitor)

---

## Core Interfaces

### Coordinator Interface

```go
package coordinator

import (
    "context"
)

// Coordinator defines the control plane interface
type Coordinator interface {
    // Start initializes and starts all coordinator components
    // This is a blocking call that runs until context is cancelled
    Start(ctx context.Context) error

    // Shutdown gracefully stops all components
    Shutdown(ctx context.Context) error

    // IsLeader returns whether this coordinator is currently the leader
    IsLeader() bool

    // Health returns the current health status
    Health() HealthStatus
}

// HealthStatus represents the health state of the coordinator
type HealthStatus struct {
    Healthy       bool              `json:"healthy"`
    IsLeader      bool              `json:"is_leader"`
    Dependencies  map[string]string `json:"dependencies"` // "connected", "degraded", "disconnected"
    Uptime        int64             `json:"uptime_seconds"`
}
```

---

## Coordinator Struct

### Main Coordinator Structure

```go
package coordinator

import (
    "context"
    "sync"
    "time"

    "github.com/yourusername/niyanta/internal/broker"
    "github.com/yourusername/niyanta/internal/health"
    "github.com/yourusername/niyanta/internal/scheduler"
    "github.com/yourusername/niyanta/internal/storage"
    "go.uber.org/zap"
)

// CoordinatorImpl is the concrete implementation of Coordinator
type CoordinatorImpl struct {
    // Configuration
    config Config

    // Core dependencies
    stateManager  storage.StateManager
    brokerClient  broker.BrokerClient
    scheduler     scheduler.Scheduler
    healthMonitor health.Monitor
    logger        *zap.Logger

    // API server
    apiServer APIServer

    // Leader election
    leaderElector LeaderElector
    isLeader      bool
    leaderMu      sync.RWMutex

    // Lifecycle management
    startTime     time.Time
    shutdownOnce  sync.Once
    wg            sync.WaitGroup
}

// Config holds coordinator configuration
type Config struct {
    // Server configuration
    ServerPort     int           `yaml:"server_port" env:"SERVER_PORT" default:"8080"`
    GRPCPort       int           `yaml:"grpc_port" env:"GRPC_PORT" default:"9090"`

    // Leader election
    EnableHA       bool          `yaml:"enable_ha" env:"ENABLE_HA" default:"false"`
    LeaderLeaseTTL time.Duration `yaml:"leader_lease_ttl" env:"LEADER_LEASE_TTL" default:"10s"`

    // Scheduler configuration
    SchedulerInterval time.Duration `yaml:"scheduler_interval" env:"SCHEDULER_INTERVAL" default:"5s"`
    BatchSize         int           `yaml:"batch_size" env:"BATCH_SIZE" default:"100"`

    // Health monitoring
    HealthCheckInterval time.Duration `yaml:"health_check_interval" env:"HEALTH_CHECK_INTERVAL" default:"30s"`
    HeartbeatTimeout    time.Duration `yaml:"heartbeat_timeout" env:"HEARTBEAT_TIMEOUT" default:"60s"`

    // Worker management
    WorkerDeadThreshold time.Duration `yaml:"worker_dead_threshold" env:"WORKER_DEAD_THRESHOLD" default:"90s"`
}

// NewCoordinator creates a new coordinator instance
func NewCoordinator(
    config Config,
    stateManager storage.StateManager,
    brokerClient broker.BrokerClient,
    scheduler scheduler.Scheduler,
    healthMonitor health.Monitor,
    logger *zap.Logger,
) (*CoordinatorImpl, error) {
    // Validate configuration
    if err := config.Validate(); err != nil {
        return nil, fmt.Errorf("invalid configuration: %w", err)
    }

    coord := &CoordinatorImpl{
        config:        config,
        stateManager:  stateManager,
        brokerClient:  brokerClient,
        scheduler:     scheduler,
        healthMonitor: healthMonitor,
        logger:        logger.With(zap.String("component", "coordinator")),
        startTime:     time.Now(),
    }

    // Initialize API server
    apiServer, err := NewAPIServer(coord, config, logger)
    if err != nil {
        return nil, fmt.Errorf("failed to create API server: %w", err)
    }
    coord.apiServer = apiServer

    // Initialize leader elector if HA is enabled
    if config.EnableHA {
        leaderElector, err := NewLeaderElector(config, stateManager, logger)
        if err != nil {
            return nil, fmt.Errorf("failed to create leader elector: %w", err)
        }
        coord.leaderElector = leaderElector
    }

    return coord, nil
}

// Validate validates the configuration
func (c *Config) Validate() error {
    if c.ServerPort < 1 || c.ServerPort > 65535 {
        return fmt.Errorf("invalid server port: %d", c.ServerPort)
    }
    if c.GRPCPort < 1 || c.GRPCPort > 65535 {
        return fmt.Errorf("invalid gRPC port: %d", c.GRPCPort)
    }
    if c.SchedulerInterval < time.Second {
        return fmt.Errorf("scheduler interval too short: %v", c.SchedulerInterval)
    }
    if c.BatchSize < 1 {
        return fmt.Errorf("invalid batch size: %d", c.BatchSize)
    }
    return nil
}
```

---

## API Server Implementation

### API Server Interface

```go
package coordinator

import (
    "context"
    "net/http"
)

// APIServer defines the HTTP/gRPC server interface
type APIServer interface {
    // Start starts the API server (blocking)
    Start(ctx context.Context) error

    // Shutdown gracefully shuts down the server
    Shutdown(ctx context.Context) error

    // Port returns the port the server is listening on
    Port() int
}
```

### Gin-Based REST API Server

```go
package coordinator

import (
    "context"
    "fmt"
    "net/http"
    "time"

    "github.com/gin-gonic/gin"
    "github.com/yourusername/niyanta/internal/auth"
    "github.com/yourusername/niyanta/internal/ratelimit"
    "github.com/yourusername/niyanta/pkg/api/v1"
    "go.uber.org/zap"
)

// APIServerImpl implements APIServer using Gin
type APIServerImpl struct {
    coordinator *CoordinatorImpl
    config      Config
    logger      *zap.Logger

    router     *gin.Engine
    httpServer *http.Server
}

// NewAPIServer creates a new API server
func NewAPIServer(coord *CoordinatorImpl, config Config, logger *zap.Logger) (*APIServerImpl, error) {
    // Set Gin mode
    if logger.Core().Enabled(zap.DebugLevel) {
        gin.SetMode(gin.DebugMode)
    } else {
        gin.SetMode(gin.ReleaseMode)
    }

    router := gin.New()

    // Global middleware
    router.Use(gin.Recovery())
    router.Use(LoggingMiddleware(logger))
    router.Use(RequestIDMiddleware())

    server := &APIServerImpl{
        coordinator: coord,
        config:      config,
        logger:      logger.With(zap.String("component", "api_server")),
        router:      router,
    }

    // Register routes
    server.registerRoutes()

    return server, nil
}

// registerRoutes sets up all API routes
func (s *APIServerImpl) registerRoutes() {
    // Health endpoints (no auth)
    s.router.GET("/health", s.handleHealth)
    s.router.GET("/ready", s.handleReady)
    s.router.GET("/metrics", s.handleMetrics)

    // API v1 routes
    v1 := s.router.Group("/v1")

    // Authentication middleware
    authMiddleware := auth.NewAPIKeyMiddleware(s.coordinator.stateManager, s.logger)
    v1.Use(authMiddleware.Authenticate())

    // Rate limiting middleware
    rateLimiter := ratelimit.NewTokenBucketLimiter(s.coordinator.stateManager, s.logger)
    v1.Use(rateLimiter.Limit())

    // Workload management
    workloads := v1.Group("/workloads")
    {
        workloads.POST("", s.handleSubmitWorkload)
        workloads.GET("", s.handleListWorkloads)
        workloads.GET("/:workload_id", s.handleGetWorkload)
        workloads.POST("/:workload_id/cancel", s.handleCancelWorkload)
        workloads.POST("/:workload_id/pause", s.handlePauseWorkload)
        workloads.POST("/:workload_id/resume", s.handleResumeWorkload)
        workloads.GET("/:workload_id/logs", s.handleGetWorkloadLogs)
        workloads.GET("/:workload_id/audit", s.handleGetWorkloadAudit)
    }

    // Workload configuration
    configs := v1.Group("/workload-configs")
    {
        configs.POST("", s.handleCreateWorkloadConfig)
        configs.GET("", s.handleListWorkloadConfigs)
        configs.GET("/:config_id", s.handleGetWorkloadConfig)
        configs.PUT("/:config_id", s.handleUpdateWorkloadConfig)
        configs.DELETE("/:config_id", s.handleDeleteWorkloadConfig)
    }

    // Customer management
    customer := v1.Group("/customer")
    {
        customer.GET("", s.handleGetCustomer)
        customer.GET("/stats", s.handleGetCustomerStats)
    }

    // Admin endpoints
    admin := v1.Group("/admin")
    admin.Use(auth.NewAdminMiddleware(s.logger).Authenticate())
    {
        admin.GET("/workers", s.handleListWorkers)
        admin.POST("/workers/:worker_id/drain", s.handleDrainWorker)
        admin.DELETE("/workers/:worker_id", s.handleRemoveWorker)
    }
}

// Start starts the API server
func (s *APIServerImpl) Start(ctx context.Context) error {
    addr := fmt.Sprintf(":%d", s.config.ServerPort)

    s.httpServer = &http.Server{
        Addr:    addr,
        Handler: s.router,

        ReadTimeout:       10 * time.Second,
        WriteTimeout:      30 * time.Second,
        IdleTimeout:       60 * time.Second,
        ReadHeaderTimeout: 5 * time.Second,
    }

    s.logger.Info("starting API server", zap.String("addr", addr))

    // Start server in goroutine
    errChan := make(chan error, 1)
    go func() {
        if err := s.httpServer.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            errChan <- fmt.Errorf("API server error: %w", err)
        }
    }()

    // Wait for context cancellation or error
    select {
    case <-ctx.Done():
        s.logger.Info("API server stopping due to context cancellation")
        return nil
    case err := <-errChan:
        return err
    }
}

// Shutdown gracefully shuts down the API server
func (s *APIServerImpl) Shutdown(ctx context.Context) error {
    if s.httpServer == nil {
        return nil
    }

    s.logger.Info("shutting down API server")
    return s.httpServer.Shutdown(ctx)
}

// Port returns the listening port
func (s *APIServerImpl) Port() int {
    return s.config.ServerPort
}

// Health check handler
func (s *APIServerImpl) handleHealth(c *gin.Context) {
    c.JSON(http.StatusOK, gin.H{
        "status":    "healthy",
        "timestamp": time.Now().Format(time.RFC3339),
    })
}

// Readiness check handler
func (s *APIServerImpl) handleReady(c *gin.Context) {
    health := s.coordinator.Health()

    if health.Healthy {
        c.JSON(http.StatusOK, health)
    } else {
        c.JSON(http.StatusServiceUnavailable, health)
    }
}

// Metrics handler (Prometheus format)
func (s *APIServerImpl) handleMetrics(c *gin.Context) {
    // Metrics are handled by prometheus handler (see 11_OBSERVABILITY.md)
    // This is a placeholder that would delegate to prometheus.Handler()
    c.String(http.StatusOK, "# Metrics endpoint\n")
}
```

---

## Leader Election

### Leader Elector Interface

```go
package coordinator

import (
    "context"
)

// LeaderElector manages leader election for HA
type LeaderElector interface {
    // Start begins the leader election process
    Start(ctx context.Context) error

    // IsLeader returns whether this instance is the leader
    IsLeader() bool

    // OnBecomeLeader registers a callback for when this instance becomes leader
    OnBecomeLeader(callback func(ctx context.Context))

    // OnLoseLeadership registers a callback for when this instance loses leadership
    OnLoseLeadership(callback func())
}
```

### PostgreSQL Advisory Lock Implementation

```go
package coordinator

import (
    "context"
    "fmt"
    "sync"
    "time"

    "github.com/yourusername/niyanta/internal/storage"
    "go.uber.org/zap"
)

// LeaderElectorImpl implements leader election using PostgreSQL advisory locks
type LeaderElectorImpl struct {
    config       Config
    stateManager storage.StateManager
    logger       *zap.Logger

    isLeader         bool
    leaderMu         sync.RWMutex
    generation       int64

    onBecomeLeader   func(ctx context.Context)
    onLoseLeadership func()
}

// NewLeaderElector creates a new leader elector
func NewLeaderElector(
    config Config,
    stateManager storage.StateManager,
    logger *zap.Logger,
) (*LeaderElectorImpl, error) {
    return &LeaderElectorImpl{
        config:       config,
        stateManager: stateManager,
        logger:       logger.With(zap.String("component", "leader_elector")),
    }, nil
}

// Start begins the leader election process
func (le *LeaderElectorImpl) Start(ctx context.Context) error {
    le.logger.Info("starting leader election")

    ticker := time.NewTicker(le.config.LeaderLeaseTTL / 2)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            le.logger.Info("stopping leader election")
            le.releaseLock(context.Background())
            return nil

        case <-ticker.C:
            le.tryAcquireLock(ctx)
        }
    }
}

// tryAcquireLock attempts to acquire or renew the leader lock
func (le *LeaderElectorImpl) tryAcquireLock(ctx context.Context) {
    wasLeader := le.IsLeader()

    // Try to acquire/renew lock
    acquired, generation, err := le.stateManager.AcquireLeaderLock(
        ctx,
        "coordinator-leader",
        le.config.LeaderLeaseTTL,
    )

    if err != nil {
        le.logger.Error("failed to acquire leader lock", zap.Error(err))
        if wasLeader {
            le.loseLeadership()
        }
        return
    }

    if acquired {
        if !wasLeader {
            le.becomeLeader(ctx, generation)
        } else {
            // Successfully renewed lock
            le.logger.Debug("renewed leader lock")
        }
    } else {
        if wasLeader {
            le.loseLeadership()
        }
    }
}

// becomeLeader transitions to leader state
func (le *LeaderElectorImpl) becomeLeader(ctx context.Context, generation int64) {
    le.leaderMu.Lock()
    le.isLeader = true
    le.generation = generation
    le.leaderMu.Unlock()

    le.logger.Info("became leader", zap.Int64("generation", generation))

    if le.onBecomeLeader != nil {
        go le.onBecomeLeader(ctx)
    }
}

// loseLeadership transitions to follower state
func (le *LeaderElectorImpl) loseLeadership() {
    le.leaderMu.Lock()
    le.isLeader = false
    le.leaderMu.Unlock()

    le.logger.Info("lost leadership")

    if le.onLoseLeadership != nil {
        go le.onLoseLeadership()
    }
}

// releaseLock releases the leader lock
func (le *LeaderElectorImpl) releaseLock(ctx context.Context) {
    if err := le.stateManager.ReleaseLeaderLock(ctx, "coordinator-leader"); err != nil {
        le.logger.Error("failed to release leader lock", zap.Error(err))
    }
}

// IsLeader returns whether this instance is the leader
func (le *LeaderElectorImpl) IsLeader() bool {
    le.leaderMu.RLock()
    defer le.leaderMu.RUnlock()
    return le.isLeader
}

// OnBecomeLeader registers a callback
func (le *LeaderElectorImpl) OnBecomeLeader(callback func(ctx context.Context)) {
    le.onBecomeLeader = callback
}

// OnLoseLeadership registers a callback
func (le *LeaderElectorImpl) OnLoseLeadership(callback func()) {
    le.onLoseLeadership = callback
}
```

---

## Component Integration

### Coordinator Start Method

```go
// Start starts all coordinator components
func (c *CoordinatorImpl) Start(ctx context.Context) error {
    c.logger.Info("starting coordinator")

    // Start leader election if HA is enabled
    if c.leaderElector != nil {
        c.leaderElector.OnBecomeLeader(c.onBecomeLeader)
        c.leaderElector.OnLoseLeadership(c.onLoseLeadership)

        c.wg.Add(1)
        go func() {
            defer c.wg.Done()
            if err := c.leaderElector.Start(ctx); err != nil {
                c.logger.Error("leader elector failed", zap.Error(err))
            }
        }()
    } else {
        // No HA, always leader
        c.setLeader(true)
        c.onBecomeLeader(ctx)
    }

    // Start API server
    c.wg.Add(1)
    go func() {
        defer c.wg.Done()
        if err := c.apiServer.Start(ctx); err != nil {
            c.logger.Error("API server failed", zap.Error(err))
        }
    }()

    // Start broker message processor
    c.wg.Add(1)
    go func() {
        defer c.wg.Done()
        c.processBrokerMessages(ctx)
    }()

    c.logger.Info("coordinator started successfully")

    // Wait for context cancellation
    <-ctx.Done()

    c.logger.Info("coordinator stopping")
    return nil
}

// onBecomeLeader is called when this instance becomes the leader
func (c *CoordinatorImpl) onBecomeLeader(ctx context.Context) {
    c.logger.Info("starting leader-only components")

    // Start scheduler
    c.wg.Add(1)
    go func() {
        defer c.wg.Done()
        if err := c.runScheduler(ctx); err != nil {
            c.logger.Error("scheduler failed", zap.Error(err))
        }
    }()

    // Start health monitor
    c.wg.Add(1)
    go func() {
        defer c.wg.Done()
        if err := c.healthMonitor.Start(ctx); err != nil {
            c.logger.Error("health monitor failed", zap.Error(err))
        }
    }()
}

// onLoseLeadership is called when this instance loses leadership
func (c *CoordinatorImpl) onLoseLeadership() {
    c.logger.Info("stopping leader-only components")
    // Components will stop when their context is cancelled
}

// runScheduler runs the scheduling loop
func (c *CoordinatorImpl) runScheduler(ctx context.Context) error {
    ticker := time.NewTicker(c.config.SchedulerInterval)
    defer ticker.Stop()

    c.logger.Info("scheduler started", zap.Duration("interval", c.config.SchedulerInterval))

    for {
        select {
        case <-ctx.Done():
            c.logger.Info("scheduler stopping")
            return nil

        case <-ticker.C:
            if !c.IsLeader() {
                continue
            }

            // Run scheduling batch
            scheduled, err := c.scheduler.ScheduleNextBatch(ctx, c.config.BatchSize)
            if err != nil {
                c.logger.Error("scheduling failed", zap.Error(err))
                continue
            }

            if scheduled > 0 {
                c.logger.Info("scheduled workloads", zap.Int("count", scheduled))
            }
        }
    }
}

// processBrokerMessages handles incoming broker messages
func (c *CoordinatorImpl) processBrokerMessages(ctx context.Context) {
    c.logger.Info("starting broker message processor")

    // Subscribe to coordinator events channel
    msgChan, err := c.brokerClient.Subscribe(ctx, "coordinator.events")
    if err != nil {
        c.logger.Error("failed to subscribe to broker", zap.Error(err))
        return
    }

    for {
        select {
        case <-ctx.Done():
            c.logger.Info("broker message processor stopping")
            return

        case msg := <-msgChan:
            c.handleBrokerMessage(ctx, msg)
        }
    }
}

// handleBrokerMessage processes a single broker message
func (c *CoordinatorImpl) handleBrokerMessage(ctx context.Context, msg broker.Message) {
    c.logger.Debug("received broker message",
        zap.String("type", msg.Type),
        zap.String("sender", msg.SenderID),
    )

    switch msg.Type {
    case "worker.register":
        c.handleWorkerRegistration(ctx, msg)
    case "worker.heartbeat":
        c.handleWorkerHeartbeat(ctx, msg)
    case "workload.completed":
        c.handleWorkloadCompletion(ctx, msg)
    case "workload.failed":
        c.handleWorkloadFailure(ctx, msg)
    case "workload.progress":
        c.handleWorkloadProgress(ctx, msg)
    default:
        c.logger.Warn("unknown message type", zap.String("type", msg.Type))
    }
}

// setLeader sets the leader status
func (c *CoordinatorImpl) setLeader(isLeader bool) {
    c.leaderMu.Lock()
    c.isLeader = isLeader
    c.leaderMu.Unlock()
}

// IsLeader returns whether this coordinator is the leader
func (c *CoordinatorImpl) IsLeader() bool {
    c.leaderMu.RLock()
    defer c.leaderMu.RUnlock()
    return c.isLeader
}
```

---

## Graceful Shutdown

```go
// Shutdown gracefully shuts down the coordinator
func (c *CoordinatorImpl) Shutdown(ctx context.Context) error {
    var shutdownErr error

    c.shutdownOnce.Do(func() {
        c.logger.Info("initiating graceful shutdown")

        // Shutdown API server first (stop accepting new requests)
        if err := c.apiServer.Shutdown(ctx); err != nil {
            c.logger.Error("API server shutdown failed", zap.Error(err))
            shutdownErr = err
        }

        // Release leader lock if we're the leader
        if c.leaderElector != nil && c.IsLeader() {
            if err := c.stateManager.ReleaseLeaderLock(ctx, "coordinator-leader"); err != nil {
                c.logger.Error("failed to release leader lock", zap.Error(err))
            }
        }

        // Wait for all goroutines to finish
        done := make(chan struct{})
        go func() {
            c.wg.Wait()
            close(done)
        }()

        select {
        case <-done:
            c.logger.Info("all components stopped")
        case <-ctx.Done():
            c.logger.Warn("shutdown timeout exceeded")
            shutdownErr = ctx.Err()
        }

        c.logger.Info("coordinator shutdown complete")
    })

    return shutdownErr
}

// Health returns the current health status
func (c *CoordinatorImpl) Health() HealthStatus {
    dependencies := make(map[string]string)

    // Check database connection
    if err := c.stateManager.Ping(context.Background()); err != nil {
        dependencies["database"] = "disconnected"
    } else {
        dependencies["database"] = "connected"
    }

    // Check broker connection
    if err := c.brokerClient.Ping(context.Background()); err != nil {
        dependencies["broker"] = "disconnected"
    } else {
        dependencies["broker"] = "connected"
    }

    // Overall health
    healthy := true
    for _, status := range dependencies {
        if status == "disconnected" {
            healthy = false
            break
        }
    }

    return HealthStatus{
        Healthy:      healthy,
        IsLeader:     c.IsLeader(),
        Dependencies: dependencies,
        Uptime:       int64(time.Since(c.startTime).Seconds()),
    }
}
```

---

## Complete Implementation Example

### Main Entry Point (cmd/coordinator/main.go)

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
    "github.com/yourusername/niyanta/internal/config"
    "github.com/yourusername/niyanta/internal/coordinator"
    "github.com/yourusername/niyanta/internal/health"
    "github.com/yourusername/niyanta/internal/observability/logger"
    "github.com/yourusername/niyanta/internal/scheduler"
    "github.com/yourusername/niyanta/internal/storage/postgres"
    "go.uber.org/zap"
)

func main() {
    // Load configuration
    cfg, err := config.LoadCoordinatorConfig()
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

    log.Info("starting niyanta coordinator",
        zap.String("version", "1.0.0"),
        zap.Bool("ha_enabled", cfg.Coordinator.EnableHA),
    )

    // Initialize state manager (PostgreSQL)
    stateManager, err := postgres.NewStateManager(cfg.Database, log)
    if err != nil {
        log.Fatal("failed to initialize state manager", zap.Error(err))
    }
    defer stateManager.Close()

    // Initialize broker client (NATS)
    brokerClient, err := nats.NewBrokerClient(cfg.Broker, log)
    if err != nil {
        log.Fatal("failed to initialize broker client", zap.Error(err))
    }
    defer brokerClient.Close()

    // Initialize scheduler
    schedulerImpl := scheduler.NewFIFOScheduler(stateManager, brokerClient, log)

    // Initialize health monitor
    healthMonitor := health.NewMonitor(cfg.Coordinator, stateManager, brokerClient, log)

    // Create coordinator
    coord, err := coordinator.NewCoordinator(
        cfg.Coordinator,
        stateManager,
        brokerClient,
        schedulerImpl,
        healthMonitor,
        log,
    )
    if err != nil {
        log.Fatal("failed to create coordinator", zap.Error(err))
    }

    // Set up context with cancellation
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // Handle signals for graceful shutdown
    sigChan := make(chan os.Signal, 1)
    signal.Notify(sigChan, syscall.SIGINT, syscall.SIGTERM)

    // Start coordinator in goroutine
    errChan := make(chan error, 1)
    go func() {
        if err := coord.Start(ctx); err != nil {
            errChan <- err
        }
    }()

    // Wait for shutdown signal or error
    select {
    case sig := <-sigChan:
        log.Info("received shutdown signal", zap.String("signal", sig.String()))
        cancel()

    case err := <-errChan:
        log.Error("coordinator error", zap.Error(err))
        cancel()
    }

    // Graceful shutdown with timeout
    shutdownCtx, shutdownCancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer shutdownCancel()

    if err := coord.Shutdown(shutdownCtx); err != nil {
        log.Error("shutdown failed", zap.Error(err))
        os.Exit(1)
    }

    log.Info("coordinator stopped successfully")
}
```

---

## Testing

### Unit Test Example

```go
package coordinator

import (
    "context"
    "testing"
    "time"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/mock"
    "github.com/yourusername/niyanta/internal/broker/mock"
    "github.com/yourusername/niyanta/internal/storage/mock"
    "go.uber.org/zap/zaptest"
)

func TestCoordinator_StartAndShutdown(t *testing.T) {
    // Create mocks
    mockStateManager := mock_storage.NewMockStateManager()
    mockBrokerClient := mock_broker.NewMockBrokerClient()
    mockScheduler := mock_scheduler.NewMockScheduler()
    mockHealthMonitor := mock_health.NewMockMonitor()

    logger := zaptest.NewLogger(t)

    // Configure mocks
    mockStateManager.On("Ping", mock.Anything).Return(nil)
    mockBrokerClient.On("Ping", mock.Anything).Return(nil)
    mockBrokerClient.On("Subscribe", mock.Anything, "coordinator.events").
        Return(make(chan broker.Message), nil)

    config := Config{
        ServerPort:        8080,
        EnableHA:          false,
        SchedulerInterval: 1 * time.Second,
        BatchSize:         10,
    }

    // Create coordinator
    coord, err := NewCoordinator(
        config,
        mockStateManager,
        mockBrokerClient,
        mockScheduler,
        mockHealthMonitor,
        logger,
    )
    assert.NoError(t, err)
    assert.NotNil(t, coord)

    // Start coordinator
    ctx, cancel := context.WithCancel(context.Background())

    go func() {
        time.Sleep(100 * time.Millisecond)
        cancel()
    }()

    err = coord.Start(ctx)
    assert.NoError(t, err)

    // Shutdown
    shutdownCtx, shutdownCancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer shutdownCancel()

    err = coord.Shutdown(shutdownCtx)
    assert.NoError(t, err)

    // Verify mocks
    mockStateManager.AssertExpectations(t)
    mockBrokerClient.AssertExpectations(t)
}

func TestCoordinator_Health(t *testing.T) {
    mockStateManager := mock_storage.NewMockStateManager()
    mockBrokerClient := mock_broker.NewMockBrokerClient()

    logger := zaptest.NewLogger(t)

    mockStateManager.On("Ping", mock.Anything).Return(nil)
    mockBrokerClient.On("Ping", mock.Anything).Return(nil)

    config := Config{
        ServerPort: 8080,
        EnableHA:   false,
    }

    coord, err := NewCoordinator(
        config,
        mockStateManager,
        mockBrokerClient,
        nil,
        nil,
        logger,
    )
    assert.NoError(t, err)

    // Check health
    health := coord.Health()
    assert.True(t, health.Healthy)
    assert.Equal(t, "connected", health.Dependencies["database"])
    assert.Equal(t, "connected", health.Dependencies["broker"])
}
```

### Integration Test Example

```go
//go:build integration

package coordinator_test

import (
    "context"
    "testing"
    "time"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
    "github.com/yourusername/niyanta/internal/coordinator"
    "github.com/yourusername/niyanta/tests/testutil"
)

func TestCoordinator_Integration(t *testing.T) {
    // Set up test environment (Postgres, NATS)
    env := testutil.SetupTestEnvironment(t)
    defer env.Teardown()

    // Create coordinator with real dependencies
    coord, err := coordinator.NewCoordinator(
        env.CoordinatorConfig,
        env.StateManager,
        env.BrokerClient,
        env.Scheduler,
        env.HealthMonitor,
        env.Logger,
    )
    require.NoError(t, err)

    // Start coordinator
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    go coord.Start(ctx)

    // Wait for coordinator to be ready
    time.Sleep(500 * time.Millisecond)

    // Verify health
    health := coord.Health()
    assert.True(t, health.Healthy)

    // Test API endpoint
    // (Add API tests here)

    // Shutdown
    shutdownCtx, shutdownCancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer shutdownCancel()

    err = coord.Shutdown(shutdownCtx)
    assert.NoError(t, err)
}
```

---

## Phase Implementation

### Phase 1: MVP (Basic Coordinator)
- Single instance coordinator (no HA)
- REST API with basic authentication
- Integration with state manager, broker, scheduler
- Basic health checks

### Phase 2: Production Hardening
- Leader election with HA support
- Graceful shutdown with drain period
- Comprehensive error handling
- Enhanced observability

### Phase 3: Advanced Features
- gRPC API support
- WebSocket support for real-time updates
- Advanced authentication (JWT, mTLS)
- Performance optimizations

---

## Next Steps

1. Implement API handlers (see [10_API_HANDLERS.md](10_API_HANDLERS.md))
2. Integrate with scheduler (see [06_SCHEDULER.md](06_SCHEDULER.md))
3. Integrate with health monitor (see [07_HEALTH_MONITOR.md](07_HEALTH_MONITOR.md))
4. Add observability (see [11_OBSERVABILITY.md](11_OBSERVABILITY.md))
5. Write comprehensive tests (see [13_TESTING_STRATEGY.md](13_TESTING_STRATEGY.md))

---

**Next**: [03_WORKER_IMPLEMENTATION.md](03_WORKER_IMPLEMENTATION.md) - Worker component design
