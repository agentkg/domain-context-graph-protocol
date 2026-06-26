# RFC: Domain Context Graph Pack Extension

**RFC ID:** DCG-001-PACK
**Status:** Draft
**Date:** 2026-06-26
**Extends:** [DCG-001](01-rfc.md)

---

## 1. Introduction

This document is an optional extension to the Domain Context Graph protocol
([DCG-001](01-rfc.md)). It specifies the pack system: a general extensibility
mechanism for distributing and loading reusable vocabulary bundles into DCG
domain projects.

Packs are distinct from `ontology_builtin` (defined in R-047 through R-048
[DCG-001]). The built-in ontology is always loaded and is not an opt-in
mechanism. Packs are explicitly declared by domain projects and loaded on
demand. The first pack type defined by this extension is `ontology`.

**Scope:** This extension covers the pack concept, pack declaration format,
pack loading semantics, ontology pack type, built-in packs, pack naming and
resolution, and pack interaction with stack composition.

**Conformance:** A conforming implementation MAY support the pack extension;
if it does, it MUST satisfy all MUST requirements in this extension.

---

## 2. Status of This Memo

This document specifies an optional extension to [DCG-001](01-rfc.md) for
declaring and loading reusable vocabulary packs. Distribution of this memo is
unlimited.

---

## 3. Terminology

The following terms are defined for use in this extension. All base terms
(Entity, Attribute, Domain, Graph Card, Ontology, `ontology_builtin`, etc.)
are defined in [DCG-001](01-rfc.md).

- **Pack**: A named, reusable bundle of protocol content that domain projects
  can opt into via their `graph_card.json`.
- **Pack Type**: A category that determines the format and semantics of a
  pack's content. The first pack type defined by this extension is `ontology`.
- **Ontology Pack**: A pack of type `ontology` — a named bundle of types,
  properties, and aliases in the same format as the `graph_card.json`
  `ontology` key.
- **Built-in Pack**: A pack shipped as part of an implementation, available
  without user configuration. Distinct from `ontology_builtin`, which is
  automatically loaded and not declared in `graph_card.json`.

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
- [Pack Concept](#5-pack-concept)
- [Pack Declaration Format](#6-pack-declaration-format)
- [Pack Loading Semantics](#7-pack-loading-semantics)
- [Ontology Pack Type](#8-ontology-pack-type)
- [Built-in Packs](#9-built-in-packs)
- [Pack Naming and Resolution](#10-pack-naming-and-resolution)
- [Stack Composition with Packs](#11-stack-composition-with-packs)
- [Security Considerations](#12-security-considerations)
- [References](#13-references)
  - [Normative](#normative)
- [Appendix A: Requirement Summary](#appendix-a-requirement-summary)

---

## 5. Pack Concept

R-001. A Pack is a named, reusable bundle of protocol content that domain
    projects can opt into via their `graph_card.json`. Packs MUST be
    identified by a string name.

R-002. The pack system MUST be extensible by pack type. The pack type
    determines the format and semantics of the pack's content. The first
    pack type defined by this extension is `ontology`.

---

## 6. Pack Declaration Format

R-003. Packs MUST be declared per domain project in `graph_card.json` via the
    `packs` key. The `packs` key MUST be a JSON object keyed by pack type,
    where each value is an array of pack names:

    ```json
    { "packs": { "ontology": ["code", "security"] } }
    ```

R-004. There MUST NOT be a stack-level `packs` key. Each domain project
    declares its own packs independently.

R-005. The `packs` key is OPTIONAL in `graph_card.json`. When absent, no
    packs are loaded beyond `ontology_builtin` (as defined in R-047 through
    R-048 [DCG-001]).

---

## 7. Pack Loading Semantics

R-006. Packs MUST be loaded before the per-project custom ontology, so that
    custom declarations can reference or override pack-provided content. Load
    order within a domain project:

    1. `ontology_builtin` (auto-loaded; see R-047 through R-048 [DCG-001])
    2. Declared packs (in array order)
    3. Custom `ontology` section (as defined in R-049 through R-050 [DCG-001])

R-007. Each domain project's pack-derived content MUST be fully self-contained
    between its packs and its custom declarations. A domain project MUST NOT
    depend on another domain project's pack declarations. Pack-loaded
    declarations follow the same persistence rules as custom ontology
    declarations (see R-051 [DCG-001]).

R-008. When a pack name in the `packs` array cannot be resolved, implementations
    MUST raise an error listing the unknown pack name and available packs.

---

## 8. Ontology Pack Type

R-009. An ontology pack MUST use the same format as the `graph_card.json`
    `ontology` key, as defined in R-049 through R-050 [DCG-001]:

    - `types` (array of `{"name", "description"}`)
    - `properties` (array of `{"name", "datatype", "description"}`)
    - `aliases` (array of `{"alias", "canonical"}`)

R-010. Ontology packs MUST be declared under the `"ontology"` key within the
    `packs` object:

    ```json
    { "packs": { "ontology": ["code", "security"] } }
    ```

---

## 9. Built-in Packs

R-011. Pack names are implementation-defined. Implementations SHOULD ship
    RECOMMENDED built-in packs for common domains.

R-012. The following built-in packs are RECOMMENDED:

    - `"code"` — source code analysis types and relations
    - `"security"` — security domain types and relations
    - `"dcg-development"` — development tooling vocabulary

    Implementations MAY ship additional packs or allow user-defined packs.

---

## 10. Pack Naming and Resolution

R-013. Pack names MUST be non-empty strings matching `[a-z][a-z0-9-]*`. Names
    are case-sensitive. Implementations MUST resolve pack names to pack content
    using an implementation-defined lookup mechanism.

---

## 11. Stack Composition with Packs

R-014. When a stack is loaded, each layer's packs MUST be loaded independently
    as part of that layer's ontology setup, before the stack-level ontology
    merge (see [DCG-001-COMP](02-rfc-composition.md)).

R-015. When multiple layers declare the same pack, the pack content MUST be
    loaded once per layer. Pack content from different layers follows the same
    merge semantics as custom ontology declarations (as defined in
    [DCG-001-COMP](02-rfc-composition.md)).

---

## 12. Security Considerations

Pack content is loaded from implementation-defined sources. Implementations
SHOULD validate pack content against the expected format before merging into
the ontology. Implementations SHOULD reject packs that contain malformed or
excessively large content. User-defined packs allow arbitrary vocabulary
extension and SHOULD be treated with the same trust level as project
configuration.

---

## 13. References

### Normative

- [RFC 2119] Bradner, S., "Key words for use in RFCs to Indicate Requirement
  Levels", BCP 14, RFC 2119, March 1997.
  <https://www.ietf.org/rfc/rfc2119.txt>
- [RFC 8174] Leiba, B., "Ambiguity of Uppercase vs Lowercase in RFC 2119 Key
  Words", BCP 14, RFC 8174, May 2017.
  <https://www.rfc-editor.org/rfc/rfc8174>
- [DCG-001] Domain Context Graph Core Protocol.
  <[01-rfc.md](01-rfc.md)>

---

## Appendix A: Requirement Summary

| Req | Section | Level | Summary |
|-----|---------|-------|---------|
| R-001 | 5. Pack Concept | MUST | A Pack is a named, reusable bundle identified by a string name |
| R-002 | 5. Pack Concept | MUST | Pack system MUST be extensible by pack type; first type is `ontology` |
| R-003 | 6. Pack Declaration Format | MUST | Packs declared in `graph_card.json` via `packs` key (object keyed by pack type) |
| R-004 | 6. Pack Declaration Format | MUST NOT | No stack-level `packs` key; each project declares independently |
| R-005 | 6. Pack Declaration Format | OPTIONAL | `packs` key is optional; absent means only `ontology_builtin` is loaded |
| R-006 | 7. Pack Loading Semantics | MUST | Packs loaded before custom ontology; order: builtin → packs → custom |
| R-007 | 7. Pack Loading Semantics | MUST | Domain project pack content self-contained; no cross-project pack dependency |
| R-008 | 7. Pack Loading Semantics | MUST | Unknown pack name MUST raise error listing the name and available packs |
| R-009 | 8. Ontology Pack Type | MUST | Ontology pack uses same format as `graph_card.json` `ontology` key |
| R-010 | 8. Ontology Pack Type | MUST | Ontology packs declared under `"ontology"` key in `packs` object |
| R-011 | 9. Built-in Packs | SHOULD | Implementations SHOULD ship RECOMMENDED built-in packs |
| R-012 | 9. Built-in Packs | RECOMMENDED | Built-in packs: `code`, `security`, `dcg-development` |
| R-013 | 10. Pack Naming and Resolution | MUST | Pack names match `[a-z][a-z0-9-]*`; case-sensitive; impl-defined resolution |
| R-014 | 11. Stack Composition with Packs | MUST | Each layer's packs loaded independently before stack-level ontology merge |
| R-015 | 11. Stack Composition with Packs | MUST | Same pack declared in multiple layers loaded once per layer; same merge semantics |
