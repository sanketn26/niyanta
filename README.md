# niyanta

An idiomatic, composable activity runner that provides execution guarantees. The engine runs activities — and their composed child activities — durably: replay-based resume, framework-managed retries, parallel execution across a worker fleet, coordination, and multi-tenant isolation. Apps are *compositions of activities* built on it.

Start with:

- [Getting started](docs/GETTING_STARTED.md)
- [Documentation index](docs/README.md)
- [Platform architecture](docs/platform/ARCHITECTURE.md) — the composable activity runner
- [Data Connector app](docs/apps/dataconnector/INGESTION_ARCHITECTURE.md) — ingestion as an activity composition
- [LLMOps app](docs/apps/llmops/ARCHITECTURE.md) — LLM-driven fleet operations as an activity composition
