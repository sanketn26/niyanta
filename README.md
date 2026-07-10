# niyanta

A durable activity execution engine. Niyanta runs activities — and their composed child activities — with execution guarantees: replay-based resume, framework-managed retries, exactly-once child dispatch, generation-fenced redistribution, durable timers and signals, parallel execution across a worker fleet, and multi-tenant isolation. The engine is use-case agnostic: activity payloads are opaque, and anything built on top is a consumer of its API.

Start with:

- [Getting started](docs/GETTING_STARTED.md)
- [Documentation index](docs/README.md)
- [Platform architecture](docs/platform/ARCHITECTURE.md) — the engine
- [Implementation plan](docs/IMPLEMENTATION_PLAN.md) — phases 0–9, including a staged Activity Manager console and mandatory production hardening gate
