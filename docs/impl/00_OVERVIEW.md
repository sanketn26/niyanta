# Niyanta Implementation Specification - Overview

**Version**: 1.0
**Last Updated**: 2025-10-29
**Status**: Implementation Ready

## Purpose

This directory contains detailed implementation specifications for Niyanta, breaking down the high-level design into concrete Go packages, structs, interfaces, and implementation details.

## Document Structure

This implementation guide is organized into the following documents:

1. **[00_OVERVIEW.md](00_OVERVIEW.md)** (this file) - Overview and navigation
2. **[01_PROJECT_STRUCTURE.md](01_PROJECT_STRUCTURE.md)** - Complete directory layout and package organization
3. **[02_COORDINATOR_IMPLEMENTATION.md](02_COORDINATOR_IMPLEMENTATION.md)** - Coordinator component design
4. **[03_WORKER_IMPLEMENTATION.md](03_WORKER_IMPLEMENTATION.md)** - Worker component design
5. **[04_STATE_MANAGER.md](04_STATE_MANAGER.md)** - State storage layer implementation
6. **[05_BROKER_CLIENT.md](05_BROKER_CLIENT.md)** - Broker communication layer
7. **[06_SCHEDULER.md](06_SCHEDULER.md)** - Scheduling algorithms and priority queues
8. **[07_HEALTH_MONITOR.md](07_HEALTH_MONITOR.md)** - Worker health monitoring and failure detection
9. **[08_CHECKPOINT_MANAGER.md](08_CHECKPOINT_MANAGER.md)** - Checkpoint creation and recovery
10. **[09_WORKLOAD_INTERFACE.md](09_WORKLOAD_INTERFACE.md)** - Workload plugin system
11. **[10_API_HANDLERS.md](10_API_HANDLERS.md)** - REST API implementation
12. **[11_OBSERVABILITY.md](11_OBSERVABILITY.md)** - Logging, metrics, and tracing
13. **[12_CONFIGURATION.md](12_CONFIGURATION.md)** - Configuration management
14. **[13_TESTING_STRATEGY.md](13_TESTING_STRATEGY.md)** - Unit, integration, and E2E tests

## How to Use This Guide

### For Implementation

When implementing a specific component:

1. Start with the relevant component specification (e.g., [02_COORDINATOR_IMPLEMENTATION.md](02_COORDINATOR_IMPLEMENTATION.md))
2. Review the interface definitions and struct layouts
3. Check dependencies on other components
4. Refer to [01_PROJECT_STRUCTURE.md](01_PROJECT_STRUCTURE.md) for package placement
5. Follow the implementation sequence suggested in each document

### For Code Review

When reviewing code:

1. Verify that package structure matches [01_PROJECT_STRUCTURE.md](01_PROJECT_STRUCTURE.md)
2. Check that interfaces are implemented as specified
3. Ensure error handling follows the patterns in each component spec
4. Verify observability hooks are in place per [11_OBSERVABILITY.md](11_OBSERVABILITY.md)

### For Architecture Questions

When clarifying design decisions:

1. Review the high-level docs first:
   - [../ARCHITECTURE.md](../ARCHITECTURE.md) - System architecture
   - [../DATA_MODELS.md](../DATA_MODELS.md) - Data structures
   - [../API_SPEC.md](../API_SPEC.md) - API contracts
2. Then dive into specific implementation details in this directory

## Key Principles

### 1. Interface-Driven Design

All major components are defined by interfaces first, enabling:
- Easy mocking for unit tests
- Flexibility to swap implementations
- Clear contracts between components

### 2. Dependency Injection

Components receive their dependencies through constructors:
```go
func NewCoordinator(
    cfg Config,
    stateManager StateManager,
    brokerClient BrokerClient,
    logger *zap.Logger,
) *Coordinator
```

### 3. Context-First APIs

All operations that perform I/O accept `context.Context` as first parameter:
```go
func (s *Scheduler) ScheduleNextBatch(ctx context.Context) error
```

### 4. Error Wrapping

Use `fmt.Errorf` with `%w` to wrap errors for better debugging:
```go
if err != nil {
    return fmt.Errorf("failed to schedule workload %s: %w", workloadID, err)
}
```

### 5. Structured Logging

Use structured logging with consistent fields:
```go
logger.Info("workload scheduled",
    zap.String("workload_id", workload.ID),
    zap.String("customer_id", workload.CustomerID),
    zap.String("worker_id", worker.ID),
)
```

## Implementation Phases

The documents in this directory support the phased implementation approach:

### Phase 1: MVP (Weeks 1-10)
Primary documents:
- [02_COORDINATOR_IMPLEMENTATION.md](02_COORDINATOR_IMPLEMENTATION.md) - Basic coordinator
- [03_WORKER_IMPLEMENTATION.md](03_WORKER_IMPLEMENTATION.md) - Basic worker
- [04_STATE_MANAGER.md](04_STATE_MANAGER.md) - PostgreSQL storage
- [05_BROKER_CLIENT.md](05_BROKER_CLIENT.md) - NATS integration
- [06_SCHEDULER.md](06_SCHEDULER.md) - FIFO scheduler
- [09_WORKLOAD_INTERFACE.md](09_WORKLOAD_INTERFACE.md) - Plugin system

### Phase 2: Production Hardening (Weeks 11-20)
Additional focus areas:
- [07_HEALTH_MONITOR.md](07_HEALTH_MONITOR.md) - Robust failure detection
- [11_OBSERVABILITY.md](11_OBSERVABILITY.md) - Full observability stack
- Leader election in [02_COORDINATOR_IMPLEMENTATION.md](02_COORDINATOR_IMPLEMENTATION.md)
- Graceful shutdown patterns across all components

### Phase 3: Advanced Features (Weeks 21-30)
Enhancements:
- Affinity scheduling in [06_SCHEDULER.md](06_SCHEDULER.md)
- Automatic checkpointing in [08_CHECKPOINT_MANAGER.md](08_CHECKPOINT_MANAGER.md)
- WebSocket support in [10_API_HANDLERS.md](10_API_HANDLERS.md)

## Cross-Cutting Concerns

### Concurrency Patterns

All components use Go's concurrency primitives safely:

1. **Worker Pools**: Bounded goroutine pools with semaphores
2. **Context Cancellation**: Proper propagation for graceful shutdown
3. **Mutex Protection**: Fine-grained locking for shared state
4. **Channel Communication**: Typed channels with select for timeouts

See individual component docs for specific patterns.

### Error Handling

Standard error handling approach:

1. **Transient Errors**: Retry with exponential backoff
2. **Permanent Errors**: Fail fast and log
3. **Context Errors**: Check `ctx.Err()` first
4. **Error Classification**: Tag errors with type for retry logic

### Resource Management

1. **Database Connections**: Use connection pooling (pgxpool)
2. **Broker Connections**: Reconnect on failure
3. **File Handles**: Always defer close
4. **Goroutines**: Use sync.WaitGroup for cleanup

## Development Workflow

### Starting a New Component

1. Create package directory under `internal/` or `pkg/`
2. Define interfaces in separate `*_interface.go` files
3. Implement in `*.go` files
4. Add tests in `*_test.go` files
5. Document in corresponding impl doc

### Adding a New Feature

1. Update relevant impl spec document first
2. Update interfaces if needed
3. Implement feature with tests
4. Update [../IMPLEMENTATION_PLAN.md](../IMPLEMENTATION_PLAN.md) if major

### Refactoring

1. Ensure interfaces remain stable (backwards compatible)
2. Update impl spec to reflect changes
3. Verify all tests still pass
4. Update integration tests if needed

## Code Organization Conventions

### Package Naming
- `internal/coordinator` - Coordinator-specific logic
- `internal/worker` - Worker-specific logic
- `internal/storage` - State management
- `internal/broker` - Broker communication
- `pkg/api` - Public API types
- `pkg/models` - Shared domain models

### File Naming
- `manager.go` - Main component implementation
- `*_interface.go` - Interface definitions
- `*_test.go` - Unit tests
- `*_integration_test.go` - Integration tests
- `*_mock.go` - Mock implementations

### Interface Naming
- Use `er` suffix: `Scheduler`, `StateManager`, `BrokerClient`
- Avoid `I` prefix: Use `Scheduler` not `IScheduler`

## Testing Guidelines

Refer to [13_TESTING_STRATEGY.md](13_TESTING_STRATEGY.md) for comprehensive testing approach.

**Quick Reference**:
- Unit tests: Test each package in isolation with mocks
- Integration tests: Test component interactions with real dependencies
- E2E tests: Test full system with docker-compose

## Documentation Standards

### Code Comments

```go
// Package coordinator implements the control plane for Niyanta.
// It handles workload scheduling, worker health monitoring, and
// lifecycle management.
package coordinator

// Scheduler assigns pending workloads to available workers based on
// capacity, affinity rules, and priority.
type Scheduler interface {
    // ScheduleNextBatch finds pending workloads and assigns them to workers.
    // Returns the number of workloads scheduled or an error.
    ScheduleNextBatch(ctx context.Context) (int, error)
}
```

### README Files

Each major package should have a README.md explaining:
- Purpose and responsibilities
- Key types and interfaces
- Usage examples
- Testing approach

## Getting Started

To begin implementing Niyanta:

1. **Read Foundation Docs**:
   - [../CLAUDE.md](../CLAUDE.md) - Project overview
   - [../ARCHITECTURE.md](../ARCHITECTURE.md) - System design
   - This overview document

2. **Set Up Project Structure**:
   - Follow [01_PROJECT_STRUCTURE.md](01_PROJECT_STRUCTURE.md)
   - Create directory layout
   - Initialize Go module

3. **Start with Core Interfaces**:
   - Read [04_STATE_MANAGER.md](04_STATE_MANAGER.md)
   - Implement state storage layer first (foundation)
   - Then broker client in [05_BROKER_CLIENT.md](05_BROKER_CLIENT.md)

4. **Build Coordinator**:
   - Follow [02_COORDINATOR_IMPLEMENTATION.md](02_COORDINATOR_IMPLEMENTATION.md)
   - Implement API handlers per [10_API_HANDLERS.md](10_API_HANDLERS.md)
   - Add scheduler per [06_SCHEDULER.md](06_SCHEDULER.md)

5. **Build Worker**:
   - Follow [03_WORKER_IMPLEMENTATION.md](03_WORKER_IMPLEMENTATION.md)
   - Implement workload interface per [09_WORKLOAD_INTERFACE.md](09_WORKLOAD_INTERFACE.md)
   - Add checkpoint manager per [08_CHECKPOINT_MANAGER.md](08_CHECKPOINT_MANAGER.md)

6. **Add Observability**:
   - Integrate logging, metrics, tracing per [11_OBSERVABILITY.md](11_OBSERVABILITY.md)

7. **Write Tests**:
   - Follow [13_TESTING_STRATEGY.md](13_TESTING_STRATEGY.md)

## Reference Links

### External Documentation
- [Go Standard Project Layout](https://github.com/golang-standards/project-layout)
- [Effective Go](https://golang.org/doc/effective_go)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [NATS Documentation](https://docs.nats.io/)
- [Prometheus Best Practices](https://prometheus.io/docs/practices/)

### Internal Documentation
- Main Docs: [../README.md](../README.md)
- Architecture: [../ARCHITECTURE.md](../ARCHITECTURE.md)
- Data Models: [../DATA_MODELS.md](../DATA_MODELS.md)
- API Spec: [../API_SPEC.md](../API_SPEC.md)
- Implementation Plan: [../IMPLEMENTATION_PLAN.md](../IMPLEMENTATION_PLAN.md)

## Contributing

When adding to this implementation guide:

1. Follow the existing document structure
2. Include code examples for clarity
3. Reference other documents when needed
4. Keep examples up-to-date with actual code
5. Add diagrams where helpful (ASCII art or mermaid)

## Version History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-10-29 | Implementation Team | Initial implementation specification |

---

**Next**: Start with [01_PROJECT_STRUCTURE.md](01_PROJECT_STRUCTURE.md) to see the complete project layout.
