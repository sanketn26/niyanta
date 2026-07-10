# Niyanta Documentation

Niyanta is a **durable activity execution engine**: it runs activities, guarantees their execution (durability, replay-based resume, framework retries, leases, fencing, durable timers, signals), and holds enough state to resume them after any failure. The engine is agnostic to what an activity does — its inputs, results, and any configuration that drives it are opaque payloads.

Anything built *on* the engine — a composition of activity types wired together for some product — is a consumer of the engine's API and SDK. Compositions are documented and planned separately; nothing in these docs depends on any particular one.

New here? Start with [GETTING_STARTED.md](GETTING_STARTED.md). Editing docs? Read [CONTRIBUTING_DOCS.md](CONTRIBUTING_DOCS.md) first — it defines the engine boundary and the checks that keep the docs from drifting.

```
docs/
  GETTING_STARTED.md            read/validate the specs; intended dev flow
  CONTRIBUTING_DOCS.md          where things go; vocabulary; pre-commit checks
  IMPLEMENTATION_PLAN.md        phased delivery roadmap (engine-only, phases 0–9)
  platform/                     the engine
    ARCHITECTURE.md             components, guarantees, HA, deployment, scaling, security
    DATA_MODELS.md              schemas, message formats, Go structs, invariants
    API_SPEC.md                 REST API contracts
    adr/                        architecture decision records
    impl/                       phase implementation guides 0–9
api/openapi/                    machine-readable engine API (niyanta-v1.yaml)
```

## Platform (the engine)

- [ARCHITECTURE.md](platform/ARCHITECTURE.md) — the runner, guarantees, HA, deployment, scaling, security
- [DATA_MODELS.md](platform/DATA_MODELS.md) — activity/attempt/call-log/signal schemas, message formats, Go structs
- [API_SPEC.md](platform/API_SPEC.md) — REST API contracts
- [adr/ADR-005-activity-execution-model.md](platform/adr/ADR-005-activity-execution-model.md) — the activity execution model (`RunChild`, replay, signals, `ContinueAsNew`)
- [impl/](platform/impl/README.md) — implementation phases 0–9

## Roadmap

- [IMPLEMENTATION_PLAN.md](IMPLEMENTATION_PLAN.md) — phased build plan: skeleton → persistence + fencing → distributed workers → composition/replay → timers/signals/`ContinueAsNew` → safe operator API → read-only console → operator workflows → observability → production hardening

## The contract in one paragraph

An **activity** is a unit of work. Activities compose: a parent calls `ctx.RunChild(...)` and the engine records the call durably, frees the parent's worker slot, runs the child, and resumes the parent by replaying its body against the call log. `Sleep` and `AwaitSignal` suspend the same way; `ContinueAsNew` bounds replay history for unbounded-lifetime activities. Retries are framework-managed per attempt; every physical attempt is audited; generation fencing guarantees a redistributed activity's stale worker can neither act nor write. Everything an activity means — its input, its result, its config — is opaque to the engine.
