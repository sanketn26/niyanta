# Phase 7: Console Operator Workflows + Live Updates

**Goal**: Authorized humans can safely change engine state from a browser, and watched resources converge after network or process interruptions.

## Browser Identity

- Prefer OIDC Authorization Code + PKCE for connected deployments. Provide a documented local-admin/session mode for air-gapped installations.
- API keys remain service credentials; never place them in browser local storage.
- Use server-side sessions with `HttpOnly`, `Secure`, `SameSite` cookies, rotation, expiry, logout, and revocation.
- Require CSRF protection for mutations and ship restrictive CSP, frame, referrer, and content-type headers.

## Authorization

Enforce capabilities in the coordinator API, not by hiding buttons:

- `activity:read`
- `payload:read`
- `activity:operate` — cancel, pause/resume, signal, retry-now
- `activity:recover` — force-fail, force-complete, truncate replay
- `worker:drain`
- `tenant:admin`

Capabilities are tenant-scoped unless explicitly system-wide. Every SSE subscription and resource read uses the same principal and tenant checks.

## Mutation Protocol

Every action includes an `Idempotency-Key`, expected generation/resource version, and current target identity. Privileged actions also require a reason. Stale preconditions return an explicit conflict and current metadata.

Recovery and drain workflows first call a preview endpoint. The server returns the expected effect and a short-lived, single-use confirmation token bound to principal, target, action, and generation. Typed confirmation is UI friction only; it does not replace server preconditions.

The state mutation and authoritative audit event commit in one database transaction. Audit records include principal, tenant, action, target, reason, before/after metadata, request ID, and timestamp. Payloads and credentials are not copied into audit rows.

## Live Update Contract

- SSE endpoint is tenant-scoped and emits versioned envelopes with monotonic event IDs, resource kind/ID, resource version, event type, and timestamp.
- Support heartbeat frames, `Last-Event-ID`, a bounded replay window, per-principal connection limits, and slow-client eviction.
- Events are invalidations, not authoritative state. The UI refetches affected resources; after a gap or expired replay window it performs a full reconciliation.
- Document buffering/timeouts for supported reverse proxies and load balancers.

## Acceptance Tests

1. Viewer, routine operator, recovery operator, and tenant admin matrices are enforced through direct API calls.
2. Two operators act on one generation: one commits; the other receives a conflict; retrying either idempotency key does not repeat an action.
3. Kill the API process after mutation commit: the audit row survives.
4. Disconnect past the SSE replay window: the client detects the gap, reconciles, and converges.
5. Cross-tenant SSE subscriptions, guessed IDs, and saved-filter access disclose nothing.
6. CSRF, session fixation/revocation, CSP, and logout tests pass.

## Done When

- Routine and recovery operations are capability-gated, concurrency-safe, idempotent, and transactionally audited.
- Live updates improve freshness but are never required for correctness.
