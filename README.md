# Domain Context Graph (DCG) Protocol

A protocol for structuring, versioning, and sharing domain knowledge so that AI systems and human developers can operate on verified context rather than statistical inference.

DCG provides a standard, tool-portable format for capturing how real-world domains (Payments, Security, Infrastructure) relate to each other and to the artifacts (code, APIs, services) that implement them. The format is readable by both humans and language models without specialized tooling.

**RFC:** [DCG-002](spec/rfc.md) | **Schemas:** [`graph_card`](schema/graph_card.schema.json) | [`graph_data`](schema/graph_data.schema.json) | [`stack`](schema/stack.schema.json)

## Why DCG?

LLMs hallucinate domain relationships. Static analysis captures code structure but not business meaning. DCG bridges the gap: a compact, JSON-based knowledge graph that captures both structural facts (this function calls that function) and domain semantics (this function is part of the Payments domain and handles PCI compliance).

DCG graphs are:
- **Local-first** -- plain JSON files in your repo, no database required
- **Layered** -- atomic versioning layers track knowledge evolution over time
- **Composable** -- stack multiple domain projects into a unified knowledge graph
- **AI-native** -- English property keys and compact attributes minimize tokens per entity

## Repository Contents

```
spec/
  rfc.md                 RFC DCG-002 -- full protocol specification (146 requirements)
  design.md              Design companion document

schema/
  graph_card.schema.json   graph_card.json metadata manifest
  graph_data.schema.json   Graph data files under graphs/
  stack.schema.json        Stack composition manifest (dcg-stack.yml)
```

## Quick Start

A minimal DCG domain project is a directory with two files:

```
my-domain/
  graph_card.json          # metadata + ontology + graphs index
  graphs/
    default.json           # entity and relation data
```

**`graph_card.json`** -- the project's metadata manifest:

```json
{
  "dcg_project": {
    "name": "payments-domain",
    "version": "0.1.0",
    "description": "Payment processing domain knowledge"
  },
  "ontology": {
    "types": [
      {"name": "Service", "description": "A backend microservice"}
    ],
    "properties": [
      {"name": "api_version", "datatype": "string", "description": "API version string"}
    ]
  },
  "graphs": [
    {"id": "default", "file": "graphs/default.json"}
  ]
}
```

**`graphs/default.json`** -- entity and relation data:

```json
{
  "schema_version": "2.0",
  "entities": {
    "dcg:sha256:a3f7c2d1e4b89012...": {
      "id": "dcg:sha256:a3f7c2d1e4b89012...",
      "label": "process_payment",
      "description": "Processes a credit card payment",
      "attributes": [
        {"property": "instance of", "ref": "dcg:meta:Function"},
        {"property": "part of", "ref": "dcg:sha256:b1c4e8f2..."},
        {"property": "file path", "string": "src/billing/charge.py"},
        {"property": "line number", "quantity": 42}
      ]
    }
  },
  "relations": {},
  "layers": []
}
```

## Composing Multiple Domains

A **stack manifest** (`dcg-stack.yml`) composes domain projects into a unified graph with cross-layer connections:

```yaml
stack: appsec-context-graph
strict: true

layers:
  - name: security
    source: ./dcg-security-domain

  - name: product
    source: ./dcg-product-domain
    extends: [security]

joins:
  - mitigates: SecurityCapability.targets_cwe -> WeaknessType.cwe_id
```

The `joins` section declares **join rules** that join entities across layers on shared property values. The stack materializes these connections at load time -- no cross-layer UIDs are stored in any layer's data files.

## Schema Validation

```bash
pip install check-jsonschema

# Validate a graph_card.json
check-jsonschema --schemafile schema/graph_card.schema.json path/to/graph_card.json

# Validate a graph data file
check-jsonschema --schemafile schema/graph_data.schema.json path/to/graphs/default.json

# Validate a stack manifest (YAML -> JSON)
python3 -c "import yaml,json,sys; json.dump(yaml.safe_load(open(sys.argv[1])),sys.stdout)" \
  dcg-stack.yml | check-jsonschema --schemafile schema/stack.schema.json -
```

---

## Ubiquitous Language

The following glossary defines every term used across the [RFC](spec/rfc.md), JSON schemas, and the [reference implementation](https://github.com/agentkg/domain-context-graph). Each term maps to its protocol definition, file/schema location, and code symbol. This is the shared vocabulary -- use these terms consistently in documentation, APIs, and conversations about DCG.

### Entity Model

| Term | Definition | Schema / File | Code Symbol |
|---|---|---|---|
| **Entity** | A knowledge item with an ID, label, description, and attributes. The atomic unit of a DCG graph. | `graph_data.schema.json` `entities` | `Entity` (model), `add_entity()` / `get_entity()` (store) |
| **Attribute** | A typed fact about an entity: a property-value pair with optional qualifiers. Attributes are intrinsic facts on entities. | `attributes` array on each entity | `Attribute` (model), attribute dicts in entity `attributes` |
| **Qualifier** | Metadata about an attribute (not the entity). Follows the same `{property, typed_key: value}` format as an attribute. | `qualifiers` array on an attribute | `qualifiers` field on `Attribute` |
| **Relation** | A typed directed link between two entities, stored separately from attributes. Has source, target, property, and optional qualifiers. | `graph_data.schema.json` `relations` | `Relation` (model), `add_relation()` / `get_relations()` (store) |
| **Entity UID** | Content-addressed identifier: `dcg:<algorithm>:<hash>`. Computed from sorted identity key-value pairs. Deterministic -- same keys always produce the same UID. | `id` field on entity | `entity_uid(**identity_keys)` |
| **Relation UID** | Deterministic identifier for a relation: `dcg:rel:<algorithm>:<hash>`. Computed from source UID, target UID, and property. | `uid` field on relation | `relation_uid(source, target, property)` |
| **Type** | A type-defining entity with a `dcg:meta:` prefixed ID (e.g., `dcg:meta:Function`). Exempt from content-addressing. Defines what an entity IS. | Built-in ontology YAML | `TYPE_REGISTRY`, `register_global_type()` |
| **Retraction** | Marking an entity or attribute as `"retracted": true` in a layer. Suppressses matching data from older layers during merge without physical deletion. | `retracted` boolean on entity or attribute | `retracted` field on `Entity`, `Attribute` |
| **Redirect** | A `"redirected to"` attribute on an old entity pointing to a new UID. Supports identity key changes (renames, moves). Followed transparently up to depth 10. | `redirected to` attribute | `redirect(old_uid, new_uid)`, `_follow_redirect()` |

### Property and Value System

| Term | Definition | Schema / File | Code Symbol |
|---|---|---|---|
| **Property** | An English-language key identifying the kind of fact an attribute asserts (e.g., `"instance of"`, `"file path"`). | `property` field on attribute | `property` field, `PROPERTY_REGISTRY` |
| **Property Alias** | An alternative name for a property (e.g., `P31` -> `instance of`). Resolved transparently by queries. | `aliases` in ontology | `register_alias()`, `resolve_property()`, `PROPERTY_ALIASES` |
| **Typed Value Key** | The single value field on an attribute, indicating its type: `"ref"` (UID reference), `"string"` (text), or `"quantity"` (number). Exactly one per attribute. | `ref` / `string` / `quantity` key on attribute | `typed_key` field on `Attribute` |
| **Core Properties** | The three properties every implementation must support: `"instance of"` (P31), `"part of"` (P361), `"subclass of"` (P279). | Built-in ontology | Hardcoded in `builtin_ontology.yaml` |

### Domain and Classification

| Term | Definition | Schema / File | Code Symbol |
|---|---|---|---|
| **Domain** | A first-class entity with `"instance of" -> "dcg:meta:Domain"`. Represents a real-world area of concern (Payments, Security). Ordinary entity -- no special store treatment. | Entity with Domain type attribute | `create_domain()` (ops/ingest) |
| **Domain Root** | A domain entity with no `"part of"` attribute to another domain. The top-level entry point for consumers. RECOMMENDED: 1-10 per project. | No parent domain | -- |
| **Domain Hierarchy** | Nested domains via `"part of"` attributes between domain entities. Hides internal categorization behind top-level domains. | `part of` attributes between domains | `query(part_of=...)` |
| **Two-Axis Classification** | Every entity is classified by structural type (`instance of`) and domain membership (`part of`), independently queryable. | `instance of` + `part of` attributes | `query(instance_of=..., part_of=...)` |

### Ontology

| Term | Definition | Schema / File | Code Symbol |
|---|---|---|---|
| **Ontology** | The set of declared entity types, properties, relations, and aliases available to a graph. Composed from built-in declarations and per-layer `graph_card.json` ontology sections. | `ontology` key in `graph_card.json` | `OntologyStore` |
| **Built-in Ontology** | Core types and properties shipped with the implementation (Domain, Type, Function, instance of, part of, etc.). Always available without explicit registration. | `builtin_ontology.yaml` | `_BUILTIN_TYPES`, `_BUILTIN_PROPERTIES` |
| **Ontology Declaration** | Domain-specific types, properties, and aliases declared in a layer's `graph_card.json` `ontology` key. The single source of truth for a layer's custom vocabulary. | `graph_card.schema.json` `ontology` | `OntologyStore.load_declaration()` |
| **Ontology Merge** | Combining ontology declarations from all layers in topological order into a unified view. Identical redeclarations are accepted; conflicting definitions raise errors. | Stack load process | `OntologyStore.merge()`, `OntologyConflictError` |
| **Strict Mode** | Validation mode (`strict: true`) that rejects unregistered types, properties, and relations at write time. Enforces minimum 2 entity-linking attributes per non-Domain entity. | `strict` in stack manifest | `OntologyStore(strict=True)`, `OntologyViolationError` |

### Versioning (Layers)

| Term | Definition | Schema / File | Code Symbol |
|---|---|---|---|
| **Layer** | An immutable, atomic snapshot of changes committed to the store. Contains entity/relation UIDs, source, pass type, timestamp. Analogous to a git commit. | `graph_data.schema.json` `layers` | `Layer` (model), `commit_layer()` |
| **Pass Type** | What a layer represents relative to history: `"full"` (complete snapshot -- merge stops here), `"incremental"` (additive update), or `"patch"` (targeted correction). | `pass_type` on layer | `pass_type` field on `Layer`, `commit_layer(pass_type=...)` |
| **Merge** | Materializing current state by replaying layers newest-to-oldest at attribute level. Attributes with different properties are preserved; same-tuple attributes resolve latest-wins. Stops at first `"full"` layer. | Layer replay logic | `merge_layers(store)` |
| **Compaction** | Squashing all layers into a single `"full"` layer. Consumes retractions, purges retracted entities/attributes. Produces clean state. | Compaction logic | `compact(store)`, `should_compact(store)` |
| **Schema Version** | Version string on each graph data file (current: `"2.0"`). Major version mismatches are rejected; minor mismatches are tolerated. | `schema_version` in graph data | `SCHEMA_VERSION`, `check_schema_version()`, `SchemaVersionError` |

### File Format

| Term | Definition | Schema / File | Code Symbol |
|---|---|---|---|
| **Domain Project** | A directory containing `graph_card.json` + `graphs/` subdirectory. The canonical on-disk format for DCG data. | Directory layout | `DcgProject` |
| **Graph Card** | The metadata manifest (`graph_card.json`) at a project's root. Contains project identity, ontology declarations, and a graphs index. MUST NOT contain entity/relation data. | `graph_card.schema.json` | `DcgProject.MANIFEST_FILE`, `DcgProject.save()` / `DcgProject.load()` |
| **Graph Data File** | A JSON file under `graphs/` (e.g., `graphs/default.json`) containing entities, relations, and layers for one named graph. | `graph_data.schema.json` | `DcgProject._load_graph_data()`, `DcgProject._graph_data_dict()` |
| **Graphs Index** | The `graphs` array in `graph_card.json` mapping graph IDs to file paths. First entry is the default graph. | `graphs` in `graph_card.json` | `DcgProject._graphs_index` |

### Composition (Stacks)

| Term | Definition | Schema / File | Code Symbol |
|---|---|---|---|
| **Stack** | A composition of multiple domain projects into a unified knowledge graph, defined by a YAML manifest. | `stack.schema.json` | `DcgStack` |
| **Stack Manifest** | The YAML file (`dcg-stack.yml`) declaring the stack name, layers, extends DAG, strict flag, and join rules. | `stack.schema.json` | `StackConfig.from_yaml()` |
| **Composition Layer** | A single domain project within a stack. Distinct from versioning layers -- composition layers separate concerns across independent projects. | `layers` array in manifest | `StackLayer` |
| **Extends DAG** | The directed acyclic graph formed by `extends` fields across composition layers. Each layer may have multiple parents. Validated for acyclicity at load time. | `extends` on layer entry | `DcgStack._extends_dag`, `_topological_sort()` |
| **Join Rule** | A declaration in the stack manifest joining entities across layers on shared property values: `property: SourceType.match_prop -> TargetType.match_prop`. | `joins` array in manifest | `JoinRule`, `parse_join_rule()` |
| **Cross-Layer Relation** | A cross-layer relation produced at stack load time by evaluating a join rule. Read-only, recomputed on every load, not stored in any layer's data files. | Runtime only | `DcgStack._materialize_joins()`, `cross_layer_relations()` |
| **Cross-Layer Resolution** | BFS traversal of the extends DAG to find an entity, starting from a named layer. Returns the first entity found (nearest layer wins). | Resolution logic | `DcgStack.resolve(layer_name, uid)` |
| **Intra-Layer Constraint** | All entity/relation UIDs in a layer's graph data files must reference only UIDs within that same layer. Cross-layer connections use join rules, not UID references. | Graph data constraint | `GraphStore.validate_intra_layer_refs()` |

### Store Protocol

| Term | Definition | Code Symbol |
|---|---|---|
| **GraphStoreProtocol** | The runtime-checkable Protocol that all conforming graph store implementations must satisfy. | `GraphStoreProtocol` |
| **GraphStore** | The reference in-memory graph store implementation with layered history, ontology validation, and redirect support. | `GraphStore` |
| **add_entity** | Add or update an entity. Idempotent -- same UID updates in place. Validates ontology in strict mode. | `GraphStore.add_entity(entity)` |
| **add_relation** | Create a typed directed relation between two entities. | `GraphStore.add_relation(source, target, property)` |
| **get_entity** | Retrieve an entity by UID. Follows redirects transparently (max depth 10). | `GraphStore.get_entity(uid)` |
| **get_relations** | Query relations by source, target, or property. Resolves property aliases. | `GraphStore.get_relations(source, target, property)` |
| **query** | Filter entities by structural type (`instance_of`) and/or domain membership (`part_of`). | `GraphStore.query(instance_of, part_of)` |
| **commit_layer** | Atomically commit pending changes as a new immutable layer. | `GraphStore.commit_layer(source, pass_type)` |
| **load** | Deserialize a graph data file into the store. Validates schema version. | `GraphStore.load(path)` |

---

## Design Principles

1. **English-first, alias-compatible.** Property keys are English strings (`"instance of"`, not `"P31"`). Wikidata P-IDs are registered as aliases for interoperability.

2. **Content-addressed identity.** Entity UIDs are deterministic hashes of identity keys. Same inputs always produce the same UID, regardless of insertion order.

3. **Attribute-level versioning.** Layers track changes at the individual attribute level, not the entity level. This enables fine-grained retraction and merge without data loss.

4. **Vocabulary coupling, not UID coupling.** Cross-layer connections match on shared property values (e.g., `cwe_id: "CWE-20"`), not on UID references. Parent layers can be rebuilt with new UIDs without breaking child layer data.

5. **Ontology-governed composition.** Every type and property referenced in join rules must exist in the merged ontology. Strict mode catches unregistered vocabulary at write time.

## Reference Implementation

The reference implementation is [domain-context-graph](https://github.com/agentkg/domain-context-graph) (`dcg` Python package).

## License

Apache-2.0
