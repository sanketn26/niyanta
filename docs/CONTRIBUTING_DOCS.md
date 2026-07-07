# Contributing to the Docs

**Status**: Draft

The docs drifted once (a workload-era data model and API survived an architecture reframe). This guide exists to keep that from recurring. It defines where things go, the vocabulary, and the checks to run before committing doc changes.

## The one rule

**Niyanta is a durable activity execution engine and nothing else.** Every doc in this repo describes the engine: activities, attempts, call log, signals, workers, scheduling, leases, fencing, versioning, the console, and the observability of all of it. The engine is agnostic to what an activity *does*.

If you're about to document a domain concept — a connector field, a pipeline stage, a runbook, anything that gives an activity's payload *meaning* — stop. That belongs to whatever product is built on the engine, documented wherever that product lives, not here. To the engine (and to these docs), such things are opaque `input_json`.

## Layout

```
docs/
  README.md              index — update when adding/moving a doc
  GETTING_STARTED.md
  CONTRIBUTING_DOCS.md   (this file)
  IMPLEMENTATION_PLAN.md roadmap (engine-only, phases 0–7)
  platform/              ARCHITECTURE, DATA_MODELS, API_SPEC, adr/, impl/
api/openapi/             machine-readable engine API (niyanta-v1.yaml)
```

## Vocabulary (canonical)

| Use | Not |
|-----|-----|
| activity, activity type, attempt, child activity | workload, workload type, job |
| call log (`activity_calls`), replay, suspend/resume | manual checkpoint loop |
| `activity_attempts` (per-attempt rows) | `retry_count` integer |
| `input_json` / opaque payload | domain config in the engine |
| execution (one replay history; `ContinueAsNew` starts a new one) | — |

The canonical activity model is [platform/adr/ADR-005-activity-execution-model.md](platform/adr/ADR-005-activity-execution-model.md). The canonical schema is [platform/DATA_MODELS.md](platform/DATA_MODELS.md). The v1→v2 migration map is the appendix there.

## Phase numbering

There are two numbering schemes — don't conflate them:

- **Delivery phases** 0–7 in [IMPLEMENTATION_PLAN.md](IMPLEMENTATION_PLAN.md), each with a guide in [platform/impl/](platform/impl/README.md).
- **Engine capability milestones** M1–M4 in ADR-005, mapped to delivery phases in the ADR's cross-walk table.

When you change phasing, update the ADR cross-walk, the plan, and the impl README together. The plan's phase numbers win.

## Source of truth for contracts

- API shapes: [api/openapi/niyanta-v1.yaml](../api/openapi/niyanta-v1.yaml) is the source of truth; [platform/API_SPEC.md](platform/API_SPEC.md) prose explains it. If you add an endpoint to one, add it to the other in the same change.
- Execution-model guarantees (exactly-once dispatch, fencing, determinism, `ContinueAsNew`): ADR-005 states them; DATA_MODELS encodes the invariants; the impl phase guides carry the acceptance tests. All three move together.

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

# 2. OpenAPI parses (ruby ships with macOS; PyYAML is not in the Python stdlib)
ruby -ryaml -e "YAML.load_file('api/openapi/niyanta-v1.yaml')"

# 3. No workload-era vocabulary crept into platform/ (allow only the migration map / ADR history)
grep -rniE 'workload' docs/platform/ | grep -viE 'v1|supersed|migration|original|Workload interface' && echo "review: stray workload terms" || echo "vocab ok"

# 4. No domain/product concepts crept back into the engine docs
grep -rniE 'connector|runbook|ingestion|llmops' docs/platform/ docs/*.md | grep -viE 'agnostic|opaque|does not|never|no longer|removed' && echo "review: possible domain leakage" || echo "boundary ok"
```

A doc PR that adds an endpoint without updating the matching OpenAPI, or changes a guarantee without updating ADR-005 + DATA_MODELS + the impl guide, is incomplete.
