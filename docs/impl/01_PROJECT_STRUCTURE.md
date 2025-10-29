# Project Structure

**Component**: Repository Organization
**Last Updated**: 2025-10-29

## Complete Directory Layout

```
niyanta/
├── cmd/
│   ├── coordinator/
│   │   ├── main.go                     # Coordinator entry point
│   │   └── config.yaml                 # Default configuration
│   └── worker/
│       ├── main.go                     # Worker entry point
│       └── config.yaml                 # Default configuration
│
├── internal/                           # Private application code
│   ├── coordinator/
│   │   ├── coordinator.go              # Main coordinator struct
│   │   ├── api_server.go               # HTTP/gRPC server
│   │   ├── leader_election.go          # HA leader election
│   │   └── README.md
│   │
│   ├── worker/
│   │   ├── worker.go                   # Main worker struct
│   │   ├── executor.go                 # Workload execution engine
│   │   ├── heartbeat.go                # Heartbeat sender
│   │   └── README.md
│   │
│   ├── scheduler/
│   │   ├── scheduler.go                # Main scheduler interface & impl
│   │   ├── scheduler_interface.go      # Interface definition
│   │   ├── fifo_scheduler.go           # FIFO implementation
│   │   ├── priority_scheduler.go       # Priority-based (Phase 3)
│   │   ├── affinity.go                 # Affinity evaluation logic
│   │   └── README.md
│   │
│   ├── health/
│   │   ├── monitor.go                  # Worker health monitor
│   │   ├── monitor_interface.go        # Interface definition
│   │   ├── detector.go                 # Failure detection
│   │   └── README.md
│   │
│   ├── storage/
│   │   ├── interface.go                # StateManager interface
│   │   ├── postgres/
│   │   │   ├── postgres.go             # PostgreSQL implementation
│   │   │   ├── workloads.go            # Workload operations
│   │   │   ├── workers.go              # Worker operations
│   │   │   ├── checkpoints.go          # Checkpoint operations
│   │   │   ├── audit.go                # Audit event operations
│   │   │   ├── migrations/             # SQL migrations
│   │   │   │   ├── 001_initial.up.sql
│   │   │   │   ├── 001_initial.down.sql
│   │   │   │   └── ...
│   │   │   └── README.md
│   │   └── mock/
│   │       └── mock_storage.go         # Mock for testing
│   │
│   ├── broker/
│   │   ├── interface.go                # BrokerClient interface
│   │   ├── nats/
│   │   │   ├── client.go               # NATS implementation
│   │   │   ├── publisher.go            # Publishing logic
│   │   │   ├── subscriber.go           # Subscription logic
│   │   │   └── README.md
│   │   ├── redis/                      # Alternative implementation
│   │   │   └── client.go
│   │   └── mock/
│   │       └── mock_broker.go          # Mock for testing
│   │
│   ├── checkpoint/
│   │   ├── manager.go                  # Checkpoint manager
│   │   ├── manager_interface.go        # Interface definition
│   │   ├── serializer.go               # Checkpoint serialization
│   │   ├── compressor.go               # Compression (Phase 4)
│   │   └── README.md
│   │
│   ├── workload/
│   │   ├── interface.go                # Workload plugin interface
│   │   ├── registry.go                 # Plugin registry
│   │   ├── context.go                  # Workload execution context
│   │   ├── plugins/
│   │   │   ├── sleep/                  # Sample workload
│   │   │   │   ├── sleep.go
│   │   │   │   └── sleep_test.go
│   │   │   └── README.md               # How to add plugins
│   │   └── README.md
│   │
│   ├── auth/
│   │   ├── middleware.go               # Authentication middleware
│   │   ├── api_key.go                  # API key validation
│   │   ├── jwt.go                      # JWT validation (Phase 2)
│   │   └── README.md
│   │
│   ├── ratelimit/
│   │   ├── limiter.go                  # Rate limiter
│   │   ├── limiter_interface.go        # Interface definition
│   │   ├── token_bucket.go             # Token bucket algorithm
│   │   └── README.md
│   │
│   ├── observability/
│   │   ├── logger/
│   │   │   ├── logger.go               # Structured logger setup
│   │   │   └── fields.go               # Common log fields
│   │   ├── metrics/
│   │   │   ├── metrics.go              # Prometheus metrics
│   │   │   ├── coordinator.go          # Coordinator-specific metrics
│   │   │   └── worker.go               # Worker-specific metrics
│   │   ├── tracing/
│   │   │   ├── tracer.go               # Distributed tracing setup
│   │   │   └── spans.go                # Common span helpers
│   │   └── README.md
│   │
│   ├── config/
│   │   ├── config.go                   # Configuration structs
│   │   ├── loader.go                   # Config loading (viper)
│   │   ├── validator.go                # Config validation
│   │   └── README.md
│   │
│   └── util/
│       ├── id.go                       # ID generation utilities
│       ├── retry.go                    # Retry logic with backoff
│       ├── errors.go                   # Error classification
│       └── time.go                     # Time utilities
│
├── pkg/                                # Public libraries (can be imported)
│   ├── api/
│   │   ├── v1/
│   │   │   ├── types.go                # API request/response types
│   │   │   ├── errors.go               # API error types
│   │   │   └── validation.go           # Request validation
│   │   └── README.md
│   │
│   ├── models/
│   │   ├── workload.go                 # Workload domain model
│   │   ├── worker.go                   # Worker domain model
│   │   ├── customer.go                 # Customer domain model
│   │   ├── checkpoint.go               # Checkpoint domain model
│   │   ├── audit.go                    # Audit event domain model
│   │   └── README.md
│   │
│   ├── client/                         # Client SDK (Phase 3)
│   │   ├── client.go                   # SDK entry point
│   │   ├── workloads.go                # Workload operations
│   │   └── README.md
│   │
│   └── proto/                          # Protocol buffer definitions (if using gRPC)
│       ├── workload_service.proto
│       └── README.md
│
├── api/                                # API specifications
│   ├── openapi/
│   │   └── niyanta-v1.yaml             # OpenAPI 3.0 spec
│   └── proto/                          # Alternative location for proto files
│       └── niyanta/
│           └── v1/
│               └── workload_service.proto
│
├── deployments/
│   ├── kubernetes/
│   │   ├── coordinator/
│   │   │   ├── deployment.yaml
│   │   │   ├── service.yaml
│   │   │   ├── configmap.yaml
│   │   │   └── hpa.yaml
│   │   ├── worker/
│   │   │   ├── deployment.yaml
│   │   │   ├── configmap.yaml
│   │   │   └── hpa.yaml
│   │   ├── postgres/
│   │   │   ├── statefulset.yaml
│   │   │   ├── service.yaml
│   │   │   └── pvc.yaml
│   │   ├── nats/
│   │   │   ├── statefulset.yaml
│   │   │   └── service.yaml
│   │   ├── monitoring/
│   │   │   ├── prometheus/
│   │   │   ├── grafana/
│   │   │   └── dashboards/
│   │   ├── kustomization.yaml
│   │   └── README.md
│   │
│   ├── terraform/                      # Infrastructure as code
│   │   ├── aws/
│   │   │   ├── main.tf
│   │   │   ├── rds.tf
│   │   │   ├── eks.tf
│   │   │   └── variables.tf
│   │   ├── gcp/
│   │   │   └── ...
│   │   └── README.md
│   │
│   └── docker-compose/                 # Local development
│       ├── docker-compose.yml          # All services
│       ├── docker-compose.dev.yml      # Development overrides
│       └── README.md
│
├── scripts/
│   ├── build.sh                        # Build binaries
│   ├── test.sh                         # Run all tests
│   ├── lint.sh                         # Run linters
│   ├── migrate.sh                      # Run database migrations
│   ├── seed-data.sh                    # Seed development data
│   ├── generate-mocks.sh               # Generate mock implementations
│   └── README.md
│
├── tests/
│   ├── integration/
│   │   ├── coordinator_test.go         # Coordinator integration tests
│   │   ├── worker_test.go              # Worker integration tests
│   │   ├── e2e_test.go                 # End-to-end tests
│   │   └── testdata/                   # Test fixtures
│   ├── fixtures/                       # Shared test fixtures
│   └── README.md
│
├── docs/
│   ├── ARCHITECTURE.md                 # System architecture
│   ├── API_SPEC.md                     # API specification
│   ├── DATA_MODELS.md                  # Data models
│   ├── IMPLEMENTATION_PLAN.md          # Implementation roadmap
│   └── impl/                           # Detailed implementation specs (this directory)
│       ├── 00_OVERVIEW.md
│       ├── 01_PROJECT_STRUCTURE.md     # This file
│       ├── 02_COORDINATOR_IMPLEMENTATION.md
│       └── ...
│
├── .github/
│   ├── workflows/
│   │   ├── ci.yml                      # CI pipeline
│   │   ├── cd.yml                      # CD pipeline
│   │   └── release.yml                 # Release pipeline
│   └── CODEOWNERS                      # Code ownership
│
├── .gitignore
├── .golangci.yml                       # Linter configuration
├── Makefile                            # Common commands
├── go.mod                              # Go module definition
├── go.sum                              # Go dependencies
├── Dockerfile.coordinator              # Coordinator container
├── Dockerfile.worker                   # Worker container
├── CLAUDE.md                           # Development guidelines
├── README.md                           # Project overview
├── LICENSE                             # License file
└── CHANGELOG.md                        # Version history
```

## Package Descriptions

### `cmd/`
**Purpose**: Application entry points (main packages)

Each subdirectory under `cmd/` produces a separate binary. These should be minimal - just configuration loading, dependency injection, and calling into `internal/` packages.

**Guidelines**:
- Keep main.go files under 200 lines
- All business logic belongs in `internal/`
- Handle signal catching (SIGTERM, SIGINT) for graceful shutdown
- Set up logging and metrics before starting components

### `internal/`
**Purpose**: Private application code (cannot be imported by external projects)

Contains all the core business logic. Organized by component/domain.

**Key Principles**:
- Each subdirectory is a cohesive package
- Define interfaces in `*_interface.go` files
- Use dependency injection (pass interfaces to constructors)
- No package should import `cmd/`

### `pkg/`
**Purpose**: Public libraries that can be imported by other projects

These are stable, well-documented packages that external consumers can use.

**Key Packages**:
- `pkg/api`: API types (shared between server and clients)
- `pkg/models`: Domain models (shared between coordinator and worker)
- `pkg/client`: Go SDK for Niyanta (Phase 3)

**Guidelines**:
- API stability matters (use semantic versioning)
- Comprehensive documentation
- Examples in `*_example_test.go` files

### `api/`
**Purpose**: API specifications and contracts

OpenAPI specs, Protocol Buffer definitions, and API documentation.

### `deployments/`
**Purpose**: Deployment configurations for various platforms

Kubernetes manifests, Terraform modules, Docker Compose files.

**Organization**:
- One subdirectory per deployment target
- Include README.md with deployment instructions
- Use Kustomize for Kubernetes overlays (dev, staging, prod)

### `scripts/`
**Purpose**: Build, test, and automation scripts

**Common Scripts**:
- `build.sh`: Build all binaries
- `test.sh`: Run unit, integration, and E2E tests
- `lint.sh`: Run golangci-lint
- `migrate.sh`: Run database migrations

### `tests/`
**Purpose**: Integration and E2E tests

**Organization**:
- `integration/`: Tests that use real dependencies (database, broker)
- `fixtures/`: Shared test data
- Use build tag `//go:build integration` for integration tests

### `docs/`
**Purpose**: Project documentation

High-level architecture, API specs, and implementation guides.

---

## Key Files

### `Makefile`

```makefile
.PHONY: help build test lint clean migrate docker-build

help: ## Show this help
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-20s\033[0m %s\n", $$1, $$2}'

build: ## Build all binaries
	@./scripts/build.sh

test: ## Run all tests
	@./scripts/test.sh

test-unit: ## Run unit tests only
	go test -v -race -short ./...

test-integration: ## Run integration tests
	go test -v -race -tags=integration ./tests/integration/...

lint: ## Run linters
	@./scripts/lint.sh

clean: ## Clean build artifacts
	rm -rf bin/

migrate: ## Run database migrations
	@./scripts/migrate.sh up

migrate-down: ## Rollback database migrations
	@./scripts/migrate.sh down

docker-build: ## Build Docker images
	docker build -f Dockerfile.coordinator -t niyanta-coordinator:latest .
	docker build -f Dockerfile.worker -t niyanta-worker:latest .

dev: ## Run local development environment
	docker-compose -f deployments/docker-compose/docker-compose.yml up

.DEFAULT_GOAL := help
```

### `go.mod`

```go
module github.com/yourusername/niyanta

go 1.21

require (
    github.com/gin-gonic/gin v1.9.1                // HTTP framework
    github.com/jackc/pgx/v5 v5.4.3                 // PostgreSQL driver
    github.com/nats-io/nats.go v1.31.0             // NATS client
    github.com/spf13/viper v1.17.0                 // Configuration
    go.uber.org/zap v1.26.0                        // Structured logging
    github.com/prometheus/client_golang v1.17.0    // Metrics
    go.opentelemetry.io/otel v1.19.0               // Tracing
    github.com/stretchr/testify v1.8.4             // Testing
    github.com/golang-migrate/migrate/v4 v4.16.2   // Migrations
)
```

### `.golangci.yml`

```yaml
linters:
  enable:
    - errcheck
    - goimports
    - golint
    - govet
    - staticcheck
    - unused
    - ineffassign
    - misspell
    - gosec

linters-settings:
  golint:
    min-confidence: 0.8

  goimports:
    local-prefixes: github.com/yourusername/niyanta

issues:
  exclude-rules:
    - path: _test\.go
      linters:
        - errcheck

run:
  timeout: 5m
  tests: true
```

---

## Import Path Conventions

### Internal Imports

```go
import (
    "context"
    "fmt"

    "github.com/yourusername/niyanta/internal/storage"
    "github.com/yourusername/niyanta/internal/broker"
    "github.com/yourusername/niyanta/pkg/models"

    "github.com/jackc/pgx/v5/pgxpool"
    "go.uber.org/zap"
)
```

**Guidelines**:
- Standard library first
- Project imports second (grouped)
- Third-party imports third

### Avoid Circular Dependencies

**Bad**:
```
internal/coordinator -> internal/worker
internal/worker -> internal/coordinator
```

**Good**:
```
internal/coordinator -> pkg/models
internal/worker -> pkg/models
```

Use `pkg/models` for shared types to break cycles.

---

## Build and Deployment

### Building Binaries

```bash
# Build coordinator
go build -o bin/coordinator ./cmd/coordinator

# Build worker
go build -o bin/worker ./cmd/worker

# Cross-compile for Linux
GOOS=linux GOARCH=amd64 go build -o bin/coordinator-linux ./cmd/coordinator
```

### Docker Images

**Dockerfile.coordinator**:
```dockerfile
FROM golang:1.21-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /coordinator ./cmd/coordinator

FROM alpine:latest
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=builder /coordinator .
EXPOSE 8080
CMD ["./coordinator"]
```

**Dockerfile.worker**:
```dockerfile
FROM golang:1.21-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /worker ./cmd/worker

FROM alpine:latest
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=builder /worker .
EXPOSE 8081
CMD ["./worker"]
```

---

## Testing Structure

### Unit Tests

Located alongside source files:
```
internal/scheduler/
├── scheduler.go
├── scheduler_test.go          # Unit tests
└── scheduler_mock.go           # Generated mocks
```

Run with:
```bash
go test ./internal/scheduler/
```

### Integration Tests

Located in `tests/integration/`:
```
tests/integration/
├── coordinator_test.go
├── worker_test.go
└── testdata/
    └── fixtures.sql
```

Run with:
```bash
go test -tags=integration ./tests/integration/
```

### E2E Tests

Located in `tests/integration/e2e_test.go`:
```go
//go:build integration

func TestWorkloadLifecycle(t *testing.T) {
    // Full system test with docker-compose
}
```

---

## Configuration Management

### Configuration Files

**cmd/coordinator/config.yaml**:
```yaml
server:
  port: 8080

database:
  url: "postgres://user:pass@localhost:5432/niyanta"
  max_connections: 20

broker:
  url: "nats://localhost:4222"

scheduler:
  interval_seconds: 5
  batch_size: 100

logging:
  level: "info"
  format: "json"
```

### Environment Variable Overrides

Use `NIYANTA_` prefix:
```bash
export NIYANTA_SERVER_PORT=9090
export NIYANTA_DATABASE_URL="postgres://..."
```

Viper automatically maps these to config struct fields.

---

## Development Workflow

### 1. Set Up Local Environment

```bash
# Clone repository
git clone https://github.com/yourusername/niyanta.git
cd niyanta

# Install dependencies
go mod download

# Start dependencies (Postgres, NATS) with Docker Compose
cd deployments/docker-compose
docker-compose up -d postgres nats

# Run migrations
cd ../..
./scripts/migrate.sh up

# Seed test data
./scripts/seed-data.sh
```

### 2. Run Coordinator

```bash
go run ./cmd/coordinator
```

### 3. Run Worker

```bash
go run ./cmd/worker
```

### 4. Run Tests

```bash
# Unit tests
make test-unit

# Integration tests
make test-integration

# All tests
make test
```

### 5. Build and Deploy

```bash
# Build binaries
make build

# Build Docker images
make docker-build

# Deploy to Kubernetes
kubectl apply -k deployments/kubernetes/
```

---

## Code Generation

### Mock Generation

Use `mockgen` to generate mocks for interfaces:

```bash
# Install mockgen
go install github.com/golang/mock/mockgen@latest

# Generate mocks
mockgen -source=internal/storage/interface.go -destination=internal/storage/mock/mock_storage.go -package=mock
```

**Add to** `scripts/generate-mocks.sh`:
```bash
#!/bin/bash
set -e

echo "Generating mocks..."

mockgen -source=internal/storage/interface.go -destination=internal/storage/mock/mock_storage.go -package=mock
mockgen -source=internal/broker/interface.go -destination=internal/broker/mock/mock_broker.go -package=mock
# ... more interfaces

echo "Done!"
```

### Protocol Buffer Generation (if using gRPC)

```bash
# Install protoc and plugins
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest

# Generate Go code from .proto files
protoc --go_out=. --go-grpc_out=. api/proto/niyanta/v1/*.proto
```

---

## Migration Management

### Creating Migrations

```bash
# Install migrate CLI
go install -tags 'postgres' github.com/golang-migrate/migrate/v4/cmd/migrate@latest

# Create new migration
migrate create -ext sql -dir internal/storage/postgres/migrations -seq add_worker_tags
```

This creates:
- `002_add_worker_tags.up.sql`
- `002_add_worker_tags.down.sql`

### Running Migrations

```bash
# Up
migrate -path internal/storage/postgres/migrations -database "postgres://localhost:5432/niyanta?sslmode=disable" up

# Down (rollback one)
migrate -path internal/storage/postgres/migrations -database "postgres://localhost:5432/niyanta?sslmode=disable" down 1
```

---

## Next Steps

Now that you understand the project structure:

1. **Read Component Specs**: Start with [02_COORDINATOR_IMPLEMENTATION.md](02_COORDINATOR_IMPLEMENTATION.md)
2. **Set Up Repository**: Create directories and initialize Go module
3. **Implement Core Interfaces**: Begin with [04_STATE_MANAGER.md](04_STATE_MANAGER.md)
4. **Build Components**: Follow the implementation specs in order

---

**Next**: [02_COORDINATOR_IMPLEMENTATION.md](02_COORDINATOR_IMPLEMENTATION.md) - Coordinator component design
