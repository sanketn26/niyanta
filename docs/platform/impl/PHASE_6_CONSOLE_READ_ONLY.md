# Phase 6: Read-Only Activity Manager Console

**Goal**: A human can find an activity and explain its state from a browser without introducing a privileged mutation path.

## Boundary

The console uses the versioned coordinator API exclusively. It has no core Postgres credentials and does not import storage repositories. This keeps tenancy, redaction, pagination, and schema compatibility in one server-side boundary.

The Go console backend serves a React + TypeScript SPA via `go:embed`, manages browser sessions/preferences, and proxies authenticated API reads. The frontend client is generated from OpenAPI and checked for drift in CI.

## Views

1. **Cluster summary**: coordinator state, workers, capacity, versions, queue state, and conditions. Charts are optional until Phase 8.
2. **Activities**: cursor-paged filters for status, type, version, condition, root, and time range. Tenant filtering is available only to principals with tenant administration capability.
3. **Activity detail**: metadata, attempts, paged call/replay timeline, signal and heartbeat metadata, `ContinueAsNew` chain, audit trail, and progressively loaded parent/child composition.
4. **Workers**: health, advertised capabilities/versions, heartbeat freshness, leases, and in-flight activity metadata.
5. **Versions**: pinned-version inventory and drain status, read-only in this phase.

## Data Safety And Browser Limits

- Inputs, results, signals, checkpoints, errors, and logs are redacted server-side by default. Revealing payloads requires `payload:read` and emits a sensitive-read audit event.
- Never render payload or log content as HTML. Apply CSP and content-type protections.
- Cursor-page or virtualize attempts, calls, audit rows, logs, and workers. Load composition children on demand. Enforce server response-size and query-time limits.
- Treat exports as sensitive payload reads and apply the same authorization, redaction, limits, and audit policy.
- Store preferences outside engine core tables. Saved filters must be tenant- and principal-scoped.

## UX Quality Bar

- Responsive desktop/tablet layout; operational desktop use is primary.
- Keyboard navigation and automated WCAG 2.2 AA checks.
- Explicit loading, empty, partial-data, stale-data, unauthorized, and error states.
- URLs preserve filters and selected activity tabs without containing secrets.

## Acceptance Tests

1. Find a suspended activity and identify its exact call index and wait reason in under one minute.
2. Navigate a synthetic 10,000-node composition through progressive loading without an unbounded query or browser freeze.
3. A principal without `payload:read` cannot retrieve payloads through UI, direct API, or export; an authorized reveal is audited.
4. Cross-tenant list/detail/audit/saved-filter requests reveal no resource existence.
5. The embedded SPA works without a CDN and passes keyboard and automated accessibility checks.
6. The console process has no direct core-database credentials.

## Done When

- The console explains state reliably using only the public/operator read API.
- Large histories remain bounded in API, server, and browser memory.
- No operator mutation is exposed yet.
