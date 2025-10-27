# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Niyanta is a distributed job system designed to support customer-specific workloads with tiered service levels and robust error handling. The architecture prioritizes minimal external dependencies while providing:

- Checkpoint-based recovery
- Automatic redistribution of failed jobs
- Comprehensive observability

## System Architecture

Niyanta is a horizontally scalable, long-running workload processor consisting of three core components:

### Core Components

#### 1. Coordinator
The coordinator acts as the control plane for the distributed system:

- **Workload Registration**: Exposes API endpoints for registering new workloads (both at compile time and runtime)
- **Scheduling & Distribution**: Makes decisions about workload placement and termination
- **Affinity Support**: Honors workload affinity preferences when distributing work among workers
- **Auto-scaling**: Monitors per-customer health metrics and scales workloads up or down as needed
- **Lifecycle Management**: Determines when workloads should be started, stopped, or redistributed

#### 2. Worker
Workers are the execution units that process workloads:

- **Control Plane Integration**: Reacts to control messages from the coordinator via the broker
- **Workload Execution**: Capable of running any registered workload type
- **Registration**: Self-registers supported workload types with the coordinator on boot
- **Capability Declaration**: Advertises its capacity and supported workload types

**Key Concepts**:
- **Workload**: A logical unit of work (e.g., specific customer job type)
- **Worker**: A deployment unit that can execute multiple workload types

#### 3. State Management
Persistent state storage is critical for recovery and coordination. Technology options:
- **etcd**: Distributed key-value store for coordination and configuration
- **PostgreSQL**: Relational database for complex state queries

**State Responsibilities**:
- Checkpoint storage per workload per customer
- Auto-scaling configuration per workload per customer
- Audit event log for workload lifecycle
- Current and historical state transitions for workloads

#### 4. Broker (Control Plane Communication)
The broker facilitates bidirectional communication between coordinators and workers.

**Requirements**:
- **Bidirectional**: Both coordinator and worker can initiate communication
- **Health & Capacity Reporting**: Workers report health status and available capacity
- **Workload Updates**: Status updates from workers to coordinator
- **Intermediate Status**: Progress reporting during long-running workloads
- **Snapshot Capability**: Workers can report current workload inventory

**Technology Options**:
- Lightweight queue mechanism (e.g., NATS, RabbitMQ)
- Redis Streams or Pub/Sub
- Custom protocol over gRPC/HTTP

### Deployment Targets

The system must support multiple deployment environments:

1. **EC2 Instances & MicroVMs**: Simpler scaling requirements, direct VM control
2. **Kubernetes**: Full-featured deployment with native K8s primitives (Deployments, StatefulSets, HPA)

## Technology Stack

- **Language**: Go
- **Architecture**: Distributed job processing system

## Common Commands

### Go Module Management
```bash
# Initialize Go module (if not already done)
go mod init github.com/yourusername/niyanta

# Download dependencies
go mod download

# Tidy up dependencies
go mod tidy

# Vendor dependencies (if needed)
go mod vendor
```

### Building
```bash
# Build the project
go build ./...

# Build specific package
go build ./cmd/niyanta

# Build with race detector
go build -race ./...
```

### Testing
```bash
# Run all tests
go test ./...

# Run tests with verbose output
go test -v ./...

# Run tests with coverage
go test -cover ./...

# Run specific test
go test -run TestFunctionName ./path/to/package

# Run tests with race detector
go test -race ./...

# Generate coverage report
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out
```

### Code Quality
```bash
# Format code
go fmt ./...

# Run linter (requires golangci-lint)
golangci-lint run

# Vet code for common issues
go vet ./...
```

## Design Principles

### 1. Minimal External Dependencies
Reduce operational complexity and potential points of failure by minimizing external dependencies. Favor bundled or lightweight solutions over complex external systems where possible.

### 2. Per-Customer Tiered Service Levels
The architecture must support different SLAs for different customers and workload types:
- **Priority Queues**: Higher-tier customers get priority scheduling
- **Resource Allocation**: SLA-based CPU/memory guarantees
- **Isolation**: Customer workloads should not impact each other
- **Monitoring**: Per-customer metrics and alerting thresholds

### 3. Checkpoint-Based Recovery
All workloads must support checkpointing for fault tolerance:

**When Implementing Workloads**:
- Define clear, logical checkpoint boundaries (not arbitrary time intervals)
- Ensure checkpoint state is atomic and consistent
- Store checkpoints in the state management layer
- Handle checkpoint restoration gracefully in workload initialization
- Consider checkpoint frequency vs. recovery time tradeoffs

**Recovery Behavior**:
- On worker failure, coordinator detects and redistributes work
- New worker picks up from last checkpoint
- Minimize duplicate work during recovery

### 4. Automatic Redistribution
The coordinator must automatically redistribute work when workers fail:

**Implementation Considerations**:
- **Lease Management**: Workers hold time-bounded leases on workloads
- **Heartbeats**: Regular health checks via the broker
- **Graceful Shutdown**: Workers notify coordinator before termination
- **Fence Tokens**: Prevent split-brain scenarios during redistribution
- **Idempotency**: Workloads should handle duplicate execution gracefully

### 5. Workload Affinity
Support both hard and soft affinity requirements:
- **Hard Affinity**: Workload must run on specific worker type/instance
- **Soft Affinity**: Preference for certain workers, but can run elsewhere
- **Anti-Affinity**: Avoid co-locating certain workloads

Use cases include: hardware requirements (GPU), data locality, compliance requirements.

### 6. Robust Error Handling
Distinguish error types and handle appropriately:

- **Transient Errors**: Network blips, temporary resource exhaustion → Retry with backoff
- **Permanent Errors**: Invalid workload configuration, missing dependencies → Alert and stop
- **Partial Failures**: Some tasks in batch fail → Continue processing others
- **Cascading Failures**: Circuit breakers to prevent system-wide impact

### 7. Comprehensive Observability
Built-in observability is non-negotiable:

**Logging**:
- Structured logging (JSON) with consistent fields
- Log levels: DEBUG, INFO, WARN, ERROR
- Correlation IDs across distributed traces

**Metrics** (per customer, per workload type):
- Workload throughput (jobs/sec)
- Latency percentiles (p50, p95, p99)
- Error rates and error types
- Queue depth and age
- Worker capacity and utilization

**Tracing**:
- Distributed tracing for workload lifecycle
- Track: submission → scheduling → execution → completion

**Health Endpoints**:
- `/health`: Basic liveness check
- `/ready`: Readiness for traffic
- `/metrics`: Prometheus-compatible metrics
