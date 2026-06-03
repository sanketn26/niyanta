# General Purpose Data Connector Specification

**Version**: 0.2
**Last Updated**: 2026-05-03
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

| Pattern | Source or destination shape | Runtime behavior | Niyanta equivalent |
|---------|-----------------------------|------------------|--------------------|
| API push | External product can send events to the platform | Expose an ingestion endpoint and validate signed requests | `push_http` connector |
| Native SaaS polling | Platform calls a remote API on a schedule | Declarative auth, request, response, paging, and DCR/destination config | `poll_http` connector |
| Function/worker polling | Custom code polls a source API | Run a packaged connector worker with code hooks | `worker_plugin` connector |
| Database snapshot or query | Source exposes SQL/JDBC-like access | Discover tables, plan snapshots or cursor queries, checkpoint per stream/table | `database_source` connector |
| Database CDC | Source exposes replication logs such as WAL/binlog/redo logs | Claim replication slots or subscriptions, read ordered changes, checkpoint LSN/SCN/binlog offsets | `cdc_source` connector |
| Blob/object ingestion | Files are written to object storage | Scan, claim, parse, checkpoint file offsets or object versions | `object_store` connector |
| Managed stream subscription | Source exposes Kafka-compatible topics, Event Hubs, Pub/Sub, Kinesis, Pulsar, RabbitMQ, or OCI Streaming | Claim partitions/subscriptions, consume batches, commit offsets only after delivery | `stream_subscription` connector |
| CEF/syslog stream | Source emits standard text events | Agent or listener receives lines, parses, normalizes, forwards | `stream_listener` connector |
| Manual/import utility | Operator uploads or replays data | Batch parse and emit records | `batch_import` connector |
| Destination writer | Platform writes records into warehouses, lakes, queues, APIs, or vector stores | Validate destination config, create or evolve target schema when allowed, batch writes, checkpoint destination commits | `destination_connector` |

Design implication: connector definitions must describe the data movement strategy explicitly, while connector runs should share the same lifecycle, checkpoint, retry, health, and delivery machinery. Source connectors produce records; destination connectors consume records. Niyanta's ingestion-first product may expose destinations as sink adapters, but the spec must also be able to model Airbyte-style destinations as independently versioned connectors when the destination behavior is complex.

Airbyte-style connector repositories commonly contain:

- a connector metadata file with connector type, subtype, Docker image, release stage, documentation, resource requirements, allowed hosts, support level, and test-suite metadata
- a declarative manifest for low-code HTTP/API sources
- optional custom Python, Java, or other runtime code
- configuration schema, connection tests, discovery/catalog behavior, stream schemas, state/checkpoint migration rules, and acceptance/integration tests

Niyanta definitions should be able to import or translate these fields without requiring the Airbyte runtime protocol to be the platform's native protocol.

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
- Database positions: table cursor, primary-key range, snapshot chunk, WAL LSN, binlog file/position, GTID set, SCN, resume token.
- Deduplication watermarks: event IDs or source hashes.
- Catalog versions: selected stream schemas, cursor fields, primary keys, and sync modes used by a run.

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
direction: source
isolation:
  minimum_level: shared
  requires_dedicated_network: false
  allows_shared_workers: true
packaging:
  mode: declarative
  runtime: niyanta
  image: null
  entrypoint: null
  allowed_hosts:
    - api.github.com
  resource_requirements:
    default:
      cpu_request: 100m
      memory_request: 256Mi
compatibility:
  protocol: niyanta
  protocol_version: 0.1
  imported_from: null
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
- `direction` is `source`, `destination`, or `source_and_destination`.
- Public definitions can be shared across tenants; private definitions must carry tenant ownership.
- Isolation requirements are part of compatibility and cannot be weakened in a patch version.
- `packaging` describes how executable behavior is delivered, including declarative manifests, native Niyanta plugins, Docker/OCI images, Java archives, Python packages, or external command protocols.
- `compatibility` declares whether the connector uses the native Niyanta runtime contract, an Airbyte-compatible command contract, or a custom adapter.

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

### `database_source`

Reads from operational databases through SQL, JDBC/ODBC-like drivers, or native database protocols.

This covers snapshot and cursor-based extraction from:

- PostgreSQL
- MySQL and MariaDB
- Microsoft SQL Server
- Oracle
- Snowflake, BigQuery, Redshift, Databricks, ClickHouse, DuckDB, and other analytical databases
- MongoDB, Cassandra, DynamoDB, Elasticsearch, and similar database-shaped sources when represented through a custom adapter

Required sections:

- provider and driver identity
- connection and network policy
- credential and TLS/SSH tunnel references
- schema discovery policy
- stream/table selection policy
- snapshot mode and chunking policy
- incremental cursor policy when CDC is not used
- checkpoint: per stream/table cursor, primary key boundary, page token, or completed snapshot chunk
- type mapping and schema evolution policy
- destination

YAML example:

```yaml
kind: database_source
provider: postgres
driver:
  type: native
  package: niyanta-postgres
connection:
  host_ref: db_host
  port: 5432
  database: app
  username_ref: db_username
  password_ref: db_password
  tls:
    mode: require
discovery:
  include_schemas:
    - public
  exclude_tables:
    - public.audit_temp
streams:
  - name: public.orders
    primary_key:
      - id
    cursor_field: updated_at
    sync_mode: incremental_append
snapshot:
  chunking:
    mode: primary_key_range
    target_rows: 50000
checkpoint:
  mode: per_stream_cursor
destination:
  type: stream
  name: database_changes
```

Rules:

- Discovery must be repeatable and versioned so a connection can explain why a stream was added, removed, or changed.
- Snapshot chunk checkpoints must be independent per stream/table.
- Incremental cursor reads must define cursor ordering, null handling, and tie-breaking with primary keys or deterministic synthetic keys.
- Schema evolution must declare whether new columns are auto-added, ignored, quarantined, or require operator approval.
- Network isolation defaults to `pooled` or `dedicated` when tenant-specific database access, private connectivity, SSH tunneling, or static egress IPs are required.

### `cdc_source`

Reads ordered database changes from replication logs.

This covers:

- PostgreSQL logical replication and WAL LSNs
- MySQL/MariaDB binlogs and GTIDs
- SQL Server CDC/Change Tracking or transaction log readers
- Oracle redo/SCN based capture
- MongoDB change streams
- database-specific managed CDC APIs

Required sections:

- provider and replication mechanism
- replication slot/subscription or change stream identity
- initial snapshot policy
- publication/table selection
- ordering and transaction boundary policy
- checkpoint: LSN, SCN, binlog file/position, GTID set, resume token, or provider cursor
- schema history storage policy
- heartbeat and lag policy
- destination

YAML example:

```yaml
kind: cdc_source
provider: postgres
replication:
  mechanism: logical_replication
  slot_name: niyanta_conn_123
  publication: niyanta_publication
  plugin: pgoutput
initial_snapshot:
  mode: consistent_snapshot_then_cdc
checkpoint:
  mode: lsn_after_delivery
  value:
    lsn: "0/16B6C50"
schema_history:
  storage: niyanta_state
  migration_policy: versioned
lag:
  heartbeat_interval_seconds: 30
destination:
  type: stream
  name: database_cdc
```

Rules:

- CDC checkpoints must advance only after all records up to the checkpoint are delivered.
- Initial snapshots and live CDC must use isolated checkpoint namespaces unless the connector proves a consistent handoff point.
- Transaction ordering must be preserved within a source partition or replication stream.
- DDL/schema-change handling must be explicit: apply, ignore, quarantine, or fail.
- Replication slots, publications, and source-side resources must be cleaned up only by an explicit lifecycle policy.

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
- Amazon Kinesis
- Apache Pulsar
- RabbitMQ streams or queues
- OCI Streaming

Required sections:

- provider: `kafka`, `event_hubs`, `gcp_pubsub`, `kinesis`, `pulsar`, `rabbitmq`, `oci_streaming`
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
| Amazon Kinesis | stream shard | consumer ARN/application name | stream, shard, sequence number |
| Apache Pulsar | topic partition | subscription | topic, partition, message ID |
| RabbitMQ | queue or stream partition | consumer tag/group where available | queue/stream, delivery tag, offset when stream-backed |
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

### `destination_connector`

Writes records into external systems when a simple sink adapter is insufficient.

This covers:

- databases and warehouses such as Postgres, MySQL, BigQuery, Snowflake, Redshift, Databricks, ClickHouse, DuckDB, Cassandra, MongoDB, and Elasticsearch
- object and lake storage such as S3, GCS, Azure Blob, R2, Iceberg, Delta, or data lake layouts
- queues and streams such as Kafka, Kinesis, Pub/Sub, Pulsar, RabbitMQ, SQS, and MQTT
- search, vector, and application destinations such as OpenSearch, Qdrant, Pinecone, Weaviate, Milvus, Chroma, Typesense, HubSpot, Customer.io, or webhook APIs

Required sections:

- destination provider and write protocol
- config schema and secret refs
- accepted input schema or stream contract
- write mode: append, overwrite, upsert, append_dedupe, exactly_once_when_supported
- batching, staging, and commit policy
- schema creation/evolution policy
- idempotency key or transactional commit behavior
- checkpoint/commit acknowledgement shape
- dead-letter behavior

YAML example:

```yaml
kind: destination_connector
direction: destination
provider: s3
format:
  type: parquet
  compression: zstd
layout:
  bucket_ref: dest_bucket
  prefix: raw/{{connection_id}}/{{stream_name}}/
  partitioning:
    - field: event_date
      transform: date
write:
  mode: append
  batch_size: 100000
  staging: niyanta_managed
  commit: atomic_manifest
checkpoint:
  mode: destination_commit_receipt
```

Rules:

- Destination connectors may be public, tenant-private, or connection-private.
- A source run may checkpoint only after the destination connector returns a durable commit receipt.
- Destination schema evolution must declare how target tables, indexes, object layouts, or vector collections are created and modified.
- Upsert and exactly-once modes require deterministic keys, transaction support, or destination-side idempotency.
- Destination connectors must support dry-run validation that does not mutate the target unless explicitly configured.

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

## Packaging And Compatibility

Connector definitions must separate source/destination semantics from executable packaging. This allows the same domain model to represent native Niyanta connectors, declarative manifests, and Airbyte-style packaged connectors.

Supported packaging modes:

- `declarative`: YAML manifest interpreted by the Niyanta generic runtime.
- `native_plugin`: compiled or interpreted plugin loaded through the Niyanta activity plugin model.
- `oci_image`: container image executed through a controlled command adapter.
- `external_command`: executable command with a declared protocol and sandbox policy.
- `airbyte_manifest`: Airbyte low-code/declarative source manifest translated or interpreted by an adapter.
- `airbyte_image`: Airbyte-compatible Docker image using Airbyte's command protocol.

Packaging shape:

```yaml
packaging:
  mode: airbyte_image
  runtime: docker
  image: airbyte/source-hubspot:6.3.5
  digest: sha256:example
  entrypoint:
    - /airbyte/integration_code/main
  command_protocol: airbyte
  command_protocol_version: v0
  allowed_hosts:
    - api.hubapi.com
  resource_requirements:
    check:
      memory_request: 1600Mi
      memory_limit: 1600Mi
    discover:
      memory_request: 1024Mi
      memory_limit: 1024Mi
    read:
      cpu_request: 500m
      memory_request: 1024Mi
  release:
    stage: generally_available
    support_level: certified
    license: ELv2
    documentation_url: https://docs.airbyte.com/integrations/sources/hubspot
```

Compatibility rules:

- Native Niyanta connectors should prefer `declarative` or `native_plugin`.
- Airbyte low-code manifests may be translated to `poll_http` when they only use supported declarative features.
- Airbyte connectors with custom components, custom error handlers, dynamic schema loaders, state migrations, or non-HTTP behavior should use `worker_plugin` or `airbyte_image` until a native translation exists.
- OCI and external command packages must declare allowed hosts, resource requirements, secret access scope, temporary filesystem policy, and network isolation.
- Imported connector metadata must preserve upstream version, image tag/digest, documentation URL, release stage, support level, breaking-change notes, and test-suite references.
- Niyanta remains responsible for tenant isolation, secret resolution, scheduling, destination commit ordering, metrics, logs, and checkpoint durability even when execution is delegated to an adapter.

## Airbyte-Compatible Runtime Mapping

Niyanta's native runtime contract is the source of truth, but the spec can model Airbyte-style connectors through an adapter layer.

Airbyte command mapping:

| Airbyte command | Niyanta equivalent | Notes |
|-----------------|--------------------|-------|
| `spec` | `GetSpec` / connector definition import | Produces configuration schema, auth fields, and documentation metadata. |
| `check` | `Test` | Validates credentials, network reachability, and required permissions. |
| `discover` | `Discover` | Produces stream catalog, schemas, primary keys, cursor fields, and supported sync modes. |
| `read` | `Run` | Emits records and state/checkpoint messages for selected streams. |
| state messages | checkpoint manager | State is normalized into Niyanta's versioned checkpoint model. |
| trace/log messages | ingestion events and logs | Secrets and payload samples are redacted according to Niyanta policy. |

Catalog model:

```yaml
catalog:
  streams:
    - name: contacts
      namespace: hubspot
      source_schema_ref: schemas/hubspot.contacts.v1
      primary_key:
        - id
      cursor_field:
        - updatedAt
      supported_sync_modes:
        - full_refresh
        - incremental
      default_sync_mode: incremental
      destination_sync_modes:
        - append
        - append_dedupe
```

Rules:

- Discovery output is stored as a versioned catalog snapshot on the connection.
- A run references a selected catalog version, not a mutable discovery result.
- Stream-level state must be preserved when translating Airbyte state into Niyanta checkpoints.
- Per-stream failures may degrade only the affected stream when the connector declares stream-isolated execution.
- Airbyte `full_refresh`, `incremental`, `append`, `overwrite`, and dedupe modes must map to explicit Niyanta run and destination policies before a connection is enabled.

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
- Multi-stream connectors must store state per stream/table/topic when streams can advance independently.
- Database and CDC connectors must preserve source ordering guarantees in their checkpoint shape.
- Imported Airbyte state must be normalized into Niyanta checkpoints without losing stream-level state, cursor precision, or state migration metadata.

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
- discovery/catalog behavior when the source or destination schema is not fully static
- sample payloads
- parser fixtures
- selected catalog fixtures for multi-stream connectors
- output schema assertions
- checkpoint round-trip tests
- checkpoint migration tests when state shape changes
- destination dry-run or write validation when `direction` includes `destination`
- dry-run mode that reads but does not commit checkpoint or deliver

Publication checklist:

- No embedded credentials.
- All secret fields use secret references.
- Schema is locked for the connector version.
- Direction is declared as `source`, `destination`, or `source_and_destination`.
- Packaging mode, runtime, allowed hosts, resource requirements, and compatibility protocol are declared.
- Sample events are scrubbed of PII.
- Transform preserves raw source data when allowed.
- Connector has sample queries or equivalent inspection recipes.
- Connector declares permissions/prerequisites.
- Connector declares source rate limits and retry policy.
- Database connectors declare discovery, schema evolution, cursor, snapshot, and CDC policies where applicable.
- Destination connectors declare write mode, schema evolution, idempotency, and commit receipt behavior.
- Connector declares spec version compatibility and checkpoint migration behavior.
- Connector declares dead-letter policy support.
- Connector declares whether secret rotation should trigger immediate connection testing.
- Imported Airbyte-style connectors preserve upstream metadata, image tag/digest, release stage, support level, breaking-change notes, and test-suite references.

## Storage Extensions

Niyanta should model connectors as first-class ingestion resources backed by the existing activity execution substrate.

```sql
CREATE TABLE connector_definitions (
    id                  VARCHAR(128) NOT NULL,
    version             VARCHAR(32) NOT NULL,
    kind                VARCHAR(64) NOT NULL,
    direction           VARCHAR(32) NOT NULL DEFAULT 'source',
    display_json        JSONB NOT NULL,
    packaging_json      JSONB NOT NULL DEFAULT '{}',
    compatibility_json  JSONB NOT NULL DEFAULT '{}',
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
    catalog_json        JSONB NOT NULL DEFAULT '{}',
    catalog_version     VARCHAR(64),
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

The full Data Connector API — request/response bodies, auth, pagination, errors, and idempotency — is specified in [API_SPEC.md](API_SPEC.md). It is the app's own surface, layered on the engine's activity API ([../../platform/API_SPEC.md](../../platform/API_SPEC.md)), and covers connector definitions, connections, test/discover/catalog, runs/backfills, controls, the customer ingestion view, and significant-event filters.

## Worker Runtime Contract

Connector workers should implement the existing activity interface and add a connector adapter:

```go
type ConnectorExecutor interface {
    GetSpec(ctx context.Context, definition ConnectorDefinition) (ConnectorSpec, error)
    Validate(ctx context.Context, connection ConnectorConnection) error
    Test(ctx context.Context, connection ConnectorConnection) ConnectorTestResult
    Discover(ctx context.Context, connection ConnectorConnection) (ConnectorCatalog, error)
    Run(ctx context.Context, run ConnectorRun, catalog ConnectorCatalog, checkpoint []byte, sink RecordSink) ([]byte, error)
}
```

The generic runtime should provide:

- secret resolution
- HTTP client with retries, timeouts, rate limiting, and redaction
- database clients, CDC readers, and stream clients where the connector kind is native
- parser library
- transform engine
- schema discovery and catalog snapshot storage
- checkpoint manager
- destination sink adapters
- destination connector commit receipts
- metrics/logging wrappers

Custom connector plugins should only implement behavior that cannot be expressed declaratively.

Airbyte-compatible adapters should implement the same interface by translating `spec`, `check`, `discover`, `read`, record, state, log, and trace messages into Niyanta's native models.

## Recommended Implementation Phases

### Phase 1: Connector Foundation And Declarative HTTP Poller

- Connector definitions and connections with `direction`, `packaging`, and `compatibility` metadata.
- `GetSpec`, `Test`, `Discover`, selected-catalog storage, and catalog-versioned runs.
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

### Phase 3: Packaging And Compatibility Foundation

- Import API for external connector metadata.
- Airbyte manifest/image metadata preservation.
- Airbyte-compatible adapter skeleton for `spec`, `check`, `discover`, `read`, state, log, and trace messages.
- OCI image packaging model, resource limits, allowed hosts, and runtime sandbox policy.
- Selected-catalog diff, approval, and audit lifecycle.

### Phase 4: Additional Connector Kinds

- Push HTTP ingestion.
- Object storage ingestion.
- Syslog/CEF stream listener.
- Worker plugin connectors.
- Database snapshot and cursor-based sources.
- CDC sources with ordered checkpointing.
- Destination connectors for complex warehouse, lake, queue, API, and vector-store writes.
- Production Airbyte image execution for selected connectors.

### Phase 5: Marketplace And Publication

- Definition signing.
- Version compatibility checks.
- UI metadata rendering.
- Sample queries and dashboards.
- Connector publication workflow.
