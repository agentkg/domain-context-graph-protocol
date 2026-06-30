# RFC: Domain Context Graph Composition Extension

**RFC ID:** DCG-001-COMP
**Status:** Draft
**Date:** 2026-06-26
**Extends:** [DCG-001](01-rfc.md)
**Schemas:** [stack](../schema/stack.schema.json)

---

## 1. Introduction

This document is an optional extension to the Domain Context Graph protocol
([DCG-001](01-rfc.md)). It specifies how multiple DCG domain projects may be
composed into a unified knowledge graph using a **stack manifest**. Each project
in the stack is a **composition layer**. Cross-layer connections are expressed as
join rules and materialized at stack load time, keeping layer data stores
independent.

**Scope:** This extension covers stack manifest format, DAG validation, join rule
syntax and evaluation, cross-layer join materialization, multi-parent entity
resolution, and cross-layer query semantics.

**Conformance:** A conforming implementation MAY support composition; if it does,
it MUST satisfy all MUST requirements in this extension.

---

## 2. Status of This Memo

This document specifies an optional extension to [DCG-001](01-rfc.md) for
composing multiple domain projects into a unified knowledge graph. Distribution
of this memo is unlimited.

---

## 3. Terminology

The following terms are defined for use in this extension. All base terms
(Entity, Attribute, Domain, Graph Card, etc.) are defined in [DCG-001](01-rfc.md).

- **Stack**: A composition of multiple DCG domain projects into a unified
  knowledge graph, defined by a YAML manifest (conventionally `dcg-stack.yml`).
- **Composition Layer**: A single DCG domain project within a stack.
  Composition layers separate concerns across independent projects. "Layer" in
  this extension always refers to a composition layer — an entry in
  `dcg-stack.yml`. There is no versioning-layer concept.
- **Extends DAG**: The directed acyclic graph formed by the `extends` fields
  across all composition layers in a stack. Each layer may declare multiple
  parents; the stack validates acyclicity at load time.
- **Join Rule**: A compact declaration in the stack manifest (`joins` section)
  that instructs the stack to materialize cross-layer relations by joining
  entities from different layers on a shared property value.
- **Cross-Layer Relation**: A relation produced at stack load time by evaluating
  a join rule. Cross-layer relations are read-only and computed, not stored in
  any layer's graph data files.

---

## 4. BCP 14 Boilerplate

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14
[RFC 2119](https://www.ietf.org/rfc/rfc2119.txt)
[RFC 8174](https://www.rfc-editor.org/rfc/rfc8174) when, and only when, they
appear in all capitals, as shown here.

---

## Table of Contents

- [Introduction](#1-introduction)
- [Status of This Memo](#2-status-of-this-memo)
- [Terminology](#3-terminology)
- [BCP 14 Boilerplate](#4-bcp-14-boilerplate)
- [Stack Manifest Format](#5-stack-manifest-format)
- [DAG Validation and Load Order](#6-dag-validation-and-load-order)
- [Join Rule Syntax](#7-join-rule-syntax)
- [Cross-Layer Join Materialization](#8-cross-layer-join-materialization)
- [Multi-Parent Entity Resolution](#9-multi-parent-entity-resolution)
- [Cross-Layer Queries](#10-cross-layer-queries)
- [Composition Guarantees](#11-composition-guarantees)
- [Security Considerations](#12-security-considerations)
- [References](#13-references)
  - [Normative](#normative)
- [Appendix A: Requirement Summary](#appendix-a-requirement-summary)
- [Appendix B: Change Log](#appendix-b-change-log)

---

## 5. Stack Manifest Format

A **stack** composes multiple DCG domain projects into a unified knowledge
graph. Each project in the stack is a **composition layer**. Composition
layers form a directed acyclic graph (DAG) — each layer may declare multiple
parent layers. Cross-layer connections are materialized from ontology-governed
join rules at stack load time, not stored in layer data.

Entities and attributes within each layer MUST conform to the format defined in
R-001 through R-036 [DCG-001].

R-001. A stack MUST be defined by a YAML manifest file (conventionally named
    `dcg-stack.yml`).

R-002. The manifest MUST contain:
    - `stack` (string — human-readable stack name)
    - `layers` (array — ordered list of composition layer entries)

R-003. The manifest MAY contain:
    - `strict` (boolean — enable strict validation; default `false`).
      When `strict: true`, every non-Domain, non-Type entity in each
      layer MUST have at least one `"instance of"` attribute AND at
      least one `"part of"` attribute; absence of either MUST be a
      load error. When `strict: false` (the default), these checks
      are advisory only (SHOULD-level per R-045 [DCG-001]).
    - `joins` (array — cross-layer join rules; see §7)

    The manifest MUST NOT contain an `ontology` block. Ontology declarations
    belong in each layer's `graph_card.json`.

R-004. Each layer entry MUST contain:
    - `name` (string — unique identifier for this composition layer)
    - `source` (string — path to the DCG domain project directory,
      resolved relative to the manifest file)

R-005. Each layer entry MAY contain:
    - `extends` (array of strings — names of layers in this stack that this
      layer inherits from for entity resolution; default `[]`)

R-006. Layer names MUST be unique within a stack manifest.

R-007. Every name in an `extends` list MUST reference a layer name declared in
    the same manifest.

R-008. The extends DAG MUST be acyclic. Implementations MUST detect and reject
     cycles at load time (see §6).

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
  - mitigates: SecurityCapability.targets_cwe -> AttackType.cwe_id
  - triggers:  AttackType.exploits_cwe -> WeaknessType.cwe_id
  - addresses: SecurityDomain.covers_owasp -> OWASPCategory.owasp_id
```

---

## 6. DAG Validation and Load Order

At load time, before loading any project data, implementations MUST perform
the following validation steps in order:

R-009. **Reference validation.** Every name in every `extends` list MUST refer
     to a layer name declared in the same manifest. Implementations MUST
     reject manifests with unresolvable extends references.

R-010. **Cycle detection.** A topological sort MUST be performed on the extends
     DAG. If a cycle is detected, implementations MUST fail with an error
     naming the layers involved in the cycle.

R-011. **Load order.** Implementations MUST load composition layers in
     topological order (parents before children). Layer ontologies MUST be
     merged incrementally in this order so that child layers can reference
     types and properties declared by parent layers.

R-012. **Join rule validation.** After merging all layer ontologies, every type
     and property referenced in `joins` rules MUST exist in the merged
     ontology. Implementations MUST reject stacks where a join rule
     references an undeclared type or property.

---

## 7. Join Rule Syntax

Join rules appear in the `joins` array of the stack manifest. Each rule
instructs the stack to join entities from different layers on a shared
property value and materialize a cross-layer relation.

R-013. Each join rule MUST be a single-key YAML mapping. The key is the
     relation property name; the value is the join expression.

R-014. The join expression MUST use the following syntax:

     ```
     [<LayerName>.]<SourceType>.<match_property> -> [<LayerName>.]<TargetType>.<match_property>
     ```

     - `LayerName` (optional) — restricts matching to the named layer
     - `SourceType` — the `instance of` type of the source entity
     - `match_property` — the property on the source whose value is matched
     - `TargetType` — the `instance of` type of the target entity
     - `match_property` — the property on the target whose value is matched

     When a layer prefix is present, entity matching for that side of the
     rule MUST be restricted to entities in the named layer only. When
     absent, all layers are searched.

R-015. The `.` character MUST NOT appear in entity type names, property
     names, or layer names. This guarantees unambiguous parsing of the
     join expression. Disambiguation between layer name and type name
     relies on case: layer names start lowercase, type names start
     uppercase.

R-016. The full join rule grammar is:

     ```
     join_rule      := property_name ":" SP join_expr
     join_expr      := type_prop SP "->" SP type_prop
     type_prop      := [LayerName "."] TypeName "." prop_name
     LayerName      := [a-z][a-z0-9_-]*
     TypeName       := [A-Z][A-Za-z0-9_-]*
     property_name  := [a-z][a-z0-9_]*( SP [a-z][a-z0-9_]*)*
     prop_name      := [a-z][a-z0-9_]*
     ```

     Implementations MUST enforce this grammar via regex or schema
     validation. Join rules that do not match MUST be rejected at
     parse time with a descriptive error.

R-017. When a join rule specifies a layer prefix, the referenced layer
     name MUST exist in the stack's `layers` array. Implementations
     MUST reject stacks where a join rule references an undeclared
     layer.

**Example join rules:**

```yaml
joins:
  - mitigates: SecurityCapability.targets_cwe -> WeaknessType.cwe_id
  - mitigates: security.SecurityCapability.targets_cwe -> AttackType.cwe_id
  - triggers:  AttackType.exploits_cwe -> security.WeaknessType.cwe_id
```

Each rule is parsed into a structured `JoinRule` with fields:
- `property` — the relation property (e.g., `"mitigates"`)
- `from_layer` — optional source layer name (e.g., `"security"` or `null`)
- `from_type` — source entity type (e.g., `"SecurityCapability"`)
- `from_prop` — source match property (e.g., `"targets_cwe"`)
- `to_layer` — optional target layer name (e.g., `"security"` or `null`)
- `to_type` — target entity type (e.g., `"WeaknessType"`)
- `to_prop` — target match property (e.g., `"cwe_id"`)

---

## 8. Cross-Layer Join Materialization

After all layers are loaded and ontologies are merged, the stack processes
join rules to produce cross-layer relations.

R-018. For each join rule, the stack MUST:
     1. Collect all entities whose `"instance of"` attribute matches `from_type`
        (source candidates). If `from_layer` is specified, restrict to that
        layer; otherwise search all layers.
     2. Collect all entities whose `"instance of"` attribute matches `to_type`
        (target candidates). If `to_layer` is specified, restrict to that
        layer; otherwise search all layers.
     3. Build an index of target candidates keyed by the value of their
        `to_prop` property.
     4. For each source entity, for each value of its `from_prop` property,
        look up matching targets in the index and emit a materialized
        cross-layer relation.

R-019. Each materialized cross-layer relation MUST be a JSON object with:
     - `dcg:type` (string — `"relation"`)
     - `uid` (string — deterministic UID computed from source ID, target ID,
       and property, following the standard relation UID scheme as defined in
       R-010 [DCG-001])
     - `source` (string — UID of the source entity)
     - `target` (string — UID of the target entity)
     - `property` (string — relation property name from the join rule)
     - `dcg:cross_layer` (boolean — `true`)
     - `dcg:matched_on` (string — the property value that caused the match,
       e.g., `"CWE-20"`)

R-020. Cross-layer relations MUST be read-only. They MUST NOT be stored in any
     layer's graph data files. They are recomputed on every stack load.

R-021. Cross-layer relations MUST be stored separately from intra-layer
     relations in the stack's runtime state.

R-022. A source entity with multiple values for `from_prop` (multi-value
     attributes, as defined in R-019 [DCG-001]) MUST produce one materialized
     cross-layer relation per matching `(from_prop_value, target)` pair.

**Example — materialized cross-layer relation:**

```json
{
  "dcg:type": "relation",
  "uid": "dcg:rel:sha256:...",
  "source": "dcg:sha256:wafsanitizer...",
  "target": "dcg:sha256:inputval...",
  "property": "mitigates",
  "dcg:cross_layer": true,
  "dcg:matched_on": "CWE-20"
}
```

---

## 9. Multi-Parent Entity Resolution

Entity resolution via the extends DAG is the core mechanism by which
composition layers share knowledge.

R-023. `resolve(layer_name, uid)` MUST perform a breadth-first search of the
     extends DAG starting from the named layer, visiting parents in the
     order declared in the `extends` list.

R-024. Resolution MUST return the **first** entity found in the BFS traversal
     (nearest layer in declaration order wins).

R-025. Resolution MUST follow entity redirects as defined in R-077 through
     R-081 [DCG-001] within each layer's store during the traversal.
     The redirect format is as defined in R-052 [DCG-001].

R-026. Resolution MUST terminate and return `None` if the DAG is fully
     traversed without finding the entity.

R-027. Resolution MUST track visited layers and terminate if a cycle is
     encountered at runtime (even if validation was performed at load time).

**Example BFS traversal (extends: [product, compliance]):**

```
resolve("infrastructure", uid):
  1. infrastructure → not found
  2. product (first parent) → not found
  3. compliance (second parent) → not found
  4. security (product's parent, reached via BFS) → FOUND
```

---

## 10. Cross-Layer Queries

R-028. Cross-layer `query()` MUST iterate layers in manifest declaration
     order (not extends DAG order).

R-029. When the same entity UID appears in multiple layers, cross-layer
     queries MUST deduplicate by UID. The entity from the
     **first-declared** layer (earliest in the manifest) MUST be returned.

R-030. Cross-layer queries MUST NOT merge attributes across layers. The
     entity dict from the winning layer is returned as-is.

R-031. Cross-layer queries SHOULD accept an optional layer filter (list of
     layer names) to restrict which layers are searched.

R-032. When relations are returned by a cross-layer query, implementations
     SHOULD include materialized cross-layer relations alongside intra-layer
     relations. Callers SHOULD be able to filter by `dcg:cross_layer: true`
     to distinguish them.

---

## 11. Composition Guarantees

Each composition layer is independently loadable. A stack's join rules
propagate parent-layer improvements automatically on the next load.

R-033. Each native composition layer (those with a `graph_card.json`)
     MUST be independently loadable at the file format level — the
     layer's `graph_card.json` and graph data files MUST parse and
     validate without access to other layers. Strict mode validation
     (R-003) MAY be skipped when loading a layer standalone.
     Adapter-backed layers (see [DCG-001-ADAPT](04-rfc-store-adaptor.md)) are
     exempt from this standalone requirement.

R-034. Writes MUST target a single composition layer at a time.

R-035. `save()` MUST operate on the active composition layer only.
     Other layers MUST NOT be modified.

R-036. Improvements committed to a parent layer (e.g., new entities with the
     same `cwe_id` values) MUST be automatically visible to the stack on
     next load via re-materialization of join rules. No changes to the stack
     manifest or child layer data are required for propagation.

R-037. Stack-level tooling that invokes `purge_retracted()`
     (R-076 [DCG-001](01-rfc.md)) MUST target each layer explicitly and
     independently. An implementation MUST NOT implicitly purge layers
     other than the one targeted.

Cross-layer connections are matched on shared property values (stable
identifiers like `cwe_id`, `owasp_id`) rather than UIDs, so parent layers can be
rebuilt without breaking cross-layer connections. All intra-layer constraints
apply as specified in R-062 [DCG-001].

**Example — 3-layer appsec composition:**

```
security layer:
  WeaknessType: "Improper Input Validation" (cwe_id: CWE-20)
  WeaknessType: "XSS" (cwe_id: CWE-79)

product layer (extends: [security]):
  SecurityCapability: "WAF Input Sanitization"
    — targets_cwe: CWE-20  (intra-layer property attribute)
    — targets_cwe: CWE-79  (intra-layer property attribute)

stack joins:
  - mitigates: SecurityCapability.targets_cwe -> WeaknessType.cwe_id

Materialized by stack:
  "WAF Input Sanitization" -[mitigates]-> "Improper Input Validation"
  "WAF Input Sanitization" -[mitigates]-> "XSS"
```

When the security layer is rebuilt with a new version where CWE entities have
different UIDs, these relations automatically re-materialize to the new UIDs on
the next stack load — no changes to the product layer are required.

---

## 12. Security Considerations

Cross-layer join rules execute at load time. Implementations SHOULD validate
that join rule expressions do not reference layers or types outside the manifest
to prevent information disclosure. Join rule grammar is fully specified in R-016;
implementations MUST enforce it at parse time to prevent injection of malformed
expressions.

---

## 13. References

### Normative

- [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt) — Key words for use in RFCs to Indicate Requirement Levels
- [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174) — Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words
- [DCG-001](01-rfc.md) — Domain Context Graph Core Protocol

---

## Appendix A: Requirement Summary

| R-NNN | Section | Level | Requirement |
|---|---|---|---|
| R-001 | 5. Stack Manifest Format | MUST | Stack defined by YAML manifest file (`dcg-stack.yml`) |
| R-002 | 5. Stack Manifest Format | MUST | Manifest contains `stack` and `layers` fields |
| R-003 | 5. Stack Manifest Format | MAY/MUST | Manifest MAY contain `strict` (defines validation thresholds) and `joins`; MUST NOT contain `ontology` block |
| R-004 | 5. Stack Manifest Format | MUST | Each layer entry contains `name` and `source` |
| R-005 | 5. Stack Manifest Format | MAY | Each layer entry MAY contain `extends` array |
| R-006 | 5. Stack Manifest Format | MUST | Layer names MUST be unique within a manifest |
| R-007 | 5. Stack Manifest Format | MUST | Every `extends` name MUST reference a declared layer |
| R-008 | 5. Stack Manifest Format | MUST | Extends DAG MUST be acyclic; cycles MUST be detected and rejected |
| R-009 | 6. DAG Validation and Load Order | MUST | Reference validation: all extends names MUST resolve |
| R-010 | 6. DAG Validation and Load Order | MUST | Cycle detection: topological sort MUST be performed; cycles MUST fail |
| R-011 | 6. DAG Validation and Load Order | MUST | Load order: layers MUST load in topological order; ontologies merged incrementally |
| R-012 | 6. DAG Validation and Load Order | MUST | Join rule validation: all join rule types/properties MUST exist in merged ontology |
| R-013 | 7. Join Rule Syntax | MUST | Join rule MUST be a single-key YAML mapping |
| R-014 | 7. Join Rule Syntax | MUST | Join expression MUST use `[Layer.]Type.prop -> [Layer.]Type.prop` syntax |
| R-015 | 7. Join Rule Syntax | MUST | `.` MUST NOT appear in type, property, or layer names |
| R-016 | 7. Join Rule Syntax | MUST | Full grammar MUST be enforced; non-matching rules rejected at parse time |
| R-017 | 7. Join Rule Syntax | MUST | Layer-prefixed join rules MUST reference a declared layer |
| R-018 | 8. Cross-Layer Join Materialization | MUST | Join evaluation: collect candidates, build index, emit cross-layer relations |
| R-019 | 8. Cross-Layer Join Materialization | MUST | Materialized relations MUST have specified JSON fields including `dcg:cross_layer: true` |
| R-020 | 8. Cross-Layer Join Materialization | MUST | Cross-layer relations MUST be read-only; MUST NOT be stored in layer data files |
| R-021 | 8. Cross-Layer Join Materialization | MUST | Cross-layer relations MUST be stored separately from intra-layer relations |
| R-022 | 8. Cross-Layer Join Materialization | MUST | Multi-value `from_prop` MUST produce one relation per matching pair |
| R-023 | 9. Multi-Parent Entity Resolution | MUST | `resolve()` MUST perform BFS of extends DAG |
| R-024 | 9. Multi-Parent Entity Resolution | MUST | Resolution MUST return first entity found (nearest layer wins) |
| R-025 | 9. Multi-Parent Entity Resolution | MUST | Resolution MUST follow entity redirects within each layer |
| R-026 | 9. Multi-Parent Entity Resolution | MUST | Resolution MUST return `None` if DAG fully traversed without finding entity |
| R-027 | 9. Multi-Parent Entity Resolution | MUST | Resolution MUST track visited layers to handle runtime cycles |
| R-028 | 10. Cross-Layer Queries | MUST | Cross-layer `query()` MUST iterate in manifest declaration order |
| R-029 | 10. Cross-Layer Queries | MUST | Duplicate UIDs MUST deduplicate; first-declared layer wins |
| R-030 | 10. Cross-Layer Queries | MUST | Queries MUST NOT merge attributes across layers |
| R-031 | 10. Cross-Layer Queries | SHOULD | Queries SHOULD accept optional layer filter |
| R-032 | 10. Cross-Layer Queries | SHOULD | Query results SHOULD include cross-layer relations; callers SHOULD be able to filter |
| R-033 | 11. Composition Guarantees | MUST | Each native layer MUST be independently loadable; adapter layers exempt |
| R-034 | 11. Composition Guarantees | MUST | Writes MUST target a single composition layer |
| R-035 | 11. Composition Guarantees | MUST | `save()` MUST operate on active layer only |
| R-036 | 11. Composition Guarantees | MUST | Parent layer improvements MUST be automatically visible on next stack load |
| R-037 | 11. Composition Guarantees | MUST | Stack-level purge MUST target each layer explicitly and independently |

---

## Appendix B: Change Log

**2026-06-30 — Strict mode definition + stack purge + adapter exemption**

- R-003: Defined strict mode validation thresholds explicitly (previously
  circular reference with DCG-001)
- R-033: Narrowed standalone loadability to native layers; adapter-backed
  layers (DCG-001-ADAPT) are exempt
- R-037: Added stack-level purge orchestration requirement (moved from
  DCG-001 R-076)
