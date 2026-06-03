# LLMOps JSON Schemas

Enforceable [JSON Schema](https://json-schema.org/) (draft 2020-12) for the LLMOps app's authored artifacts and model outputs.

| Schema | Validates |
|--------|-----------|
| [runbook.schema.json](runbook.schema.json) | A runbook (`when` → `diagnose` → `respond`). Validated on `POST`/`PUT`; rejected with `400 INVALID_INPUT` + offending path. |
| [finding.schema.json](finding.schema.json) | A `Finding`. The model's structured output **must** validate against this or the diagnosis attempt fails and retries — there are no malformed findings. |

The `finding.schema.json` is also the contract for structured-output parsing: it doubles as the response schema handed to the model and the validator applied to its output. `auto_actionable` is intentionally not derived from the model — the guardrail evaluator sets it.

Planned: `playbook.schema.json`, `policy.schema.json`.

These are the **app's** schemas. The engine validates only the activity submission shape (`api/openapi/niyanta-v1.yaml`); a runbook is an opaque payload to the engine.
