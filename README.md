# DCG Protocol

Protocol specification and JSON schemas for **Domain Context Graph (DCG)** — a protocol for structuring, versioning, and sharing domain knowledge so that AI systems can operate on verified context.

## Contents

```
spec/
  rfc.md              RFC DCG-001 — full protocol specification
  design.md           Design companion document

schema/
  graph.schema.json   Entry point manifest (graph.json) — project metadata + graph index + inline data
  subgraph.schema.json  Linked subgraph files — standalone entity/relation/layer stores
  stack.schema.json   Stack composition manifest (dcg-stack.yml) — multi-project layering
```

## Schemas

### `graph.schema.json` — Domain Project Entry Point

Validates `graph.json`, the root file of every DCG domain project. It contains:

- **`dcg_project`** — project metadata (name, version, description)
- **`graphs`** — index linking to all graph files in the project
- **`entities`** — main graph entities (Wikibase JSON-compatible, keyed by content-addressed UID)
- **`relations`** — typed directed relationships between entities
- **`layers`** — ordered commit history (append-only, supports compaction)

### `subgraph.schema.json` — Linked Subgraph Files

Validates standalone graph files referenced from `graph.json`'s `graphs` index (e.g., `graphs/appsec.json`). Contains only graph data — no project metadata.

### `stack.schema.json` — Composition Manifest

Validates `dcg-stack.yml` (parsed as JSON). Defines how multiple DCG domain projects compose into a layered stack with cross-layer entity resolution.

## Validation

```bash
# Validate a graph.json against the schema
pip install check-jsonschema
check-jsonschema --schemafile schema/graph.schema.json path/to/graph.json

# Validate a subgraph file
check-jsonschema --schemafile schema/subgraph.schema.json path/to/graphs/appsec.json

# Validate a stack manifest (convert YAML to JSON first)
python3 -c "import yaml,json,sys; json.dump(yaml.safe_load(open(sys.argv[1])),sys.stdout)" dcg-stack.yml | \
  check-jsonschema --schemafile schema/stack.schema.json -
```

## Reference Implementation

The reference implementation is [domain-context-graph](https://github.com/agentkg/domain-context-graph) (`dcg` package).

## License

Apache-2.0
