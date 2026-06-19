# RFC: Domain Context Graph Protocol Specification

**RFC ID:** DCG-002
**Status:** Draft
**Date:** 2026-06-19
**Supersedes:** DCG-001
**Companion:** [Design Spec](design.md)
**Schemas:** [`graph.schema.json`](../schema/graph.schema.json) | [`subgraph.schema.json`](../schema/subgraph.schema.json) | [`stack.schema.json`](../schema/stack.schema.json)

---

## 1. Abstract

Domain Context Graph (DCG) is a protocol for structuring, versioning, and
sharing domain knowledge so that AI systems and human developers can operate on
verified context rather than statistical inference. It provides a standard,
tool-portable format for capturing how real-world domains (Payments, Security,
Infrastructure) relate to each other and to the artifacts (code, APIs, services)
that implement them.

The protocol uses a compact, English-first entity format stored in a local
file-based format (`graph.json`) optimized for code-based read and write. The
format is designed to be readable by both humans and language models without
specialized tooling. Wikidata-compatible import and export is a first-class
capability via a bidirectional adapter.

The local format supports multi-file projects, layered knowledge evolution, and
stack-based composition of multiple domain projects. This document specifies the
protocol using RFC 2119 requirement keywords.

---

## 2. Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

Additional definitions:

- **Entity**: An item with an ID, label, description, and claims.
- **Claim**: A typed fact about an entity — a property-value pair with optional
  qualifiers.
- **Domain**: A first-class entity representing a real-world area of concern.
- **Domain Root** (Top-Level Domain): A domain entity with no parent domain.
  The primary entry point for consumers exploring a project's knowledge.
- **Layer**: A snapshot of changes committed atomically to the store.
- **Retraction**: Marking an entity or claim as retracted in a layer via the
  `"retracted": true` flag. Retractions suppress matching data from older layers
  during merge without physically deleting it.
- **Graph**: A store containing entities and relations, persisted as JSON files.
- **Domain Project**: A directory containing a `graph.json` entry point plus
  optional linked subgraph files — the canonical on-disk format for DCG data.
- **Meta-Entity**: A type-defining entity (e.g., `dcg:meta:Function`).
- **Stack**: A composition of multiple DCG domain projects into a layered
  knowledge structure, defined by a YAML manifest.
- **Composition Layer**: A single DCG domain project within a stack. Distinct
  from a versioning layer (Section 7.2) — composition layers separate concerns
  across independent projects, while versioning layers track changes within a
  single project.

---

## 3. Conformance

A conforming implementation MUST satisfy all MUST requirements in Sections 4–10
of this specification.

---

## 4. Entity Model

### 4.1 Entity Structure

1. Every entity MUST be a JSON object containing the following fields:
   - `id` (string, REQUIRED)
   - `label` (string, REQUIRED — English display name)
   - `description` (string, REQUIRED — English description)
   - `claims` (array, REQUIRED — flat list of claim objects)

2. Entities SHOULD include an `aliases` field (array of strings). If omitted,
   implementations MUST treat it as an empty array.

3. All extension fields MUST be prefixed with `dcg:` to prevent collision with
   future protocol fields.

**Example entity:**

```json
{
  "id": "dcg:sha256:a3f7c2d1e4b89012f4c5d6e7a8b90123",
  "label": "process_payment",
  "description": "Processes a credit card payment",
  "aliases": ["processPayment"],
  "claims": [
    {"property": "instance of", "entity": "dcg:meta:Function"},
    {"property": "part of", "entity": "dcg:sha256:b1c4e8f2a7d30956"},
    {"property": "file path", "string": "src/billing/charge.py"},
    {"property": "line number", "quantity": 42}
  ]
}
```

### 4.2 Wikidata Structural Compatibility

DCG uses a compact English-first format internally. For Wikidata interoperability,
the compact format maps bidirectionally to the Wikibase Action API JSON format
(Section 9).

4. The `label` field (string) is shorthand for the Wikibase labels structure:
   `"label": "foo"` expands to `{"en": {"language": "en", "value": "foo"}}`.

5. The `description` field (string) is shorthand for the Wikibase descriptions
   structure, following the same expansion as labels.

6. The `aliases` field (array of strings) is shorthand for the Wikibase aliases
   structure: `["a", "b"]` expands to
   `{"en": [{"language": "en", "value": "a"}, {"language": "en", "value": "b"}]}`.

7. For multilingual support, implementations MAY use the full Wikibase
   language-map structure (`labels`, `descriptions`, `aliases` as objects) instead
   of the shorthand. If both `label` and `labels` are present, `labels` takes
   precedence. Property keys remain English regardless of multilingual labels.

### 4.3 Content-Addressed Entity IDs

8. Entity IDs MUST use the self-describing format: `dcg:<algorithm>:<hash>`.

9. Implementations MUST support `sha256` as the hash algorithm.

10. Implementations MAY support additional hash algorithms (e.g., `blake3`).

11. The hash digest MUST be at least 128 bits (32 hex characters). 64-bit
    truncation is explicitly prohibited — the birthday bound at 64 bits causes
    collisions at ~10M entities.

12. The hash MUST be computed from sorted identity key-value pairs joined by
    `:`, where each pair is formatted as `key=value`:

    ```
    digest = SHA256("key1=val1:key2=val2:...".encode())[:32]
    id = f"dcg:sha256:{digest}"
    ```

13. Implementations MUST produce identical IDs for identical identity keys
    regardless of insertion order.

14. Meta-entity IDs (e.g., `dcg:meta:Function`) are exempt from
    content-addressing. They MUST use the literal prefix `dcg:meta:`.

### 4.4 Schema Version

15. Every graph (not every entity) MUST include a `schema_version` field at the
    top level of `graph.json` and each subgraph file.

16. The initial schema version is `"1.0"`.

17. Implementations MUST reject graphs whose major version exceeds the
    implementation's supported major version.

18. Implementations SHOULD accept graphs with a higher minor version than
    supported, ignoring unknown fields and preserving unknown value types
    on round-trip.

19. Major version bumps indicate breaking changes. Minor version bumps indicate
    additive, backward-compatible changes (e.g., new value types).

---

## 5. Claim and Property Model

### 5.1 Claim Format

20. Claims MUST be a flat array on the entity. Each claim MUST be a JSON object
    containing:
    - `property` (string, REQUIRED — English property key)
    - Exactly one typed value key (see Section 5.3)

21. Claims MAY include a `qualifiers` array. Each qualifier follows the same
    format as a claim (property + typed value key).

22. Multiple claims with the same property are allowed (multi-value). A property
    can have multiple values — e.g., an entity can have two `"part of"` claims
    pointing to different domains.

23. Claim identity is the tuple `(property, typed_key, value)`. Two claims are
    considered the same if all three match.

### 5.2 Property Keys

24. Property keys MUST be English-language strings (e.g., `"instance of"`,
    not `"P31"`).

25. The following core properties MUST be supported by all implementations:
    - `"instance of"` — structural type classification (Wikidata P31)
    - `"part of"` — domain membership and composition (Wikidata P361)
    - `"subclass of"` — type hierarchy (Wikidata P279)

26. Implementations SHOULD maintain a property alias registry that maps
    English keys to Wikidata P-IDs and vice versa.

27. Property aliases MUST be resolved transparently by `get_relations()` and
    `query()`. A query for `"P31"` MUST return the same results as
    `"instance of"` if the alias is registered.

28. When a property is renamed, the old name MUST be retained as an alias.

### 5.3 Value Types

29. Each claim MUST have exactly one typed value key. An entity MUST NOT have
    more than one typed value key per claim.

30. Implementations MUST support these core value types:
    - `"entity"` (string) — UID reference to another entity.
      Maps to Wikibase `wikibase-entityid`.
    - `"string"` (string) — plain text value.
      Maps to Wikibase `string`.
    - `"quantity"` (number) — numeric value.
      Maps to Wikibase `quantity`.

31. Implementations MAY support additional value types (e.g., `"time"`,
    `"coordinate"`, `"url"`, `"boolean"`).

32. Implementations MUST preserve unknown typed value keys on round-trip. An
    implementation that encounters a claim with an unrecognized typed key
    MUST store it as-is and include it in `to_dict()` output. It MUST NOT
    drop, error on, or silently discard unknown types.

### 5.4 Qualifiers

33. Qualifiers provide additional context on a claim. They follow the same
    format as claims: `{"property": "<key>", "<typed_key>": <value>}`.

34. Qualifiers are metadata about the claim, not about the entity. Example:
    `{"property": "calls", "entity": "dcg:sha256:xyz",
      "qualifiers": [{"property": "line number", "quantity": 55}]}`
    — the line number qualifies the "calls" relationship, not the entity.

### 5.5 Retraction

35. Entities and claims can be retracted in a layer by setting
    `"retracted": true` on the entity or claim object.

36. An entity-level retraction suppresses the entire entity from all older
    layers during merge:
    ```json
    {"id": "dcg:sha256:abc123", "retracted": true}
    ```

37. A claim-level retraction suppresses matching claims from all older layers
    during merge:
    ```json
    {"property": "part of", "entity": "dcg:sha256:payments", "retracted": true}
    ```

38. A retracted claim MUST specify the same `(property, typed_key, value)` tuple
    as the claim being retracted. The merge process matches on this tuple.

39. Compacted layers MUST NOT contain `retracted` flags — retractions are
    consumed by compaction (see Section 7.5).

---

## 6. Domains and Ontology

### 6.1 Domains as Entities

40. A Domain MUST be represented as a standard entity with an
    `"instance of" → "dcg:meta:Domain"` claim.

41. Domain entities MUST NOT receive special treatment by the store. They are
    ordinary items that participate in queries and relations like any other
    entity.

42. Entity membership in a domain MUST be expressed via `"part of"` claims
    pointing to the domain entity.

43. An entity MAY belong to multiple domains via multiple `"part of"` claims.

### 6.2 Meta-Entity Types

44. Meta-entity types MUST be structural — they describe what an entity IS,
    independent of which domain it belongs to.

45. The following meta-entities MUST be defined by all implementations:
    - `dcg:meta:Domain` — a real-world area of concern
    - `dcg:meta:Type` — a classification of entities

46. Implementations MAY define additional built-in meta-entities (e.g.,
    `dcg:meta:Function`, `dcg:meta:Class`).

47. Consumers MUST be able to register new meta-entity types at runtime
    via a public registration API.

48. `register_meta_entity()` SHOULD reject registration if the ID already
    exists with different semantics, unless the caller explicitly allows
    overwrite.

### 6.3 Two-Axis Classification

49. Every non-Domain, non-Type entity MUST have at least one `"instance of"`
    claim (structural type).

50. Every non-Domain, non-Type entity SHOULD have at least one `"part of"`
    claim (domain membership).

51. Implementations MUST support querying by structural type (`instance_of`)
    and by domain membership (`part_of`) independently and in combination.

### 6.4 Domain Hierarchy and Abstraction

52. A domain entity MAY have `"part of"` claims pointing to other domain
    entities, forming a domain hierarchy. A domain with no parent domain is
    a **top-level domain** (domain root).

53. A DCG project SHOULD expose a small number of top-level domains
    (RECOMMENDED: 1–10) as the primary entry points for consumers.

54. Top-level domains serve as abstraction barriers. Consumers querying by
    top-level domain SHOULD receive a simplified view of the graph. Internal
    sub-domains and their categorization links are implementation detail that
    top-level domains hide.

55. Implementations MUST support querying all entities within a domain hierarchy
    — a query for a top-level domain SHOULD include entities belonging to any
    of its sub-domains.

**Example — security domain hierarchy:**

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

A consumer querying `part_of=Security` sees just "Security" and its direct
children. Drilling into sub-domains reveals the internal categorization. The
complexity of 50+ CWE entries is hidden behind the "Security" top-level domain.

---

## 7. Graph Store

### 7.1 GraphStoreProtocol

56. A conforming implementation MUST satisfy the `GraphStoreProtocol`
    runtime-checkable Protocol.

57. The protocol MUST define these operations:
    - `add_entity(entity: dict) -> str`
    - `add_relation(source, target, property, qualifiers=None) -> str`
    - `remove(uid: str) -> None`
    - `get_entity(uid: str) -> dict | None`
    - `get_relations(source=None, target=None, property=None) -> list[dict]`
    - `query(instance_of=None, part_of=None, **property_filters) -> list[dict]`
    - `commit_layer(source: str, pass_type: str = "incremental") -> dict`
    - `get_layers() -> list[dict]`
    - `to_dict() -> dict`
    - `to_json(path) -> None`

58. `add_entity()` MUST be idempotent — adding an entity with an existing
    UID updates the entity in place.

59. `get_entity()` MUST follow entity redirects (Section 7.4) transparently,
    up to a maximum depth of 10.

60. `get_relations()` MUST resolve property aliases (Section 5.2) before
    matching.

61. `query()` MUST support filtering by `instance_of` and `part_of`
    independently and in combination.

### 7.2 Layer Model

A **layer** is an atomic unit of change — a batch of entity additions, relation
additions, and retractions committed together. Layers are the building blocks of
DCG's versioning model. They are analogous to git commits: immutable, ordered,
and replayable.

62. A layer MUST be an immutable snapshot of changes committed atomically.

63. Each layer MUST contain:
    - `layer_id` (string, content-addressed)
    - `parent_id` (string or null — the preceding layer's ID)
    - `source` (string — what produced this layer, e.g. `"code-parser"`,
      `"security-scanner"`, `"domain-team"`)
    - `pass_type` (string — see Section 7.2.1)
    - `schema_version` (string)
    - `timestamp` (string, ISO 8601)
    - `entity_uids` (list of strings — entities added, modified, or retracted)
    - `relation_uids` (list of strings — relations added or modified)

64. Layers SHOULD additionally include:
    - `entity_count` (integer — total entities added/modified)
    - `relation_count` (integer — total relations added/modified)
    - `duration_ms` (integer — extraction duration, if available)

65. `commit_layer()` MUST clear pending state after creating the layer.

#### 7.2.1 Pass Types

The `pass_type` field declares **what the layer represents relative to the
store's history**.

| pass_type | Meaning | Merge behavior |
|---|---|---|
| `"full"` | A **complete snapshot** of everything the source knows. If an entity or claim existed before but is NOT restated in this layer, it is implicitly gone. | Merge MUST stop here — layers before a full pass are superseded. |
| `"incremental"` | An **additive update** — only new or changed entities and claims. Anything not mentioned is assumed unchanged. | Merge continues walking past this layer. |
| `"patch"` | A **targeted correction** — explicit fixes to specific claims, often including retractions. | Same merge behavior as incremental, but semantically signals a correction. |

66. A `"full"` pass type indicates a complete snapshot. Merge operations
    MUST NOT look past a full layer.

### 7.3 Merge Semantics

Merge materializes the current state of the graph by replaying layers from
newest to oldest. It operates at the **claim level** — individual facts about
entities — not at the entity level.

67. Layer merge MUST operate at the **claim level**. When two layers modify
    the same entity:
    - Claims with different property keys MUST all be preserved.
    - Claims with the same `(property, typed_key, value)` tuple MUST resolve
      to the latest layer's version (latest-wins).
    - Claims with the same property key but different values MUST all be
      preserved (multi-value).

68. Entity-level retractions (`"retracted": true` on the entity) MUST suppress
    the entity and all its claims from all older layers.

69. Claim-level retractions (`"retracted": true` on a claim) MUST suppress
    matching claims (by `(property, typed_key, value)` tuple) from all older
    layers.

70. Merge MUST walk layers from newest to oldest and MUST stop at the first
    `"full"` layer.

**Example — knowledge evolving via layers:**

```
Layer 1 (full, source: "code-parser"):
  fn_A → instance of: Function, part of: Payments, file path: "src/pay.py"
  fn_B → instance of: Function, part of: Payments

Layer 2 (incremental, source: "domain-team"):
  fn_A → tested by: test_fn_A.py          ← new claim added

Layer 3 (patch, source: "domain-team"):
  fn_A → part of: Payments [RETRACTED]     ← old claim retracted
  fn_A → part of: Billing                 ← new claim
  fn_B [RETRACTED]                         ← entire entity deleted
```

Merged state (walking L3 → L2 → L1):
- `fn_A`: instance of Function, part of Billing, file path src/pay.py,
  tested by test_fn_A.py
- `fn_B`: gone (retracted in L3)

### 7.4 Entity Redirects

When an entity's identity keys change (e.g., a function is renamed or a file is
moved), its content-addressed UID changes. Redirects ensure that references to
the old UID continue to resolve.

71. Implementations MUST support entity redirects for identity key changes.

72. A redirect MUST be stored as a `"redirected to"` claim on the old entity,
    pointing to the new entity UID.

73. `get_entity()` MUST follow redirects transparently. `get_entity(old_uid)`
    MUST return the entity at the redirect target.

74. Redirect chains MUST be followed up to a maximum depth of 10.
    Implementations MUST return `None` if the chain exceeds this depth.

75. Redirects MUST survive compaction.

### 7.5 Compaction

Compaction materializes the merged state into a single `"full"` layer, consuming
all retractions and intermediate state.

76. `compact()` MUST produce a new `"full"` layer containing only the merged
    state — no retracted entities, no retracted claims, no `"retracted"` flags.

77. After compaction, `load()` MUST NOT resurrect pre-compaction retractions.
    The compacted full layer is authoritative.

78. Implementations SHOULD trigger compaction when either:
    - The number of retracted entities/claims exceeds a threshold
      (RECOMMENDED default: 1000), OR
    - `len(layers) > layer_threshold` (RECOMMENDED default: 50)

79. Each layer SHOULD include a `compacted_through` field indicating the
    latest layer ID incorporated by the most recent compaction, so loaders
    know which prior layers are safe to skip.

### 7.6 Domain Project Format

A **DCG domain project** is a directory that stores one or more DCG graphs in a
local file-based format optimized for code-based read and write. The `graph.json`
file at the project root is both the entry point and the primary datastore.

This format is the **canonical on-disk representation** of DCG data. All tools
that read or write DCG graphs MUST use this format as the authoritative store.

**JSON Schemas:** The file formats are formally defined by JSON Schema:
- [`graph.schema.json`](../schema/graph.schema.json) — `graph.json` entry point
- [`subgraph.schema.json`](../schema/subgraph.schema.json) — linked subgraph files
- [`stack.schema.json`](../schema/stack.schema.json) — `dcg-stack.yml` manifest

#### 7.6.1 Entry Point — `graph.json`

80. A DCG domain project MUST contain a `graph.json` file at the project root.
    This file serves three roles:
    1. **Project metadata** — identity, version, description
    2. **Graph index** — links to all graph files in the project
    3. **Primary datastore** — inline entities, relations, and layers

81. `graph.json` MUST contain a `dcg_project` object with at minimum:
    - `name` (string — project identifier)
    - `version` (string — semver)

82. `graph.json` MUST contain a `graphs` array indexing all graph files in
    the project. Each entry MUST have:
    - `id` (string — graph identifier, e.g., `"default"`)
    - `file` (string — path relative to project root)
    The first entry SHOULD be `{"id": "default", "file": "graph.json"}`.

83. `graph.json` MUST also contain the main graph data inline:
    `schema_version`, `entities`, `relations`, and `layers`.

84. Implementations MUST treat a directory as a valid DCG domain project
    if and only if it contains a `graph.json` with a `dcg_project` field.

85. `dcg_project` SHOULD additionally contain:
    - `description` (string — human-readable project description)

**Example `graph.json`:**

```json
{
  "dcg_project": {
    "name": "app-security-graph",
    "version": "0.1.0",
    "description": "Application security domain knowledge graph"
  },
  "graphs": [
    {"id": "default", "file": "graph.json"},
    {"id": "appsec", "file": "graphs/appsec.json"}
  ],
  "schema_version": "1.0",
  "entities": {
    "dcg:sha256:a1b2c3d4...": {
      "id": "dcg:sha256:a1b2c3d4...",
      "label": "AppSecurity",
      "description": "Application security domain",
      "claims": [
        {"property": "instance of", "entity": "dcg:meta:Domain"}
      ]
    }
  },
  "relations": {},
  "layers": [
    {"layer_id": "...", "source": "wikidata", "pass_type": "full",
     "timestamp": "...", "entity_count": 155}
  ]
}
```

#### 7.6.2 Multi-File Projects

A project MAY split its graph data across multiple files.

86. Additional graphs MAY be stored as separate JSON files referenced by
    the `graphs` index. Each subgraph file MUST contain only graph data
    (`schema_version`, `entities`, `relations`, `layers`) — no `dcg_project`
    or `graphs` wrapper.

87. Implementations MUST load all subgraph files listed in the `graphs`
    index during `load()`. Each subgraph MUST be registered in the
    project's registry under its `id`.

88. When saving, implementations MUST persist each subgraph to its
    referenced `file` path and update `graph.json` to reflect the
    current `graphs` index.

#### 7.6.3 Design Rationale

The `graph.json` format is designed for local, code-based workflows:

- **Single entry point** — one file to discover the entire project
- **Human and LLM readable** — JSON with English property keys is inspectable
  by both humans and language models without specialized tooling
- **Git-friendly** — the project directory can be a git repository; JSON diffs
  are meaningful in code review
- **Splittable** — large graphs can be partitioned across files without
  changing the protocol
- **Token-efficient** — the compact claim format minimizes tokens per entity,
  maximizing the knowledge that fits in an LLM's context window

### 7.7 Composition (Stack Manifests)

A **stack** composes multiple DCG domain projects into a layered knowledge
structure. Each project in the stack is a **composition layer** (distinct from
the versioning layers in Section 7.2). Composition allows separating concerns
into independent projects while enabling cross-layer entity resolution.

#### 7.7.1 Stack Manifest Format

89. A stack MUST be defined by a YAML manifest file (conventionally named
    `dcg-stack.yml`).

90. The manifest MUST contain:
    - `stack` (string — human-readable stack name)
    - `layers` (array — ordered list of composition layer entries)

91. Each layer entry MUST contain:
    - `name` (string — unique identifier for this composition layer)
    - `source` (string — path to the DCG domain project directory,
      resolved relative to the manifest file)

92. Each layer entry MAY contain:
    - `extends` (string — name of another layer in this stack that this
      layer inherits from for entity resolution)
    - `pass_type` (string — default pass type hint; defaults to `"full"`)

93. Layer names MUST be unique within a stack manifest.

94. The `extends` field MUST reference a layer name that exists in the
    same manifest.

95. The `extends` graph MUST be acyclic. Implementations MUST detect and
    reject cycles in the extends chain at load time.

**Example `dcg-stack.yml`:**

```yaml
stack: appsec-ctx-graph
layers:
  - name: security
    source: ./graphs/security-domain
    pass_type: full

  - name: product
    source: ./graphs/product-domain
    extends: security
    pass_type: full

  - name: code
    source: ./graphs/code
    extends: product
    pass_type: incremental
```

#### 7.7.2 Loading

96. Implementations MUST load all layers in manifest declaration order.
    Each layer's DCG domain project MUST be loaded independently —
    composition layers do not share a GraphStore.

97. Implementations MUST register each layer's GraphStore in a shared
    registry keyed by layer name, enabling cross-layer entity resolution.

#### 7.7.3 Cross-Layer Entity Resolution

Entity resolution via the `extends` chain is the core mechanism by which
composition layers share knowledge.

98. `resolve(layer_name, uid)` MUST walk the `extends` chain starting
    from the named layer. For each layer in the chain, it MUST check
    that layer's GraphStore for the entity.

99. Resolution MUST return the **first** entity found in the chain
    (nearest layer wins).

100. Resolution MUST follow entity redirects (Section 7.4) within each
     layer's store during the chain walk.

101. Resolution MUST terminate and return `None` if the chain is
     exhausted without finding the entity.

102. Resolution MUST track visited layers and terminate if a cycle is
     encountered, even if validation was skipped at load time.

#### 7.7.4 Cross-Layer Queries

103. Cross-layer `query()` MUST iterate layers in manifest declaration
     order (not extends chain order).

104. When the same entity UID appears in multiple layers, cross-layer
     queries MUST deduplicate by UID. The entity from the
     **first-declared** layer (earliest in the manifest) MUST be returned.

105. Cross-layer queries MUST NOT merge claims across layers. The
     entity dict from the winning layer is returned as-is.

106. Cross-layer queries SHOULD accept an optional layer filter (list of
     layer names) to restrict which layers are searched.

#### 7.7.5 Composition Guarantees

107. Each composition layer MUST be independently loadable. A layer
     MUST NOT fail to load because its parent layer is unavailable.

108. Writes MUST target a single composition layer at a time.

109. `commit_layer()` and `save()` MUST operate on the active
     composition layer only. Other layers MUST NOT be modified.

#### 7.7.6 Composition Patterns

Composition layers create new knowledge by introducing relationships between
entities that may live in different layers. This is the primary value of
stack-based composition.

110. Entities in a child layer MAY reference entities from parent layers
     via the extends chain in their claims. The referenced entity MUST be
     resolvable via `resolve()`.

111. Relations created in a child layer are stored in the child layer's
     GraphStore, even when the source or target entity lives in a parent
     layer. The child layer owns the relationship.

112. Improvements committed to a parent layer MUST be automatically
     visible to child layers on next load via the extends chain. No
     changes to the child layer are required for propagation.

**Example — 3-layer composition:**

```
L1 (security-domain):
  CWE-79: XSS (entity, instance of Vulnerability)
  CWE-89: SQL Injection (entity, instance of Vulnerability)

L2 (product-domain, extends: security):
  CheckoutService (entity, instance of Class)
  — "vulnerability in" → CWE-79  ← cross-layer relation, resolves via extends

L3 (code, extends: product):
  process_payment() (entity, instance of Function)
  — "part of" → CheckoutService  ← resolves via extends to L2
  — "calls" → update_stock()     ← local relation within L3
```

When L1 adds a new vulnerability `CWE-352: CSRF`, L2 and L3 automatically
see it on next load — no changes needed in L2 or L3. L2 can then create a
`"vulnerability in" → CWE-352` relation in its own store without touching L1.

---

## 8. Builder Helpers

113. Implementations SHOULD provide builder functions that generate valid
     entity JSON without requiring callers to construct the full claim
     structure manually.

114. Builders MUST produce output that satisfies all entity and claim
     requirements in Sections 4 and 5.

115. The following builder functions are RECOMMENDED:
     - `item(uid, label, description, claims)` → entity dict
     - `claim(property, value)` → claim dict (auto-detects value type)
     - `entity_value(uid)` → `{"entity": uid}`
     - `string_value(text)` → `{"string": text}`
     - `quantity_value(amount)` → `{"quantity": amount}`

---

## 9. Wikidata Interoperability

DCG entities are designed to be bidirectionally convertible with Wikibase
Action API JSON format. The compact DCG format is the canonical internal
representation; Wikibase JSON is an import/export format.

### 9.1 Compatibility

116. DCG entity JSON, when expanded to Wikibase format via the adapter
     (Section 9.2), MUST be parseable by `qwikidata` (the standard
     offline Wikidata JSON parser) with no modifications beyond ID format.

117. Implementations SHOULD provide a `wikidata` alias field in the
     property registry for properties that correspond to Wikidata P-IDs.

### 9.2 Adapter Specification

118. Implementations SHOULD provide an `export_to_wikidata()` function
     that converts DCG compact format to Wikibase Action API JSON:
     - `label` → `labels` (language-map structure)
     - `description` → `descriptions` (language-map structure)
     - `aliases` (array) → `aliases` (language-map structure)
     - Flat claims → grouped-by-property snak structure (with `type`,
       `mainsnak`, `snaktype`, `datavalue`, `rank`)
     - Add `"type": "item"` field
     - English property keys → P-IDs via property registry
     - `dcg:` UIDs → placeholder Q-numbers (or kept as custom IDs)

119. Implementations SHOULD provide an `import_from_wikidata()` function
     that converts Wikibase Action API JSON to DCG compact format:
     - Q-numbers → `dcg:sha256:` UIDs (computed from entity label +
       Q-number as identity keys). Original Q-number preserved as alias.
     - P-IDs → English property keys via property registry
     - Strip `sitelinks`, `lastrevid`, `modified`
     - Compress snak structure to flat claims with typed keys
     - `"type": "item"` field removed

### 9.3 Round-Trip Fidelity

120. A DCG entity exported to Wikibase JSON and re-imported MUST produce
     an entity with identical claims (same property-value pairs). Label,
     description, and alias content MUST be preserved.

121. Entity UIDs are NOT preserved on round-trip through Wikidata (Q-numbers
     ≠ dcg: UIDs). Implementations MUST re-compute UIDs on import and
     create redirects for any references that changed.

---

## 10. Security Considerations

122. Entity IDs MUST NOT be derived from user-supplied free text without
     sanitization. Identity keys MUST be validated before hashing.

123. Implementations SHOULD validate that entity JSON does not exceed
     a configurable maximum size (RECOMMENDED default: 1 MB per entity).

---

## 11. References

### Normative

- [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt) — Key words for use in RFCs
- [Wikibase JSON (Action API)](https://doc.wikimedia.org/Wikibase/master/php/docs_topics_json.html)
- [Wikibase Data Model](https://www.mediawiki.org/wiki/Wikibase/DataModel/JSON)

### Informative

- [IPFS Content Identifiers (CIDs)](https://docs.ipfs.tech/concepts/content-addressing/)
- [qwikidata — Offline Wikidata JSON](https://pypi.org/project/qwikidata/)
- [Wikidata Property P31 (instance of)](https://www.wikidata.org/wiki/Property:P31)
- [Domain-Driven Design — Bounded Contexts](https://martinfowler.com/bliki/BoundedContext.html)

---

## Appendix A: Requirement Summary

| # | Section | Level | Requirement |
|---|---|---|---|
| 1–3 | Entity Structure | MUST/SHOULD | Top-level fields (id, label, description, claims), aliases, dcg: prefix |
| 4–7 | Wikidata Compat | MUST/MAY | Shorthand labels → Wikibase expansion, multilingual support |
| 8–14 | Entity IDs | MUST/MAY | Self-describing, sha256, ≥128 bits, deterministic, meta-entity exempt |
| 15–19 | Schema Version | MUST/SHOULD | Per-graph, version validation, forward compat |
| 20–23 | Claim Format | MUST/MAY | Flat array, typed value keys, multi-value, claim identity |
| 24–28 | Property Keys | MUST/SHOULD | English keys, alias registry, safe renames |
| 29–32 | Value Types | MUST/MAY | Core 3 types, open extension, preserve unknown |
| 33–34 | Qualifiers | — | Flat format, claim metadata |
| 35–39 | Retraction | MUST | Unified `retracted: true`, claim matching, compaction consumption |
| 40–43 | Domains | MUST/MAY | First-class entities, "part of" membership |
| 44–48 | Meta-Entities | MUST/SHOULD/MAY | Structural types, runtime registration |
| 49–51 | Classification | MUST/SHOULD | Two-axis: instance_of + part_of |
| 52–55 | Domain Hierarchy | MUST/SHOULD/RECOMMENDED | Nesting, top-level domains, count guidance, hierarchy queries |
| 56–61 | GraphStore | MUST | Protocol operations, idempotent add, redirects, alias resolution |
| 62–66 | Layers | MUST/SHOULD | Immutable, required fields, pass types |
| 67–70 | Merge | MUST | Claim-level, retraction precedence, full-layer stop |
| 71–75 | Redirects | MUST | Transparent follow, max depth 10, survive compaction |
| 76–79 | Compaction | MUST/SHOULD | Clean state, no retractions, threshold triggers |
| 80–88 | Domain Project | MUST/SHOULD | graph.json format, multi-file, subgraph loading |
| 89–95 | Stack Manifest | MUST/MAY | YAML format, extends chain, cycle detection |
| 96–97 | Stack Loading | MUST | Declaration-order, registry integration |
| 98–102 | Cross-Layer Resolution | MUST | Extends chain walk, first-found-wins, redirect follow |
| 103–106 | Cross-Layer Queries | MUST/SHOULD | Declaration-order, UID dedup, no claim merge |
| 107–109 | Composition Guarantees | MUST | Independent layers, single-layer writes |
| 110–112 | Composition Patterns | MUST | Cross-layer refs, child owns relation, propagation |
| 113–115 | Builders | SHOULD/RECOMMENDED | Helper functions |
| 116–121 | Wikidata Interop | MUST/SHOULD | qwikidata compat, adapter, round-trip fidelity |
| 122–123 | Security | MUST/SHOULD | Input validation, size limits |

**Total: 123 requirements (78 MUST, 28 SHOULD, 11 MAY/RECOMMENDED, 6 MUST NOT)**
