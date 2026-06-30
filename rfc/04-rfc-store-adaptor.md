# RFC: Domain Context Graph Adapter Extension

**RFC ID:** DCG-001-ADAPT
**Status:** Draft
**Date:** 2026-06-30
**Extends:** [DCG-001](01-rfc.md) | [DCG-001-COMP](02-rfc-composition.md)

---

**Table of Contents**

- [Introduction](#1-introduction)
- [Status of This Memo](#2-status-of-this-memo)
- [Terminology](#3-terminology)
- [BCP 14 Boilerplate](#4-bcp-14-boilerplate)
- [Layer Adapter Protocol](#5-layer-adapter-protocol)
  - [Source Anchor Identity](#51-source-anchor-identity)
- [Stack Manifest Integration](#6-stack-manifest-integration)
- [Enrichment Overlay](#7-enrichment-overlay)
  - [Runtime Overlay Application](#71-runtime-overlay-application)
- [Security Considerations](#8-security-considerations)
- [References](#9-references)
- [Appendix A: Requirement Summary](#appendix-a-requirement-summary)
- [Appendix B: Overlay File Example](#appendix-b-overlay-file-example)
- [Appendix C: Overlay Application Diagram](#appendix-c-overlay-application-diagram)
- [Appendix D: Change Log](#appendix-d-change-log)

---

## 1. Introduction

External code knowledge graph tools (tree-sitter-based analyzers, language
servers, static analysis frameworks) produce structured graphs that capture code
entities and their relationships. These graphs use tool-specific formats and
identifier systems that differ from DCG's content-addressed entity model.

This extension defines how such external graphs can back a DCG composition layer
at runtime, and how enrichments (new attributes, new entities, new relations) can
be persisted without modifying the external source.

**Scope:** This extension covers the layer adapter load protocol, source anchor
identity for stable entity matching, stack manifest integration for
adapter-backed layers, enrichment overlay persistence format, and runtime overlay
application semantics including two-phase entity matching.

**Conformance:** A conforming implementation MAY support adapters; if it does, it
MUST satisfy all MUST requirements in this extension.

---

## 2. Status of This Memo

This document specifies an optional extension to [DCG-001](01-rfc.md) for
loading external knowledge graphs into DCG composition layers via adapters and
persisting enrichments via overlay files. Distribution of this memo is unlimited.

---

## 3. Terminology

The following terms are defined for use in this extension. All base terms
(Entity, Attribute, Domain, Graph Card, etc.) are defined in [DCG-001](01-rfc.md).
Composition terms (Stack, Composition Layer, Extends DAG, Join Rule) are defined
in [DCG-001-COMP](02-rfc-composition.md).

- **Layer Adapter**: A read-only component that populates a GraphStore from an
  external source format. Adapters translate tool-specific graph formats into
  DCG entities and relations.
- **Source Anchor**: A string attribute on each adapter-produced entity that
  carries the external source's own stable identifier for that entity (e.g.,
  node ID, qualified name, URI). Used as a fallback matching key when UID-based
  overlay matching fails due to identity drift.
- **Enrichment Overlay**: A DCG-native JSON file (`.dcg/overlay.json`) that
  stores only the delta between the adapter-loaded baseline and user
  modifications. The overlay is applied on top of the baseline at load time.
- **Baseline**: The store state immediately after adapter load, before overlay
  application. The baseline is the adapter's read-only contribution; enrichments
  are everything added on top.
- **Adapter-Backed Layer**: A composition layer loaded via adapter rather than
  from a `graph_card.json` domain project. Adapter-backed layers participate in
  the same composition semantics as native layers.

---

## 4. BCP 14 Boilerplate

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14
[RFC 2119](https://www.ietf.org/rfc/rfc2119.txt)
[RFC 8174](https://www.rfc-editor.org/rfc/rfc8174) when, and only when, they
appear in all capitals, as shown here.

---

## 5. Layer Adapter Protocol

R-001. A Layer Adapter MUST implement:
    - `load(source, store, config)` — populate a GraphStore from external data
      located at `source`, using adapter-specific `config` options
    - `source_exists(source)` — return a boolean indicating whether the source
      path contains valid data for this adapter

R-002. `load()` MUST populate the provided GraphStore with entities and relations
    conforming to [DCG-001](01-rfc.md) (R-001, R-084–R-086). Every entity added
    to the store MUST be a valid DCG entity.

R-003. `load()` MUST NOT modify the external source. The adapter is read-only
    with respect to the external data it translates.

R-004. Every entity produced by `load()` MUST conform to DCG entity format — a
    JSON object with `id`, `label`, `description`, and `attributes` as an
    atomic unit (R-084 [DCG-001](01-rfc.md)).

R-005. Entity UIDs MUST use the standard content-addressed scheme
    (R-004–R-009 [DCG-001](01-rfc.md)).

R-006. The adapter MUST register any domain-specific types and properties in the
    store's ontology before adding entities that reference them. Entities MUST
    NOT reference unregistered types or properties.

### 5.1 Source Anchor Identity

External source formats have their own identifier systems (node IDs, URIs,
database keys) that are independent of DCG's content-addressed UIDs. Because
DCG UIDs are computed from entity properties (name, file path), they change
when those properties change — for example, when a file is renamed. Source
anchors bridge external identity and DCG identity, enabling stable overlay
matching across source changes.

R-007. Every entity produced by `load()` MUST carry a `"source_anchor"`
    attribute — a string value that uniquely identifies this entity within
    the external source. The anchor MUST be the external system's own stable
    identifier (e.g., node ID, qualified name, URI).

R-008. Source anchors MUST be stable across reloads of the same external
    source — the same logical entity MUST produce the same anchor value even
    if other properties (like file path) change. When the external source
    provides no stable identifier, the adapter SHOULD construct one from the
    most stable available properties (e.g., fully qualified name).

R-009. The `"source_anchor"` property MUST be registered in the store's
    ontology as a string-typed property.

---

## 6. Stack Manifest Integration

R-010. Layer entries in `dcg-stack.yml` ([DCG-001-COMP](02-rfc-composition.md))
    MAY include an `adapter` field (string — adapter identifier). When
    present, the layer is loaded via adapter rather than as a native DCG
    domain project.

R-011. When `adapter` is present, `source` MUST point to the external data
    directory (the adapter's input), not a DCG domain project directory.

R-012. Layer entries with `adapter` MAY include a `mapping` field (object).
    The mapping is adapter-specific configuration passed as `config` to
    `load()`. The stack MUST pass the mapping verbatim without
    interpretation.

R-013. Adapter-backed layers MUST participate in the same DAG validation,
    ontology merge, join materialization, and query semantics as native
    layers — all requirements from [DCG-001-COMP](02-rfc-composition.md)
    apply.

R-014. An adapter-backed layer without a `graph_card.json` MUST still
    function. The stack MUST construct layer metadata (name, version) from
    the layer entry in the stack manifest.

**Example stack manifest with adapter layer:**

```yaml
stack: appsec-ctx-graph
layers:
  - name: security
    source: ./dcg-appsec/graphs/security
  - name: product
    source: ./dcg-appsec/graphs/product-domain
    extends: [security]
  - name: code
    adapter: graphify
    source: ./graphify-out
    extends: [product]
    mapping:
      code_only: true
joins:
  - implements: Function.addresses_domain -> SecurityDomain.label
```

---

## 7. Enrichment Overlay

R-015. Adapter-backed layers MUST support enrichment — adding new entities,
    modifying attributes on existing entities, and adding new relations —
    via normal GraphStore operations (`add_entity()`, `add_relation()`).

R-016. Enrichments MUST be persisted in a DCG-native overlay file at
    `<source>/.dcg/overlay.json`, where `<source>` is the adapter layer's
    source directory.

R-017. The overlay file MUST use DCG graph data format
    (R-059 [DCG-001](01-rfc.md)) with `schema_version`, `entities`, and
    `relations` keys. Note: `schema_version` is REQUIRED in overlay
    files, upgrading the SHOULD in R-060 [DCG-001](01-rfc.md), to
    ensure overlay readers can reject version-incompatible overlays.

R-018. The overlay MUST contain only the delta vs. the adapter-loaded
    baseline — new or modified entities and new relations. Unmodified
    baseline entities MUST NOT appear in the overlay.

R-019. `to_json()` (R-083 [DCG-001](01-rfc.md)) on an adapter-backed layer
    MUST write only to the overlay file. The external source MUST NOT be
    modified. Implementations MAY expose this as a `save()` convenience
    method; when present, `save()` MUST delegate to the overlay write
    path defined here.

R-020. Implementations MUST track the baseline state to compute the delta
    on save. Content hashing (e.g., MD5 or SHA-256 of the JSON-serialized
    entity dict) is RECOMMENDED for detecting modifications.

R-021. When the external source changes between loads (e.g., after a code
    regeneration), the overlay MUST still apply on next load. Overlay
    entities whose UIDs no longer exist in the new baseline are matched
    via source anchor fallback (R-023) or inserted as new entities.

### 7.1 Runtime Overlay Application

R-022. On load, the overlay MUST be applied after the adapter populates the
    store. The load sequence is:

    1. Adapter `load()` populates the GraphStore (baseline)
    2. Baseline snapshot is captured — UID-to-content-hash mapping, and
       source-anchor-to-UID index
    3. Overlay file is read (if present at `<source>/.dcg/overlay.json`)
    4. Each overlay entity is matched and applied (see R-023)
    5. Each overlay relation is applied (see R-026)

R-023. Overlay entity matching MUST use a two-phase strategy:

    1. **Primary: UID match** — if the overlay entity's UID exists in the
       baseline, replace that baseline entity (entity-level replacement
       per R-025).

    2. **Fallback: Source anchor match** — if the overlay entity's UID is
       NOT in the baseline, extract its `source_anchor` attribute and
       search the baseline's source-anchor index for an entity with the
       same `source_anchor` value. If found, the overlay entity replaces
       the matched baseline entity AND the overlay entity's UID is
       updated to the baseline entity's current UID (see R-024).

       If the overlay entity has no `source_anchor` attribute (e.g., a
       user-created entity not originating from the adapter), phase 2
       MUST be skipped and the entity proceeds directly to step 3.

       If multiple baseline entities share the same `source_anchor`
       value, the first match in iteration order MUST be used.
       Implementations SHOULD log a warning when anchor collisions are
       detected.

    3. **No match** — if neither UID nor source anchor matches a baseline
       entity, the overlay entity is inserted as a new entity in the
       merged store.

R-024. When a source-anchor fallback match updates an overlay entity's
    UID (R-023 phase 2), the updated UID MUST be persisted back to the
    overlay file on the next save. This ensures subsequent reloads use
    the current UID directly (phase 1) rather than repeatedly falling
    back to anchor matching.

R-025. Overlay MUST use **entity-level replacement** semantics. When an
    overlay entity matches a baseline entity (by UID or source anchor),
    the overlay entity replaces the baseline entity entirely.
    Attribute-level merge MUST NOT be used. The overlay entity carries the
    complete entity dict (all attributes — both original and enriched).

R-026. Overlay relations MUST be **additive**. Each overlay relation is
    inserted into the store. Relations that duplicate existing relations
    (same UID) MUST be treated as idempotent updates
    (R-066 [DCG-001](01-rfc.md) governs `add_entity()` idempotency;
    implementations MUST apply equivalent idempotent semantics to
    `add_relation()`).

---

## 8. Security Considerations

Adapter code has read access to arbitrary file paths specified in the stack
manifest's `source` field. Implementations SHOULD validate source paths and
SHOULD NOT follow symlinks outside the project root without explicit
configuration.

Overlay files are user-modifiable JSON. Implementations SHOULD validate overlay
file structure and entity format on load to prevent injection of malformed
entities into the graph.

---

## 9. References

### Normative

- [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt) — Key words for use in RFCs to Indicate Requirement Levels
- [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174) — Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words
- [DCG-001](01-rfc.md) — Domain Context Graph Core Protocol
- [DCG-001-COMP](02-rfc-composition.md) — Domain Context Graph Composition Extension

---

## Appendix A: Requirement Summary

| R-NNN | Section | Level | Requirement |
|---|---|---|---|
| R-001 | 5 Layer Adapter Protocol | MUST | Adapter implements `load(source, store, config)` and `source_exists(source)` |
| R-002 | 5 Layer Adapter Protocol | MUST | `load()` populates store with DCG-conformant entities and relations |
| R-003 | 5 Layer Adapter Protocol | MUST | `load()` MUST NOT modify the external source |
| R-004 | 5 Layer Adapter Protocol | MUST | Entities conform to DCG entity format (id, label, description, attributes) |
| R-005 | 5 Layer Adapter Protocol | MUST | Entity UIDs use content-addressed scheme |
| R-006 | 5 Layer Adapter Protocol | MUST | Adapter registers types/properties in ontology before use |
| R-007 | 5.1 Source Anchor Identity | MUST | Every entity MUST carry `source_anchor` attribute from external source ID |
| R-008 | 5.1 Source Anchor Identity | MUST/SHOULD | Source anchors MUST be stable across reloads; SHOULD construct from stable properties |
| R-009 | 5.1 Source Anchor Identity | MUST | `source_anchor` property registered in ontology as string type |
| R-010 | 6 Stack Manifest Integration | MAY | Layer entries MAY include `adapter` field |
| R-011 | 6 Stack Manifest Integration | MUST | When adapter present, `source` points to external data directory |
| R-012 | 6 Stack Manifest Integration | MAY | Layer entries MAY include `mapping` field (adapter config) |
| R-013 | 6 Stack Manifest Integration | MUST | Adapter-backed layers participate in full composition semantics |
| R-014 | 6 Stack Manifest Integration | MUST | Adapter-backed layers work without `graph_card.json` |
| R-015 | 7 Enrichment Overlay | MUST | Adapter layers support enrichment via normal GraphStore operations |
| R-016 | 7 Enrichment Overlay | MUST | Enrichments persisted at `<source>/.dcg/overlay.json` |
| R-017 | 7 Enrichment Overlay | MUST | Overlay uses DCG graph data format with schema_version |
| R-018 | 7 Enrichment Overlay | MUST | Overlay contains only the delta vs. baseline |
| R-019 | 7 Enrichment Overlay | MUST | `to_json()` on adapter-backed layer writes only to overlay; `save()` MAY alias |
| R-020 | 7 Enrichment Overlay | MUST | Baseline state tracked for delta computation; content hashing RECOMMENDED |
| R-021 | 7 Enrichment Overlay | MUST | Overlay applies on reload even when external source changes |
| R-022 | 7.1 Runtime Overlay Application | MUST | Overlay applied after adapter load; defined 5-step sequence |
| R-023 | 7.1 Runtime Overlay Application | MUST | Two-phase matching: UID primary, source anchor fallback (skip if no anchor; first match on collision), insert on no match |
| R-024 | 7.1 Runtime Overlay Application | MUST | UID rewrite from anchor fallback MUST be persisted to overlay on next save |
| R-025 | 7.1 Runtime Overlay Application | MUST | Entity-level replacement semantics; no attribute-level merge |
| R-026 | 7.1 Runtime Overlay Application | MUST | Overlay relations are additive; idempotent on duplicate UID |

---

## Appendix B: Overlay File Example

```json
{
  "schema_version": "2.0",
  "entities": {
    "dcg:sha256:a3f7c2d1e4b89012f4c5d6e7a8b90123": {
      "id": "dcg:sha256:a3f7c2d1e4b89012f4c5d6e7a8b90123",
      "label": "AuthService",
      "description": "Handles authentication",
      "attributes": [
        {"property": "instance of", "ref": "dcg:meta:Class"},
        {"property": "file path", "string": "src/auth.py"},
        {"property": "source_anchor", "string": "n1"},
        {"property": "line number", "quantity": 10},
        {"property": "addresses_domain", "string": "Identity & Access Security"}
      ]
    }
  },
  "relations": {}
}
```

The entity above shows an enriched adapter-loaded entity. The `source_anchor`
(`"n1"`) is the external source's node ID. The `addresses_domain` attribute is
the enrichment added by a user or AI agent. The entire entity is stored in the
overlay — not just the enriched attribute — per entity-level replacement
semantics (R-025).

---

## Appendix C: Overlay Application Diagram

```
Load sequence:

  External Source          Overlay (.dcg/overlay.json)
  (e.g. graph.json)       (DCG-native delta)
        |                         |
        v                         |
  adapter.load()                  |
  (each entity gets               |
   source_anchor attr)             |
        |                         |
        v                         |
  +---------------+               |
  |  GraphStore   | <- baseline   |
  |  (N entities) |   snapshot +  |
  |               |   anchor idx  |
  +-------+-------+               |
          |                       v
          |             +------------------------+
          |             |  Apply overlay          |
          |             |  Phase 1: UID match ->  |
          |             |    replace entity       |
          |             |  Phase 2: anchor        |
          |             |    fallback -> replace   |
          |             |    + rewrite UID        |
          |             |  No match -> insert     |
          |             +-----------+------------+
          |                         |
          v                         v
  +--------------------------------------+
  |  Merged GraphStore                   |
  |  (baseline + enrichments)            |
  |  Ready for queries, joins, MCP       |
  +--------------------------------------+
```

---

## Appendix D: Change Log

**2026-06-30 — DCG-001-ADAPT initial draft**

Initial extension RFC. Defines layer adapter protocol (R-001–R-006), source
anchor identity for stable overlay matching (R-007–R-009), stack manifest
integration (R-010–R-014), enrichment overlay format and persistence
(R-015–R-021), and runtime overlay application with two-phase entity matching
and UID rewrite persistence (R-022–R-026).
