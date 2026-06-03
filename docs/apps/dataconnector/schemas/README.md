# Data Connector JSON Schemas

Enforceable [JSON Schema](https://json-schema.org/) (draft 2020-12) for the Data Connector app's authored artifacts and request bodies. These make the YAML examples in [../DATA_CONNECTOR_SPEC.md](../DATA_CONNECTOR_SPEC.md) and [../API_SPEC.md](../API_SPEC.md) validatable rather than illustrative.

| Schema | Validates |
|--------|-----------|
| [connector-definition.schema.json](connector-definition.schema.json) | A `ConnectorDefinition` (the reusable template) |
| [connector-connection.schema.json](connector-connection.schema.json) | A `ConnectorConnection` create/update body — secrets must be references |
| [checkpoint.schema.json](checkpoint.schema.json) | A connector checkpoint blob |

Authored connector definitions/connections are YAML; convert to JSON (or use a YAML-aware validator) before validating. The Data Connector composition validates against these on `POST`/`PATCH` and rejects with `400 INVALID_INPUT` plus the offending JSON path.

Planned (stubs to add as kinds land): per-kind config schemas (`poll_http`, `database_source`, `cdc_source`, `stream_subscription`, `destination_connector`), selected-catalog schema, and control-config schemas.

These are the **app's** schemas. The engine validates only its own activity submission shape (see `api/openapi/niyanta-v1.yaml`); a connector definition is an opaque payload to the engine.
