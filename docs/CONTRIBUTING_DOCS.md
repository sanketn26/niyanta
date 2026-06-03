# Contributing to the Docs

**Status**: Draft

The docs drifted once (a workload-era data model and API survived an architecture reframe). This guide exists to keep that from recurring. It defines where things go, the vocabulary, and the checks to run before committing doc changes.

## The one rule

**The platform is a composable activity runner. Apps are compositions of activities.** Keep them separate:

- Engine concerns → `docs/platform/` (activities, attempts, call log, signals, workers, scheduling, leases, health). Engine-agnostic to what an activity *does*.
- App concerns → `docs/apps/<app>/` (connector definitions, runbooks, ingestion planning, diagnosis). Each app owns its config language and state.

If you're about to add a connector field or a runbook concept to a `platform/` doc, stop — it belongs in `apps/`.

## Layout

```
docs/
  README.md              index — update when adding/moving a doc or app
  GETTING_STARTED.md
  CONTRIBUTING_DOCS.md   (this file)
  IMPLEMENTATION_PLAN.md cross-cutting roadmap
  platform/              engine: ARCHITECTURE, DATA_MODELS, API_SPEC, adr/, impl/
  apps/<app>/            ARCHITECTURE, *_SPEC, API_SPEC, OPERATIONS..., impl/, schemas/, fixtures/
api/openapi/             machine-readable API stubs (engine + per app)
```

When you add an app, mirror the dataconnector/llmops shape: an architecture doc, a spec, an API spec, operations/failure semantics, `impl/` phases + README, `schemas/`. Add it to [README.md](README.md) and to the platform ARCHITECTURE §Activity Compositions list.

## Vocabulary (canonical)

| Use | Not |
|-----|-----|
| activity, activity type, attempt, child activity | workload, workload type, job |
| call log (`activity_calls`), replay, suspend/resume | manual checkpoint loop |
| `activity_attempts` (per-attempt rows) | `retry_count` integer |
| `input_json` / opaque payload | workload config in the engine |
| composition / cookbook | "app layer" / "app SPI" / "app runtime" |

The canonical activity model is [platform/adr/ADR-005-activity-execution-model.md](platform/adr/ADR-005-activity-execution-model.md). The canonical schema is [platform/DATA_MODELS.md](platform/DATA_MODELS.md) (v2). The v1→v2 map is the appendix there.

## Phase numbering

There are two numbering schemes — don't conflate them:

- **System delivery phases** 0–6 in [IMPLEMENTATION_PLAN.md](IMPLEMENTATION_PLAN.md) (platform 0–4, dataconnector 5–6, llmops 1–3 within its own track).
- **Engine capability milestones** M1–M4 in ADR-005, mapped to system phases in the ADR's cross-walk table.

When you change phasing, update the ADR cross-walk and the plan together. The plan's phase numbers win.

## Config languages are the apps', not the engine's

The connector YAML and LLMOps runbook/policy languages are **not Niyanta config**. The engine never parses them. Document them under the owning app; never add them to platform docs as engine features.

## Source of truth for contracts

- API shapes: the OpenAPI files in `api/openapi/` are the source of truth; the `*_SPEC.md` / `API_SPEC.md` prose explains them. Keep them in sync — if you add an endpoint to a doc, add it to the matching OpenAPI stub.
- Authored artifacts (connector definitions, runbooks, checkpoints, findings): the JSON Schemas in `docs/apps/*/schemas/` are enforceable; YAML examples in prose must validate against them.
- Redaction: [apps/llmops/REDACTION_CONTRACT.md](apps/llmops/REDACTION_CONTRACT.md) + its `fixtures/`.

## Before you commit: checks

```bash
# 1. All relative markdown links resolve
#    (fails loudly if any [text](path.md) target is missing)
while IFS= read -r src; do
  dir=$(dirname "$src")
  grep -oE '\]\([^)]*\.md[^)]*\)' "$src" | sed -E 's/^\]\(//; s/\)$//' | while read -r link; do
    case "$link" in http*) continue;; esac
    target="${link%%#*}"; [ -z "$target" ] && continue
    [ -f "$(cd "$dir" && readlink -f "$target" 2>/dev/null)" ] || echo "BROKEN: $src -> $link"
  done
done < <(git ls-files '*.md')

# 2. OpenAPI + JSON Schema parse
for f in api/openapi/*.yaml; do python3 -c "import yaml; yaml.safe_load(open('$f'))"; done
for f in docs/apps/*/schemas/*.json; do python3 -c "import json; json.load(open('$f'))"; done

# 3. No workload-era vocabulary crept into platform/ (allow only the migration map / ADR history)
grep -rniE 'workload' docs/platform/ | grep -viE 'v1|supersed|migration|original|Workload interface' && echo "review: stray workload terms" || echo "vocab ok"
```

A doc PR that adds an endpoint, artifact, or app without updating the matching OpenAPI/schema and the index is incomplete.
