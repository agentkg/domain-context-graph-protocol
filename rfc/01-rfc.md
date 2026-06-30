# RFC: Domain Context Graph Core Protocol

**RFC ID:** DCG-001
**Status:** Draft
**Date:** 2026-06-26
**Extensions:** [DCG-001-COMP](02-rfc-composition.md) | [DCG-001-PACK](03-rfc-packs.md) | [DCG-001-ADAPT](04-rfc-store-adaptor.md)
**Schemas:** [graph_card](../schema/graph_card.schema.json) | [graph_data](../schema/graph_data.schema.json)

---

## 1. Abstract

Domain Context Graph (DCG) is a protocol for structuring, curating, and
sharing domain knowledge so that AI systems and human developers can operate on
verified context rather than statistical inference. It provides a standard,
tool-portable format for capturing how real-world domains (Payments, Security,
Infrastructure) relate to each other and to the artifacts (code, APIs, services)
that implement them.

The protocol uses a compact, English-first entity format stored in a local
file-based format (`graph_card.json` metadata manifest plus `graphs/` data
files) optimized for code-based read and write.
The format is designed to be readable by both humans and language models without
specialized tooling. Wikidata-compatible property aliases are built in;
optional import/export adapter conventions are described in Appendix B.

The local format supports multi-file projects. Stack-based composition of
multiple domain projects is specified in [DCG-001-COMP](02-rfc-composition.md).

---

## 2. Status of This Memo

This document specifies a protocol for the Domain Context Graph (DCG) data
format and store operations. It defines the wire format, data model, and
persistence semantics for DCG domain projects. Distribution of this memo is
unlimited.

---

## 3. Terminology

Additional definitions:

- **Entity**: An item with an ID, label, description, and attributes.
- **Attribute**: A typed fact about an entity — a property-value pair with optional
  qualifiers.
- **Domain**: A first-class entity representing a real-world area of concern.
- **Domain Root** (Top-Level Domain): A domain entity with no parent domain.
  The primary entry point for consumers exploring a project's knowledge.
- **Retraction**: Marking an entity or attribute as retracted via the
  `"retracted": true` flag. Retractions soft-delete data without physically
  removing it. Purge permanently removes retracted data.
- **Graph**: A store containing entities and relations, persisted as JSON files.
- **Domain Project**: A directory containing a `graph_card.json` entry point
  plus graph data files under a `graphs/` subdirectory — the canonical on-disk
  format for DCG data.
- **Graph Card**: The metadata manifest (`graph_card.json`) of a DCG domain
  project. Contains project identity, ontology declarations, and a `graphs`
  index pointing to graph data files. Does NOT contain entities or relations.
- **Graph Data File**: A JSON file under `graphs/` (e.g., `graphs/default.json`)
  containing the entities and relations for one named graph.
- **Type**: A type-defining entity (e.g., `dcg:meta:Function`).
- **Ontology**: The set of declared entity types, properties,
  relations, and aliases available to a graph. Composed from the built-in
  (`ontology_builtin`) and per-project `graph_card.json` ontology sections.
- **`ontology_builtin`**: The always-loaded base ontology containing RFC-mandatory
  entity types (Domain, Type) and cross-domain general vocabulary with Wikidata
  mappings. Not an opt-in pack; defined in this RFC.

---

## 4. Conformance

A conforming implementation MUST satisfy all MUST requirements in this
specification. Conformance to extensions [DCG-001-COMP](02-rfc-composition.md),
[DCG-001-PACK](03-rfc-packs.md), and [DCG-001-ADAPT](04-rfc-store-adaptor.md) is
independent and optional.

---

## 5. BCP 14 Boilerplate

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14
[RFC 2119](https://www.ietf.org/rfc/rfc2119.txt)
[RFC 8174](https://www.rfc-editor.org/rfc/rfc8174) when, and only when, they
appear in all capitals, as shown here.

---

## Table of Contents

- [Abstract](#1-abstract)
- [Status of This Memo](#2-status-of-this-memo)
- [Terminology](#3-terminology)
- [Conformance](#4-conformance)
- [BCP 14 Boilerplate](#5-bcp-14-boilerplate)
- [Entity Model](#6-entity-model)
  - [Entity Structure](#61-entity-structure)
  - [Content-Addressed Entity IDs](#62-content-addressed-entity-ids)
  - [Schema Version](#63-schema-version)
- [Attribute and Property Model](#7-attribute-and-property-model)
  - [Attribute Format](#71-attribute-format)
  - [Property Keys](#72-property-keys)
  - [Value Types](#73-value-types)
  - [Qualifiers](#74-qualifiers)
  - [Retraction](#75-retraction)
- [Domains and Ontology](#8-domains-and-ontology)
  - [Domains as Entities](#81-domains-as-entities)
  - [Entity Types](#82-entity-types)
  - [Two-Axis Classification](#83-two-axis-classification)
  - [Domain Hierarchy](#84-domain-hierarchy)
  - [Ontology Declaration](#85-ontology-declaration)
    - [Built-in Ontology (`ontology_builtin`)](#851-built-in-ontology-ontology_builtin)
    - [Per-Project Ontology](#852-per-project-ontology)
    - [Ontology Persistence Format](#853-ontology-persistence-format)
    - [Redirect Format](#854-redirect-format)
- [Domain Project Format](#9-domain-project-format)
  - [Overview](#91-overview)
  - [Metadata Manifest — `graph_card.json`](#92-metadata-manifest--graph_cardjson)
  - [Graph Data Files](#93-graph-data-files)
  - [Intra-Layer Constraint](#94-intra-layer-constraint)
- [Graph Store Protocol](#10-graph-store-protocol)
  - [Store Operations](#101-store-operations)
  - [Retraction and Purge](#102-retraction-and-purge)
  - [Entity Redirects](#103-entity-redirects)
  - [I/O Operations](#104-io-operations)
  - [Entity and Attribute Atomicity](#105-entity-and-attribute-atomicity)
  - [Relation Classification](#106-relation-classification)
- [Security Considerations](#11-security-considerations)
- [References](#12-references)
  - [Normative](#normative)
  - [Informative](#informative)
- [Appendix A: Requirement Summary](#appendix-a-requirement-summary)
- [Appendix B: Wikidata Alias Mapping (Informative)](#appendix-b-wikidata-alias-mapping-informative)
- [Appendix C: Design Rationale (Informative)](#appendix-c-design-rationale-informative)
- [Appendix D: Change Log](#appendix-d-change-log)

---

## 6. Entity Model

### 6.1 Entity Structure

R-001. Every entity MUST be a JSON object containing the following fields:
   - `id` (string, REQUIRED)
   - `label` (string, REQUIRED — English display name)
   - `description` (string, REQUIRED — English description)
   - `attributes` (array, REQUIRED — flat list of attribute objects)

R-002. Entities SHOULD include an `aliases` field (array of strings). If omitted,
   implementations MUST treat it as an empty array.

R-003. All extension fields MUST be prefixed with `dcg:` to prevent collision with
   future protocol fields.

**Example entity:**

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

### 6.2 Content-Addressed Entity IDs

R-004. Entity IDs MUST use the self-describing format: `dcg:<algorithm>:<hash>`.

R-005. Implementations MUST support `sha256` as the hash algorithm.

R-006. Implementations MAY support additional hash algorithms (e.g., `blake3`).

R-007. The hash digest MUST be at least 128 bits (32 hex characters). 64-bit
    truncation is explicitly prohibited — the birthday bound at 64 bits causes
    collisions at ~10M entities.

R-008. The hash MUST be computed from sorted identity key-value pairs joined by
    `:`, where each pair is formatted as `key=value`:

    ```
    digest = SHA256("key1=val1:key2=val2:...".encode())[:32]
    id = f"dcg:sha256:{digest}"
    ```

R-009. Implementations MUST produce identical IDs for identical identity keys
    regardless of insertion order.

R-010. Relation UIDs MUST use the prefix `dcg:rel:sha256:` and MUST be computed
    from sorted key-value pairs `property=<name>:source=<uid>:target=<uid>`,
    hashed and truncated following the same scheme as entity UIDs (R-007, R-008).

R-011. Entity type IDs (e.g., `dcg:meta:Function`) are exempt from
    content-addressing. They MUST use the literal prefix `dcg:meta:`.

### 6.3 Schema Version

R-012. Every graph data file SHOULD include a `schema_version` field at the top
    level. When present, implementations MUST validate it per R-014 through R-016.

R-013. The current schema version is `"2.0"`. Version `"2.0"` reflects the
    breaking changes introduced in DCG-001 (graph_card.json + graphs/ layout,
    multi-parent DAG extends, join rules).

R-014. When `schema_version` is present, implementations MUST reject graphs whose
    major version exceeds the implementation's supported major version.

R-015. Implementations SHOULD accept graphs with a higher minor version than
    supported, ignoring unknown fields and preserving unknown value types
    on round-trip.

R-016. Major version bumps indicate breaking changes. Minor version bumps indicate
    additive, backward-compatible changes (e.g., new value types).

---

## 7. Attribute and Property Model

### 7.1 Attribute Format

R-017. Attributes MUST be a flat array on the entity. Each attribute MUST be a JSON object
    containing:
    - `property` (string, REQUIRED — English property key)
    - Exactly one typed value key (see §7.3)

R-018. Attributes MAY include a `qualifiers` array. Each qualifier follows the same
    format as an attribute (property + typed value key).

R-019. Multiple attributes with the same property are allowed (multi-value). A property
    can have multiple values — e.g., an entity can have two `"part of"` attributes
    pointing to different domains.

R-020. Attribute identity is the tuple `(property, typed_key, value)`. Two attributes are
    considered the same if all three match.

### 7.2 Property Keys

R-021. Property keys MUST be English-language strings (e.g., `"instance of"`,
    not `"P31"`).

R-022. The following core properties MUST be supported by all implementations:
    - `"instance of"` — structural type classification (Wikidata P31)
    - `"part of"` — domain membership and composition (Wikidata P361)
    - `"subclass of"` — type hierarchy (Wikidata P279). Note: "supported"
      means property alias resolution and storage; no dedicated query axis
      is required.

R-023. Implementations SHOULD maintain a property alias registry that maps
    canonical English property keys to their aliases and vice versa. See
    Appendix B for Wikidata-specific alias naming conventions.

R-024. Property aliases MUST be resolved transparently by `get_relations()` and
    `query()`. A query for `"wikidata_P31"` MUST return the same results as
    `"instance of"` if the alias is registered.

R-025. When a property is renamed, the old name MUST be retained as an alias.

### 7.3 Value Types

R-026. Each attribute MUST have exactly one typed value key. An entity MUST NOT have
    more than one typed value key per attribute.

R-027. Implementations MUST support these core value types:
    - `"ref"` (string) — UID reference to another entity.
      Maps to Wikibase `wikibase-entityid`.
    - `"string"` (string) — plain text value.
      Maps to Wikibase `string`.
    - `"quantity"` (number) — numeric value.
      Maps to Wikibase `quantity`.

R-028. Implementations MAY support additional value types (e.g., `"time"`,
    `"coordinate"`, `"url"`, `"boolean"`).

R-029. Implementations MUST preserve unknown typed value keys on round-trip. An
    implementation that encounters an attribute with an unrecognized typed key
    MUST store it as-is and include it in `to_dict()` output. It MUST NOT
    drop, error on, or silently discard unknown types.

### 7.4 Qualifiers

R-030. Qualifiers provide additional context on an attribute. They MUST follow the
    same format as attributes: `{"property": "<key>", "<typed_key>": <value>}`.

R-031. Qualifiers are metadata about the attribute, not about the entity. Example:
    `{"property": "calls", "ref": "dcg:sha256:xyz",
      "qualifiers": [{"property": "line number", "quantity": 55}]}`
    — the line number qualifies the "calls" relation, not the entity.

### 7.5 Retraction

R-032. Entities and attributes can be retracted by setting `"retracted": true`
    on the entity or attribute object.

R-033. An entity-level retraction marks the entire entity as soft-deleted:
    ```json
    {"id": "dcg:sha256:abc123", "retracted": true}
    ```

R-034. An attribute-level retraction marks a specific attribute as soft-deleted:
    ```json
    {"property": "part of", "ref": "dcg:sha256:payments", "retracted": true}
    ```

R-035. A retracted attribute MUST specify the same `(property, typed_key, value)` tuple
    as the attribute being retracted.

R-036. After purge, the store MUST NOT contain `retracted` flags — retractions
    are consumed by `purge_retracted()` (see §10.2) — except for redirect
    source tombstones preserved per R-081.

---

## 8. Domains and Ontology

### 8.1 Domains as Entities

R-037. A Domain MUST be represented as a standard entity with an
    `"instance of" → "dcg:meta:Domain"` attribute.

R-038. Domain entities MUST NOT receive special treatment by the store. They are
    ordinary items that participate in queries and relations like any other
    entity.

R-039. Entity membership in a domain MUST be expressed via `"part of"` attributes
    pointing to the domain entity.

R-040. An entity MAY belong to multiple domains via multiple `"part of"` attributes.

### 8.2 Entity Types

R-041. Entity types MUST be structural — they describe what an entity IS,
    independent of which domain it belongs to.

R-042. The following entity types MUST be defined by all implementations:
    - `dcg:meta:Domain` — a real-world area of concern
    - `dcg:meta:Type` — a classification of entities

R-043. Implementations MAY define additional built-in entity types (e.g.,
    `dcg:meta:Function`, `dcg:meta:Class`).

### 8.3 Two-Axis Classification

R-044. Every non-Domain, non-Type entity MUST have at least one `"instance of"`
    attribute (structural type).

R-045. Every non-Domain, non-Type entity SHOULD have at least one `"part of"`
    attribute (domain membership). Implementations SHOULD report
    non-Domain, non-Type entities with fewer than
    2 entity-linking attributes (e.g., `"instance of"` + `"part of"`)
    as validation warnings.

### 8.4 Domain Hierarchy

R-046. A domain entity MAY have `"part of"` attributes pointing to other domain
    entities, forming a domain hierarchy. A domain with no parent domain is
    a **top-level domain** (domain root).

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

### 8.5 Ontology Declaration

Domain-specific types, properties, and aliases MUST be declared in a structured
format before use. This enables validation, self-documenting schemas, and
conflict detection across composed layers.

#### 8.5.1 Built-in Ontology (`ontology_builtin`)

R-047. Implementations MUST ship an **`ontology_builtin`** defining the core entity
     types and properties specified in §8.2 and §7.2, plus cross-domain
     vocabulary with Wikidata mappings.

R-048. Built-in types, properties, and aliases from `ontology_builtin` MUST be
     available in every conforming implementation without requiring explicit
     registration.

#### 8.5.2 Per-Project Ontology

R-049. A DCG domain project MUST declare its domain-specific types, properties,
     relations, and aliases in the `ontology` key of `graph_card.json`.
     There is no separate `ontology.yaml` file — `graph_card.json` is the
     single source of truth for a layer's ontology. Projects MAY also
     reference ontology packs via the `packs` key (see [DCG-001-PACK](03-rfc-packs.md)).

R-050. The `ontology` key in `graph_card.json` MUST use the following sub-keys,
     all optional:
     - `types` (array of `{name, description}` objects)
     - `properties` (array of `{name, datatype, description}` objects —
       `datatype` defaults to `"string"`)
     - `aliases` (array of `{alias, canonical}` objects)

#### 8.5.3 Ontology Persistence Format

R-051. When `to_dict()` serializes graph card state, the ontology MUST be
     included under an `"ontology"` key in `graph_card.json` containing only
     custom (non-built-in) types, properties, and aliases.
     Built-in declarations MUST NOT be serialized. See
     [DCG-001-PACK](03-rfc-packs.md) for pack-specific serialization
     exclusion and `packs` key persistence requirements.

#### 8.5.4 Redirect Format

R-052. A redirect MUST be stored as a `"redirected to"` attribute on the old entity,
    pointing to the new entity UID.

---

## 9. Domain Project Format

### 9.1 Overview

A **DCG domain project** is a directory that stores one or more DCG graphs in a
local file-based format optimized for code-based read and write. The
`graph_card.json` file at the project root is the metadata manifest; entity and
relation data lives in separate graph data files under a `graphs/` subdirectory.

JSON is the **canonical on-disk format** for DCG data. All conforming
implementations MUST support JSON for reading and writing DCG graphs.
Implementations MAY additionally support other text-based serialization
formats (e.g., YAML, TOML) for human authoring convenience, provided they
can round-trip losslessly to the canonical JSON representation.

**JSON Schemas:** The file formats are formally defined by JSON Schema:
- [`graph_card.schema.json`](../schema/graph_card.schema.json) — `graph_card.json` metadata manifest
- [`graph_data.schema.json`](../schema/graph_data.schema.json) — graph data files under `graphs/`

**Directory layout:**

```
dcg-security-domain/
  graph_card.json          # metadata + ontology + graphs index (NO entity/relation data)
  graphs/
    default.json           # entities + relations (the main graph)
    threat-model.json      # optional additional subgraph
```

### 9.2 Metadata Manifest — `graph_card.json`

R-053. A DCG domain project MUST contain a `graph_card.json` file at the project
    root. This file serves three roles:
    1. **Project metadata** — identity, version, description
    2. **Ontology declarations** — domain-specific types, properties, aliases
    3. **Graph index** — links to all graph data files in the project

    `graph_card.json` MUST NOT contain entity or relation data. All graph data
    lives in files referenced by the `graphs` index.

R-054. `graph_card.json` MUST contain a `dcg_project` object with at minimum:
    - `name` (string — project identifier)
    - `version` (string — strict semver: `MAJOR.MINOR.PATCH`)

R-055. `graph_card.json` SHOULD contain a `graphs` array indexing all graph data
    files in the project. If the `graphs` array is present, it MUST contain
    at least one entry. Each entry MUST have:
    - `id` (string — graph identifier, e.g., `"default"`)
    - `file` (string — path relative to project root, e.g., `"graphs/default.json"`)
    The default graph is the entry with `"id": "default"`. If no entry has
    `"id": "default"`, the first entry MUST be treated as the default.

R-056. `graph_card.json` MUST NOT contain `entities` or `relations` keys.
    Graph data is stored exclusively in the graph data files referenced
    by the `graphs` index.

R-057. Implementations MUST treat a directory as a valid DCG domain project
    if and only if it contains a `graph_card.json` with a `dcg_project` field.

R-058. `dcg_project` SHOULD additionally contain:
    - `description` (string — human-readable project description)

**Example `graph_card.json`:**

```json
{
  "dcg_project": {
    "name": "app-security-graph",
    "version": "0.2.0",
    "description": "Application security domain knowledge graph"
  },
  "packs": {
    "ontology": ["security"]
  },
  "ontology": {
    "types": [
      {"name": "WeaknessType", "description": "A CWE-catalogued software weakness pattern"},
      {"name": "AttackType", "description": "A named attack technique"}
    ],
    "properties": [
      {"name": "cwe_id", "datatype": "string"},
      {"name": "owasp_id", "datatype": "string"}
    ]
  },
  "graphs": [
    {"id": "default", "file": "graphs/default.json"},
    {"id": "appsec", "file": "graphs/appsec.json"}
  ]
}
```

### 9.3 Graph Data Files

Graph data files contain the actual entity and relation data. They live under
the `graphs/` subdirectory and are referenced by the `graph_card.json` index.

R-059. Graph data files MUST contain:
    - `entities` (object — map of UID to entity object)
    - `relations` (object — map of UID to relation object)

R-060. Graph data files SHOULD contain:
    - `schema_version` (string)

R-061. Graph data files MUST NOT contain `dcg_project`, `ontology`, or `graphs`
    keys — those belong exclusively in `graph_card.json`.

**Example `graphs/default.json`:**

```json
{
  "schema_version": "2.0",
  "entities": {
    "dcg:sha256:a1b2c3d4...": {
      "id": "dcg:sha256:a1b2c3d4...",
      "label": "Improper Input Validation",
      "description": "Failure to validate input before processing",
      "attributes": [
        {"property": "instance of", "ref": "dcg:meta:WeaknessType"},
        {"property": "part of", "ref": "dcg:sha256:appsec-domain..."},
        {"property": "cwe_id", "string": "CWE-20"}
      ]
    }
  },
  "relations": {}
}
```

### 9.4 Intra-Layer Constraint

R-062. All entities and relations stored in a layer's graph data files MUST
    reference only UIDs that exist within that same layer. Cross-layer UID
    references in graph data files are NOT permitted.

R-063. Cross-layer connections MUST NOT be stored in any layer's graph data
    files. See [DCG-001-COMP](02-rfc-composition.md) for join rule
    expression and materialization requirements.

---

## 10. Graph Store Protocol

### 10.1 Store Operations

R-064. A conforming implementation MUST provide a graph store component that
    satisfies the operations defined in R-065.

R-065. The graph store MUST provide at minimum these operations (names are
    logical — implementations MAY use language-idiomatic naming):
    - **add_entity**(entity) — insert or update an entity; return its UID
    - **add_relation**(source, target, property, qualifiers?) — insert a
      relation; return its UID
    - **remove**(uid) — retract an entity by UID
    - **get_entity**(uid) — retrieve an entity by UID, or null if absent
    - **get_relations**(source?, target?, property?) — query relations by
      any combination of source, target, and property filters
    - **query**(instance_of?, part_of?) — query entities by type and/or
      domain membership
    - **to_dict**() — serialize graph card state to an in-memory structure
    - **load**(path) — deserialize a domain project from disk
    - **to_json**(path) — serialize and persist a domain project to disk

R-066. `add_entity()` and `add_relation()` MUST be idempotent — adding an
    entity or relation with an existing UID updates it in place.

R-067. `get_entity()` MUST follow entity redirects (§10.3) transparently,
    up to an implementation-defined maximum depth (RECOMMENDED: 10).

R-068. `get_relations()` MUST resolve property aliases (§7.2) before
    matching.

R-069. `query()` MUST support filtering by `instance_of` and `part_of`
    independently and in combination.

### 10.2 Retraction and Purge

Entities and relations can be soft-deleted (retracted) and later permanently
removed (purged). Retraction marks data as logically deleted without physical
removal; purge permanently removes retracted data.

R-070. `remove(uid)` MUST mark the entity or relation as retracted by setting
    `"retracted": true` on the stored object.

R-071. `get_entity(uid)` MUST return `None` for retracted entities.

R-072. `query()` MUST exclude retracted entities from results.

R-073. `get_relations()` MUST exclude retracted relations from results.

R-074. `purge_retracted()` MUST permanently remove all entities and relations
    with `"retracted": true` from the store, except as specified by R-081.
    Implementations MAY expose this as a store method or a standalone function.

R-075. After purge, `load()` MUST NOT resurrect previously retracted data.

R-076. `purge_retracted()` MUST operate on a single domain project at a time.
    See [DCG-001-COMP](02-rfc-composition.md) for stack-level purge
    orchestration requirements.

### 10.3 Entity Redirects

When an entity's identity keys change (e.g., a function is renamed or a file is
moved), its content-addressed UID changes. Redirects ensure that references to
the old UID continue to resolve. Redirect storage format is defined in R-052.

R-077. Implementations MUST support entity redirects for identity key changes.

R-078. `get_entity()` MUST follow redirects transparently. `get_entity(old_uid)`
    MUST return the entity at the redirect target.

R-079. Redirect chains MUST be followed up to the implementation-defined
    maximum depth (RECOMMENDED: 10). Implementations MUST return `None`
    (or the language-equivalent null value) if the chain exceeds this depth.

R-080. Redirect source entities MUST survive purge operations. `purge_retracted()`
    MUST NOT remove an entity that carries a `"redirected to"` attribute, even
    if the entity is also marked retracted. `remove(uid)` on a redirect source
    MUST preserve the `"redirected to"` attribute in the retraction tombstone.

R-081. If a redirect target entity is retracted and then purged, the redirect
    source entity's `"redirected to"` attribute becomes a dangling pointer.
    Implementations MUST NOT raise an error when following a dangling redirect.
    `get_entity()` MUST return `None` for any redirect chain that terminates at
    a missing or retracted entity (per R-071).

### 10.4 I/O Operations

R-082. Implementations MUST load all graph data files listed in the `graphs`
    index during `load()`. Each file MUST be registered in the project's
    registry under its `id`. If the `graphs` array is absent, `load()` MUST
    treat the project as having an empty graph (no entities, no relations).

R-083. When saving, implementations MUST persist each graph data file to its
    referenced `file` path and update `graph_card.json` to reflect the
    current `graphs` index.

### 10.5 Entity and Attribute Atomicity

R-084. An entity and its attributes MUST be stored and retrieved as one
    atomic unit. Implementations MUST NOT support partial attribute
    persistence — storing an entity MUST persist all its attributes;
    retrieving an entity MUST return all its attributes.

R-085. The entity dict (`id`, `label`, `description`, `attributes`, `aliases`)
    is the **primary data unit** of a DCG graph. All other stored data
    (relations, ontology) is either derived from or supplements entity data.

R-086. `add_entity()` MUST accept a complete entity dict and persist it
    atomically. An implementation MUST NOT accept attributes separately
    from their parent entity.

### 10.6 Relation Classification

R-087. Relations MUST be classified as one of:

- **Derived relations**: automatically materialized from ref-typed
  attributes on entities. When an entity has an attribute with a
  `"ref"` value and a property not in the exempt set
  (`"instance of"`, `"redirected to"`), the store MUST be capable of
  producing a corresponding relation. Derived relations are
  reconstructable from entity data alone.

- **Explicit relations**: created directly via `add_relation()` and
  not derivable from any entity's attributes. Explicit relations
  represent user-asserted facts.

R-088. Explicit relations MUST be persisted and MUST survive round-trip
    (`load()` → `to_json()` → `load()`). Implementations MUST NOT discard
    explicit relations on save.

R-089. Derived relations MAY be stored alongside entities in persistence
    files as a query optimization. When stored, they MUST be
    reconstructable — on `load()`, if derived relations are absent,
    implementations MUST re-derive them from entity attributes.

R-090. Implementations MAY store additional index structures (e.g.,
    relation lookup tables, type indexes) alongside the primary entity
    data to optimize queries. Such structures are supplementary and MUST
    be reconstructable from entity data.

---

## 11. Security Considerations

Entity IDs MUST NOT be derived from user-supplied free text without sanitization.
Identity keys MUST be validated before hashing. Implementations SHOULD validate
that entity JSON does not exceed a configurable maximum size (RECOMMENDED default:
1 MB per entity). These are informative guidance items; conforming implementations
are encouraged to apply defensive input handling beyond what is strictly required
by the data model requirements above.

---

## 12. References

### Normative

- [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt) — Key words for use in RFCs to Indicate Requirement Levels
- [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174) — Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words
- [JSON Schema](https://json-schema.org/) — A Media Type for Describing JSON Documents

### Informative

- [Wikibase JSON (Action API)](https://doc.wikimedia.org/Wikibase/master/php/docs_topics_json.html)
- [Wikibase Data Model](https://www.mediawiki.org/wiki/Wikibase/DataModel/JSON)
- [IPFS Content Identifiers (CIDs)](https://docs.ipfs.tech/concepts/content-addressing/)
- [qwikidata — Offline Wikidata JSON](https://pypi.org/project/qwikidata/)
- [Wikidata Property P31 (instance of)](https://www.wikidata.org/wiki/Property:P31)
- [Domain-Driven Design — Bounded Contexts](https://martinfowler.com/bliki/BoundedContext.html)

---

## Appendix A: Requirement Summary

| R-NNN | Section | Level | Requirement |
|---|---|---|---|
| R-001 | 6.1 Entity Structure | MUST | Entity is a JSON object with id, label, description, attributes |
| R-002 | 6.1 Entity Structure | SHOULD/MUST | aliases field (SHOULD include; treat absent as empty array) |
| R-003 | 6.1 Entity Structure | MUST | Extension fields MUST use `dcg:` prefix |
| R-004 | 6.2 Content-Addressed Entity IDs | MUST | Entity IDs use `dcg:<algorithm>:<hash>` format |
| R-005 | 6.2 Content-Addressed Entity IDs | MUST | sha256 algorithm MUST be supported |
| R-006 | 6.2 Content-Addressed Entity IDs | MAY | Additional hash algorithms may be supported |
| R-007 | 6.2 Content-Addressed Entity IDs | MUST | Hash digest MUST be ≥128 bits (32 hex chars); 64-bit truncation prohibited |
| R-008 | 6.2 Content-Addressed Entity IDs | MUST | Hash computed from sorted identity key-value pairs joined by `:` |
| R-009 | 6.2 Content-Addressed Entity IDs | MUST | Identical IDs for identical identity keys regardless of insertion order |
| R-010 | 6.2 Content-Addressed Entity IDs | MUST | Relation UIDs use `dcg:rel:sha256:` prefix; computed from property+source+target |
| R-011 | 6.2 Content-Addressed Entity IDs | MUST | Entity type IDs exempt from content-addressing; use `dcg:meta:` prefix |
| R-012 | 6.3 Schema Version | SHOULD/MUST | schema_version SHOULD be present; MUST validate when present |
| R-013 | 6.3 Schema Version | — | Current schema version is `"2.0"` |
| R-014 | 6.3 Schema Version | MUST | Reject graphs whose major version exceeds supported major version |
| R-015 | 6.3 Schema Version | SHOULD | Accept higher minor version; preserve unknown fields on round-trip |
| R-016 | 6.3 Schema Version | — | Major bump = breaking; minor bump = additive backward-compatible |
| R-017 | 7.1 Attribute Format | MUST | Attributes are flat array; each has property + exactly one typed value key |
| R-018 | 7.1 Attribute Format | MAY | Attributes may include qualifiers array |
| R-019 | 7.1 Attribute Format | — | Multi-value attributes allowed (same property, different values) |
| R-020 | 7.1 Attribute Format | — | Attribute identity = (property, typed_key, value) |
| R-021 | 7.2 Property Keys | MUST | Property keys MUST be English-language strings |
| R-022 | 7.2 Property Keys | MUST | Core properties (instance of, part of, subclass of) MUST be supported |
| R-023 | 7.2 Property Keys | SHOULD | Property alias registry; `wikidata_P<ID>` naming convention |
| R-024 | 7.2 Property Keys | MUST | Property aliases MUST resolve transparently in get_relations() and query() |
| R-025 | 7.2 Property Keys | MUST | Renamed properties MUST retain old name as alias |
| R-026 | 7.3 Value Types | MUST | Exactly one typed value key per attribute |
| R-027 | 7.3 Value Types | MUST | Core value types: ref, string, quantity |
| R-028 | 7.3 Value Types | MAY | Additional value types may be supported |
| R-029 | 7.3 Value Types | MUST | Unknown typed value keys MUST be preserved on round-trip |
| R-030 | 7.4 Qualifiers | MUST | Qualifiers follow attribute format (property + typed value key) |
| R-031 | 7.4 Qualifiers | — | Qualifiers are metadata about the attribute, not the entity |
| R-032 | 7.5 Retraction | — | Retraction via `"retracted": true` on entity or attribute |
| R-033 | 7.5 Retraction | — | Entity-level retraction marks entire entity as soft-deleted |
| R-034 | 7.5 Retraction | — | Attribute-level retraction marks specific attribute as soft-deleted |
| R-035 | 7.5 Retraction | MUST | Retracted attribute MUST specify same (property, typed_key, value) tuple |
| R-036 | 7.5 Retraction | MUST | After purge, store MUST NOT contain retracted flags (except redirect tombstones per R-081) |
| R-037 | 8.1 Domains as Entities | MUST | Domain = entity with `"instance of" → "dcg:meta:Domain"` |
| R-038 | 8.1 Domains as Entities | MUST | Domain entities receive no special store treatment |
| R-039 | 8.1 Domains as Entities | MUST | Entity membership expressed via `"part of"` attributes |
| R-040 | 8.1 Domains as Entities | MAY | Entity may belong to multiple domains |
| R-041 | 8.2 Entity Types | MUST | Entity types are structural (what an entity IS) |
| R-042 | 8.2 Entity Types | MUST | dcg:meta:Domain and dcg:meta:Type MUST be defined |
| R-043 | 8.2 Entity Types | MAY | Additional built-in entity types may be defined |
| R-044 | 8.3 Two-Axis Classification | MUST | Non-Domain/Type entities MUST have at least one `"instance of"` |
| R-045 | 8.3 Two-Axis Classification | SHOULD | Non-Domain/Type entities SHOULD have `"part of"`; SHOULD report as validation warnings |
| R-046 | 8.4 Domain Hierarchy | MAY | Domain entities may reference other domains via `"part of"` |
| R-047 | 8.5.1 Built-in Ontology | MUST | ontology_builtin MUST be shipped with core types, properties, Wikidata mappings |
| R-048 | 8.5.1 Built-in Ontology | MUST | ontology_builtin declarations available without explicit registration |
| R-049 | 8.5.2 Per-Project Ontology | MUST | Domain-specific ontology declared in `graph_card.json` ontology key |
| R-050 | 8.5.2 Per-Project Ontology | MUST | ontology key uses types/properties/aliases sub-keys |
| R-051 | 8.5.3 Ontology Persistence | MUST | to_dict() serializes only custom declarations; built-in excluded; packs per DCG-001-PACK |
| R-052 | 8.5.4 Redirect Format | MUST | Redirect stored as `"redirected to"` attribute on old entity |
| R-053 | 9.2 Metadata Manifest | MUST | graph_card.json MUST be at project root; no entity/relation data |
| R-054 | 9.2 Metadata Manifest | MUST | dcg_project object MUST contain name and version (strict semver) |
| R-055 | 9.2 Metadata Manifest | SHOULD/MUST | graphs array SHOULD be present; MUST have ≥1 entry with id and file; id "default" is default graph |
| R-056 | 9.2 Metadata Manifest | MUST | graph_card.json MUST NOT contain entities or relations keys |
| R-057 | 9.2 Metadata Manifest | MUST | Directory is valid project iff graph_card.json has dcg_project field |
| R-058 | 9.2 Metadata Manifest | SHOULD | dcg_project SHOULD additionally contain description |
| R-059 | 9.3 Graph Data Files | MUST | Graph data files MUST contain entities and relations |
| R-060 | 9.3 Graph Data Files | SHOULD | Graph data files SHOULD contain schema_version |
| R-061 | 9.3 Graph Data Files | MUST | Graph data files MUST NOT contain dcg_project, ontology, or graphs keys |
| R-062 | 9.4 Intra-Layer Constraint | MUST | All UID refs in a layer MUST be intra-layer only |
| R-063 | 9.4 Intra-Layer Constraint | MUST | Cross-layer connections MUST NOT be stored in graph data; join rules per DCG-001-COMP |
| R-064 | 10.1 Store Operations | MUST | Implementation MUST provide a graph store satisfying R-065 operations |
| R-065 | 10.1 Store Operations | MUST | Graph store MUST provide listed logical operations (language-neutral) |
| R-066 | 10.1 Store Operations | MUST | add_entity() and add_relation() MUST be idempotent |
| R-067 | 10.1 Store Operations | MUST | get_entity() MUST follow redirects up to impl-defined depth (RECOMMENDED 10) |
| R-068 | 10.1 Store Operations | MUST | get_relations() MUST resolve property aliases before matching |
| R-069 | 10.1 Store Operations | MUST | query() MUST support filtering by instance_of and part_of |
| R-070 | 10.2 Retraction and Purge | MUST | remove(uid) MUST mark entity/relation as retracted |
| R-071 | 10.2 Retraction and Purge | MUST | get_entity(uid) MUST return None for retracted entities |
| R-072 | 10.2 Retraction and Purge | MUST | query() MUST exclude retracted entities |
| R-073 | 10.2 Retraction and Purge | MUST | get_relations() MUST exclude retracted relations |
| R-074 | 10.2 Retraction and Purge | MUST | purge_retracted() MUST permanently remove retracted data (except R-081 tombstones) |
| R-075 | 10.2 Retraction and Purge | MUST | After purge, load() MUST NOT resurrect retracted data |
| R-076 | 10.2 Retraction and Purge | MUST | purge_retracted() MUST operate on single domain project; stack orchestration per DCG-001-COMP |
| R-077 | 10.3 Entity Redirects | MUST | Implementations MUST support entity redirects |
| R-078 | 10.3 Entity Redirects | MUST | get_entity() MUST follow redirects transparently |
| R-079 | 10.3 Entity Redirects | MUST | Redirect chains MUST be followed up to impl-defined depth (RECOMMENDED 10); return null if exceeded |
| R-080 | 10.3 Entity Redirects | MUST | Redirect source entities MUST survive purge; "redirected to" preserved in tombstone |
| R-081 | 10.3 Entity Redirects | MUST | Dangling redirects MUST NOT raise errors; get_entity() returns None for missing/retracted target |
| R-082 | 10.4 I/O Operations | MUST | load() MUST load all graph data files in graphs index; absent graphs array = empty graph |
| R-083 | 10.4 I/O Operations | MUST | to_json(path) MUST persist each graph data file and update graph_card.json graphs index |
| R-084 | 10.5 Entity/Attribute Atomicity | MUST | Entity and attributes stored/retrieved as one atomic unit; no partial attribute persistence |
| R-085 | 10.5 Entity/Attribute Atomicity | — | Entity dict is the primary data unit; relations and ontology are derived/supplementary |
| R-086 | 10.5 Entity/Attribute Atomicity | MUST | add_entity() MUST accept complete entity dict atomically; no separate attribute persistence |
| R-087 | 10.6 Relation Classification | MUST | Relations classified as derived (from ref attributes, reconstructable) or explicit (via add_relation()) |
| R-088 | 10.6 Relation Classification | MUST | Explicit relations MUST survive round-trip (load → save → load) |
| R-089 | 10.6 Relation Classification | MAY/MUST | Derived relations MAY be stored; MUST be reconstructable if absent on load |
| R-090 | 10.6 Relation Classification | MAY/MUST | Additional index structures MAY be stored; MUST be reconstructable from entity data |

---

## Appendix B: Wikidata Alias Mapping (Informative)

This appendix describes optional Wikidata interoperability patterns. These are
non-normative — implementations MAY support them but are not required to.

DCG uses a compact English-first format internally. For Wikidata interoperability,
the compact format maps bidirectionally to the Wikibase Action API JSON format:

- The `label` field (string) is shorthand for the Wikibase labels structure:
  `"label": "foo"` expands to `{"en": {"language": "en", "value": "foo"}}`.
- The `description` field (string) is shorthand for the Wikibase descriptions
  structure, following the same expansion as labels.
- The `aliases` field (array of strings) is shorthand for the Wikibase aliases
  structure: `["a", "b"]` expands to
  `{"en": [{"language": "en", "value": "a"}, {"language": "en", "value": "b"}]}`.

Implementations MAY use these mappings for import/export with Wikidata systems.
The `ontology_builtin` includes `wikidata_P<ID>` aliases for common properties
(see §8.5.1).

### B.1 Offline Import via qwikidata

Implementations MAY use the `qwikidata` library (or equivalent) to import
Wikidata entities from offline JSON dumps. The adapter SHOULD convert
Wikibase JSON to DCG entity format, mapping `wikidata_P31` to `instance of`,
`wikidata_P279` to `subclass of`, and other well-known properties to their
canonical DCG names via the property alias registry.

### B.2 Round-Trip Fidelity

Implementations that support Wikidata export SHOULD preserve enough
metadata to allow round-trip conversion: DCG → Wikibase JSON → DCG
without semantic loss. Qualifiers, references, and rank information
SHOULD be preserved in attribute qualifiers where possible.

### B.3 Property Alias Conventions

The `ontology_builtin` ships with Wikidata property aliases using the
`wikidata_P<ID>` naming convention (e.g., `wikidata_P31` → `instance of`,
`wikidata_P279` → `subclass of`). Implementations MAY register additional
Wikidata aliases via `register_alias()` for project-specific properties,
following the same `wikidata_P<ID>` convention.

---

## Appendix C: Design Rationale (Informative)

The `graph_card.json` + `graphs/` layout is designed for local, code-based
workflows:

- **Clean separation of concerns** — metadata and ontology in `graph_card.json`;
  entity data in graph data files; no mixed responsibilities
- **Single entry point** — one file to discover the entire project's metadata
  and graph index
- **Human and LLM readable** — JSON with English property keys is inspectable
  by both humans and language models without specialized tooling
- **Git-friendly** — the project directory can be a git repository; JSON diffs
  are meaningful in code review
- **Splittable** — large graphs can be partitioned across multiple graph data
  files without changing the protocol
- **Token-efficient** — the compact attribute format minimizes tokens per entity,
  maximizing the knowledge that fits in an LLM's context window
- **Swappable** — because cross-layer connections are expressed as
  vocabulary-based join rules (not UID references), a parent project can be
  rebuilt with new UIDs without breaking child project data

---

## Appendix D: Change Log

**2026-06-30 — Core protocol relaxation for extension-neutral minimalism**

Relaxed requirements to keep the core protocol minimal and delegate
extension-specific concerns to the appropriate extension RFCs:
- R-023: Generalized property alias registry (Wikidata naming moved to Appendix B)
- R-045: Validation reporting relaxed from MUST to SHOULD; strict mode deferred to DCG-001-COMP
- R-051: Pack-specific serialization exclusion deferred to DCG-001-PACK
- R-055: Default graph identified by `"id": "default"` instead of array position
- R-063: Join-rule mandate moved to DCG-001-COMP; core retains only storage prohibition
- R-064/R-065: Language-neutral store protocol descriptions (removed Python-specific signatures)
- R-067/R-079: Redirect depth made implementation-defined (RECOMMENDED 10)
- R-076: Stack-level purge orchestration deferred to DCG-001-COMP
- §9.1: JSON canonical format with MAY for alternative text-based formats

**2026-06-30 — §10.5 Entity/Attribute Atomicity + §10.6 Relation Classification**

Added R-084..R-090. Formalizes that entities and their attributes are one
atomic unit in persistence (§10.5), and that relations are classified as
derived (reconstructable from entity ref attributes) or explicit (user-asserted
via `add_relation()`) (§10.6). These clarifications support the adapter
extension [DCG-001-ADAPT](04-rfc-store-adaptor.md) which depends on entity atomicity
for enrichment overlay semantics.

**2026-06-26 — DCG-001 initial split**

Split from the monolithic DCG-002 protocol RFC. This document contains:
- Wire format and data model (§6–§8)
- Domain project on-disk format (§9)
- Graph Store Protocol including retraction, purge, redirects, and I/O (§10)

Content moved to companion documents:
- Stack composition → [DCG-001-COMP](02-rfc-composition.md)
- Pack system → [DCG-001-PACK](03-rfc-packs.md)
- Implementation guidance and runtime behavior → DCG-001-IMP (spec)

Requirements renumbered sequentially R-001..R-083. Wikidata structural
compatibility (old R-4..R-6) moved to Appendix B (informative, no formal
requirements). Schema version reference updated from "DCG-002" to "DCG-001".
