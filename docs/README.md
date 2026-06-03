# Niyanta Documentation

Niyanta is an **idiomatic, composable activity runner that provides execution guarantees**. The docs separate that engine from the things built on it:

1. **Platform** — the activity runner: runtime, execution guarantees (durability, replay, retries), state for resume, parallel execution across a worker fleet, coordination, and isolation. App-agnostic. The engine's only concern is running an activity and resuming it if it gets stuck.
2. **Apps** — **compositions of activities** ("cookbooks") built on the platform. They are not a special platform layer and there is no app SPI; each is just activity types wired together, leaning on the platform's guarantees, with its own state and (optionally) its own config language.

A composition can itself be a platform: the Data Connector exposes a YAML connector language, and LLMOps exposes a YAML/JSON runbook language. Those languages are **the apps' own config, not Niyanta's** — the engine never parses them. See [platform/ARCHITECTURE.md](platform/ARCHITECTURE.md) §Activity Compositions and §What Is And Is Not Niyanta's Concern.

New here? Start with [GETTING_STARTED.md](GETTING_STARTED.md). Editing docs? Read [CONTRIBUTING_DOCS.md](CONTRIBUTING_DOCS.md) first — it defines the platform/app boundary and the checks that keep the docs from drifting.

```
docs/
  GETTING_STARTED.md            read/validate the specs; intended dev flow
  CONTRIBUTING_DOCS.md          where things go; vocabulary; pre-commit checks
  IMPLEMENTATION_PLAN.md        cross-cutting, phased delivery roadmap
  platform/                     the composable activity runner + guarantees
    ARCHITECTURE.md
    DATA_MODELS.md
    API_SPEC.md
    adr/                        architecture decision records
    impl/                       phase guides 0–4
  apps/
    dataconnector/              ingestion as an activity composition (+ YAML connector language)
      INGESTION_ARCHITECTURE.md
      DATA_CONNECTOR_SPEC.md
      API_SPEC.md
      OPERATIONS_AND_FAILURE_SEMANTICS.md
      schemas/                  enforceable JSON Schemas for connector artifacts
      impl/                     phase guides 5–6
    llmops/                     LLM-driven fleet operations as an activity composition (+ YAML runbooks)
      ARCHITECTURE.md
      LLMOPS_SPEC.md
      OPERATIONS_AND_FAILURE_SEMANTICS.md
      REDACTION_CONTRACT.md
      schemas/                  JSON Schemas (runbook, finding)
      fixtures/redaction/       redaction test fixtures
      impl/                     phase guides 1–3
api/openapi/                    machine-readable API stubs (engine + apps)
```

## Platform (the engine)

- [ARCHITECTURE.md](platform/ARCHITECTURE.md) — the runner, guarantees, the composition model, HA, deployment, scaling, security
- [DATA_MODELS.md](platform/DATA_MODELS.md) — activity/attempt/call-log/signal schemas, broker messages, Go structs
- [API_SPEC.md](platform/API_SPEC.md) — REST/gRPC API contracts
- [adr/ADR-005-activity-execution-model.md](platform/adr/ADR-005-activity-execution-model.md) — the durable activity substrate (RunChild, replay, signals)
- [impl/](platform/impl/README.md) — implementation phases 0–4

## Data Connector (activity composition)

Self-orchestrated, multi-tenant ingestion. Supervisor → plan → partition-run → deliver, expressed as composed activities, with a declarative YAML connector language on top.

- [INGESTION_ARCHITECTURE.md](apps/dataconnector/INGESTION_ARCHITECTURE.md) — control loops, partitioning, multi-tenancy
- [DATA_CONNECTOR_SPEC.md](apps/dataconnector/DATA_CONNECTOR_SPEC.md) — connector model, kinds, declarative spec, Airbyte compatibility
- [API_SPEC.md](apps/dataconnector/API_SPEC.md) — connector/connection/control/ingestion-view API
- [OPERATIONS_AND_FAILURE_SEMANTICS.md](apps/dataconnector/OPERATIONS_AND_FAILURE_SEMANTICS.md) — delivery guarantees, checkpoint order, dead letters, blast radius
- [schemas/](apps/dataconnector/schemas/README.md) — enforceable JSON Schemas
- [impl/](apps/dataconnector/impl/README.md) — implementation phases 5–6

## LLMOps for Data Connectors (activity composition)

Uses LLMs to detect issues, diagnose lag/failures, and operationalize connectors. Sweep → diagnose → (tool lookups) → remediate → verify, expressed as composed activities that read Data Connector telemetry and act through its control API. Behavior is driven by declarative YAML/JSON runbooks.

- [ARCHITECTURE.md](apps/llmops/ARCHITECTURE.md) — the activities, what they read/write, guarantees relied on, guardrails
- [LLMOPS_SPEC.md](apps/llmops/LLMOPS_SPEC.md) — domain model, runbooks, storage, APIs, activity contract
- [OPERATIONS_AND_FAILURE_SEMANTICS.md](apps/llmops/OPERATIONS_AND_FAILURE_SEMANTICS.md) — action safety, confidence, guardrails, verify/rollback, audit
- [REDACTION_CONTRACT.md](apps/llmops/REDACTION_CONTRACT.md) — what may reach a prompt, prompt logging, provider failure matrix, fixtures
- [schemas/](apps/llmops/schemas/README.md) — JSON Schemas (runbook, finding)
- [impl/](apps/llmops/impl/README.md) — implementation phases 1–3

## Roadmap

- [IMPLEMENTATION_PLAN.md](IMPLEMENTATION_PLAN.md) — phased build plan

## How they relate

The platform runs activities and guarantees they resume if stuck. Each app is a composition of those activities:

- **Data Connector**: an ingestion run *is* an activity; page fetch / transform batch / destination flush *are* child activities.
- **LLMOps**: a diagnosis *is* an activity; a tool lookup and a model call *are* child activities; a remediation *is* an activity that calls the Data Connector control API.

The engine is agnostic to all of this — to it, a connector definition and a runbook are opaque payloads inside an activity's state.
