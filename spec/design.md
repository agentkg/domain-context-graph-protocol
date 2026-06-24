# Domain Context Graph — Design Spec

**Status:** Draft
**Date:** 2026-06-08
**Updated:** 2026-06-09

---

## 1. Problem Statement

There is no lightweight, embeddable, in-process graph store that:

1. Speaks a **Wikidata-compatible entity format** (labels, descriptions, aliases, typed attributes)
2. Supports **layered evolution** — each data source produces a layer that stacks on previous state
3. Is **git-backed** — content-addressed, diffable, serializable to a dedicated git repo
4. Works **in-memory first** — no server, no external DB required for pipeline use
5. Is **domain-agnostic** — the same protocol handles code graphs, city knowledge graphs, or any entity-relation data

### What exists today

| Tool | Entity Format | Embeddable? | Versioned? | Layered? |
|---|---|---|---|---|
| **Wikibase** | Wikibase JSON (canonical) | No — PHP/MediaWiki stack | Yes (revisions) | No |
| **Oxigraph** | RDF/SPARQL | Yes (pyoxigraph) | No | No |
| **TerminusDB** | JSON-LD/RDF | No — server process | Yes (git-like branching) | Yes |
| **qwikidata** | Wikibase JSON (read-only) | Yes | No | No |
| **KuzuDB/Neo4j** | Property graph (Cypher) | Yes/No | No | No |

**The gap:** A Wikibase JSON-compatible entity store that is embeddable, versioned, and layered.

KnowledgeContextGraph fills this gap.

---

## 2. Design Principles

1. **Wikibase JSON-compatible** — entity structure follows the [Wikibase JSON spec](https://doc.wikimedia.org/Wikibase/master/php/docs_topics_json.html). Tools like `qwikidata` can parse our entities.
2. **In-memory primary** — `GraphStore` is a dict-backed in-process store. No disk, no server.
3. **Git persistence optional** — `GitGraphStore` serializes to a dedicated git repo via `dulwich`. Layers = git commits.
4. **Protocol-first** — Python Protocols define the contract. Implementations are pluggable.
5. **Domain-agnostic** — the core knows nothing about code, cities, or any specific domain. Consumers register their own ontology entries.
6. **Wikidata semantics** — typing via "instance of" (P31), hierarchy via "subclass of" (P279), English relation keys.

---

## 3. Entity Model

### 3.1 Wikibase JSON Compatibility

KnowledgeContextGraph entities follow the Wikibase JSON structure with targeted simplifications for embedded use. The format is a subset that any Wikibase-aware tool can read.

#### Wikibase canonical format (reference)

```json
{
  "id": "Q42",
  "type": "item",
  "labels": {"en": {"language": "en", "value": "Douglas Adams"}},
  "descriptions": {"en": {"language": "en", "value": "English author"}},
  "aliases": {"en": [{"language": "en", "value": "Douglas Noel Adams"}]},
  "claims": {
    "P31": [{
      "id": "Q42$statement-id",
      "type": "statement",
      "mainsnak": {
        "snaktype": "value",
        "property": "P31",
        "datavalue": {"type": "wikibase-entityid", "value": {"id": "Q5"}}
      },
      "qualifiers": {},
      "references": [],
      "rank": "normal"
    }]
  }
}
```

#### KnowledgeContextGraph format

```json
{
  "id": "dcg:sha256:a3f7c2d1e4b89012f4c5d6e7",
  "type": "item",
  "labels": {"en": {"language": "en", "value": "list_pets"}},
  "descriptions": {"en": {"language": "en", "value": "FastAPI endpoint handler for listing pets"}},
  "aliases": {"en": [{"language": "en", "value": "listPets"}]},
  "attributes": {
    "instance of": [{
      "type": "statement",
      "mainsnak": {
        "snaktype": "value",
        "property": "instance of",
        "datavalue": {"type": "wikibase-entityid", "value": {"id": "dcg:meta:Function"}}
      },
      "rank": "normal"
    }],
    "line number": [{
      "type": "statement",
      "mainsnak": {
        "snaktype": "value",
        "property": "line number",
        "datavalue": {"type": "quantity", "value": {"amount": 42}}
      },
      "rank": "normal"
    }],
    "file path": [{
      "type": "statement",
      "mainsnak": {
        "snaktype": "value",
        "property": "file path",
        "datavalue": {"type": "string", "value": "src/main.py"}
      },
      "rank": "normal"
    }]
  },
  "dcg:schema_version": "1.0"
}
```

### 3.2 Key Differences from Wikibase

| Aspect | Wikibase | KnowledgeContextGraph |
|---|---|---|
| Entity IDs | `Q` + integer (`Q42`) | Self-describing content hash (`dcg:sha256:a3f7c2d1...`) |
| Property IDs | `P` + integer (`P31`) | English keys (`instance of`) |
| Statement IDs | Random GUID | Omitted (derived from entity + property) |
| Qualifiers | Full snak structure | Same (optional) |
| References | Full snak groups | Same (optional) |
| Sitelinks | Wikipedia page links | Omitted |
| `lastrevid` / `modified` | MediaWiki revision | Omitted (git commit provides this) |
| Schema version | Implicit | Explicit `dcg:schema_version` field |

**Why English property keys instead of P-IDs:** KnowledgeContextGraph is designed for developer consumption, not Wikipedia. `"instance of"` is immediately readable; `"P31"` requires a lookup. For Wikidata interop, a property alias registry maps between the two.

### 3.3 Content-Addressing (UIDs)

Entity IDs are **self-describing content-addressed hashes**: the algorithm is encoded in the ID so the scheme can evolve without breaking existing references (following the IPFS CID/multihash pattern).

Format: `dcg:<algorithm>:<hash>`

```python
def entity_uid(**identity_keys) -> str:
    """Deterministic, self-describing content-addressed UID."""
    parts = [f"{k}={v}" for k, v in sorted(identity_keys.items())]
    digest = hashlib.sha256(":".join(parts).encode()).hexdigest()[:32]
    return f"dcg:sha256:{digest}"

# Code entity — identity is (name, path, line_number)
entity_uid(name="list_pets", path="src/main.py", line_number=42)
# → "dcg:sha256:a3f7c2d1e4b89012f4c5d6e7a8b90123"

# Domain entity — identity is (domain)
entity_uid(domain="payments")
# → "dcg:sha256:b1c4e8f2a7d30956e1f2a3b4c5d60789"
```

**Why 128-bit (32 hex chars)?** The birthday bound at 64 bits (~2^32) means collisions become likely at ~10M entities — well within range of a large monorepo. At 128 bits, the birthday bound is ~2^64, safe to ~10^18 entities. A collision at 64 bits causes **silent entity merge with no error signal** — the worst possible failure mode.

**Why self-describing?** If the hash algorithm ever needs to change (e.g., SHA-256 → BLAKE3), old IDs remain valid because the algorithm is encoded in the ID. Systems that embed opaque hashes face a full-rewrite migration. The `dcg:sha256:` prefix costs 11 bytes but buys permanent evolvability.

Same identity keys always produce the same UID. This enables deduplication across layers and deterministic merge.

---

## 4. Relation Model

Relations are expressed as attributes on the source entity, following the Wikibase statement structure. The target is referenced by entity ID in the `datavalue`.

```json
{
  "id": "dcg:sha256:a3f7c2d1e4b89012f4c5d6e7",
  "attributes": {
    "calls": [{
      "type": "statement",
      "mainsnak": {
        "snaktype": "value",
        "property": "calls",
        "datavalue": {"type": "wikibase-entityid", "value": {"id": "dcg:sha256:def456a1b2c3d4e5f6a7b8c9"}}
      },
      "qualifiers": {
        "line number": [{"snaktype": "value", "property": "line number",
                         "datavalue": {"type": "quantity", "value": {"amount": 55}}}]
      },
      "rank": "normal"
    }]
  }
}
```

### 4.1 Standalone Relation Objects

For consumers that prefer explicit edge objects (graph DBs, visualization tools), KnowledgeContextGraph also supports a standalone relation format:

```json
{
  "dcg:type": "relation",
  "uid": "dcg:rel:c8d9e0f1a2b34567",
  "source": "dcg:sha256:a3f7c2d1e4b89012f4c5d6e7",
  "target": "dcg:sha256:def456a1b2c3d4e5f6a7b8c9",
  "property": "calls",
  "qualifiers": {"line number": 55, "confidence": "resolved"},
  "schema_version": "1.0"
}
```

The standalone format is a denormalized view of the attribute — both representations are equivalent and convertible.

### 4.2 Property Registry

Properties are registered with English keys, descriptions, and expected datavalue types:

```python
PROPERTY_REGISTRY = {
    # Wikidata-universal
    "instance of":      {"datatype": "wikibase-entityid", "wikidata": "P31",
                         "description": "entity is an instance of this type"},
    "subclass of":      {"datatype": "wikibase-entityid", "wikidata": "P279",
                         "description": "type hierarchy"},
    "part of":          {"datatype": "wikibase-entityid", "wikidata": "P361",
                         "description": "entity belongs to a domain or composite"},
    "depends on":       {"datatype": "wikibase-entityid", "wikidata": "P1269",
                         "description": "domain or entity depends on another"},
    "triggers":         {"datatype": "wikibase-entityid",
                         "description": "entity causes an effect in another entity"},

    # Code-specific
    "calls":            {"datatype": "wikibase-entityid",
                         "description": "function invokes another function"},
    "contains":         {"datatype": "wikibase-entityid",
                         "description": "parent contains child"},
    "imports":          {"datatype": "wikibase-entityid",
                         "description": "file imports a module"},
    "inherits from":    {"datatype": "wikibase-entityid",
                         "description": "class extends another class"},
    "handles route":    {"datatype": "wikibase-entityid",
                         "description": "function handles an HTTP endpoint"},
    "tested by":        {"datatype": "wikibase-entityid",
                         "description": "production code tested by test function"},
    "vulnerability in": {"datatype": "wikibase-entityid",
                         "description": "security issue found in function"},

    # Value properties
    "line number":      {"datatype": "quantity",
                         "description": "source code line number"},
    "file path":        {"datatype": "string",
                         "description": "file system path"},
    "language":         {"datatype": "string",
                         "description": "programming language"},
    "http method":      {"datatype": "string",
                         "description": "HTTP method (GET, POST, etc.)"},
    "http path":        {"datatype": "string",
                         "description": "HTTP route path"},
}
```

Consumers extend the registry at runtime:

```python
from dcg.core import register_property

register_property("population", datatype="quantity",
                  description="number of inhabitants", wikidata="P1082")
```

---

## 5. Domains, Types, and Ontology

### 5.1 Domains as First-Class Entities

A **Domain** is a real-world area of concern — a bounded context whose entities interact with and impact entities in other domains. Domains are not namespaces or technical categories. They are **entities in the graph**, just like functions, classes, or buildings.

In a software system, the interlinked domains might include:

```
┌─────────────┐    depends on    ┌─────────────┐    impacts     ┌──────────────┐
│ Payments    │ ───────────────▶ │ Inventory   │ ◀──────────── │ Warehouse    │
│ (domain)    │                  │ (domain)    │               │ (domain)     │
└──────┬──────┘                  └──────┬──────┘               └──────┬───────┘
       │                                │                             │
       │ implemented by                 │ implemented by              │ monitored by
       ▼                                ▼                             ▼
  process_payment()              update_stock()               warehouse_sensor()
  PaymentGateway class           InventoryDB class            InfraMonitor class
  POST /api/pay endpoint         GET /api/stock endpoint      Grafana dashboard
```

Every domain is an instance of `dcg:meta:Domain`. Domains participate in the graph like any other entity — they have labels, descriptions, and attributes expressing their relations to other domains and entities:

```python
TYPE_REGISTRY = {
    # The Domain meta-type itself
    "dcg:meta:Domain":      {"labels": {"en": "Domain"},
                             "description": "A real-world area of concern"},
    "dcg:meta:Type":        {"labels": {"en": "Type"},
                             "description": "A classification of entities"},
}
```

Creating a domain:

```python
from dcg.core import GraphStore, entity_uid, builders as wb

store = GraphStore()

# Domains are entities — created just like any other entity
payments_uid = entity_uid(domain="payments")
store.add_entity(wb.item(
    uid=payments_uid,
    label="Payments",
    description="Payment processing, billing, and financial transactions",
    attributes=[
        wb.attribute("instance of", wb.ref_value("dcg:meta:Domain")),
    ],
))

inventory_uid = entity_uid(domain="inventory")
store.add_entity(wb.item(
    uid=inventory_uid,
    label="Inventory",
    description="Stock management, warehousing, and supply chain",
    attributes=[
        wb.attribute("instance of", wb.ref_value("dcg:meta:Domain")),
    ],
))

# Domain-to-domain relation: Payments depends on Inventory
store.add_relation(payments_uid, inventory_uid, "depends on")
```

### 5.2 Entity Membership in Domains

Entities belong to domains via **"part of"** attributes. An entity can belong to **multiple domains** — an API endpoint serving both Payments and Inventory has two "part of" attributes:

```python
# A function that belongs to the Payments domain
fn_uid = entity_uid(name="process_payment", path="src/billing.py", line_number=42)
store.add_entity(wb.item(
    uid=fn_uid,
    label="process_payment",
    description="Processes a credit card payment",
    attributes=[
        wb.attribute("instance of", wb.ref_value("dcg:meta:Function")),
        wb.attribute("part of", wb.ref_value(payments_uid)),
        wb.attribute("file path", wb.string_value("src/billing.py")),
    ],
))

# An endpoint shared across Payments AND Inventory domains
endpoint_uid = entity_uid(name="POST /api/checkout", source="openapi")
store.add_entity(wb.item(
    uid=endpoint_uid,
    label="POST /api/checkout",
    description="Checkout endpoint — charges payment and updates stock",
    attributes=[
        wb.attribute("instance of", wb.ref_value("dcg:meta:Endpoint")),
        wb.attribute("part of", wb.ref_value(payments_uid)),
        wb.attribute("part of", wb.ref_value(inventory_uid)),
    ],
))
```

The graph encodes domain membership as data, not metadata:

```
process_payment   → instance of → Function     (structural type)
                  → part of     → Payments     (domain membership)

POST /api/checkout → instance of → Endpoint    (structural type)
                   → part of     → Payments    (domain membership)
                   → part of     → Inventory   (domain membership)

Payments          → instance of → Domain       (it's a domain)
                  → depends on  → Inventory    (domain relation)
```

### 5.3 Type Types (Structural)

Entity type types are **structural** — they describe what an entity IS, independent of which domain it belongs to. A Function is a Function whether it lives in Payments or Inventory.

```python
TYPE_REGISTRY = {
    # Meta-types
    "dcg:meta:Domain":         {"labels": {"en": "Domain"},
                                "description": "A real-world area of concern"},
    "dcg:meta:Type":           {"labels": {"en": "Type"},
                                "description": "A classification of entities"},

    # Structural types — code
    "dcg:meta:Function":       {"labels": {"en": "Function"},
                                "description": "A callable code unit"},
    "dcg:meta:Class":          {"labels": {"en": "Class"},
                                "description": "An object-oriented class"},
    "dcg:meta:Struct":         {"labels": {"en": "Struct"},
                                "description": "A composite data type"},
    "dcg:meta:Interface":      {"labels": {"en": "Interface"},
                                "description": "An interface contract"},
    "dcg:meta:Trait":          {"labels": {"en": "Trait"},
                                "description": "A trait/mixin type"},
    "dcg:meta:File":           {"labels": {"en": "File"},
                                "description": "A source file"},
    "dcg:meta:Module":         {"labels": {"en": "Module"},
                                "description": "An importable module"},

    # Structural types — API
    "dcg:meta:Endpoint":       {"labels": {"en": "Endpoint"},
                                "description": "An API endpoint"},

    # Structural types — security
    "dcg:meta:Vulnerability":  {"labels": {"en": "Vulnerability"},
                                "description": "A security vulnerability"},

    # Consumers register their own structural types:
    # "dcg:meta:Building", "dcg:meta:Sensor", "dcg:meta:Pipeline", etc.
}
```

Registering new structural types:

```python
from dcg.core import register_global_type

register_global_type("dcg:meta:Building",
                     label="Building", description="A physical structure")
register_global_type("dcg:meta:Sensor",
                     label="Sensor", description="An IoT sensor device")
```

### 5.4 Two Dimensions of Classification

Every entity has two orthogonal classification axes:

| Axis | Relationship | Question answered | Example |
|---|---|---|---|
| **Structural type** | `instance of` | *What is it?* | Function, Class, Endpoint, Building |
| **Domain membership** | `part of` | *What real-world concern does it belong to?* | Payments, Inventory, Infrastructure |

This separation means:
- **Querying by type** across all domains: "show me all Functions" → `store.query(instance_of="dcg:meta:Function")`
- **Querying by domain**: "show me everything in Payments" → `store.get_relations(target=payments_uid, property="part of")`
- **Cross-cutting queries**: "show me all Endpoints in both Payments and Inventory" → intersect domain membership
- **Domain impact analysis**: "what domains does Payments depend on?" → `store.get_relations(source=payments_uid, property="depends on")`

---

## 6. In-Memory Store (GraphStore)

The primary store is in-memory. No disk, no server. Optimized for pipeline use.

```python
from typing import Protocol, runtime_checkable

@runtime_checkable
class GraphStoreProtocol(Protocol):
    """Core protocol — any implementation must satisfy this."""

    def add_entity(self, entity: dict) -> str: ...
    def add_relation(self, source: str, target: str, property: str,
                     qualifiers: dict | None = None) -> str: ...
    def remove(self, uid: str) -> None: ...

    def get_entity(self, uid: str) -> dict | None: ...
    def get_relations(self, source: str = None, target: str = None,
                      property: str = None) -> list[dict]: ...
    def query(self, instance_of: str = None, **property_filters) -> list[dict]: ...

    def commit_layer(self, source: str, pass_type: str = "incremental") -> dict: ...
    def get_layers(self) -> list[dict]: ...

    def to_dict(self) -> dict: ...
    def to_json(self, path) -> None: ...
```

### 6.1 GraphStore (dict-backed implementation)

```python
class GraphStore:
    """In-memory graph store with layered history."""

    def __init__(self, schema_version: str = "1.0"):
        self._entities: dict[str, dict] = {}       # uid → Wikibase JSON entity
        self._relations: dict[str, dict] = {}       # uid → standalone relation
        self._pending_entities: dict[str, dict] = {} # uncommitted
        self._pending_relations: dict[str, dict] = {}
        self._pending_tombstones: set[str] = set()
        self._layers: list[dict] = []
        self._schema_version = schema_version

    def add_entity(self, entity: dict) -> str:
        """Add or update an entity. Returns UID."""
        uid = entity["id"]
        self._pending_entities[uid] = entity
        self._entities[uid] = entity
        return uid

    def commit_layer(self, source: str, pass_type: str = "incremental") -> dict:
        """Snapshot pending changes as a new layer."""
        layer = {
            "layer_id": self._compute_layer_id(),
            "parent_id": self._layers[-1]["layer_id"] if self._layers else None,
            "source": source,
            "pass_type": pass_type,
            "schema_version": self._schema_version,
            "timestamp": datetime.utcnow().isoformat(),
            "entity_uids": list(self._pending_entities.keys()),
            "relation_uids": list(self._pending_relations.keys()),
            "tombstones": list(self._pending_tombstones),
        }
        self._layers.append(layer)
        self._pending_entities.clear()
        self._pending_relations.clear()
        self._pending_tombstones.clear()
        return layer
```

### 6.2 Usage Example

```python
from dcg.core import GraphStore, entity_uid

store = GraphStore()

# Add a function entity
fn_uid = entity_uid(name="list_pets", path="src/main.py", line_number=42)
store.add_entity({
    "id": fn_uid,
    "type": "item",
    "labels": {"en": {"language": "en", "value": "list_pets"}},
    "descriptions": {"en": {"language": "en", "value": "Lists all pets"}},
    "aliases": {},
    "attributes": {
        "instance of": [{"type": "statement", "mainsnak": {
            "snaktype": "value", "property": "instance of",
            "datavalue": {"type": "wikibase-entityid",
                          "value": {"id": "dcg:meta:Function"}}
        }, "rank": "normal"}],
        "file path": [{"type": "statement", "mainsnak": {
            "snaktype": "value", "property": "file path",
            "datavalue": {"type": "string", "value": "src/main.py"}
        }, "rank": "normal"}],
    },
    "dcg:schema_version": "1.0",
})

# Add a relation
store.add_relation(fn_uid, other_fn_uid, "calls", qualifiers={"line number": 55})

# Commit as a layer
layer = store.commit_layer(source="tree-sitter-parser")

# Query
functions = store.query(instance_of="dcg:meta:Function")
callers = store.get_relations(target=fn_uid, property="calls")

# Export
store.to_json(Path("graph.json"))
```

---

## 7. Git Persistence (GitGraphStore)

Optional layer. Extends `GraphStore` to serialize layers as git commits in a dedicated repo.

### 7.1 Repo Layout

```
.dcg/                              # dedicated graph repo
├── .git/
├── meta.json                      # store metadata
├── ontology/                      # entity types (committed once)
│   ├── Function.json
│   ├── Class.json
│   └── ...
├── entities/                      # by UID prefix (2-char bucketing)
│   ├── a3/
│   │   └── f7c2d1e4b89012.json   # full Wikibase JSON entity
│   ├── b1/
│   │   └── c4e8f2a7d30956.json
│   └── ...
├── relations/                     # by property type
│   ├── calls/
│   │   └── <uid>.json
│   ├── instance_of/
│   │   └── <uid>.json
│   └── ...
└── tombstones/
    └── <uid>.tomb
```

### 7.2 GitGraphStore

```python
class GitGraphStore(GraphStore):
    """GraphStore with git-backed persistence via dulwich."""

    def __init__(self, repo_path: str | Path, **kwargs):
        super().__init__(**kwargs)
        self._repo_path = Path(repo_path)
        self._repo = self._init_or_open()

    def commit_layer(self, source: str, pass_type: str = "incremental") -> dict:
        layer = super().commit_layer(source, pass_type)
        self._write_files(layer)
        self._git_commit(layer)
        return layer

    def load(self) -> None:
        """Rebuild in-memory state from git repo."""
        ...

    def _git_commit(self, layer: dict) -> str:
        """Create a git commit via dulwich. Returns commit SHA."""
        ...
```

### 7.3 Layer Evolution

Each `commit_layer()` = a git commit. The commit message includes source and pass type.

```
commit abc123  "layer: tree-sitter-parser (full)"
commit def456  "layer: openapi-extractor (incremental)"
commit ghi789  "layer: resolver-pass (incremental)"
commit jkl012  "layer: sarif-scanner (patch)"
```

`git log` in `.dcg/` shows the full evolution. `git diff` between commits shows exactly what changed in each layer. Git's content-addressing means unchanged entities take zero additional storage.

### 7.4 Layer Merge & Compaction

```python
def merge_layers(store: GraphStore) -> dict:
    """Materialize the current state by replaying layers (latest-wins)."""
    ...

def compact(store: GitGraphStore) -> dict:
    """Squash all layers into a single 'full' snapshot commit."""
    ...
```

**Merge semantics:**
- Walk layers newest → oldest
- First occurrence of a UID wins (latest-wins)
- Tombstone marker = entity/relation deleted
- Stop at a "full" layer (complete snapshot)

**Compaction** periodically squashes the layer chain into a fresh "full" commit, keeping the chain bounded.

---

## 8. Builder Helpers

To avoid verbose Wikibase JSON construction, KnowledgeContextGraph provides builder functions:

```python
from dcg.core import builders as wb

# Create an entity using builders
entity = wb.item(
    uid=entity_uid(name="list_pets", path="src/main.py", line_number=42),
    label="list_pets",
    description="FastAPI endpoint handler for listing pets",
    attributes=[
        wb.attribute("instance of", wb.ref_value("dcg:meta:Function")),
        wb.attribute("file path", wb.string_value("src/main.py")),
        wb.attribute("line number", wb.quantity_value(42)),
        wb.attribute("language", wb.string_value("python")),
    ],
)

# Create a relation using builders
relation = wb.relation(
    source=fn_uid, target=endpoint_uid,
    property="handles route",
    qualifiers={"confidence": "inferred"},
)
```

---

## 9. Integration Points

KnowledgeContextGraph is domain-agnostic. Consumers plug in via:

### 9.1 Ontology Extension

```python
from dcg.core import register_global_type, register_property, builders as wb

# Register structural types (domain-independent)
register_global_type("dcg:meta:Building",
                     label="Building", description="A physical structure")
register_property("address", datatype="string",
                  description="Street address")
register_property("population", datatype="quantity",
                  description="Number of inhabitants", wikidata="P1082")

# Create the domain as an entity in the graph
geo_uid = entity_uid(domain="geospatial")
store.add_entity(wb.item(
    uid=geo_uid, label="Geospatial",
    description="Physical locations and spatial relations",
    attributes=[wb.attribute("instance of", wb.ref_value("dcg:meta:Domain"))],
))
```

### 9.2 Translator Pattern

Each consumer provides a translator that converts domain-specific data to KnowledgeContextGraph format:

```python
class Translator(Protocol):
    def translate(self, raw_data: Any, store: GraphStore) -> int:
        """Translate domain data into entities/relations in the store.
        Returns count of entities added."""
        ...
```

**CGC code translator** (in `codegraphcontext/graphlayer/translators/code.py`):
```python
class CodeTranslator:
    def translate(self, file_data: dict, store: GraphStore) -> int:
        """Translate CGC parser output → KnowledgeContextGraph entities."""
        count = 0
        for fn in file_data.get("functions", []):
            store.add_entity(wb.item(
                uid=entity_uid(name=fn["name"], path=file_data["path"],
                              line_number=fn["line_number"]),
                label=fn["name"],
                description="",
                attributes=[
                    wb.attribute("instance of", wb.ref_value("dcg:meta:Function")),
                    wb.attribute("file path", wb.string_value(file_data["path"])),
                    wb.attribute("line number", wb.quantity_value(fn["line_number"])),
                    wb.attribute("language", wb.string_value(file_data.get("lang", ""))),
                ],
            ))
            count += 1
        # ... classes/structs/interfaces/traits similarly
        return count
```

### 9.3 Materializer Pattern

Materializers read from `GraphStore` and write to external systems:

```python
class Materializer(Protocol):
    def materialize(self, store: GraphStore) -> int:
        """Write graph state to an external system. Returns count written."""
        ...
```

**KuzuDB materializer** (in `codegraphcontext/graphlayer/materialize.py`):
```python
class KuzuMaterializer:
    def materialize(self, store: GraphStore, db_manager) -> int:
        for entity in store.query():
            instance_of = self._get_instance_of(entity)
            label = instance_of.split(":")[-1]  # "dcg:meta:Function" → "Function"
            props = self._extract_properties(entity)
            db_manager.execute_query(
                f"MERGE (n:{label} {{uid: $uid}}) SET n += $props",
                {"uid": entity["id"], "props": props}
            )
        ...
```

---

## 10. Wikidata Compatibility

### 10.1 What works out of the box

- `qwikidata` can parse KnowledgeContextGraph entity JSON (same top-level structure)
- Labels, descriptions, aliases follow exact Wikibase format
- Attributes use the same snak structure as Wikibase claims (mainsnak, qualifiers, references, rank)

### 10.2 What differs

- Entity IDs use `dcg:` prefix instead of `Q` numbers
- Property keys are English strings instead of `P` numbers
- `sitelinks` are omitted
- `lastrevid`/`modified` are omitted (git provides this)
- Schema version is an extension field (`dcg:schema_version`)

### 10.3 Wikidata Property Mapping

For entities that correspond to Wikidata items, the property registry includes `wikidata` aliases:

```python
# KnowledgeContextGraph property → Wikidata P-ID
"instance of"  → "P31"
"subclass of"  → "P279"
"part of"      → "P361"
"population"   → "P1082"
```

A future `export_to_wikidata()` function could convert KnowledgeContextGraph entities to Wikidata-compatible format by replacing English keys with P-IDs.

---

## 11. Schema Versioning

The `dcg:schema_version` field on every entity and layer enables forward compatibility:

- **1.0**: Initial format (this spec)
- Major version bump = breaking change (requires re-extraction)
- Minor version bump = additive change (new properties, backward-compatible)

Consumers check `schema_version` before processing. No migration shims — breaking changes require a full re-index (consistent with pre-1.0 policy).

---

## 12. Package Structure

```
kcg/
├── __init__.py                     # convenience re-exports from dcg.core
├── core/                           # INDEPENDENT — zero external domain deps
│   ├── __init__.py                 # public API exports
│   ├── model.py                    # Entity, Relation, Layer dataclasses
│   ├── ontology.py                 # TYPE_REGISTRY (incl. Domain), PROPERTY_REGISTRY, register_*
│   ├── store.py                    # GraphStore (in-memory)
│   ├── merge.py                    # merge_layers, compact
│   ├── git_store.py                # GitGraphStore (dulwich)
│   ├── federation.py               # GraphRegistry, BoundaryResolver protocol
│   ├── builders.py                 # entity(), attribute(), relation() helpers
│   └── uid.py                      # entity_uid(), relation_uid()
└── mcp/                            # optional — requires mcp>=1.27
    ├── __init__.py
    ├── server.py                   # 14 FastMCP tools over stdio
    └── __main__.py                 # python -m kcg.mcp entry point
```

**Dependencies:**
- Core: none (pure Python)
- Git persistence: `dulwich` (optional, Apache 2.0)

---

## 13. Graph Federation Protocol

KnowledgeContextGraph graphs can connect across domain boundaries. Since domains are first-class entities (Section 5), the federation protocol connects **domain-scoped graphs** — each graph contains a domain entity and the entities that belong to it. Cross-graph references follow **open-world semantics**: a reference to an entity in an unavailable graph is valid but unresolvable, not an error. Missing knowledge is not a failure.

### 13.1 Architecture

Real-world systems are composed of interlinked domains. Each domain is a graph that can exist independently but gains value through cross-domain connections:

```
┌──────────────────┐  depends on  ┌──────────────────┐  monitored by ┌──────────────────┐
│  Payments        │ ───────────▶ │  Inventory       │ ◀──────────── │  Infrastructure  │
│  (payments.dcg/) │              │  (inventory.dcg/)│              │  (infra.dcg/)    │
│                  │              │                  │              │                  │
│  process_payment │              │  update_stock    │              │  k8s_deployment  │
│  PaymentGateway  │              │  InventoryDB     │              │  grafana_board   │
│  POST /api/pay   │              │  GET /api/stock  │              │  alert_rule      │
└────────┬─────────┘              └────────┬─────────┘              └────────┬─────────┘
         │                                 │                                 │
         └─────────────────────────────────┼─────────────────────────────────┘
                                           │
                                  ┌────────▼─────────┐
                                  │  GraphRegistry    │
                                  │  (resolves cross- │
                                  │   domain refs)    │
                                  └──────────────────┘
```

**Each graph contains:**
- A **domain entity** (instance of `dcg:meta:Domain`) — the anchor
- **Entities** that belong to the domain via "part of" attributes
- **Intra-domain relations** between those entities
- **Cross-domain references** to entities in peer graphs

**Domains are the organizing principle, not types.** A graph is not "the Function graph" — it's "the Payments graph" which happens to contain Functions, Classes, Endpoints, and any other entity type relevant to Payments.

### 13.2 Graph Manifest

Each graph's `.dcg/` repo root contains a `manifest.json` declaring identity, the domain it represents, exports, and peer connections:

```json
{
  "graph_id": "dcg:graph:payments",
  "domain_entity": "dcg:d1a2b3c4e5f60789",
  "name": "Payments Domain Graph",
  "schema_version": "1.0",

  "exports": {
    "meta_entities": [
      "dcg:meta:Function",
      "dcg:meta:Class",
      "dcg:meta:Endpoint"
    ]
  },

  "peers": [
    {
      "graph_id": "dcg:graph:inventory",
      "domain": "Inventory",
      "repo": "../inventory.kcg",
      "commit": "a1b2c3d4e5f6"
    },
    {
      "graph_id": "dcg:graph:infra",
      "domain": "Infrastructure",
      "repo": "git@github.com:ops-team/infra.kcg.git",
      "commit": "f6e5d4c3b2a1"
    }
  ]
}
```

**Export rules (type-level):**
- Only entities whose "instance of" attribute targets an exported entity type type are visible to peer graphs
- Internal entities (helpers, intermediate parse artifacts) are opaque to other graphs
- The export list is the graph's **public API surface** — changing it is a contract change
- Entities with multi-domain membership are exported from each graph they belong to

### 13.3 Cross-Graph Entity References

When an entity in the Payments graph references an entity in the Inventory graph, the relation's datavalue includes the target graph:

```json
{
  "id": "dcg:sha256:a3f7c2d1e4b89012f4c5d6e7",
  "attributes": {
    "triggers": [{
      "type": "statement",
      "mainsnak": {
        "snaktype": "value",
        "property": "triggers",
        "datavalue": {
          "type": "wikibase-entityid",
          "value": {
            "id": "dcg:sha256:b1c4e8f2a7d30956e1f2a3b4",
            "graph": "dcg:graph:inventory"
          }
        }
      },
      "rank": "normal"
    }]
  }
}
```

The `graph` field in `datavalue.value` is **optional**:
- **Absent** → entity is local to this graph
- **Present** → entity lives in the named peer graph

### 13.4 Open-World Resolution

Cross-graph references are resolved through a `GraphRegistry`. The key semantic: **unavailable knowledge is not an error**.

```python
@runtime_checkable
class GraphRegistry(Protocol):
    def register(self, graph_id: str, store: GraphStore) -> None:
        """Register a graph store for resolution."""
        ...

    def resolve(self, graph_id: str, entity_uid: str) -> dict | None:
        """Resolve a foreign entity. Returns None if graph unavailable
        or entity not exported — never raises."""
        ...

    def resolve_relation(self, source_graph: str, source_uid: str,
                         target_graph: str, target_uid: str,
                         property: str) -> dict | None:
        """Resolve a cross-graph relation. Returns enriched relation
        with resolved entity data, or None if peer unavailable."""
        ...

    def list_peers(self) -> list[dict]:
        """List all registered peer graphs."""
        ...
```

**Resolution rules:**
1. `get_entity(uid)` first checks the local store
2. If not found, checks if any cross-graph reference points to `uid` in a peer graph
3. If peer graph is registered in the registry → resolve from peer store
4. If peer graph is **not** registered → return `None` (knowledge not available, not an error)
5. Relations to unresolved entities are preserved — they become resolvable when the peer graph connects

```python
class LocalGraphRegistry:
    """In-process registry for co-located graph stores."""

    def __init__(self):
        self._stores: dict[str, GraphStore] = {}

    def register(self, graph_id: str, store: GraphStore) -> None:
        self._stores[graph_id] = store

    def resolve(self, graph_id: str, entity_uid: str) -> dict | None:
        store = self._stores.get(graph_id)
        if store is None:
            return None  # graph not available — open world
        entity = store.get_entity(entity_uid)
        if entity is None:
            return None  # entity not found or not exported
        return entity
```

### 13.5 Boundary Resolver Pattern

Cross-graph relations are discovered by **BoundaryResolvers** — components that examine entities in two connected graphs and create edges between them.

```python
class BoundaryResolver(Protocol):
    """Discovers and creates cross-graph relations."""

    source_domain: str  # domain this resolver reads FROM
    target_domain: str  # domain this resolver connects TO

    def resolve(self, source_store: GraphStore,
                target_store: GraphStore,
                output_store: GraphStore) -> int:
        """Scan source and target graphs, write cross-graph relations
        to output_store. Returns count of relations created."""
        ...
```

**Key design:** Cross-graph relations are stored in the **source graph** (the graph that "reaches out"). The source graph owns the edge. This follows the same principle as hyperlinks — the linking page owns the link, not the linked page.

Example: a resolver that connects Payments domain functions to Inventory domain entities when a payment triggers stock updates:

```python
class PaymentsToInventoryResolver:
    source_domain = "Payments"
    target_domain = "Inventory"

    def resolve(self, payments_store, inventory_store, output_store):
        count = 0
        stock_fns = {e["id"]: e for e in inventory_store.query(
            instance_of="dcg:meta:Function")}

        for fn in payments_store.query(instance_of="dcg:meta:Function"):
            # Match by call graph, naming convention, or config
            matched = self._match_stock_calls(fn, stock_fns)
            if matched:
                output_store.add_relation(
                    fn["id"], matched["id"],
                    property="triggers",
                    qualifiers={
                        "confidence": "inferred",
                        "target_graph": "dcg:graph:inventory",
                    },
                )
                count += 1
        return count
```

### 13.6 Git Multi-Repo Layout

Each domain graph is an independent git repo. The domain entity lives inside the graph — the repo is named after the domain it represents:

```
project/
├── .dcg/
│   ├── payments.dcg/          # Payments domain → own git repo
│   │   ├── .git/
│   │   ├── manifest.json
│   │   ├── ontology/
│   │   ├── entities/
│   │   └── relations/
│   ├── inventory.dcg/         # Inventory domain → own git repo
│   │   ├── .git/
│   │   ├── manifest.json
│   │   └── ...
│   ├── infra.dcg/             # Infrastructure domain → own git repo
│   │   ├── .git/
│   │   ├── manifest.json
│   │   └── ...
│   └── registry.json          # local graph registry index
```

Or distributed across teams/organizations:

```
payments.dcg/      → git@github.com:org/payments-graph.kcg.git
inventory.dcg/     → git@github.com:org/inventory-graph.kcg.git
infra.dcg/         → git@github.com:ops-team/infra-graph.kcg.git
```

The `registry.json` at the workspace level maps graph IDs to repo locations:

```json
{
  "graphs": [
    {"graph_id": "dcg:graph:payments",  "domain": "Payments",      "repo": "./payments.kcg"},
    {"graph_id": "dcg:graph:inventory", "domain": "Inventory",     "repo": "./inventory.kcg"},
    {"graph_id": "dcg:graph:infra",     "domain": "Infrastructure","repo": "git@github.com:ops-team/infra-graph.kcg.git"}
  ]
}
```

### 13.7 Federation Pipeline

A complete multi-domain pipeline for an e-commerce system:

```python
from dcg.core import GraphStore, GitGraphStore, LocalGraphRegistry, entity_uid, builders as wb

# 1. Create domain-scoped stores — each is its own git repo
payments_store = GitGraphStore(".dcg/payments.dcg/")
inventory_store = GitGraphStore(".dcg/inventory.dcg/")

# 2. Each store contains its domain entity
payments_uid = entity_uid(domain="payments")
payments_store.add_entity(wb.item(
    uid=payments_uid, label="Payments",
    description="Payment processing and billing",
    attributes=[wb.attribute("instance of", wb.ref_value("dcg:meta:Domain"))],
))

inventory_uid = entity_uid(domain="inventory")
inventory_store.add_entity(wb.item(
    uid=inventory_uid, label="Inventory",
    description="Stock management and supply chain",
    attributes=[wb.attribute("instance of", wb.ref_value("dcg:meta:Domain"))],
))

# 3. Register in a shared registry
registry = LocalGraphRegistry()
registry.register("dcg:graph:payments", payments_store)
registry.register("dcg:graph:inventory", inventory_store)

# 4. Each domain runs its own extraction pipeline
for file_data in payments_code_files:
    code_translator.translate(file_data, payments_store, domain_uid=payments_uid)
payments_store.commit_layer(source="tree-sitter-parser", pass_type="full")

for file_data in inventory_code_files:
    code_translator.translate(file_data, inventory_store, domain_uid=inventory_uid)
inventory_store.commit_layer(source="tree-sitter-parser", pass_type="full")

# 5. Boundary resolution — discovers cross-domain edges
resolver = PaymentsToInventoryResolver()
edges_found = resolver.resolve(payments_store, inventory_store,
                                output_store=payments_store)
payments_store.commit_layer(source="boundary-resolver")

# 6. Cross-graph queries work through the registry
fn = payments_store.get_entity("dcg:sha256:abc123d4e5f6a7b8c9d0e1f2")
stock_ref = get_attribute_target(fn, "triggers")
# stock_ref = {"id": "dcg:def456", "graph": "dcg:graph:inventory"}

# Resolve through registry — returns None if inventory_store not available
stock_fn = registry.resolve("dcg:graph:inventory", stock_ref["id"])

# 7. Domain impact analysis — which domains does Payments depend on?
domain_deps = payments_store.get_relations(
    source=payments_uid, property="depends on")
# → [{"target": inventory_uid, ...}]
```

### 13.8 Federation Guarantees

| Property | Guarantee |
|---|---|
| **Availability** | Each graph is independently available. Peer unavailability degrades cross-graph queries gracefully (returns None), never fails |
| **Consistency** | Commit pinning in manifests ensures reproducible cross-graph resolution at a known point in time |
| **Ownership** | Each entity is stored in the graph where it was created. Entities with multi-domain membership appear in each graph via cross-graph references. Cross-graph relations are owned by the source graph |
| **Isolation** | Graphs cannot modify each other's entities. A graph can only add relations from its own entities to foreign targets |
| **Evolution** | Graphs evolve independently. A peer's new layer doesn't affect consumers until they update the commit pin in their manifest |

---

## 14. Examples

### Code Graph (CGC)

```python
store = GraphStore()

# Layer 1: Parse code
for file_data in all_file_data:
    code_translator.translate(file_data, store)
store.commit_layer(source="tree-sitter-parser", pass_type="full")

# Layer 2: Extract OpenAPI specs
for spec_file in openapi_files:
    openapi_translator.translate(spec_file, store)
store.commit_layer(source="openapi-extractor")

# Layer 3: Resolve cross-domain edges
resolver.resolve(store)  # adds "handles route", "tested by" relations
store.commit_layer(source="resolver-pass")

# Materialize to DB
kuzu_materializer.materialize(store, db_manager)

# Or export to JSON
store.to_json(Path("code-graph.json"))
```

### City Knowledge Graph (Multi-Domain)

```python
# Two domain graphs — Urban Planning and Transit
urban_store = GitGraphStore(".dcg/urban.dcg/")
transit_store = GitGraphStore(".dcg/transit.dcg/")

register_global_type("dcg:meta:Building", label="Building")
register_global_type("dcg:meta:Road", label="Road")
register_global_type("dcg:meta:BusStop", label="Bus Stop")
register_property("address", datatype="string")
register_property("nearest to", datatype="wikibase-entityid")

# Create domain entities
urban_uid = entity_uid(domain="urban-planning")
urban_store.add_entity(wb.item(
    uid=urban_uid, label="Urban Planning",
    description="Zoning, buildings, and land use",
    attributes=[wb.attribute("instance of", wb.ref_value("dcg:meta:Domain"))],
))

transit_uid = entity_uid(domain="transit")
transit_store.add_entity(wb.item(
    uid=transit_uid, label="Public Transit",
    description="Bus routes, stops, and schedules",
    attributes=[wb.attribute("instance of", wb.ref_value("dcg:meta:Domain"))],
))

# Layer 1: OSM building data → Urban Planning domain
for building in osm_buildings:
    urban_store.add_entity(wb.item(
        uid=entity_uid(source="osm", osm_id=building["id"]),
        label=building["name"],
        attributes=[
            wb.attribute("instance of", wb.ref_value("dcg:meta:Building")),
            wb.attribute("part of", wb.ref_value(urban_uid)),
            wb.attribute("address", wb.string_value(building["address"])),
        ],
    ))
urban_store.commit_layer(source="openstreetmap", pass_type="full")

# Layer 2: Transit data → Transit domain
for stop in transit_stops:
    transit_store.add_entity(wb.item(
        uid=entity_uid(source="transit", stop_id=stop["id"]),
        label=stop["name"],
        attributes=[
            wb.attribute("instance of", wb.ref_value("dcg:meta:BusStop")),
            wb.attribute("part of", wb.ref_value(transit_uid)),
        ],
    ))
transit_store.commit_layer(source="transit-authority", pass_type="full")

# Layer 3: Boundary resolution — connect bus stops to nearest buildings
resolver = TransitToUrbanResolver()
resolver.resolve(transit_store, urban_store, output_store=transit_store)
transit_store.commit_layer(source="boundary-resolver")
```

### Git Persistence

```python
# Use GitGraphStore instead of GraphStore
store = GitGraphStore(".dcg/")

# Same API — layers automatically become git commits
store.add_entity(...)
store.commit_layer(source="tree-sitter-parser")
# → .dcg/ now has a git commit with the entity files

# Later: reload from disk
store2 = GitGraphStore(".dcg/")
store2.load()
print(store2.get_layers())  # shows full layer history
```
