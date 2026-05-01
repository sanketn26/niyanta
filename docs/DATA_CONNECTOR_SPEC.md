# General Purpose Data Connector Specification

**Version**: 0.1
**Last Updated**: 2026-05-01
**Status**: Draft

## Overview

This document defines a general-purpose data connector model for Niyanta's self-orchestrated ingestion platform. It is based on the recurring patterns in Microsoft Sentinel's `DataConnectors` area and its published connector guidance: connectors are packaged metadata plus executable ingestion behavior, separated into user-facing configuration, source access rules, parsing/transformation, delivery, state checkpoints, health, and validation.

The goal is to support connectors for API polling, push ingestion, file/blob ingestion, syslog-like streams, agent or function based collectors, and custom enrichment workflows without hard-coding a connector per vendor. Customers configure connector connections; Niyanta plans and runs ingestion continuously based on schedules, lag, checkpoints, backfills, source limits, destination pressure, and health policy.

Connector definitions and connection manifests are authored as **YAML**. Niyanta may parse and store these documents internally as `JSONB` in Postgres for indexing and querying, but the human-facing specification format is YAML.

Sources:
- Azure Sentinel `DataConnectors` guide: https://github.com/Azure/Azure-Sentinel/tree/master/DataConnectors
- Microsoft Sentinel Codeless Connector Framework: https://learn.microsoft.com/en-us/azure/sentinel/create-codeless-connector
- CCF UI definitions reference: https://learn.microsoft.com/en-us/azure/sentinel/data-connector-ui-definitions-reference
- RestApiPoller connection rules reference: https://learn.microsoft.com/en-us/azure/sentinel/data-connector-connection-rules-reference
- Niyanta ingestion architecture: [INGESTION_ARCHITECTURE.md](INGESTION_ARCHITECTURE.md)

## Design Posture

Connectors are not submitted as one-off jobs. A connector definition describes what can be ingested, a connector connection describes a customer's configured source, and the ingestion control plane decides what activity attempts should run next.

The connector system must be:

- **Self-orchestrated**: supervisors, planners, durable timers, and signals create runs from state.
- **Multi-tenant**: every definition, connection, run, checkpoint, destination, and secret reference is tenant-scoped unless explicitly public.
- **High-density**: connectors should run safely in shared regional worker pools without requiring a customer-specific deployment.
- **Isolated**: connector definitions declare compute, network, secret, and state isolation requirements.
- **Observable**: health, lag, checkpoint age, parser drops, delivery failures, and source throttling are first-class outputs.
- **Scalable**: every connector declares its partitioning and concurrency boundaries.
- **Adaptable**: common sources use declarative specs; uncommon sources use plugins behind the same runtime contract.
- **Recoverable**: checkpoint commits happen only after confirmed delivery.

## Extracted Connector Patterns

### 1. Connector Type Is Chosen By Ingestion Shape

The Sentinel guide separates connectors by how data moves:

| Pattern | Source shape | Runtime behavior | Niyanta equivalent |
|---------|--------------|------------------|--------------------|
| API push | External product can send events to the platform | Expose an ingestion endpoint and validate signed requests | `push_http` connector |
| Native SaaS polling | Platform calls a remote API on a schedule | Declarative auth, request, response, paging, and DCR/destination config | `poll_http` connector |
| Function/worker polling | Custom code polls a source API | Run a packaged connector worker with code hooks | `worker_plugin` connector |
| Blob/object ingestion | Files are written to object storage | Scan, claim, parse, checkpoint file offsets or object versions | `object_store` connector |
| Managed stream subscription | Source exposes Kafka-compatible topics, Event Hubs, Pub/Sub, or OCI Streaming | Claim partitions/subscriptions, consume batches, commit offsets only after delivery | `stream_subscription` connector |
| CEF/syslog stream | Source emits standard text events | Agent or listener receives lines, parses, normalizes, forwards | `stream_listener` connector |
| Manual/import utility | Operator uploads or replays data | Batch parse and emit records | `batch_import` connector |

Design implication: connector definitions must describe the ingestion strategy explicitly, while connector runs should share the same lifecycle, checkpoint, retry, health, and delivery machinery.

### 2. Connector UX Metadata Is Separate From Connection Behavior

Sentinel's CCF separates the data connector UI definition from connection rules. The UI definition carries title, publisher, description, prerequisites, sample queries, health queries, and instruction widgets. Connection rules carry auth, request, response, paging, and DCR delivery details.

Niyanta should keep the same separation:

- `ConnectorDefinition`: reusable product/source template.
- `ConnectorConnection`: customer-specific configured instance.
- `ConnectorRun`: scheduled or triggered execution of a connection.

This allows one connector definition to support multiple customer connections, multiple accounts, multiple regions, and multiple endpoints.

### 3. The Durable State Boundary Is The Cursor/Checkpoint

API pollers and blob readers both depend on persisted progress:

- Time windows: `start_time`, `end_time`, `query_window`.
- Pagination cursors: next page token, link header, offset, page number.
- Stream offsets: file offset, partition offset, sequence number.
- Object position: bucket, key, version, etag, byte range.
- Deduplication watermarks: event IDs or source hashes.

In Niyanta, these are connector checkpoints. They must be small, durable, versioned structured blobs stored through the existing checkpoint model. They may be represented as YAML in examples and stored as parsed `JSONB` internally.

### 4. Parsing And Normalization Are First-Class

Sentinel connectors either land data in standard tables such as CEF/ASIM, land raw custom tables, or transform via DCR KQL. The general pattern is:

1. Read source payload.
2. Extract events from a response envelope.
3. Parse format-specific records.
4. Normalize to a declared schema.
5. Preserve unmapped source fields.
6. Emit sample queries and validation rules.

Niyanta should model this as a transform pipeline rather than a one-off code path per connector.

### 5. Operational Health Is Part Of The Contract

Sentinel connector definitions include connectivity criteria, last-data-received queries, graph queries, and validation steps. Niyanta should expose similar health signals:

- configured, connected, degraded, failing, paused, disabled
- last successful poll
- last event received
- lag behind source
- records read, dropped, transformed, delivered
- current checkpoint age
- auth failures and rate limit backoff

## Core Domain Model

### ConnectorDefinition

A versioned, tenant-independent connector template.

```yaml
id: github_audit
version: 1.0.0
kind: poll_http
display:
  title: GitHub Audit
  publisher: GitHub
  description: Collects organization audit events.
  logo_ref: logos/github.svg
  docs_url: https://docs.github.com
capabilities:
  - scheduled
  - multi_connection
  - checkpointed
isolation:
  minimum_level: shared
  requires_dedicated_network: false
  allows_shared_workers: true
configuration_schema: {}
source_schema: {}
output_schema: {}
auth_templates: []
request_templates: []
response_templates: []
paging_templates: []
transform_pipeline: []
health: {}
sample_queries: []
validation: {}
```

Rules:

- `id` is stable and globally unique.
- `version` follows semantic versioning.
- Breaking schema or checkpoint changes require a new major version.
- Definitions are immutable after publication except metadata-only patch versions.
- Public definitions can be shared across tenants; private definitions must carry tenant ownership.
- Isolation requirements are part of compatibility and cannot be weakened in a patch version.

### ConnectorConnection

A tenant-specific instance of a definition.

```yaml
id: conn_123
customer_id: cust_abc
definition_id: github_audit
definition_version: 1.0.0
name: prod-org-audit
status: enabled
controls:
  effective_status: enabled
  temporary_disabled_until: null
  rate_limit_override: null
  last_control_reason: null
isolation:
  level: shared
  worker_pool: default
  egress_policy: regional-shared
config:
  org: example
  region: global
  poll_interval_seconds: 300
secret_refs:
  api_token: secret://cust_abc/github-prod-token
destination:
  type: webhook
  endpoint_ref: dest_security_lake
created_at: 2026-05-01T00:00:00Z
updated_at: 2026-05-01T00:00:00Z
```

Rules:

- Secrets are never embedded in `config`; only references are stored.
- Multiple connections may reference the same definition.
- Connections can be enabled, disabled, paused, or quarantined independently.
- The requested isolation level must be equal to or stronger than the connector definition's minimum.
- A connection cannot reference destinations, secrets, or checkpoints outside its tenant.

### ConnectorRun

An execution instance created by the ingestion planner and executed by Niyanta's activity engine.

```yaml
id: run_123
connection_id: conn_123
customer_id: cust_abc
status: RUNNING
trigger: schedule
window:
  start: 2026-05-01T10:00:00Z
  end: 2026-05-01T10:05:00Z
checkpoint_in: {}
checkpoint_out: {}
metrics:
  records_read: 1000
  records_emitted: 998
  records_dropped: 2
```

Rules:

- Runs are idempotent for the same connection, trigger, and source window.
- Runs must be cancellable and checkpointable.
- A run may emit zero records and still be successful.

## Connector Kinds

## Tenant Isolation Contract

Connector definitions must declare their isolation requirements so the scheduler can place runs safely in dense deployments.

```yaml
isolation:
  minimum_level: shared
  allowed_levels:
    - shared
    - pooled
    - dedicated
  state_scope: tenant
  secret_scope: tenant
  network_scope: regional-shared
  max_connections_per_worker: 100
```

Rules:

- `shared` is the default for declarative connectors that do not need tenant-specific network access.
- `pooled` is used for higher-volume tenants, tier-specific pools, or segmented egress.
- `dedicated` is required for tenant-specific network paths, regulated data, high-risk plugins, or connectors with unsafe shared-state behavior.
- Workers must not receive raw secrets in assignment payloads; they receive secret references and resolve them through tenant-scoped providers.
- Checkpoints, dedupe keys, temporary files, and dead-letter records must be namespaced by tenant and connection.
- Metrics and logs include tenant identifiers but must not include event payloads or secrets.

### `poll_http`

Polls an HTTP API on a schedule.

Required sections:

- `auth`
- `request`
- `response`
- `checkpoint`
- `schedule`
- `destination`

Supported auth types:

- `none`
- `basic`
- `api_key`
- `oauth2_client_credentials`
- `oauth2_authorization_code`
- `jwt_token`
- `custom_plugin`

Request options:

- HTTP method: `GET`, `POST`
- headers, query parameters, request body template
- time window attributes
- retry count and timeout
- rate limit QPS
- rate limit extraction from response headers

Response options:

- format: `json`, `csv`, `xml`, `text`
- event extraction paths
- success status path/value
- gzip/deflate handling
- child-object-to-array conversion

Paging options:

- link header
- next page URL
- next page token
- persistent token
- offset
- count/page number

### `push_http`

Exposes a customer-scoped ingestion endpoint.

Required sections:

- endpoint route
- authentication and signing policy
- payload format
- validation schema
- transform pipeline
- destination

This is best for products that can send events directly and do not require Niyanta to manage polling credentials.

### `object_store`

Reads files or objects from storage.

Required sections:

- provider: `s3`, `gcs`, `azure_blob`, `filesystem`
- location selectors
- object claim strategy
- parser
- checkpoint: object key/version/etag plus byte offset when needed
- archival or processed-marker policy

### `stream_listener`

Receives syslog, CEF, raw TCP/UDP, or agent-forwarded streams.

Required sections:

- listener protocol
- bind policy or external agent config
- framing rules
- parser
- backpressure policy
- destination

This connector should support standard parsers for syslog and CEF, plus custom parser plugins for vendor-specific raw logs.

### `stream_subscription`

Consumes from broker-backed event streams where the source owns durable topics/subscriptions. This covers:

- Apache Kafka
- Confluent-compatible Kafka
- Azure Event Hubs Kafka endpoint or native Event Hubs consumer groups
- Google Cloud Pub/Sub subscriptions
- OCI Streaming

Required sections:

- provider: `kafka`, `event_hubs`, `gcp_pubsub`, `oci_streaming`
- subscription identity: topic/subscription/stream name and consumer group where applicable
- partition or shard discovery policy
- consumer group ownership mode
- starting position policy
- max batch size and max wait
- checkpoint mode
- parser
- transform pipeline
- destination

YAML example:

```yaml
kind: stream_subscription
provider: kafka
subscription:
  bootstrap_servers:
    - broker-1.example:9092
    - broker-2.example:9092
  topics:
    - audit.events
  consumer_group: niyanta-cust-abc-conn-123
  group_ownership: niyanta_managed
  start_position: committed_or_latest
  security:
    protocol: SASL_SSL
    sasl_mechanism: PLAIN
    username_ref: kafka_username
    password_ref: kafka_password
partitioning:
  discovery_interval_seconds: 60
  max_parallel_partitions: 16
  lease_seconds: 60
batch:
  max_records: 1000
  max_wait_seconds: 5
checkpoint:
  mode: offset_after_delivery
  commit_to_source: true
  commit_to_niyanta: true
parser:
  format: json
  events_path: $
destination:
  type: stream
  name: security_events
```

Provider mapping:

| Provider | Source unit | Consumer identity | Checkpoint value |
|----------|-------------|-------------------|------------------|
| Kafka | topic partition | consumer group | topic, partition, offset |
| Azure Event Hubs | event hub partition | consumer group | namespace, event hub, partition, offset/sequence number |
| GCP Pub/Sub | subscription stream | subscription | ack ID plus message ID/order key when available |
| OCI Streaming | stream partition | group cursor | stream OCID, partition, cursor/offset |

Rules:

- Use one logical connection per source subscription or consumer group.
- Niyanta should generate a default consumer group per customer connection unless the customer provides one.
- A partition/shard has one active Niyanta owner at a time, even when ingestion spans many worker processes.
- Offsets/cursors are committed only after destination delivery succeeds.
- Store offsets in Niyanta checkpoints even when committing to the source broker, so recovery and the CID ingestion view remain consistent.
- Rebalances are handled by Niyanta leases, not by trusting worker-local ownership alone.
- Backfills or replays use a separate consumer group or isolated checkpoint namespace.

Multi-process execution:

- The coordinator discovers source partitions/shards and creates one partition lease per source unit.
- Workers acquire partition assignments through scheduler commands, not by independently joining the same shared worker-local consumer group.
- Each worker process may own many partitions, but each partition is fenced by `connection_id`, `partition_id`, `lease_owner`, and `generation`.
- If a worker dies or stops heartbeating, the coordinator expires its partition leases and assigns those partitions to other worker processes.
- A resumed worker with an old generation must stop consuming and must not commit offsets.
- Scale-out is achieved by increasing worker count and raising `max_parallel_partitions`, bounded by tenant quota, source limits, and destination pressure.
- Scale-in gracefully drains partition leases: stop fetching, deliver buffered records, commit checkpoint, release lease.

Lease shape:

```yaml
connection_id: conn_123
partition_id: kafka:audit.events:7
lease_owner: worker-5
generation: 42
lease_expires_at: 2026-05-02T12:00:00Z
checkpoint:
  topic: audit.events
  partition: 7
  offset: 123456789
```

### `worker_plugin`

Runs custom code when declarative polling is insufficient.

Required sections:

- plugin package identity
- runtime requirements
- config schema
- secret refs
- checkpoint schema
- input/output contracts

This maps directly to the Niyanta activity plugin model.

## Declarative Connector Spec

### Auth

```yaml
type: api_key
secret_ref: api_token
placement: header
name: Authorization
prefix: Bearer
```

Requirements:

- Secret values are resolved only inside the worker.
- Auth refresh tokens are persisted as encrypted connector state.
- Auth failures should trip a degraded/quarantined state after a configurable threshold.

### Request

```yaml
method: GET
url: https://api.vendor.com/events
headers:
  Accept: application/json
query:
  from: "{{window.start}}"
  to: "{{window.end}}"
window:
  interval_seconds: 300
  lookback_seconds: 60
  format: rfc3339
rate_limit:
  qps: 5
  on_429: retry_after_header
retry:
  max_attempts: 3
  backoff: exponential
timeout_seconds: 30
```

### Response

```yaml
format: json
events_path: $.items
success:
  status_codes:
    - 200
  json_path: $.status
  equals: success
compression: gzip
```

### Paging

```yaml
type: next_page_token
token_path: $.next
request_parameter: cursor
placement: query
page_size: 500
page_size_parameter: limit
```

### Transform Pipeline

```yaml
- type: extract
  path: $
- type: map
  fields:
    event_time: $.created_at
    event_type: $.action
    actor.id: $.actor.id
    source.raw: $
- type: validate
  schema_ref: schemas/security_event.v1
- type: dedupe
  key: "{{event_type}}:{{event_time}}:{{actor.id}}"
```

Built-in transform stages:

- `extract`
- `parse_json`
- `parse_csv`
- `parse_xml`
- `parse_syslog`
- `parse_cef`
- `map`
- `rename`
- `coerce`
- `enrich`
- `filter`
- `dedupe`
- `validate`
- `preserve_raw`

### Destination

```yaml
type: stream
name: security_events
delivery:
  mode: at_least_once
  batch_size: 1000
  flush_interval_seconds: 10
```

Supported destinations:

- Niyanta event stream
- webhook
- object storage
- database sink
- message broker topic
- custom worker plugin sink

## Execution Semantics

Operational correctness rules for delivery guarantees, checkpoint commit order, dead letters, versioning, secret rotation, backfills, and blast-radius controls are defined in [OPERATIONS_AND_FAILURE_SEMANTICS.md](OPERATIONS_AND_FAILURE_SEMANTICS.md).

### Scheduling

Each enabled connection is reconciled by an ingestion supervisor. The planner creates `ConnectorRun` records and activity attempts according to connector policy, checkpoint state, source lag, durable timers, explicit backfills, push signals, and destination pressure. The coordinator assigns attempts to workers using connector capability, tenant quota, rate limit, affinity, and SLA policy.

Scheduling constraints:

- Do not run two mutable checkpoint runs for the same connection concurrently unless the connector definition declares partition-safe execution.
- Allow backfill runs when they use isolated checkpoint namespaces.
- Apply customer tier, rate limits, and queue depth from the existing Niyanta model.
- Prefer lagging connections over healthy idle connections within the same tenant quota.
- Respect source `Retry-After` and destination backpressure when planning future runs.

### Self-Orchestration

Each connection has a logical supervisor that can be implemented as a durable activity:

```go
func IngestionSupervisor(ctx activity.Context, conn ConnectorConnection) error {
    for {
        plan := ctx.RunChild("plan_ingestion", conn.ID)
        for _, partition := range plan.Partitions {
            ctx.RunChild("run_ingestion_partition", partition)
        }
        ctx.Sleep(plan.NextWakeAfter)
    }
}
```

Rules:

- The supervisor owns durable scheduling for a connection.
- Partition runs own source reads, transforms, delivery, and checkpoint commit.
- Config changes, backfill requests, and push events can wake a supervisor through signals.
- The planner may optimize away physical supervisor goroutines, but the durable semantics must remain the same.

### Adaptation

The runtime must feed observations back into planning:

- Empty windows can increase poll interval within policy bounds.
- Repeated lag can increase priority or split windows.
- Source throttling can reduce concurrency and QPS.
- Destination errors can pause checkpoint advancement.
- Auth failures can quarantine a connection.
- Parser drops can degrade health without halting ingestion when policy allows dead-lettering.

### Checkpointing

Checkpoint shape:

```yaml
version: 1
mode: time_window_with_cursor
last_successful_window_end: 2026-05-01T10:05:00Z
cursor: abc123
dedupe_watermark:
  ttl_seconds: 86400
  keys_ref: state://conn_123/dedupe/2026-05-01
```

Checkpoint rules:

- Commit checkpoint only after destination delivery succeeds and Niyanta is sure ingestion has completed for that batch/page/offset.
- For paginated APIs, checkpoint after each page when the destination supports idempotent writes.
- Keep prior checkpoint until the new checkpoint is committed.
- Version checkpoint schemas and provide migration hooks.

### Delivery Guarantees

Default guarantee is **at-least-once**. Exactly-once requires destination-side idempotency, transactional writes, or deterministic upserts.

Connector definitions must declare:

- source event ID path, if available
- deterministic dedupe key, if no event ID exists
- destination idempotency key behavior
- retryable and non-retryable error classes

## Health And Observability

Connector health is computed from run and data signals.

Statuses:

- `configured`: connection exists but has not run
- `connected`: recent successful run and data received when expected
- `idle`: successful run but no source data, within expected behavior
- `degraded`: transient failures, throttling, partial delivery, or lag
- `failing`: repeated auth/source/parser/destination failures
- `paused`: operator paused
- `disabled`: disabled by configuration
- `quarantined`: automatically stopped to protect system or source

Required metrics:

- `connector_runs_total`
- `connector_run_duration_seconds`
- `connector_records_read_total`
- `connector_records_emitted_total`
- `connector_records_dropped_total`
- `connector_delivery_failures_total`
- `connector_source_lag_seconds`
- `connector_checkpoint_age_seconds`
- `connector_rate_limit_backoffs_total`

Required logs:

- run started/completed/failed
- auth refresh failure
- source request failure
- parser failure sample with redacted payload
- delivery failure
- checkpoint commit failure

## Validation Rules

Every connector definition must include:

- config schema validation
- secret reference validation
- connectivity test
- sample payloads
- parser fixtures
- output schema assertions
- checkpoint round-trip tests
- dry-run mode that reads but does not commit checkpoint or deliver

Publication checklist:

- No embedded credentials.
- All secret fields use secret references.
- Schema is locked for the connector version.
- Sample events are scrubbed of PII.
- Transform preserves raw source data when allowed.
- Connector has sample queries or equivalent inspection recipes.
- Connector declares permissions/prerequisites.
- Connector declares source rate limits and retry policy.
- Connector declares spec version compatibility and checkpoint migration behavior.
- Connector declares dead-letter policy support.
- Connector declares whether secret rotation should trigger immediate connection testing.

## Storage Extensions

Niyanta should model connectors as first-class ingestion resources backed by the existing activity execution substrate.

```sql
CREATE TABLE connector_definitions (
    id                  VARCHAR(128) NOT NULL,
    version             VARCHAR(32) NOT NULL,
    kind                VARCHAR(64) NOT NULL,
    display_json        JSONB NOT NULL,
    spec_json           JSONB NOT NULL,
    status              VARCHAR(32) NOT NULL DEFAULT 'draft',
    created_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    PRIMARY KEY (id, version)
);

CREATE TABLE connector_connections (
    id                  VARCHAR(64) PRIMARY KEY,
    customer_id         VARCHAR(64) NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    definition_id       VARCHAR(128) NOT NULL,
    definition_version  VARCHAR(32) NOT NULL,
    name                VARCHAR(255) NOT NULL,
    status              VARCHAR(32) NOT NULL,
    config_json         JSONB NOT NULL,
    secret_refs_json    JSONB NOT NULL DEFAULT '{}',
    destination_json    JSONB NOT NULL,
    health_json         JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    FOREIGN KEY (definition_id, definition_version)
        REFERENCES connector_definitions(id, version)
);

CREATE TABLE connector_runs (
    id                  VARCHAR(64) PRIMARY KEY,
    connection_id       VARCHAR(64) NOT NULL REFERENCES connector_connections(id) ON DELETE CASCADE,
    customer_id         VARCHAR(64) NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    activity_id         VARCHAR(64) REFERENCES activities(id) ON DELETE SET NULL,
    trigger_type        VARCHAR(32) NOT NULL,
    status              activity_status NOT NULL,
    window_start        TIMESTAMP WITH TIME ZONE,
    window_end          TIMESTAMP WITH TIME ZONE,
    metrics_json        JSONB NOT NULL DEFAULT '{}',
    error_json          JSONB,
    started_at          TIMESTAMP WITH TIME ZONE,
    completed_at        TIMESTAMP WITH TIME ZONE,
    created_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE TABLE ingestion_events (
    id                  VARCHAR(64) PRIMARY KEY,
    customer_id         VARCHAR(64) NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    connection_id       VARCHAR(64) REFERENCES connector_connections(id) ON DELETE CASCADE,
    run_id              VARCHAR(64) REFERENCES connector_runs(id) ON DELETE SET NULL,
    severity            VARCHAR(32) NOT NULL,
    event_type          VARCHAR(128) NOT NULL,
    significant         BOOLEAN NOT NULL DEFAULT FALSE,
    message             TEXT NOT NULL,
    details_json        JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_ingestion_events_customer_time
    ON ingestion_events(customer_id, created_at DESC);

CREATE INDEX idx_ingestion_events_significant
    ON ingestion_events(customer_id, significant, created_at DESC)
    WHERE significant = TRUE;

CREATE TABLE ingestion_event_filters (
    id                  VARCHAR(64) PRIMARY KEY,
    customer_id         VARCHAR(64) NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    name                VARCHAR(255) NOT NULL,
    enabled             BOOLEAN NOT NULL DEFAULT TRUE,
    scope_json          JSONB NOT NULL DEFAULT '{}',
    match_json          JSONB NOT NULL,
    action_json         JSONB NOT NULL DEFAULT '{"mark_significant": true}',
    created_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_ingestion_event_filters_customer
    ON ingestion_event_filters(customer_id, enabled);

CREATE TABLE connection_controls (
    id                  VARCHAR(64) PRIMARY KEY,
    customer_id         VARCHAR(64) NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    connection_id       VARCHAR(64) NOT NULL REFERENCES connector_connections(id) ON DELETE CASCADE,
    control_type        VARCHAR(64) NOT NULL,
    config_json         JSONB NOT NULL DEFAULT '{}',
    reason              TEXT NOT NULL,
    created_by          VARCHAR(128) NOT NULL,
    expires_at          TIMESTAMP WITH TIME ZONE,
    cleared_at          TIMESTAMP WITH TIME ZONE,
    created_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_connection_controls_active
    ON connection_controls(customer_id, connection_id, control_type)
    WHERE cleared_at IS NULL;
```

`control_type` values:

- `rate_limit_override`
- `temporary_disable`
- `pause`
- `quarantine`

Example authored control config for `rate_limit_override`:

```yaml
qps: 0.25
burst: 1
max_parallel_partitions: 1
```

Example authored control config for `temporary_disable`:

```yaml
resume_strategy: auto_on_expiry
```

## Customer Ingestion View

The customer ingestion view is a tenant-scoped read model for operators and support tooling.

For a `customer_id`, it should include:

- connection counts by health state
- active controls by connection
- top lagging connections
- latest significant events
- latest failed runs
- source rate-limit/backoff state
- checkpoint age by connection
- per-connection records read/emitted/dropped over the selected window

Example response shape:

```yaml
customer_id: cust_abc
summary:
  connections_total: 42
  connected: 35
  degraded: 4
  failing: 2
  temporarily_disabled: 1
connections:
  - connection_id: conn_123
    name: prod-github-audit
    connector_id: github_audit
    health: degraded
    effective_status: rate_limited
    source_lag_seconds: 900
    checkpoint_age_seconds: 300
    active_controls:
      - control_type: rate_limit_override
        expires_at: 2026-05-01T18:00:00Z
        reason: source 429 spike
    latest_significant_event:
      event_type: source_rate_limited
      severity: warning
      message: Source returned repeated 429 responses
```

## API Extensions

Minimum coordinator APIs:

- `POST /connector-definitions`
- `GET /connector-definitions`
- `GET /connector-definitions/{id}/versions/{version}`
- `POST /connector-connections`
- `GET /connector-connections`
- `GET /connector-connections/{id}`
- `PATCH /connector-connections/{id}`
- `POST /connector-connections/{id}:test`
- `POST /connector-connections/{id}:run`
- `POST /connector-connections/{id}:pause`
- `POST /connector-connections/{id}:resume`
- `POST /connector-connections/{id}:rate-limit`
- `POST /connector-connections/{id}:temporary-disable`
- `POST /connector-connections/{id}:clear-control`
- `GET /connector-connections/{id}/controls`
- `GET /connector-connections/{id}/runs`
- `GET /connector-runs/{id}`
- `GET /customers/{customer_id}/ingestion-view`
- `POST /customers/{customer_id}/ingestion-event-filters`
- `GET /customers/{customer_id}/ingestion-event-filters`
- `PATCH /customers/{customer_id}/ingestion-event-filters/{filter_id}`
- `GET /customers/{customer_id}/ingestion-events`

## Worker Runtime Contract

Connector workers should implement the existing activity interface and add a connector adapter:

```go
type ConnectorExecutor interface {
    Validate(ctx context.Context, connection ConnectorConnection) error
    Test(ctx context.Context, connection ConnectorConnection) ConnectorTestResult
    Run(ctx context.Context, run ConnectorRun, checkpoint []byte, sink RecordSink) ([]byte, error)
}
```

The generic runtime should provide:

- secret resolution
- HTTP client with retries, timeouts, rate limiting, and redaction
- parser library
- transform engine
- checkpoint manager
- destination sink adapters
- metrics/logging wrappers

Custom connector plugins should only implement behavior that cannot be expressed declaratively.

## Recommended Implementation Phases

### Phase 1: Declarative HTTP Poller

- Connector definitions and connections.
- API key/basic/OAuth2 client credentials.
- JSON response extraction.
- Time window checkpointing.
- At-least-once delivery to a Niyanta stream.
- Basic health and dry-run testing.
- At-least-once delivery semantics, dead-letter sink support, connector spec versioning, and secret-rotation wakeups.

### Phase 2: Robust Source Handling

- Pagination strategies.
- CSV/XML parsing.
- Rate limit header extraction.
- Dedupe keys.
- Backfill runs.
- Connection test API.

### Phase 3: Additional Connector Kinds

- Push HTTP ingestion.
- Object storage ingestion.
- Syslog/CEF stream listener.
- Worker plugin connectors.

### Phase 4: Marketplace/Packaging

- Definition signing.
- Version compatibility checks.
- UI metadata rendering.
- Sample queries and dashboards.
- Connector publication workflow.
