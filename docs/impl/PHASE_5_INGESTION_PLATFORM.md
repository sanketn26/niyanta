# Phase 5: Production Ingestion Platform

**Goal**: Build the actual product layer: connector definitions, customer connections, selected catalogs, ingestion planner, tenant isolation, dense shared execution, observability, API, and the first declarative HTTP poller.

Connector definitions, connection manifests, event filters, and operator-authored policies are authored as YAML. The implementation parses YAML into typed Go structs and may persist the normalized representation in Postgres `JSONB` columns for indexing and queryability.

Phase 5 must implement the correctness rules in [../OPERATIONS_AND_FAILURE_SEMANTICS.md](../OPERATIONS_AND_FAILURE_SEMANTICS.md): at-least-once delivery, checkpoint-after-confirmed-ingestion, dead-letter sinks, connector spec versioning, secret-rotation wakeups, backfill isolation, and blast-radius controls.

## Files To Add

```text
internal/connector/models.go
internal/connector/registry.go
internal/connector/runtime.go
internal/connector/catalog.go
internal/connector/http_poller.go
internal/ingest/planner.go
internal/ingest/supervisor.go
internal/tenant/isolation.go
internal/api/connectors.go
internal/observability/metrics.go
```

## Connector Models

```go
package connector

type IsolationLevel string

const (
    IsolationShared    IsolationLevel = "shared"
    IsolationPooled    IsolationLevel = "pooled"
    IsolationDedicated IsolationLevel = "dedicated"
)

type Definition struct {
    ID            string
    Version       string
    Kind          string
    Direction     Direction
    Display       Display
    Isolation     IsolationRequirement
    Packaging     Packaging
    Compatibility Compatibility
    Spec          Spec
}

type Direction string

const (
    DirectionSource               Direction = "source"
    DirectionDestination          Direction = "destination"
    DirectionSourceAndDestination Direction = "source_and_destination"
)

type IsolationRequirement struct {
    MinimumLevel             IsolationLevel
    RequiresDedicatedNetwork bool
    AllowsSharedWorkers      bool
}

type Connection struct {
    ID                string
    CustomerID        string
    DefinitionID      string
    DefinitionVersion string
    Name              string
    Status            string
    Controls          ControlState
    Isolation         IsolationPolicy
    Config            map[string]any
    SecretRefs        map[string]string
    Destination       Destination
    Catalog           Catalog
    CatalogVersion    string
}

type Packaging struct {
    Mode                 string
    Runtime              string
    Image                string
    Entrypoint           []string
    AllowedHosts         []string
    ResourceRequirements map[string]ResourceRequirement
}

type Compatibility struct {
    Protocol        string
    ProtocolVersion string
    ImportedFrom    string
}

type Catalog struct {
    Streams []CatalogStream
}

type CatalogStream struct {
    Name                 string
    Namespace            string
    SourceSchemaRef      string
    PrimaryKey           []string
    CursorField          []string
    SupportedSyncModes   []string
    DefaultSyncMode      string
    DestinationSyncModes []string
}

type ResourceRequirement struct {
    CPURequest    string
    CPULimit      string
    MemoryRequest string
    MemoryLimit   string
}

type ControlState struct {
    EffectiveStatus          string
    TemporaryDisabledUntil   *time.Time
    RateLimitOverride        *RateLimitOverride
    LastControlReason        string
}

type RateLimitOverride struct {
    QPS                   float64
    Burst                 int
    MaxParallelPartitions int
}

type IsolationPolicy struct {
    Level        IsolationLevel
    WorkerPool   string
    EgressPolicy string
}
```

## Storage Schema

```sql
CREATE TABLE connector_definitions (
    id              TEXT NOT NULL,
    version         TEXT NOT NULL,
    kind            TEXT NOT NULL,
    direction       TEXT NOT NULL DEFAULT 'source',
    owner_customer_id TEXT,
    display         JSONB NOT NULL,
    isolation       JSONB NOT NULL,
    packaging       JSONB NOT NULL DEFAULT '{}',
    compatibility   JSONB NOT NULL DEFAULT '{}',
    spec            JSONB NOT NULL,
    status          TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (id, version)
);

CREATE TABLE connector_connections (
    id                  TEXT PRIMARY KEY,
    customer_id          TEXT NOT NULL,
    definition_id        TEXT NOT NULL,
    definition_version   TEXT NOT NULL,
    name                TEXT NOT NULL,
    status              TEXT NOT NULL,
    isolation           JSONB NOT NULL,
    config              JSONB NOT NULL,
    secret_refs         JSONB NOT NULL,
    destination         JSONB NOT NULL,
    catalog             JSONB NOT NULL DEFAULT '{}',
    catalog_version     TEXT,
    health              JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    FOREIGN KEY (definition_id, definition_version)
        REFERENCES connector_definitions(id, version)
);

CREATE TABLE connector_runs (
    id              TEXT PRIMARY KEY,
    customer_id      TEXT NOT NULL,
    connection_id    TEXT NOT NULL REFERENCES connector_connections(id) ON DELETE CASCADE,
    activity_id      TEXT REFERENCES activities(id) ON DELETE SET NULL,
    partition_key    TEXT NOT NULL,
    status           TEXT NOT NULL,
    trigger_type     TEXT NOT NULL,
    window_start     TIMESTAMPTZ,
    window_end       TIMESTAMPTZ,
    checkpoint_in    JSONB NOT NULL DEFAULT '{}',
    checkpoint_out   JSONB NOT NULL DEFAULT '{}',
    metrics          JSONB NOT NULL DEFAULT '{}',
    error            JSONB,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    started_at       TIMESTAMPTZ,
    completed_at     TIMESTAMPTZ
);

CREATE INDEX idx_connector_connections_tenant
    ON connector_connections(customer_id, status);

CREATE INDEX idx_connector_runs_connection
    ON connector_runs(customer_id, connection_id, created_at DESC);

CREATE TABLE stream_partition_leases (
    id              TEXT PRIMARY KEY,
    customer_id     TEXT NOT NULL,
    connection_id   TEXT NOT NULL REFERENCES connector_connections(id) ON DELETE CASCADE,
    partition_id    TEXT NOT NULL,
    lease_owner     TEXT,
    generation      BIGINT NOT NULL DEFAULT 1,
    lease_expires_at TIMESTAMPTZ,
    checkpoint      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(customer_id, connection_id, partition_id)
);

CREATE INDEX idx_stream_partition_leases_due
    ON stream_partition_leases(customer_id, connection_id, lease_expires_at);

CREATE TABLE ingestion_events (
    id              TEXT PRIMARY KEY,
    customer_id     TEXT NOT NULL,
    connection_id   TEXT REFERENCES connector_connections(id) ON DELETE CASCADE,
    run_id          TEXT REFERENCES connector_runs(id) ON DELETE SET NULL,
    severity        TEXT NOT NULL,
    event_type      TEXT NOT NULL,
    significant     BOOLEAN NOT NULL DEFAULT FALSE,
    message         TEXT NOT NULL,
    details         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_ingestion_events_customer_time
    ON ingestion_events(customer_id, created_at DESC);

CREATE INDEX idx_ingestion_events_significant
    ON ingestion_events(customer_id, significant, created_at DESC)
    WHERE significant = TRUE;

CREATE TABLE ingestion_event_filters (
    id              TEXT PRIMARY KEY,
    customer_id     TEXT NOT NULL,
    name            TEXT NOT NULL,
    enabled         BOOLEAN NOT NULL DEFAULT TRUE,
    scope           JSONB NOT NULL DEFAULT '{}',
    match           JSONB NOT NULL,
    action          JSONB NOT NULL DEFAULT '{"mark_significant": true}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_ingestion_event_filters_customer
    ON ingestion_event_filters(customer_id, enabled);

CREATE TABLE connection_controls (
    id              TEXT PRIMARY KEY,
    customer_id     TEXT NOT NULL,
    connection_id   TEXT NOT NULL REFERENCES connector_connections(id) ON DELETE CASCADE,
    control_type    TEXT NOT NULL,
    config          JSONB NOT NULL DEFAULT '{}',
    reason          TEXT NOT NULL,
    created_by      TEXT NOT NULL,
    expires_at      TIMESTAMPTZ,
    cleared_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_connection_controls_active
    ON connection_controls(customer_id, connection_id, control_type)
    WHERE cleared_at IS NULL;
```

## Tenant Isolation Guard

```go
package tenant

import "context"

type key struct{}

func WithCustomerID(ctx context.Context, customerID string) context.Context {
    return context.WithValue(ctx, key{}, customerID)
}

func CustomerID(ctx context.Context) (string, bool) {
    v, ok := ctx.Value(key{}).(string)
    return v, ok
}

func RequireCustomerID(ctx context.Context) (string, error) {
    id, ok := CustomerID(ctx)
    if !ok || id == "" {
        return "", ErrMissingCustomer
    }
    return id, nil
}
```

Every store method for tenant data starts with `tenant.RequireCustomerID(ctx)` and includes `customer_id = $n` in SQL.

## Significant Event Filters

Significant event filters classify ingestion events for a customer. They are used by the ingestion view and alerting; they do not directly pause or rate-limit a connection.

```go
type IngestionEvent struct {
    ID            string
    CustomerID    string
    ConnectionID  string
    RunID         string
    Severity      string
    EventType     string
    Significant   bool
    Message       string
    Details       map[string]any
    CreatedAt     time.Time
}

type EventFilter struct {
    ID         string
    CustomerID string
    Name       string
    Enabled    bool
    Scope      FilterScope
    Match      EventMatch
    Action     FilterAction
}

type FilterScope struct {
    ConnectorID  string
    ConnectionID string
}

type EventMatch struct {
    EventTypes []string
    Severities []string
    MinCount   int
    Window     time.Duration
}

type FilterAction struct {
    MarkSignificant bool
    Alert           bool
}
```

Event recording applies filters before insert:

```go
func (s *EventService) Record(ctx context.Context, event IngestionEvent) error {
    customerID, err := tenant.RequireCustomerID(ctx)
    if err != nil {
        return err
    }
    event.CustomerID = customerID

    filters, err := s.filters.ListEnabled(ctx, customerID)
    if err != nil {
        return err
    }
    for _, f := range filters {
        if f.Matches(event) && f.Action.MarkSignificant {
            event.Significant = true
        }
    }
    return s.events.Insert(ctx, event)
}
```

Example filter:

```yaml
name: Repeated source throttling
scope:
  connector_id: github_audit
match:
  event_types:
    - source_rate_limited
  severities:
    - warning
    - error
  min_count: 3
  window_seconds: 300
action:
  mark_significant: true
  alert: true
```

## Erring Connection Controls

Controls are active, tenant-scoped overrides applied by an operator or automated policy.

```go
type ConnectionControl struct {
    ID           string
    CustomerID   string
    ConnectionID string
    ControlType  string
    Config       map[string]any
    Reason       string
    CreatedBy    string
    ExpiresAt    *time.Time
    ClearedAt    *time.Time
}

type ControlStore interface {
    Add(ctx context.Context, c ConnectionControl) error
    Clear(ctx context.Context, customerID, connectionID, controlType, reason string) error
    ListActive(ctx context.Context, customerID, connectionID string) ([]ConnectionControl, error)
}
```

Planner usage:

```go
func (p *Planner) planConnection(ctx context.Context, conn connector.Connection) error {
    controls, err := p.controls.ListActive(ctx, conn.CustomerID, conn.ID)
    if err != nil {
        return err
    }

    effective := ApplyControls(conn, controls, time.Now())
    switch effective.Status {
    case "quarantined", "disabled", "temporarily_disabled", "paused":
        return p.connections.ScheduleNextReview(ctx, conn.ID, effective.NextReviewAt)
    }

    if effective.RateLimitOverride != nil {
        conn.Spec.RateLimit = mergeRateLimit(conn.Spec.RateLimit, *effective.RateLimitOverride)
        conn.Spec.MaxParallelPartitions = min(
            conn.Spec.MaxParallelPartitions,
            effective.RateLimitOverride.MaxParallelPartitions,
        )
    }

    return p.planRunnableConnection(ctx, conn)
}
```

Temporary disable handler:

```go
func (h *Handler) TemporaryDisableConnection(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    customerID, _ := tenant.RequireCustomerID(ctx)
    connectionID := chi.URLParam(r, "connection_id")

    var req struct {
        Reason           string `json:"reason"`
        DurationSeconds  int    `json:"duration_seconds"`
    }
    _ = json.NewDecoder(r.Body).Decode(&req)

    expiresAt := time.Now().UTC().Add(time.Duration(req.DurationSeconds) * time.Second)
    control := ConnectionControl{
        ID:           h.ids.New("ctrl"),
        CustomerID:   customerID,
        ConnectionID: connectionID,
        ControlType:  "temporary_disable",
        Config:       map[string]any{"resume_strategy": "auto_on_expiry"},
        Reason:       req.Reason,
        CreatedBy:    h.auth.Subject(ctx),
        ExpiresAt:    &expiresAt,
    }

    if err := h.controls.Add(ctx, control); err != nil {
        writeError(w, err)
        return
    }
    h.signals.WakeConnectionSupervisor(ctx, connectionID, "control_changed")
    writeJSON(w, http.StatusAccepted, control)
}
```

Rate-limit override handler:

```go
func (h *Handler) RateLimitConnection(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    customerID, _ := tenant.RequireCustomerID(ctx)
    connectionID := chi.URLParam(r, "connection_id")

    var req struct {
        QPS                  float64 `json:"qps"`
        Burst                int     `json:"burst"`
        MaxParallelPartitions int    `json:"max_parallel_partitions"`
        DurationSeconds      int     `json:"duration_seconds"`
        Reason               string  `json:"reason"`
    }
    _ = json.NewDecoder(r.Body).Decode(&req)

    expiresAt := time.Now().UTC().Add(time.Duration(req.DurationSeconds) * time.Second)
    control := ConnectionControl{
        ID:           h.ids.New("ctrl"),
        CustomerID:   customerID,
        ConnectionID: connectionID,
        ControlType:  "rate_limit_override",
        Config: map[string]any{
            "qps": req.QPS,
            "burst": req.Burst,
            "max_parallel_partitions": req.MaxParallelPartitions,
        },
        Reason:    req.Reason,
        CreatedBy: h.auth.Subject(ctx),
        ExpiresAt: &expiresAt,
    }

    if err := h.controls.Add(ctx, control); err != nil {
        writeError(w, err)
        return
    }
    h.signals.WakeConnectionSupervisor(ctx, connectionID, "control_changed")
    writeJSON(w, http.StatusAccepted, control)
}
```

## Customer Ingestion View

```go
type CustomerIngestionView struct {
    CustomerID   string
    Summary      IngestionSummary
    Connections  []ConnectionView
    Events       []IngestionEvent
}

type IngestionSummary struct {
    ConnectionsTotal       int
    Connected              int
    Degraded               int
    Failing                int
    TemporarilyDisabled    int
    Quarantined            int
}

type ConnectionView struct {
    ConnectionID             string
    Name                     string
    ConnectorID              string
    Health                   string
    EffectiveStatus          string
    SourceLagSeconds         int64
    CheckpointAgeSeconds     int64
    RecordsRead             int64
    RecordsEmitted          int64
    RecordsDropped          int64
    ActiveControls           []ConnectionControl
    LatestSignificantEvent   *IngestionEvent
}
```

Read path:

```go
func (s *ViewService) GetCustomerIngestionView(ctx context.Context, customerID string, window time.Duration) (CustomerIngestionView, error) {
    ctx = tenant.WithCustomerID(ctx, customerID)

    conns, err := s.connections.ListForCustomer(ctx)
    if err != nil {
        return CustomerIngestionView{}, err
    }
    controls, err := s.controls.ListActiveForCustomer(ctx, customerID)
    if err != nil {
        return CustomerIngestionView{}, err
    }
    metrics, err := s.metrics.LoadConnectionRollups(ctx, customerID, window)
    if err != nil {
        return CustomerIngestionView{}, err
    }
    events, err := s.events.ListSignificant(ctx, customerID, 100)
    if err != nil {
        return CustomerIngestionView{}, err
    }

    return BuildIngestionView(customerID, conns, controls, metrics, events), nil
}
```

## Ingestion Planner

```go
type Planner struct {
    connections ConnectionStore
    checkpoints CheckpointStore
    runs        RunStore
    activities  ActivityStore
    quotas      QuotaService
    ids         IDGenerator
}

func (p *Planner) PlanDueConnections(ctx context.Context, limit int) error {
    conns, err := p.connections.ListDue(ctx, limit)
    if err != nil {
        return err
    }

    for _, conn := range conns {
        if err := p.planConnection(ctx, conn); err != nil {
            p.connections.MarkDegraded(ctx, conn.ID, err)
        }
    }
    return nil
}

func (p *Planner) planConnection(ctx context.Context, conn connector.Connection) error {
    if err := p.quotas.CanPlan(ctx, conn.CustomerID); err != nil {
        return err
    }

    checkpoint, err := p.checkpoints.GetLatest(ctx, conn.CustomerID, conn.ID, "live")
    if err != nil {
        return err
    }

    plan := BuildPlan(conn, checkpoint)
    for _, part := range plan.Partitions {
        runID := p.ids.New("run")
        activityID := p.ids.New("act")
        if err := p.runs.Create(ctx, ConnectorRunFromPartition(runID, activityID, conn, part)); err != nil {
            return err
        }
        if err := p.activities.Create(ctx, ActivityFromRun(activityID, conn.CustomerID, "run_ingestion_partition", part)); err != nil {
            return err
        }
    }
    return nil
}
```

## HTTP Poller Runtime

```go
type ConnectorExecutor interface {
    GetSpec(ctx context.Context, definition connector.Definition) (connector.Spec, error)
    Validate(ctx context.Context, conn connector.Connection) error
    Test(ctx context.Context, conn connector.Connection) connector.TestResult
    Discover(ctx context.Context, conn connector.Connection) (connector.Catalog, error)
    Run(ctx context.Context, run connector.Run, catalog connector.Catalog, checkpoint []byte, sink RecordSink) ([]byte, error)
}

type HTTPPoller struct {
    client  *http.Client
    secrets SecretResolver
    sink    RecordSink
}

func (p *HTTPPoller) GetSpec(ctx context.Context, definition connector.Definition) (connector.Spec, error) {
    return definition.Spec, nil
}

func (p *HTTPPoller) Discover(ctx context.Context, conn connector.Connection) (connector.Catalog, error) {
    // Phase 5 supports static catalogs declared by the connector definition.
    // Dynamic discovery and Airbyte-compatible discovery arrive in Phase 5A/5B.
    return catalogFromDefinition(conn.DefinitionID, conn.DefinitionVersion), nil
}

func (p *HTTPPoller) Run(ctx context.Context, run connector.Run, catalog connector.Catalog, checkpoint []byte) ([]byte, error) {
    req, err := p.buildRequest(ctx, run, checkpoint)
    if err != nil {
        return nil, err
    }

    resp, err := p.client.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    if resp.StatusCode == http.StatusTooManyRequests {
        return nil, SourceRateLimited{RetryAfter: parseRetryAfter(resp)}
    }
    if resp.StatusCode < 200 || resp.StatusCode >= 300 {
        return nil, fmt.Errorf("source returned %d", resp.StatusCode)
    }

    records, nextCheckpoint, err := p.parseAndTransform(resp.Body, run)
    if err != nil {
        return nil, err
    }

    if err := p.sink.Write(ctx, run.CustomerID, run.ConnectionID, records); err != nil {
        return nil, err
    }

    // The caller commits this checkpoint only after this method returns success.
    return nextCheckpoint, nil
}
```

Phase 5 stores and uses selected catalogs, but only for the native `poll_http` runtime. Airbyte-compatible command execution, database discovery, CDC discovery, and destination connector discovery are Phase 5A/5B work.

## Dead Letter Sink

```go
type DeadLetterSink interface {
    Write(ctx context.Context, record DeadLetterRecord) error
}

type DeadLetterRecord struct {
    CustomerID    string
    ConnectionID  string
    RunID         string
    PartitionID   string
    SourceMeta    map[string]any
    Payload       []byte
    Error         string
    ConnectorID   string
    ConnectorVer  string
    CreatedAt     time.Time
}
```

Supported implementations:

- object/file sink for S3, GCS, Azure Blob, OCI Object Storage, or filesystem
- stream sink for Niyanta stream, Kafka, Event Hubs, Pub/Sub, or OCI Streaming

Dead-letter sink failure fails the run by default.

## Secret Rotation Wakeups

```go
func (m *ConnectionManager) OnSecretUpdated(ctx context.Context, customerID, secretRef string) error {
    ctx = tenant.WithCustomerID(ctx, customerID)

    conns, err := m.connections.ListUsingSecret(ctx, secretRef)
    if err != nil {
        return err
    }
    for _, conn := range conns {
        if err := m.audit.Write(ctx, AuditEvent{
            Action: "secret_ref_updated",
            ResourceType: "connector_connection",
            ResourceID: conn.ID,
        }); err != nil {
            return err
        }
        if err := m.signals.WakeConnectionSupervisor(ctx, conn.ID, "secret_rotated"); err != nil {
            return err
        }
    }
    return nil
}
```

## Connector Spec Versioning

```go
type ConnectorVersion struct {
    ID          string
    Version     semver.Version
    Compatibility Compatibility
}

type Compatibility struct {
    ConfigSchemaCompatible     bool
    OutputSchemaCompatible     bool
    CheckpointSchemaCompatible bool
    RequiresMigration          bool
}
```

Existing connections pin `definition_id` and `definition_version`. Upgrades are explicit and audited.

## Stream Subscription Runtime

Kafka, Event Hubs, GCP Pub/Sub, and OCI Streaming use the same internal pattern: discover partitions, lease partitions, consume batches, deliver records, then commit Niyanta checkpoint and source offset/cursor.

```go
type StreamProvider interface {
    DiscoverPartitions(ctx context.Context, conn connector.Connection) ([]StreamPartition, error)
    OpenReader(ctx context.Context, conn connector.Connection, partition StreamPartition, checkpoint []byte) (StreamReader, error)
    Commit(ctx context.Context, conn connector.Connection, partition StreamPartition, checkpoint []byte) error
}

type StreamReader interface {
    ReadBatch(ctx context.Context, maxRecords int, maxWait time.Duration) ([]StreamMessage, error)
    Close() error
}

type StreamMessage struct {
    Key        []byte
    Value      []byte
    Headers    map[string]string
    Checkpoint []byte
}
```

Partition lease acquisition:

```go
func (s *StreamLeaseStore) Claim(ctx context.Context, customerID, connectionID, workerID string, limit int) ([]StreamPartitionLease, error) {
    return s.query(ctx, `
        WITH due AS (
            SELECT id
            FROM stream_partition_leases
            WHERE customer_id = $1
              AND connection_id = $2
              AND (lease_owner IS NULL OR lease_expires_at < NOW())
            ORDER BY updated_at, id
            LIMIT $3
            FOR UPDATE SKIP LOCKED
        )
        UPDATE stream_partition_leases l
        SET lease_owner = $4,
            generation = generation + 1,
            lease_expires_at = NOW() + INTERVAL '60 seconds',
            updated_at = NOW()
        FROM due
        WHERE l.id = due.id
        RETURNING l.*
    `, customerID, connectionID, limit, workerID)
}
```

Worker process loop:

```go
func (r *StreamRuntime) RunPartition(ctx context.Context, lease StreamPartitionLease) error {
    reader, err := r.provider.OpenReader(ctx, lease.Connection, lease.Partition, lease.Checkpoint)
    if err != nil {
        return err
    }
    defer reader.Close()

    for ctx.Err() == nil {
        if err := r.leases.Heartbeat(ctx, lease.ID, lease.Generation); err != nil {
            return err
        }

        batch, err := reader.ReadBatch(ctx, r.cfg.MaxRecords, r.cfg.MaxWait)
        if err != nil {
            return err
        }
        if len(batch) == 0 {
            continue
        }

        records, checkpoint, err := r.transform(batch)
        if err != nil {
            return err
        }
        if err := r.sink.Write(ctx, lease.CustomerID, lease.ConnectionID, records); err != nil {
            return err
        }
        if err := r.leases.CommitCheckpoint(ctx, lease.ID, lease.Generation, checkpoint); err != nil {
            return err
        }
        if err := r.provider.Commit(ctx, lease.Connection, lease.Partition, checkpoint); err != nil {
            return err
        }
    }
    return ctx.Err()
}
```

Provider mapping:

| Provider | Reader opens | Checkpoint contains |
|----------|--------------|---------------------|
| Kafka | topic partition reader for assigned partition | topic, partition, offset |
| Event Hubs | partition receiver for assigned partition and consumer group | namespace, hub, partition, offset/sequence |
| GCP Pub/Sub | streaming pull on subscription with bounded concurrency | ack IDs plus message IDs/order keys |
| OCI Streaming | cursor for stream partition | stream OCID, partition, cursor/offset |

## Connection Supervisor Activity

```go
func (a SupervisorActivity) Execute(ctx activity.Context, input []byte) ([]byte, error) {
    var req SupervisorInput
    if err := json.Unmarshal(input, &req); err != nil {
        return nil, err
    }

    planBytes, err := ctx.RunChild("plan_connection", input, activity.ChildOptions{})
    if err != nil {
        return nil, err
    }

    var plan ingest.Plan
    if err := json.Unmarshal(planBytes, &plan); err != nil {
        return nil, err
    }

    for _, part := range plan.Partitions {
        payload, _ := json.Marshal(part)
        if _, err := ctx.RunChild("run_ingestion_partition", payload, activity.ChildOptions{}); err != nil {
            return nil, err
        }
    }

    return nil, ctx.Sleep(plan.NextWakeAfter)
}
```

## Metrics

```go
var (
    RunsTotal = promauto.NewCounterVec(prometheus.CounterOpts{
        Name: "niyanta_ingestion_runs_total",
        Help: "Total ingestion runs by tenant, connector, connection, and result.",
    }, []string{"customer_id", "connector_id", "connection_id", "result"})

    SourceLag = promauto.NewGaugeVec(prometheus.GaugeOpts{
        Name: "niyanta_ingestion_source_lag_seconds",
        Help: "Current source lag for a connector connection.",
    }, []string{"customer_id", "connector_id", "connection_id"})
)
```

## Acceptance Tests

1. Create a `poll_http` definition and connection.
2. Planner creates runs without manual activity submission.
3. Shared worker pool runs multiple tenants while respecting per-tenant concurrency caps.
4. Dedicated isolation routes a connection only to its approved worker pool.
5. Checkpoint is committed only after sink write succeeds.
6. Tenant A cannot list Tenant B's connections, runs, checkpoints, or logs.
7. A significant-event filter marks repeated source throttling as significant for one customer only.
8. A rate-limit override reduces planning concurrency/QPS for an erring connection.
9. Temporary disable stops new run planning until `expires_at`, then planning resumes.
10. Customer ingestion view returns connection health, active controls, latest significant events, lag, checkpoint age, and run metrics for the requested `cid`.
11. Secret rotation wakes affected connection supervisors.
12. Connector version upgrade is explicit, audited, and validates checkpoint compatibility.
13. Malformed records can be dead-lettered to object storage or a stream sink.

## Done When

- The product flow is connector-first.
- Ingestion is self-orchestrated.
- Multi-tenant high-density execution is safe by default.
- Connector definitions include direction, packaging, and compatibility metadata.
- Connections can discover, store, select, and run against a versioned catalog for the native `poll_http` runtime.
- Broader connector families are explicitly deferred: Airbyte-compatible command execution to Phase 5A/5B, and `database_source`, `cdc_source`, and complex `destination_connector` runtimes to Phase 5B.
