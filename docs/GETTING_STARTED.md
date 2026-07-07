# Getting Started

**Status**: Draft — the repository is currently **docs-first**. The design is specified; the Go implementation lands phase by phase per [IMPLEMENTATION_PLAN.md](IMPLEMENTATION_PLAN.md). This guide covers what you can do today (read and validate the docs/specs) and the intended developer flow as code arrives.

## What Niyanta is (in one paragraph)

Niyanta is a durable activity execution engine. It runs activities, guarantees their execution (retries, replay-based resume, leases, fencing, durable timers, signals), and resumes them if they get stuck — while staying agnostic to what any activity does. Start with [README.md](README.md) for the map, then [platform/ARCHITECTURE.md](platform/ARCHITECTURE.md) for the engine and [platform/adr/ADR-005-activity-execution-model.md](platform/adr/ADR-005-activity-execution-model.md) for the activity model.

## Prerequisites

For working on the docs/specs today:

- `git`
- `ruby` (ships with macOS; used to validate YAML) — or `python3` **with PyYAML installed** (`pip install pyyaml`; the stdlib alone cannot parse YAML)
- Optionally an OpenAPI linter (e.g. `redocly lint` / `spectral lint`)

For building the engine once code exists (target stack, per CLAUDE.md and the impl phases):

- Go (toolchain version per `go.mod` when it lands)
- PostgreSQL 15+ (state store — the only required external dependency through Phase 6)
- `golang-migrate` (migrations), `golangci-lint` (lint)
- Prometheus + Grafana (Phase 6 observability stack; shipped in the dev docker-compose)

## Today: read and validate the specs

```bash
# Validate the OpenAPI spec parses (ruby ships with macOS)
ruby -ryaml -e "YAML.load_file('api/openapi/niyanta-v1.yaml'); puts 'ok'"

# alternative, if PyYAML is installed:
# python3 -c "import yaml; yaml.safe_load(open('api/openapi/niyanta-v1.yaml'))" && echo ok

# (optional) lint OpenAPI
# npx @redocly/cli lint api/openapi/niyanta-v1.yaml
```

There is also a docs link-check you can run (see [CONTRIBUTING_DOCS.md](CONTRIBUTING_DOCS.md)).

## Intended developer flow (as code lands)

The commands below are the **target** flow from CLAUDE.md; they become live as the corresponding phase is implemented.

```bash
# Build / test / lint
go build ./...
go test ./...
go vet ./...
golangci-lint run

# Run the Phase 0 single-process activity runner (target: Phase 0)
go run ./cmd/niyanta

# Bring up dependencies for Phase 1+ (Postgres) — compose file added with Phase 1
# docker compose up -d postgres
# migrate -path ./migrations -database "$POSTGRES_URL" up
```

### Phase 0 demo (target)

Per [platform/impl/PHASE_0_SKELETON.md](platform/impl/PHASE_0_SKELETON.md), Phase 0 is a single-process, in-memory activity runner: register an activity type, submit an activity, watch it run to completion. No Postgres yet. This is the smallest end-to-end loop and the first thing to stand up.

## Build order

Read the phases in order — each ends with a runnable system:

1. [platform/impl/README.md](platform/impl/README.md) — phases 0–7 (engine, production surface, console, hardening)

The roadmap with effort estimates and gap-resolution mapping is [IMPLEMENTATION_PLAN.md](IMPLEMENTATION_PLAN.md).

## Where things live

```
docs/        design docs — start at README.md
api/openapi/ machine-readable engine API (niyanta-v1.yaml)
```

(Go source under `cmd/`, `internal/`, `pkg/` arrives with the phases; see [platform/impl/README.md](platform/impl/README.md) §Project Layout.)
