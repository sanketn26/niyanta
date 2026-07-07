# Phase 5: Production Engine Surface — API Completeness, Multi-Tenancy, Deployment Safety

**Goal**: The engine is safely consumable by untrusted tenants and operable by humans: complete API, auth, isolation, fairness, and the tooling to deploy new activity code without stranding suspended work.

## Files To Add

```text
internal/api/server.go
internal/api/middleware/auth.go
internal/api/middleware/ratelimit.go
internal/api/handlers/activities.go
internal/api/handlers/recovery.go
internal/api/handlers/workers.go
internal/scheduler/priority.go
internal/audit/writer.go
cmd/niyanta/versions.go
```

## API Completeness

Everything in [../../../api/openapi/niyanta-v1.yaml](../../../api/openapi/niyanta-v1.yaml), plus the endpoints that close the contract gaps:

| Endpoint | Closes |
|----------|--------|
| `POST /activities/{id}:pause` / `:resume` | `PAUSED` state finally has producers |
| `GET /activities/{id}?wait=30s` | long-poll on terminal state — the default client interaction stops being a poll loop |
| `POST /activities/{id}:force-fail` | G6 recovery |
| `POST /activities/{id}:force-complete` (operator-supplied result) | G6 recovery |
| `POST /activities/{id}:truncate-replay` (drop call-log suffix from divergent index; requires `confirm` token; writes audit event) | G6 recovery |
| `GET /activities/{id}/attempts`, `/calls`, `/signals` | inspection (console data source; `curl`-debuggable) |
| `GET /activities/{id}/executions` | `ContinueAsNew` chain |
| `POST /activities/{id}:signal` with `Idempotency-Key` | signal dedupe (Phase 4 schema) |

Update the OpenAPI file in the same change — it is the source of truth.

## Auth And Tenancy

- API keys, hashed at rest (`api_keys` table: hash, customer_id, role, created/revoked).
- Customer extraction middleware; **every** storage query tenant-filtered at the boundary.
- Cross-tenant access returns `NOT_FOUND`, never `FORBIDDEN` — no existence leaks. Contract-tested.

## Fairness And Protection

- Priority scheduling with tier weights and aging (+priority per minute waited; nothing starves > 30 minutes).
- Token-bucket rate limit per tenant; `429` + `Retry-After`.
- Queue-depth backpressure on submit; `503` + `Retry-After`.
- Per-tenant concurrency caps enforced at admission (the Phase 2 scheduler's admission step grows teeth).

## Deployment Safety (G1 tooling)

```bash
niyanta versions list
# TYPE            VERSION   PINNED  RUNNING  SUSPENDED  PARKED
# sample_pipeline 1.0.0     41      3        38         0

niyanta versions drain sample_pipeline@1.0.0
# stops new pins to 1.0.0; reports until zero remain
```

- `no_compatible_worker` parked count exposed as a gauge + alert condition.
- Drain completes when every pinned activity finishes or crosses a `ContinueAsNew` boundary (which re-pins onto current code).

## Cross-Cutting

- Idempotent submission (`Idempotency-Key`), keys retained 24h, nightly cleanup.
- Structured logging (zerolog/zap) with correlation IDs across activity → attempt → call.
- Audit events written asynchronously (channel + writer goroutine) — audit never blocks the hot path.

## Acceptance Tests

1. Submit with idempotency key → long-poll wait → completion; duplicate submit returns the same activity.
2. Tenant A cannot see tenant B's activities (`NOT_FOUND`), metrics, or rate-limit budget.
3. Higher-tier activity submitted later executes first; aging prevents starvation past 30 minutes.
4. Rate-limit and backpressure paths return correct codes and `Retry-After`.
5. **Poison-recovery drill**: force a determinism violation → not retried; `:truncate-replay` via API → activity resumes and completes; audit trail shows the intervention.
6. **Drain drill**: pins on `1.0.0` drain to zero via completion/`ContinueAsNew`; old workers removed with nothing parked.

## Done When

- The OpenAPI file and the running server agree exactly.
- Every operator escape hatch is an audited API call, not a SQL session.
- A tenant cannot observe, starve, or affect another tenant.
