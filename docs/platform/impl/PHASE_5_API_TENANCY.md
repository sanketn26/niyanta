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
internal/api/handlers/cluster.go
internal/api/handlers/versions.go
internal/api/handlers/events.go
internal/auth/capabilities.go
internal/audit/transactional.go
internal/scheduler/priority.go
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
| `GET /activities/{id}/composition` | cursor-paged/lazy parent-child traversal for the console |
| `GET /cluster`, `/versions`, `/workers/{id}/leases` | bounded cluster, version, and lease inspection |
| `POST /activities/{id}:retry-now` | routine operator action with state preconditions |
| `POST /workers/{id}:drain`, `/versions/{type}/{version}:drain` | deployment operations with preview/progress |

`GET /activities` also supports `pinned_version`, `condition`, `created_after`, `created_before`, `updated_after`, `updated_before`, and admin-only `customer_id`. All collection endpoints use stable cursor pagination; offset pagination is not used by the console.

Update the OpenAPI file in the same change — it is the source of truth.

## State Access Boundary

The CLI and console consume this API only. They never import storage implementations or connect to core Postgres tables. This keeps tenant filtering, redaction, pagination, and migration compatibility at one boundary.

## Auth, Capabilities, And Tenancy

- API keys, hashed at rest (`api_keys` table: hash, customer_id, capabilities, created/revoked). These are service credentials, not browser session tokens.
- Customer extraction middleware; **every** storage query tenant-filtered at the boundary.
- Cross-tenant access returns `NOT_FOUND`, never `FORBIDDEN` — no existence leaks. Contract-tested.
- Define capabilities now: `activity:read`, `payload:read`, `activity:operate`, `activity:recover`, `worker:drain`, and `tenant:admin`. Phase 7 adds browser identity/session handling but reuses these server-enforced capabilities.

## Safe Operator Mutations

- Every mutation requires `Idempotency-Key` and `expected_generation`, or an equivalent `If-Match` resource version.
- A stale precondition returns `409 CONFLICT` with current non-sensitive metadata; it never silently applies to a newer state.
- Recovery and drain endpoints expose a preview. Execution requires a short-lived, single-use confirmation token bound to principal, tenant, action, target, and generation, plus an operator reason.
- The state change and authoritative audit event commit in the same database transaction. Async audit may be used for telemetry only.
- Audit rows record principal, tenant, request ID, action, target, reason, and before/after metadata without copying payloads or credentials.

## Payload And Query Safety

- Inputs, results, signals, checkpoints, errors, and logs are redacted server-side by default. `payload:read` is required to reveal them; sensitive reads are audited.
- Every response and field has a size cap. Logs, calls, attempts, audit rows, children, workers, and leases are cursor-paged.
- Composition traversal is lazy; no endpoint recursively materializes an unbounded tree.

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
- Non-authoritative diagnostic events may be asynchronous. Privileged mutations use transactional audit as defined above.

## Acceptance Tests

1. Submit with idempotency key → long-poll wait → completion; duplicate submit returns the same activity.
2. Tenant A cannot see tenant B's activities (`NOT_FOUND`), metrics, or rate-limit budget.
3. Higher-tier activity submitted later executes first; aging prevents starvation past 30 minutes.
4. Rate-limit and backpressure paths return correct codes and `Retry-After`.
5. **Poison-recovery drill**: force a determinism violation → not retried; `:truncate-replay` via API → activity resumes and completes; audit trail shows the intervention.
6. **Drain drill**: pins on `1.0.0` drain to zero via completion/`ContinueAsNew`; old workers removed with nothing parked.
7. Two operators mutate the same generation: one succeeds; one receives `409`; idempotent retry does not duplicate the action.
8. Kill the API immediately after a recovery commit; its audit row is present after restart.
9. A caller without `payload:read` cannot retrieve sensitive fields through list, detail, logs, signals, composition, or export routes.
10. OpenAPI user-journey tests generate and compile the Phase 6 TypeScript client.

## Done When

- The OpenAPI file and the running server agree exactly.
- Every operator escape hatch is an audited API call, not a SQL session.
- A tenant cannot observe, starve, or affect another tenant.
