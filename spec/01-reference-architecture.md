# DCG Reference Architecture

**Spec ID:** DCG-001-IMP
**Status:** Draft
**Date:** 2026-06-26
**Implements:** [DCG-001](../rfc/01-rfc.md) | [DCG-001-COMP](../rfc/02-rfc-composition.md) | [DCG-001-PACK](../rfc/03-rfc-packs.md)

---

## Table of Contents

**Part I: Data Model & Design Rationale**
- [1. Problem Statement](#1-problem-statement)
- [2. Design Principles](#2-design-principles)
- [3. Entity Model Rationale](#3-entity-model-rationale)
- [4. Relation Model](#4-relation-model)
- [5. Domains, Types, and Ontology](#5-domains-types-and-ontology)
- [6. Wikidata Compatibility](#6-wikidata-compatibility)
- [7. Integration Patterns](#7-integration-patterns)
- [8. Examples](#8-examples)

**Part II: Runtime Implementation Guidance**
- [9. In-Memory Store](#9-in-memory-store)
- [10. Ontology Runtime](#10-ontology-runtime)
- [11. Stack Loader](#11-stack-loader)
- [12. Query Behavior](#12-query-behavior)
- [13. Git Persistence](#13-git-persistence)
- [14. Builder Helpers](#14-builder-helpers)
- [15. Schema Versioning](#15-schema-versioning)
- [16. Package Structure](#16-package-structure)
- [17. Security Considerations](#17-security-considerations)
- [18. References](#18-references)

---

# Part I: Data Model & Design Rationale

---

## 1. Problem Statement

There is no lightweight, embeddable, in-process graph store that:

1. Speaks a **Wikidata-compatible entity format** (labels, descriptions, aliases, typed attributes)
2. Is **file-backed** — content-addressed, diffable, serializable to a local directory
3. Works **in-memory first** — no server, no external DB required for pipeline use
4. Is **domain-agnostic** — the same protocol handles code graphs, city knowledge graphs, or any entity-relation data

### What exists today

| Tool | Entity Format | Embeddable? | Versioned? |
|---|---|---|---|
| **Wikibase** | Wikibase JSON (canonical) | No — PHP/MediaWiki stack | Yes (revisions) |
| **Oxigraph** | RDF/SPARQL | Yes (pyoxigraph) | No |
| **TerminusDB** | JSON-LD/RDF | No — server process | Yes (git-like branching) |
| **qwikidata** | Wikibase JSON (read-only) | Yes | No |
| **KuzuDB/Neo4j** | Property graph (Cypher) | Yes/No | No |

**The gap:** A Wikibase JSON-compatible entity store that is embeddable, file-versioned, and composable across multiple domain projects.

DCG fills this gap.

---

## 2. Design Principles

> **Implements:** foundational principles governing all requirements in [DCG-001](../rfc/01-rfc.md), [DCG-001-COMP](../rfc/02-rfc-composition.md), [DCG-001-PACK](../rfc/03-rfc-packs.md)

1. **Wikibase JSON-compatible** — entity structure follows the [Wikibase JSON spec](https://doc.wikimedia.org/Wikibase/master/php/docs_topics_json.html). The compact DCG format maps bidirectionally to Wikibase Action API JSON.
2. **In-memory primary** — `GraphStore` is a dict-backed in-process store. No disk, no server.
3. **File persistence optional** — `DomainProject` serializes to a `graph_card.json` + `graphs/` directory layout.
4. **Protocol-first** — Python Protocols define the contract. Implementations are pluggable.
5. **Domain-agnostic** — the core knows nothing about code, cities, or any specific domain. Consumers register their own ontology entries.
6. **Wikidata semantics** — typing via "instance of" (P31), hierarchy via "subclass of" (P279), English relation keys.
7. **Composition over federation** — multiple domain projects compose into a unified stack via a YAML manifest and ontology-governed join rules.

---

## 3. Entity Model Rationale

> **Implements:** R-001 through R-011 [DCG-001] — entity structure, content-addressed IDs

### 3.1 Wikibase JSON Compatibility

DCG entities follow the Wikibase JSON structure with targeted simplifications for embedded use. The format is a subset that any Wikibase-aware tool can read. The compact DCG format expands losslessly to full Wikibase Action API JSON.

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

#### DCG compact format

```json
{
  "id": "dcg:sha256:a3f7c2d1e4b89012f4c5d6e7a8b90123",
  "label": "process_payment",
  "description": "Processes a credit card payment",
  "aliases": ["processPayment"],
  "attributes": [
    {"property": "instance of", "ref": "dcg:meta:Function"},
    {"property": "part of", "ref": "dcg:sha256:b1c4e8f2a7d30956"},
    {"property": "file path", "string": "src/billing/charge.py"},
    {"property": "line number", "quantity": 42}
  ]
}
```

The `label` field expands to `{"en": {"language": "en", "value": "..."}}`, `description` similarly, and `aliases` to a Wikibase aliases structure. This compact → full expansion is lossless.

### 3.2 Key Differences from Wikibase

| Aspect | Wikibase | DCG |
|---|---|---|
| Entity IDs | `Q` + integer (`Q42`) | Self-describing content hash (`dcg:sha256:a3f7c2d1...`) |
| Property IDs | `P` + integer (`P31`) | English keys (`instance of`) |
| Statement IDs | Random GUID | Omitted (derived from entity + property) |
| Qualifiers | Full snak structure | Flat array format |
| References | Full snak groups | Same (optional) |
| Sitelinks | Wikipedia page links | Omitted |
| `lastrevid` / `modified` | MediaWiki revision | Omitted (file/git provides this) |
| Schema version | Implicit | Explicit `schema_version` field |

**Why English property keys instead of P-IDs:** DCG is designed for developer consumption, not Wikipedia. `"instance of"` is immediately readable; `"P31"` requires a lookup. For Wikidata interop, a property alias registry maps between the two.

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

Same identity keys always produce the same UID. This enables deduplication across composition layers and deterministic identity.

**Relation UIDs** follow the same scheme, with prefix `dcg:rel:sha256:` computed from `property=<name>:source=<uid>:target=<uid>` sorted and hashed.

---

## 4. Relation Model

> **Implements:** R-017 through R-031 [DCG-001] — attribute format, property keys, value types, qualifiers

Relations in DCG are expressed as attributes on the source entity, following the flat attribute array format. The target is referenced by entity ID via a `ref` typed value.

```json
{
  "id": "dcg:sha256:a3f7c2d1e4b89012f4c5d6e7",
  "label": "process_payment",
  "description": "Processes a credit card payment",
  "attributes": [
    {
      "property": "calls",
      "ref": "dcg:sha256:def456a1b2c3d4e5f6a7b8c9",
      "qualifiers": [
        {"property": "line number", "quantity": 55}
      ]
    }
  ]
}
```

### 4.1 Standalone Relation Objects

For consumers that prefer explicit edge objects (graph DBs, visualization tools), DCG also supports standalone relation objects stored in the `relations` map of a graph data file:

```json
{
  "dcg:type": "relation",
  "uid": "dcg:rel:sha256:c8d9e0f1a2b34567",
  "source": "dcg:sha256:a3f7c2d1e4b89012f4c5d6e7",
  "target": "dcg:sha256:def456a1b2c3d4e5f6a7b8c9",
  "property": "calls",
  "qualifiers": [{"property": "line number", "quantity": 55}]
}
```

The standalone format is a denormalized view of the attribute — both representations are equivalent and convertible. Cross-layer relations produced by join rules use this format with `"dcg:cross_layer": true` added.

### 4.2 Property Registry

Properties are registered with English keys, descriptions, and expected value types:

```python
PROPERTY_REGISTRY = {
    # Wikidata-universal
    "instance of":      {"datatype": "ref", "wikidata": "P31",
                         "description": "entity is an instance of this type"},
    "subclass of":      {"datatype": "ref", "wikidata": "P279",
                         "description": "type hierarchy"},
    "part of":          {"datatype": "ref", "wikidata": "P361",
                         "description": "entity belongs to a domain or composite"},
    "depends on":       {"datatype": "ref", "wikidata": "P1269",
                         "description": "domain or entity depends on another"},
    "triggers":         {"datatype": "ref",
                         "description": "entity causes an effect in another entity"},

    # Code-specific
    "calls":            {"datatype": "ref",
                         "description": "function invokes another function"},
    "contains":         {"datatype": "ref",
                         "description": "parent contains child"},
    "imports":          {"datatype": "ref",
                         "description": "file imports a module"},
    "inherits from":    {"datatype": "ref",
                         "description": "class extends another class"},
    "handles route":    {"datatype": "ref",
                         "description": "function handles an HTTP endpoint"},
    "tested by":        {"datatype": "ref",
                         "description": "production code tested by test function"},
    "vulnerability in": {"datatype": "ref",
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

Consumers extend the registry at runtime (see §10.2).

---

## 5. Domains, Types, and Ontology

> **Implements:** R-037 through R-046 [DCG-001] — domains as entities, entity types, two-axis classification, domain hierarchy (rationale; ontology mechanics R-047–R-052 in §10)

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
from dcg.core import GraphStore, entity_uid, builders as wb

store = GraphStore()

# Domains are entities — created just like any other entity
payments_uid = entity_uid(domain="payments")
store.add_entity(wb.entity(
    uid=payments_uid,
    label="Payments",
    description="Payment processing, billing, and financial transactions",
    attributes=[
        wb.attribute("instance of", wb.ref_value("dcg:meta:Domain")),
    ],
))

inventory_uid = entity_uid(domain="inventory")
store.add_entity(wb.entity(
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
store.add_entity(wb.entity(
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
store.add_entity(wb.entity(
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

### 5.3 Structural Types

Entity types are **structural** — they describe what an entity IS, independent of which domain it belongs to. A Function is a Function whether it lives in Payments or Inventory.

The built-in (`ontology_builtin`) defines the mandatory core:

```python
TYPE_REGISTRY = {
    # Mandatory meta-types (all implementations)
    "dcg:meta:Domain":         {"label": "Domain",
                                "description": "A real-world area of concern"},
    "dcg:meta:Type":           {"label": "Type",
                                "description": "A classification of entities"},
}
```

Domain-specific structural types (Function, Class, Endpoint, Vulnerability, etc.) are provided via ontology packs (see §10.6) or registered by consumers. The "code" pack ships:

```python
# "code" ontology pack — not built-in, declared in graph_card.json packs
"dcg:meta:Function":       {"label": "Function", "description": "A callable code unit"},
"dcg:meta:Class":          {"label": "Class",    "description": "An object-oriented class"},
"dcg:meta:Struct":         {"label": "Struct",   "description": "A composite data type"},
"dcg:meta:Interface":      {"label": "Interface","description": "An interface contract"},
"dcg:meta:Trait":          {"label": "Trait",    "description": "A trait/mixin type"},
"dcg:meta:File":           {"label": "File",     "description": "A source file"},
"dcg:meta:Module":         {"label": "Module",   "description": "An importable module"},
"dcg:meta:Endpoint":       {"label": "Endpoint", "description": "An API endpoint"},
```

Registering new structural types at runtime:

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

This separation enables:
- **Querying by type** across all domains: `store.query(instance_of="dcg:meta:Function")`
- **Querying by domain**: `store.query(part_of=payments_uid)`
- **Cross-cutting queries**: intersect domain membership with type filter
- **Domain impact analysis**: `store.get_relations(source=payments_uid, property="depends on")`

### 5.5 Domain Hierarchy

Domains can nest. A top-level domain (domain root) has no parent domain. Sub-domains declare themselves via `"part of"` pointing to their parent domain:

```
Security (top-level domain, domain root)
├── OWASP Categories (sub-domain, part of Security)
│   ├── A01: Broken Access Control (entity, part of OWASP Categories)
│   └── A03: Injection (entity, part of OWASP Categories)
├── CWE Weaknesses (sub-domain, part of Security)
│   ├── CWE-79: XSS (entity, part of CWE Weaknesses)
│   └── CWE-89: SQL Injection (entity, part of CWE Weaknesses)
└── Vulnerability Instances (sub-domain, part of Security)
    └── CVE-2024-1234 (entity, part of Vulnerability Instances)
```

A consumer querying `part_of=Security` sees the top-level domain and its direct members. The complexity of 50+ CWE entries is hidden behind the "Security" top-level domain until explicitly drilled into.

---

## 6. Wikidata Compatibility

> **Implements:** Appendix B [DCG-001] — Wikidata interoperability conventions

### 6.1 What Works Out of the Box

- `qwikidata` can read DCG entity JSON — after compact-to-full expansion, the top-level structure matches Wikibase
- Labels, descriptions, aliases follow exact Wikibase format after expansion
- Attributes use the same semantic model as Wikibase claims (subject-property-value with optional qualifiers)

### 6.2 What Differs

- Entity IDs use `dcg:` prefix instead of `Q` numbers
- Property keys are English strings instead of `P` numbers
- `sitelinks` are omitted
- `lastrevid`/`modified` are omitted (file system / git provides this)
- Schema version is an extension field

### 6.3 Wikidata Property Mapping

For entities that correspond to Wikidata items, the property alias registry includes `wikidata_P<ID>` aliases:

```python
# DCG English key → Wikidata P-ID alias
"instance of"  → wikidata alias: "wikidata_P31"
"subclass of"  → wikidata alias: "wikidata_P279"
"part of"      → wikidata alias: "wikidata_P361"
"population"   → wikidata alias: "wikidata_P1082"
```

The built-in ontology ships these aliases. Querying for `wikidata_P31` resolves to the same results as querying `instance of`. A future `export_to_wikidata()` function could convert DCG entities to Wikidata-compatible format by replacing English keys with P-IDs.

### 6.4 Import from Wikidata

Implementations may use `qwikidata` or equivalent to import Wikidata entities from offline JSON dumps. The adapter converts Wikibase JSON to DCG compact format, mapping `P31` → `instance of`, `P279` → `subclass of`, etc. via the alias registry.

---

## 7. Integration Patterns

> **Implements:** informative — no direct RFC requirements; design rationale for consumer integration points

DCG is domain-agnostic. Consumers plug in via three patterns:

### 7.1 Ontology Extension

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
store.add_entity(wb.entity(
    uid=geo_uid, label="Geospatial",
    description="Physical locations and spatial relations",
    attributes=[wb.attribute("instance of", wb.ref_value("dcg:meta:Domain"))],
))
```

### 7.2 Translator Pattern

Each consumer provides a translator that converts domain-specific data to DCG format:

```python
class Translator(Protocol):
    def translate(self, raw_data: Any, store: GraphStore) -> int:
        """Translate domain data into entities/relations in the store.
        Returns count of entities added."""
        ...
```

**Code translator** example (in `codegraphcontext/graphlayer/translators/code.py`):

```python
class CodeTranslator:
    def translate(self, file_data: dict, store: GraphStore) -> int:
        """Translate code parser output → DCG entities."""
        count = 0
        for fn in file_data.get("functions", []):
            store.add_entity(wb.entity(
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
        return count
```

### 7.3 Materializer Pattern

Materializers read from `GraphStore` and write to external systems:

```python
class Materializer(Protocol):
    def materialize(self, store: GraphStore) -> int:
        """Write graph state to an external system. Returns count written."""
        ...
```

**KuzuDB materializer** example:

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

## 8. Examples

### 8.1 Code Graph (CGC)

```python
store = GraphStore()

# Pass 1: Parse code
for file_data in all_file_data:
    code_translator.translate(file_data, store)

# Pass 2: Extract OpenAPI specs
for spec_file in openapi_files:
    openapi_translator.translate(spec_file, store)

# Pass 3: Resolve cross-domain edges
resolver.resolve(store)  # adds "handles route", "tested by" relations

# Materialize to DB
kuzu_materializer.materialize(store, db_manager)

# Or export to JSON
store.to_json(Path("code-graph.json"))
```

### 8.2 City Knowledge Graph (Multi-Domain)

```python
# Two domain projects — Urban Planning and Transit
urban_project = DomainProject(".dcg/urban/")
transit_project = DomainProject(".dcg/transit/")

register_global_type("dcg:meta:Building", label="Building")
register_global_type("dcg:meta:Road", label="Road")
register_global_type("dcg:meta:BusStop", label="Bus Stop")
register_property("address", datatype="string")
register_property("nearest to", datatype="ref")

# Create domain entities
urban_uid = entity_uid(domain="urban-planning")
urban_project.add_entity(wb.entity(
    uid=urban_uid, label="Urban Planning",
    description="Zoning, buildings, and land use",
    attributes=[wb.attribute("instance of", wb.ref_value("dcg:meta:Domain"))],
))

transit_uid = entity_uid(domain="transit")
transit_project.add_entity(wb.entity(
    uid=transit_uid, label="Public Transit",
    description="Bus routes, stops, and schedules",
    attributes=[wb.attribute("instance of", wb.ref_value("dcg:meta:Domain"))],
))

# Pass 1: OSM building data → Urban Planning domain
for building in osm_buildings:
    urban_project.add_entity(wb.entity(
        uid=entity_uid(source="osm", osm_id=building["id"]),
        label=building["name"],
        attributes=[
            wb.attribute("instance of", wb.ref_value("dcg:meta:Building")),
            wb.attribute("part of", wb.ref_value(urban_uid)),
            wb.attribute("address", wb.string_value(building["address"])),
        ],
    ))
urban_project.save()

# Pass 2: Transit data → Transit domain
for stop in transit_stops:
    transit_project.add_entity(wb.entity(
        uid=entity_uid(source="transit", stop_id=stop["id"]),
        label=stop["name"],
        attributes=[
            wb.attribute("instance of", wb.ref_value("dcg:meta:BusStop")),
            wb.attribute("part of", wb.ref_value(transit_uid)),
        ],
    ))
transit_project.save()
```

### 8.3 File Persistence

```python
# Use DomainProject instead of bare GraphStore
project = DomainProject(".dcg/")

# Same GraphStore API — save() writes JSON files
project.add_entity(...)
project.save()
# → .dcg/graph_card.json and .dcg/graphs/default.json updated

# Later: reload from disk
project2 = DomainProject(".dcg/")
project2.load()
```

---

# Part II: Runtime Implementation Guidance

---

## 9. In-Memory Store

> **Implements:** R-064 through R-083 [DCG-001] — GraphStoreProtocol, operations, retraction, redirects, I/O

The primary store is in-memory. No disk, no server. Optimized for pipeline use.

### 9.1 GraphStoreProtocol

```python
from typing import Protocol, runtime_checkable

@runtime_checkable
class GraphStoreProtocol(Protocol):
    """Core protocol — any implementation must satisfy this."""

    def add_entity(self, entity: dict) -> str: ...
    def add_relation(self, source: str, target: str, property: str,
                     qualifiers: list | None = None) -> str: ...
    def remove(self, uid: str) -> None: ...

    def get_entity(self, uid: str) -> dict | None: ...
    def get_relations(self, source: str = None, target: str = None,
                      property: str = None) -> list[dict]: ...
    def query(self, instance_of: str = None, part_of: str = None,
              **property_filters) -> list[dict]: ...

    def to_dict(self) -> dict: ...
    def load(self, path) -> None: ...
    def to_json(self, path) -> None: ...
```

### 9.2 GraphStore (dict-backed implementation)

```python
class GraphStore:
    """In-memory graph store."""

    def __init__(self):
        self._entities: dict[str, dict] = {}       # uid → entity dict
        self._relations: dict[str, dict] = {}       # uid → standalone relation

    def add_entity(self, entity: dict) -> str:
        """Add or update an entity. Idempotent — updates entity in place. Returns UID."""
        uid = entity["id"]
        self._entities[uid] = entity
        return uid
```

### 9.3 Operation Semantics

**Guidance:**

1. `add_entity()` is idempotent — adding an entity with an existing UID replaces the stored object in full (R-066).
2. `get_entity()` follows entity redirects transparently, up to a depth of 10 (R-067, R-079). Returns `None` for retracted or missing entities (R-071).
3. `get_relations()` resolves property aliases before matching (R-068). Excludes retracted relations (R-073).
4. `query()` supports filtering by `instance_of` and `part_of` independently and in combination (R-069). Excludes retracted entities (R-072).
5. `remove(uid)` marks the entity or relation as retracted (`"retracted": true`) — it does not physically delete it (R-070).
6. `purge_retracted()` permanently removes all retracted entities and relations, except redirect source tombstones (R-074, R-080).
7. Attribute-level retraction is distinct from entity-level retraction (R-032–R-035 [DCG-001]). To retract a single attribute, add `"retracted": true` to that attribute object in the attributes array — the entry remains in place as a soft-deleted marker. The identity tuple for a retracted attribute is `(property, typed_key, value)` so it can be correlated on re-load. `to_dict()` MUST include retracted attribute entries in the output (they must survive round-trips until explicitly purged). After `purge_retracted()`, no `"retracted": true` flags MUST remain on attributes — redirect tombstones are the only retraction-related entries exempt from purge (R-036 [DCG-001]).
8. After `purge_retracted()` is called, a subsequent `load()` from the persisted graph files MUST NOT resurrect retracted entities, relations, or attributes. The persistence layer (see §13) is responsible for ensuring retracted records are omitted from written files before any reload occurs (R-075 [DCG-001]).
9. `purge_retracted()` MUST operate on a single domain project's store only. There is no implicit cross-layer purge. When operating on a stack, tooling MUST iterate each composition layer's store and call `purge_retracted()` on each independently (R-076 [DCG-001]).

### 9.4 Usage Example

```python
from dcg.core import GraphStore, entity_uid

store = GraphStore()

# Add a function entity
fn_uid = entity_uid(name="list_pets", path="src/main.py", line_number=42)
store.add_entity({
    "id": fn_uid,
    "label": "list_pets",
    "description": "Lists all pets",
    "attributes": [
        {"property": "instance of", "ref": "dcg:meta:Function"},
        {"property": "file path", "string": "src/main.py"},
    ],
})

# Add a relation
store.add_relation(fn_uid, other_fn_uid, "calls",
                   qualifiers=[{"property": "line number", "quantity": 55}])

# Query
functions = store.query(instance_of="dcg:meta:Function")
callers = store.get_relations(target=fn_uid, property="calls")

# Export
store.to_json(Path("graph.json"))
```

---

## 10. Ontology Runtime

> **Implements:** R-041 through R-052 [DCG-001] — see sub-sections for per-range claims; DCG-001-COMP and DCG-001-PACK coverage is in §11 (Stack Loader)

### 10.1 Built-in Ontology (`ontology_builtin`)

> **Implements:** R-047 through R-048 [DCG-001] — mandatory shipped ontology, always available

The `ontology_builtin` is the always-loaded base ontology. It is not an opt-in pack.

**Guidance:**

1. The `ontology_builtin` MUST define at minimum `dcg:meta:Domain` and `dcg:meta:Type` (the two mandatory types per R-042).
2. The `ontology_builtin` SHOULD include the core property vocabulary: `"instance of"` (P31), `"part of"` (P361), `"subclass of"` (P279), with Wikidata alias mappings.
3. The `ontology_builtin` MUST be available in every conforming implementation without any user declaration — it loads before any project or pack configuration.
4. Domain-specific types (Function, Class, Vulnerability, etc.) belong in ontology packs, not `ontology_builtin`. This keeps the core lean and extensible.

### 10.2 Entity Type Registration

> **Implements:** R-041 through R-043 [DCG-001] — structural types, runtime registration API

**Guidance:**

1. Implementations MUST expose a `register_global_type(type_id, label, description)` API for registering new structural types at runtime.
2. `register_global_type()` SHOULD reject duplicate registrations with different semantics unless an explicit `overwrite=True` parameter is passed.
3. New types registered this way follow the same `dcg:meta:` prefix convention (e.g., `dcg:meta:Building`).
4. Types registered at runtime are project-scoped and SHOULD be persisted in `graph_card.json` under the `ontology.types` key (not in `ontology_builtin`).

```python
from dcg.core import register_global_type, register_property

register_global_type("dcg:meta:Building",
                     label="Building", description="A physical structure")
register_property("population", datatype="quantity",
                  description="number of inhabitants", wikidata="P1082")
```

### 10.3 Project Load Order

> **Implements:** R-049 through R-051 [DCG-001] — per-project ontology, persistence

**Guidance:**

1. When loading a domain project, the implementation MUST first load the `packs` declared in `graph_card.json` (see §10.6), then load the `ontology` key custom declarations, merging into the project's ontology registry (the combined set of types, properties, and aliases from `ontology_builtin` + packs + custom declarations).
2. Load order within a domain project is: `ontology_builtin` → declared packs (array order) → custom `ontology` section.
3. When saving (`to_dict()`), only custom declarations (not `ontology_builtin` contents, not pack contents) MUST be written to the `ontology` key in `graph_card.json`. The `packs` key MUST be persisted so packs are re-loaded on deserialization.
4. On deserialization (`load()`), the implementation MUST first load packs from the `packs` key, then load the `ontology` key, restoring the full ontology state.

### 10.4 Stack Ontology Merge

> **Implements:** R-010, R-012 [DCG-001-COMP] — ontology merge semantics only; DAG validation and load order in §11.2

**Guidance:**

1. When a stack is loaded, layer ontologies MUST be merged in topological order (parents before children) (R-011 [DCG-001-COMP]).
2. When the same type name, property name, or alias is declared in multiple layers with **different definitions**, implementations MUST log a warning and use the last-declared definition — they MUST NOT raise an error (R-010 [DCG-001-COMP]).
3. Identical re-declarations across layers MUST be silently accepted (no warning).
4. Built-in ontology declarations MUST NOT trigger warnings when a user declaration shares the same name — user declarations silently win (R-012 [DCG-001-COMP]).

The merge order (later overrides earlier):
```
1. ontology_builtin (core pack, auto-loaded)
2. Per-layer ontology packs, in declaration order
3. Per-layer graph_card.json custom ontology, in topological load order
```

### 10.5 Strict Mode

> **Implements:** R-044 through R-045 [DCG-001] — validate_ontology(), permissive writes

**Guidance:**

1. Strict mode is a validation mode, not a write guard. `add_entity()` and `add_relation()` MUST accept any type, property, or relation regardless of whether strict mode is enabled — writes are always permissive.
2. Implementations MUST provide a `validate_ontology()` method that inspects all entities and relations in the store and returns a list of violation descriptions (empty list = valid).
3. `validate_ontology()` checks MUST include:
   - Entity `"instance of"` references an unregistered entity type
   - Entity attributes reference unregistered property names
   - Non-Domain, non-Type entities with fewer than 2 entity-linking attributes (i.e., ref-typed attributes such as `"instance of"` and `"part of"`)
4. When strict mode is enabled at the stack level (`strict: true` in `dcg-stack.yml`), implementations SHOULD run `validate_ontology()` at load time and log violations as warnings (not errors).
5. When strict mode is disabled (default), validation is not automatically run but MAY be invoked explicitly.

```python
# Explicit validation — works in any mode
violations = store.validate_ontology()
for v in violations:
    print(f"Violation: {v}")
```

### 10.6 Ontology Pack Loading

> **Implements:** R-006 through R-008, R-013 through R-015 [DCG-001-PACK] — pack loading, recommended built-ins, name validation, per-layer stack loading

**Guidance:**

1. Packs are declared per domain project in `graph_card.json` via the `packs` key. There is no stack-level `packs` key — each domain project declares its own packs independently.
2. Implementations SHOULD ship the following RECOMMENDED built-in packs:
   - `"code"` — source code analysis types (Function, Class, Struct, Interface, Trait, File, Module, Endpoint) and relations (calls, imports, inherits from, handles route, tested by, etc.)
   - `"security"` — security domain types (Vulnerability, WeaknessType, AttackType) and relations (vulnerability in, exploits, mitigates)
   - `"dcg-development"` — development tooling vocabulary
3. Packs MUST be loaded before the per-project custom ontology so that custom declarations can reference or override pack-provided types and properties.
4. Each domain project's ontology MUST be fully self-contained between its packs and its custom ontology section — it MUST NOT depend on another domain project's pack declarations.
5. When a pack name in the `packs` array cannot be resolved, the implementation MUST raise an error that includes the unknown pack name and lists the available pack names (R-008 [DCG-001-PACK]).
6. Pack names MUST match the regex `[a-z][a-z0-9-]*` and are case-sensitive. Implementations MUST validate pack names at declaration time and reject names that do not conform (R-013 [DCG-001-PACK]).
7. When a stack is loaded, each layer's packs are loaded independently as part of that layer's own ontology setup, before the stack-level ontology merge described in §10.4. Each layer brings its own pack-derived vocabulary into the merge (R-014 [DCG-001-PACK]).
8. When the same pack is declared by multiple layers in a stack, the pack content is loaded once per layer. The resulting per-layer pack content enters the stack-level ontology merge using the same merge semantics as custom declarations — later layers override earlier layers on conflict, with a warning (R-015 [DCG-001-PACK]).

```json
{
  "dcg_project": {"name": "my-code-graph", "version": "0.1.0"},
  "packs": {
    "ontology": ["code", "security"]
  },
  "ontology": {
    "types": [{"name": "MyCustomType", "description": "..."}]
  }
}
```

---

## 11. Stack Loader

> **Implements:** R-001 through R-036 [DCG-001-COMP] — stack manifest parsing, DAG validation, join rules, join materialization, entity resolution, cross-layer queries, composition guarantees

The Stack Loader is the runtime component responsible for loading a `dcg-stack.yml`, validating the composition DAG, loading each layer's domain project, materializing join rules, and exposing a unified cross-layer query interface. It is the primary entry point for any multi-layer DCG composition.

### 11.1 Stack Manifest Parsing

> **Implements:** R-001 through R-008 [DCG-001-COMP] — manifest format, required fields, layer entries, uniqueness, acyclicity

**Guidance:**

1. Parse the manifest from a YAML file (conventionally `dcg-stack.yml`). The YAML loader MUST treat the file as strict UTF-8 and reject non-YAML content (R-001 [DCG-001-COMP]).
2. After parsing, validate that the top-level object contains the required fields `stack` (string) and `layers` (array). A manifest missing either MUST be rejected with a descriptive error naming the missing field (R-002 [DCG-001-COMP]).
3. The manifest MUST NOT contain a top-level `ontology` key. Ontology declarations belong in each layer's `graph_card.json`. If an `ontology` key is found at the manifest level, reject the manifest with an error (R-003 [DCG-001-COMP]).
4. Optional manifest-level fields are `strict` (boolean, default `false`) and `joins` (array of join rule objects). Any other unrecognized top-level key SHOULD produce a warning but MUST NOT be treated as a fatal error (R-003 [DCG-001-COMP]).
5. For each entry in the `layers` array, validate that `name` (string) and `source` (string path) are present. Layer entries missing either field MUST be rejected (R-004 [DCG-001-COMP]).
6. Each layer entry MAY contain an `extends` field — an array of layer name strings identifying parent layers for entity resolution (R-005 [DCG-001-COMP]).
7. Layer names MUST be unique within the manifest. After parsing all layer entries, check for duplicates and reject the manifest if any two layers share a `name` (R-006 [DCG-001-COMP]).
8. Every name that appears in any layer's `extends` list MUST match a layer name declared in the same manifest. Unresolvable extends references MUST be rejected at parse time, before any project data is loaded (R-007, R-008 [DCG-001-COMP]).

**Example `dcg-stack.yml`:**

```yaml
stack: appsec-ctx-graph
strict: true

layers:
  - name: security
    source: ./dcg-security-domain

  - name: product
    source: ./dcg-product-domain
    extends: [security]

  - name: compliance
    source: ./dcg-compliance
    extends: [security]

  - name: infrastructure
    source: ./dcg-infrastructure
    extends: [product, compliance]

joins:
  - mitigates: SecurityCapability.targets_cwe -> WeaknessType.cwe_id
  - triggers:  AttackType.exploits_cwe -> WeaknessType.cwe_id
```

### 11.2 DAG Validation and Load Order

> **Implements:** R-009 through R-012 [DCG-001-COMP] — reference validation, cycle detection, load order, join rule validation

**Guidance:**

1. Before loading any project data, perform a topological sort on the extends DAG. Use the `layers` array order as the canonical declaration order and as the tiebreaker for BFS (R-009, R-010 [DCG-001-COMP]).
2. If a cycle is detected during topological sort, reject the stack with an error that names every layer involved in the cycle. Do not partially load layers before the cycle is confirmed (R-010 [DCG-001-COMP]).
3. Load each layer's `DomainProject` (graph data + ontology) in topological order — parents before children. Each layer's packs are loaded as part of that layer's setup (see §10.6 point 7), then ontologies are merged incrementally so that child layers can reference types and properties declared in parent layers (R-011 [DCG-001-COMP]).
4. After all layers are loaded and ontologies are merged, validate every type and property name referenced in `joins` rules against the merged ontology. Any join rule referencing an undeclared type or property MUST cause the stack load to fail with an error that identifies the specific undeclared name (R-012 [DCG-001-COMP]).
5. The `strict` flag at the manifest level propagates to per-layer validation: when `strict: true`, run `validate_ontology()` for each layer after loading and log all violations. Violations in strict mode SHOULD be warnings, not fatal errors, to allow partially populated graphs to be inspected.

### 11.3 Join Rule Parser

> **Implements:** R-013 through R-017 [DCG-001-COMP] — join rule format, expression grammar, dot constraint, full grammar enforcement, layer prefix validation

**Guidance:**

1. Each entry in the `joins` array MUST be a single-key YAML mapping. The key is the relation property name (the relation that will be materialized); the value is the join expression string. Entries with more than one key MUST be rejected (R-013 [DCG-001-COMP]).
2. Parse the join expression string using the grammar defined in R-016 [DCG-001-COMP]:
   ```
   join_expr  := type_prop SP "->" SP type_prop
   type_prop  := [LayerName "."] TypeName "." prop_name
   LayerName  := [a-z][a-z0-9_-]*
   TypeName   := [A-Z][A-Za-z0-9_-]*
   prop_name  := [a-z][a-z0-9_]*
   ```
   Disambiguation between an optional `LayerName` prefix and the `TypeName` uses the case rule: layer names begin with a lowercase letter, type names begin with an uppercase letter. Implement this as a regex or a simple two-pass tokenizer that splits on `.` and classifies tokens by first character (R-014, R-015, R-016 [DCG-001-COMP]).
3. The `.` character MUST NOT appear in entity type names, property names, or layer names. Validate all names after splitting and reject expressions where any split token contains a `.` (R-015 [DCG-001-COMP]).
4. Join expressions that do not match the grammar MUST be rejected at parse time with a human-readable error message that includes the expression string and the point of failure (R-016 [DCG-001-COMP]).
5. When a layer prefix is present in a join expression, the named layer MUST exist in the stack's `layers` array. Reject the stack if a join rule references an undeclared layer (R-017 [DCG-001-COMP]).
6. Build a structured `JoinRule` object for each parsed join expression. The `JoinRule` holds: `property` (string), `from_layer` (string or `None`), `from_type` (string), `from_prop` (string), `to_layer` (string or `None`), `to_type` (string), `to_prop` (string).
7. Note on dual naming: property names in join expressions (`prop_name` grammar — lowercase with underscores, e.g., `cwe_id`) are the short identifiers used in join rules. These differ from the English property keys used in entity wire format (e.g., `"cwe id"`). Implementations SHOULD document the mapping between join-expression identifiers and wire-format property keys so that authors of join rules know which identifier to use for a given property.

### 11.4 Join Materialization Algorithm

> **Implements:** R-018 through R-022 [DCG-001-COMP] — 4-step evaluation, relation JSON fields, read-only, separate storage, multi-value

**Guidance:**

1. After all layers are loaded and join rules are parsed and validated, execute join materialization. For each `JoinRule`, apply the following 4-step algorithm (R-018 [DCG-001-COMP]):
   - **Step 1 — Collect source candidates:** Query for all entities whose `"instance of"` attribute matches `from_type`. If `from_layer` is set, restrict to that layer's store; otherwise search all layers' stores.
   - **Step 2 — Collect target candidates:** Query for all entities whose `"instance of"` attribute matches `to_type`. If `to_layer` is set, restrict to that layer's store; otherwise search all layers' stores.
   - **Step 3 — Build index:** Index target candidates by the value of their `to_prop` property. For multi-value properties, index once per value.
   - **Step 4 — Emit relations:** For each source entity, retrieve the value(s) of its `from_prop` property and look up each value in the target index. For each match, emit one cross-layer relation object.
2. Each materialized cross-layer relation MUST be a JSON object with the following fields (R-019 [DCG-001-COMP]):
   - `"dcg:type"`: `"relation"`
   - `"uid"`: deterministic UID computed from source ID, target ID, and property using the standard relation UID scheme (see §3.3)
   - `"source"`: UID of the source entity
   - `"target"`: UID of the target entity
   - `"property"`: the relation property name from the join rule (e.g., `"mitigates"`)
   - `"dcg:cross_layer"`: `true`
   - `"dcg:matched_on"`: the property value that caused the match (e.g., `"CWE-20"`)
3. Cross-layer relations are read-only computed artifacts — they MUST NOT be written to any layer's graph data files. They are recomputed on every stack load (R-020 [DCG-001-COMP]).
4. Store cross-layer relations in a separate runtime collection from each layer's intra-layer relations, so that callers can distinguish them by the presence of `"dcg:cross_layer": true` (R-021 [DCG-001-COMP]).
5. A source entity with multiple values for `from_prop` (multi-value attributes) MUST produce one materialized cross-layer relation per matching `(from_prop_value, target)` pair (R-022 [DCG-001-COMP]).

**Example — materialized relation:**

```json
{
  "dcg:type": "relation",
  "uid": "dcg:rel:sha256:c8d9e0f1a2b34567",
  "source": "dcg:sha256:wafsanitizer...",
  "target": "dcg:sha256:inputval...",
  "property": "mitigates",
  "dcg:cross_layer": true,
  "dcg:matched_on": "CWE-20"
}
```

### 11.5 Entity Resolution

> **Implements:** R-023 through R-027 [DCG-001-COMP] — BFS algorithm, first-found wins, redirect following, None return, cycle tracking

**Guidance:**

1. Implement `resolve(layer_name, uid)` as a separate operation from per-layer `get_entity()`. Its purpose is to find an entity by UID across the extends DAG starting from the named layer (R-023 [DCG-001-COMP]).
2. The algorithm is breadth-first search. Starting at the named layer, visit parent layers in the order declared in that layer's `extends` list. Use a queue initialized with `[layer_name]` (R-023 [DCG-001-COMP]).
3. At each layer in the BFS traversal, call `get_entity(uid)` on that layer's store. If the entity is found, follow any redirects within that layer (as specified by R-077–R-081 [DCG-001]) and return the resolved entity (R-024, R-025 [DCG-001-COMP]).
4. If the entity is not found in the current layer, enqueue that layer's parent layers (in `extends` declaration order) for the next BFS iteration, skipping any already-visited layers.
5. If the BFS traversal completes without finding the entity, return `None` (R-026 [DCG-001-COMP]).
6. Maintain a `visited` set of layer names throughout the traversal. Before visiting any layer, check the `visited` set and skip already-visited layers. This guarantees termination even if the DAG contains a runtime cycle (R-027 [DCG-001-COMP]).

**Example BFS traversal** for `resolve("infrastructure", uid)` with `infrastructure` extending `[product, compliance]` and both `product` and `compliance` extending `[security]`:
```
Queue: [infrastructure]
  infrastructure → not found → enqueue [product, compliance]
Queue: [product, compliance]
  product → not found → enqueue [security] (skip if already visited)
  compliance → not found → security already enqueued
Queue: [security]
  security → FOUND → return entity
```

### 11.6 Cross-Layer Queries

> **Implements:** R-028 through R-032 [DCG-001-COMP] — manifest order, UID deduplication, no attribute merge, layer filter, cross-layer relations in results

**Guidance:**

1. Cross-layer `query()` iterates layers in **manifest declaration order** (the order entries appear in the `layers` array), not extends DAG order (R-028 [DCG-001-COMP]).
2. Collect results from each layer's store in declaration order. When the same entity UID appears in more than one layer, the entity from the **first-declared layer** is used and subsequent occurrences are discarded (R-029 [DCG-001-COMP]).
3. Cross-layer queries MUST NOT merge attributes across layers. The entity dict returned for a given UID is the complete, unmodified dict from the winning layer's store. No attribute from a later layer is added to it (R-030 [DCG-001-COMP]).
4. Implement an optional `layers` filter parameter: when provided as a list of layer names, restrict the query to only those layers (in their declaration order). Layers named in the filter but not in the manifest SHOULD raise an error (R-031 [DCG-001-COMP]).
5. When returning relation results from a cross-layer query, include materialized cross-layer relations (those stored in the separate cross-layer collection from §11.4) alongside intra-layer relations. Callers can distinguish cross-layer relations by checking for `"dcg:cross_layer": true` in the relation object (R-032 [DCG-001-COMP]).

### 11.7 Composition Guarantees

> **Implements:** R-033 through R-036 [DCG-001-COMP] — independent layer loading, single-layer writes, save isolation, parent propagation

**Guidance:**

1. Each composition layer MUST be independently loadable at the file format level — a layer's `graph_card.json` and all graph data files in its `graphs/` directory MUST parse and validate without access to any other layer. Implementers SHOULD verify independent loadability during testing by loading each layer in isolation (R-033 [DCG-001-COMP]).
2. Writes to a stack MUST target one composition layer at a time. The stack loader MUST expose the concept of an "active layer" and route all `add_entity()`, `add_relation()`, and `remove()` calls to that layer's store exclusively (R-034 [DCG-001-COMP]).
3. `save()` on a stack MUST operate on the active composition layer only. No other layer's graph data files or `graph_card.json` MUST be modified by a `save()` call (R-035 [DCG-001-COMP]).
4. Cross-layer join connections are matched on stable property values (e.g., `cwe_id`, `owasp_id`) rather than UIDs. When a parent layer is rebuilt (new entities, new UIDs), join rules re-materialize automatically to the new UIDs on the next stack load. No changes to the stack manifest or child layer data are needed (R-036 [DCG-001-COMP]).

---

## 12. Query Behavior

> **Implements:** R-044 through R-046 [DCG-001] — two-axis query, instance_of, part_of

**Guidance:**

1. `query()` MUST support filtering by structural type (`instance_of`) using an exact match against the `"instance of"` attribute value.
2. `query()` MUST support filtering by domain membership (`part_of`) using the `"part of"` attribute.
3. Both filters MAY be combined: `query(instance_of="dcg:meta:Function", part_of=payments_uid)` returns only Functions in the Payments domain.
4. Without filters, `query()` MUST return all non-retracted entities in the store.
5. Property alias resolution MUST apply to filter arguments — querying `instance_of="wikidata_P31"` MUST resolve identically to `instance_of="instance of"` if the alias is registered.
6. Hierarchy traversal for domain queries (querying a top-level domain and receiving all sub-domain entities) is RECOMMENDED but not required by the core protocol. Implementations SHOULD support `part_of_recursive` or equivalent.

> **Cross-layer note:** The query behavior above applies to single-layer stores. When operating on a stack, cross-layer queries follow the additional semantics specified in §11.6: manifest declaration order iteration, UID deduplication (first-declared layer wins), no attribute merging across layers, optional layer filter, and inclusion of materialized cross-layer relations (R-028–R-032 [DCG-001-COMP]). See §11 Stack Loader for the full specification.

**Example queries:**

```python
# All functions across all domains
functions = store.query(instance_of="dcg:meta:Function")

# Everything in the Payments domain
payments_entities = store.query(part_of=payments_uid)

# Functions specifically in Payments
payment_fns = store.query(instance_of="dcg:meta:Function", part_of=payments_uid)

# Find what domains Payments depends on
domain_deps = store.get_relations(source=payments_uid, property="depends on")

# Find all callers of a function
callers = store.get_relations(target=fn_uid, property="calls")
```

---

## 13. Git Persistence

> **Implements:** R-053 through R-063 [DCG-001] — graph_card.json, graphs/ layout, intra-layer constraint

Implementations may provide a git-backed store (for example `GitGraphStore`) as an implementation of `GraphStoreProtocol` that persists each domain project as a git repository. The reference implementation ships this under `dcg/git/` (see §16).

### 13.1 Directory Layout

```
my-domain-project/
├── graph_card.json          # metadata + ontology declarations + graphs index
└── graphs/
    ├── default.json         # main graph data (entities + relations)
    └── secondary.json       # optional additional subgraph
```

`graph_card.json` is the sole entry point for project metadata and the ontology. It MUST NOT contain entity or relation data.

### 13.2 graph_card.json

```json
{
  "dcg_project": {
    "name": "payments-domain-graph",
    "version": "0.1.0",
    "description": "Payment processing and billing domain knowledge graph"
  },
  "packs": {
    "ontology": ["code"]
  },
  "ontology": {
    "types": [
      {"name": "PaymentMethod", "description": "A supported payment method"}
    ],
    "properties": [
      {"name": "gateway", "datatype": "string", "description": "Payment gateway provider"}
    ]
  },
  "graphs": [
    {"id": "default", "file": "graphs/default.json"}
  ]
}
```

### 13.3 Graph Data Files

```json
{
  "schema_version": "2.0",
  "entities": {
    "dcg:sha256:a1b2c3d4...": {
      "id": "dcg:sha256:a1b2c3d4...",
      "label": "process_payment",
      "description": "Processes a credit card payment",
      "attributes": [
        {"property": "instance of", "ref": "dcg:meta:Function"},
        {"property": "part of", "ref": "dcg:sha256:payments-domain..."},
        {"property": "file path", "string": "src/billing/charge.py"},
        {"property": "line number", "quantity": 42}
      ]
    }
  },
  "relations": {}
}
```

### 13.4 Intra-Layer Constraint

**Guidance:**

1. All entities and relations stored in a layer's graph data files MUST reference only UIDs that exist within that same layer. Cross-layer UID references in graph data files are not permitted.
2. Cross-layer connections MUST be expressed as join rules in the stack manifest (see [DCG-001-COMP](../rfc/02-rfc-composition.md)) and materialized at stack load time — not stored in any layer's graph data files.
3. This constraint enables each layer to be loaded, validated, and rebuilt independently, without requiring access to other layers.

---

## 14. Builder Helpers

> **Implements:** R-001 through R-003 [DCG-001] — entity structure conformance via helpers

**Guidance:**

1. Implementations SHOULD provide builder functions that generate valid entity JSON without requiring callers to construct the full attribute structure manually.
2. Builders MUST produce output that satisfies all entity and attribute requirements in DCG-001 §6 and §7.
3. The following builder functions are RECOMMENDED:

```python
from dcg.core import builders as wb

# Create an entity
entity = wb.entity(
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

# Create a standalone relation
relation = wb.relation(
    source=fn_uid, target=endpoint_uid,
    property="handles route",
    qualifiers=[wb.attribute("confidence", wb.string_value("inferred"))],
)
```

**Builder function signatures:**
- `entity(uid, label, description, attributes=None, aliases=None) → dict`
- `attribute(property, value) → dict` — auto-detects value type from the value object
- `ref_value(uid) → {"ref": uid}`
- `string_value(text) → {"string": text}`
- `quantity_value(amount) → {"quantity": amount}`
- `relation(source, target, property, qualifiers=None) → dict`

---

## 15. Schema Versioning

> **Implements:** R-012 through R-016 [DCG-001] — schema_version field, version validation, migration strategy

**Guidance:**

1. Every graph data file SHOULD include a `schema_version` field at the top level.
2. The current schema version is `"2.0"`. Version 2.0 reflects the `graph_card.json` + `graphs/` layout introduced in DCG-001.
3. Major version bumps indicate breaking changes requiring re-extraction. Minor version bumps indicate additive, backward-compatible changes.
4. When `schema_version` is present, implementations MUST reject graphs whose major version exceeds the implementation's supported major version.
5. Implementations SHOULD accept graphs with a higher minor version than supported, preserving unknown fields on round-trip.
6. No migration shims — breaking changes require a full re-index. This is consistent with the pre-1.0 policy of no backward compatibility guarantees.

**Version history:**
- `"2.0"` — `graph_card.json` + `graphs/` layout (current, DCG-001)
- `"1.0"` — legacy single-file `graph.json` format (DCG pre-001; not supported)

---

## 16. Package Structure

> **Implements:** module organization guidance (informative)

```
dcg/
├── __init__.py                     # convenience re-exports from dcg.core
├── core/                           # INDEPENDENT — zero external domain deps
│   ├── __init__.py                 # public API exports
│   ├── model.py                    # Entity, Relation dataclasses
│   ├── ontology.py                 # ontology_builtin, register_global_type, register_property
│   ├── store.py                    # GraphStore (in-memory), GraphStoreProtocol
│   ├── purge.py                    # purge_retracted
│   ├── project.py                  # DomainProject (graph_card.json + graphs/ persistence)
│   ├── builders.py                 # entity(), attribute(), relation() helpers
│   ├── uid.py                      # entity_uid(), relation_uid()
│   ├── stack.py                    # StackLoader, DAG validator, join materializer, cross-layer query
│   └── packs.py                    # PackRegistry, pack loader, built-in pack catalog
├── git/                            # optional — git-backed store
│   ├── __init__.py
│   └── store.py                    # GitGraphStore (git-backed GraphStoreProtocol implementation)
└── mcp/                            # optional — requires mcp>=1.27
    ├── __init__.py
    ├── server.py                   # FastMCP tools over stdio
    └── __main__.py                 # python -m dcg.mcp entry point
```

**Dependencies:**
- Core: none (pure Python)
- File persistence: stdlib `json`, `pathlib` (no additional deps)
- Git integration: `dulwich` (optional, Apache 2.0) — if consumers want git-commit semantics on top of file persistence

**Public API surface** (exported from `dcg.core`):
- `GraphStore` — in-memory store
- `DomainProject` — file-backed store
- `GraphStoreProtocol` — protocol for type-checking
- `entity_uid()`, `relation_uid()` — UID generation
- `register_global_type()`, `register_property()` — ontology extension
- `register_alias(alias, canonical)` — register a property alias (e.g., Wikidata P-IDs; see §6 and DCG-001 Appendix B.3)
- `purge_retracted()` — retraction cleanup
- `StackLoader` — stack manifest loading, DAG validation, join materialization, cross-layer query
- `builders` module — `entity()`, `attribute()`, `relation()`, `ref_value()`, `string_value()`, `quantity_value()`

---

## 17. Security Considerations

> **Implements:** DCG-001 §11 (Security Considerations — informative; no formal R-NNN requirements)

**Guidance:**

1. Entity IDs MUST NOT be derived from user-supplied free text without sanitization. Identity keys MUST be validated before hashing — malformed keys could produce unintended collisions or inject content into hash inputs.
2. Implementations SHOULD validate that entity JSON does not exceed a configurable maximum size. The RECOMMENDED default is 1 MB per entity.
3. Graph data files loaded from external sources SHOULD be parsed defensively — unknown fields MUST be preserved but MUST NOT be executed or interpreted as code.
4. Ontology packs loaded from user-defined sources carry the same trust level as project configuration — implementations SHOULD validate pack content against the expected format before merging.
5. Content-addressed UIDs are deterministic; implementations SHOULD NOT use UIDs as security tokens or access control identifiers.

---

## 18. References

### Normative

- [DCG-001](../rfc/01-rfc.md) — Domain Context Graph Core Protocol
- [DCG-001-COMP](../rfc/02-rfc-composition.md) — DCG Composition Extension (Stack Manifests)
- [DCG-001-PACK](../rfc/03-rfc-packs.md) — DCG Pack Extension (Ontology Packs)

### Informative

- [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt) — Key words for use in RFCs
- [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174) — Uppercase vs Lowercase in RFC 2119 Key Words
- [Wikibase JSON (Action API)](https://doc.wikimedia.org/Wikibase/master/php/docs_topics_json.html)
- [Wikibase Data Model](https://www.mediawiki.org/wiki/Wikibase/DataModel/JSON)
- [IPFS Content Identifiers (CIDs)](https://docs.ipfs.tech/concepts/content-addressing/)
- [qwikidata — Offline Wikidata JSON](https://pypi.org/project/qwikidata/)
- [Wikidata Property P31 (instance of)](https://www.wikidata.org/wiki/Property:P31)
- [Domain-Driven Design — Bounded Contexts](https://martinfowler.com/bliki/BoundedContext.html)
- [JSON Schema](https://json-schema.org/) — A Media Type for Describing JSON Documents
