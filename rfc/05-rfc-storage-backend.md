# RFC: Domain Context Graph Storage Backend Extension

**RFC ID:** DCG-001-STORE
**Status:** Draft
**Date:** 2026-07-06
**Extends:** [DCG-001](01-rfc.md)

---

**Table of Contents**

- [Introduction](#1-introduction)
- [Status of This Memo](#2-status-of-this-memo)
- [Terminology](#3-terminology)
- [BCP 14 Boilerplate](#4-bcp-14-boilerplate)
- [Storage Backend Protocol](#5-storage-backend-protocol)
  - [Entity Operations](#51-entity-operations)
  - [Relation Operations](#52-relation-operations)
  - [Bulk Operations](#53-bulk-operations)
  - [Optional Optimization](#54-optional-optimization)
- [Backend Transparency](#6-backend-transparency)
- [In-Memory Backend](#7-in-memory-backend)
- [Graph Store Integration](#8-graph-store-integration)
- [Security Considerations](#9-security-considerations)
- [References](#10-references)
- [Appendix A: Requirement Summary](#appendix-a-requirement-summary)
- [Appendix B: In-Memory Backend Reference](#appendix-b-in-memory-backend-reference)

---

## 1. Introduction

The DCG graph store ([DCG-001](01-rfc.md) §10) defines protocol-level
operations that include validation, ontology resolution, redirect following,
and retraction semantics. Beneath this protocol layer, implementations need
raw storage primitives for entities and relations.

This extension separates raw storage (get, set, delete, iterate, count) from
protocol logic (validation, retraction, redirects). The result is a
**storage backend protocol** — a minimal interface that any storage system
(in-memory dict, SQLite, Neo4j, cloud database) can implement to serve as the
persistence layer for a DCG graph store.

**Scope:** This extension covers the storage backend protocol, its required
operations, backend transparency guarantees, the in-memory backend
specification, and integration points with the graph store.

**Conformance:** A conforming implementation MAY support pluggable storage
backends; if it does, it MUST satisfy all MUST requirements in this extension.

---

## 2. Status of This Memo

This document specifies an optional extension to [DCG-001](01-rfc.md) for
pluggable storage backends behind the graph store. Distribution of this memo
is unlimited.

---

## 3. Terminology

The following terms are defined for use in this extension. All base terms
(Entity, Attribute, Graph, Graph Store, Retraction, etc.) are defined in
[DCG-001](01-rfc.md).

- **Storage Backend**: A component that provides raw entity and relation
  storage primitives. The backend stores and retrieves opaque dicts by UID
  without interpreting their contents.
- **In-Memory Backend**: A storage backend that uses language-native
  dictionary structures for O(1) key-value access. The default backend.
- **Backend Protocol**: The set of operations a storage backend MUST
  implement to serve as the persistence layer for a graph store.
  Implementations SHOULD name this type `StorageBackend`.

---

## 4. BCP 14 Boilerplate

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14
[RFC 2119](https://www.ietf.org/rfc/rfc2119.txt)
[RFC 8174](https://www.rfc-editor.org/rfc/rfc8174) when, and only when, they
appear in all capitals, as shown here.

---

## 5. Storage Backend Protocol

A storage backend provides raw key-value storage for entity dicts and
relation dicts, both keyed by UID strings. The backend is not aware of
DCG semantics — it stores and retrieves opaque dicts.

### 5.1 Entity Operations

R-001. A storage backend MUST implement `get_entity(uid)`. It MUST return
    the entity dict stored at the given UID, or null if no record exists.
    The returned dict includes tombstones (entities with
    `"retracted": true`) — the backend does not filter by retraction state.

R-002. A storage backend MUST implement `set_entity(uid, entity)`. It MUST
    upsert the entity dict at the given UID — inserting if absent,
    replacing if present. The backend MUST store the dict as-is without
    modification.

R-003. A storage backend MUST implement `delete_entity(uid)`. It MUST
    permanently remove the entity record at the given UID. If no record
    exists, the operation MUST be a no-op (no error).

R-004. A storage backend MUST implement `has_entity(uid)`. It MUST return
    true if any record exists for the given UID, including tombstones.
    This operation MUST be efficient — O(1) for dict-based backends.

R-005. A storage backend MUST implement `iter_entities()`. It MUST return
    an iterator yielding (uid, entity_dict) pairs for every entity record,
    including retracted records. Iteration order is implementation-defined.

R-006. A storage backend MUST implement `entity_count()`. It MUST return
    the total number of entity records, including retracted records.

### 5.2 Relation Operations

R-007. A storage backend MUST implement `get_relation(uid)`. Semantics
    are identical to `get_entity(uid)` (R-001) but for relation records.

R-008. A storage backend MUST implement `set_relation(uid, relation)`.
    Semantics are identical to `set_entity(uid, entity)` (R-002) but for
    relation records.

R-009. A storage backend MUST implement `delete_relation(uid)`. Semantics
    are identical to `delete_entity(uid)` (R-003) but for relation records.

R-010. A storage backend MUST implement `has_relation(uid)`. Semantics
    are identical to `has_entity(uid)` (R-004) but for relation records.

R-011. A storage backend MUST implement `iter_relations()`. Semantics
    are identical to `iter_entities()` (R-005) but for relation records.

R-012. A storage backend MUST implement `relation_count()`. Semantics
    are identical to `entity_count()` (R-006) but for relation records.

### 5.3 Bulk Operations

R-013. A storage backend MUST implement
    `bulk_load(entities, relations)`. It MUST atomically replace all
    stored entity and relation records with the provided maps. The maps
    are keyed by UID string with dict values. After `bulk_load()`,
    `iter_entities()` and `iter_relations()` MUST yield exactly the
    provided records.

R-014. A storage backend MUST implement `clear()`. It MUST remove all
    entity and relation records. After `clear()`, `entity_count()` and
    `relation_count()` MUST return 0.

### 5.4 Optional Optimization

R-015. A storage backend MAY implement
    `filter_relations(source?, target?, property?)`. When implemented, it
    MUST return an iterator yielding (uid, relation_dict) pairs for
    relations matching the provided filters (conjunctive AND). If the
    backend does not support server-side filtering, it MUST return null
    to signal that the graph store should fall back to client-side
    filtering via `iter_relations()`.

---

## 6. Backend Transparency

R-016. The backend MUST NOT interpret entity or relation dict contents. It
    stores and retrieves opaque dicts keyed by UID. Retraction state
    (`"retracted": true`) is a dict key like any other — the backend has
    no retraction awareness. All retraction semantics (soft-delete,
    tombstone preservation, purge) are the graph store's responsibility.

---

## 7. In-Memory Backend

R-017. Conforming implementations MUST provide an in-memory backend as the
    default storage backend. When the graph store is constructed without
    an explicit backend, it MUST use the in-memory backend.

R-018. The in-memory backend MUST use dictionary structures keyed by UID
    string with O(1) average-case complexity for `get_entity()`,
    `set_entity()`, `delete_entity()`, `has_entity()`, `get_relation()`,
    `set_relation()`, `delete_relation()`, and `has_relation()`.

R-019. The in-memory backend's `filter_relations()` MUST return null
    (no server-side filtering). The graph store falls back to iterating
    all relations and filtering in the protocol layer.

---

## 8. Graph Store Integration

R-020. The graph store (R-065 [DCG-001](01-rfc.md)) MUST expose
    `entity_count()`, `relation_count()`, `iter_entities()`,
    `iter_relations()`, and `has_entity(uid)` as public methods that
    delegate to the storage backend.

R-021. `get_relations()` (R-065 [DCG-001](01-rfc.md)) MUST check the
    backend's `filter_relations()` first. If it returns null, the graph
    store MUST fall back to iterating all relations via
    `iter_relations()` and filtering in the protocol layer (applying
    retraction exclusion, property alias resolution, and conjunctive
    filters).

R-022. `to_dict()` MUST use `iter_entities()` and `iter_relations()` to
    build the output dict. `load()` MUST use `bulk_load()` to populate
    the backend from deserialized data.

---

## 9. Security Considerations

Backend implementations that connect to external systems (databases, network
stores) MUST validate connection parameters. Implementations SHOULD NOT
follow symlinks or load data from untrusted paths without explicit
configuration. Credentials for remote backends SHOULD be handled via
environment variables or secure configuration, not embedded in stack
manifests or graph card files.

---

## 10. References

### Normative

- [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt) — Key words for use in RFCs to Indicate Requirement Levels
- [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174) — Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words
- [DCG-001](01-rfc.md) — Domain Context Graph Core Protocol

---

## Appendix A: Requirement Summary

| R-NNN | Section | Level | Requirement |
|---|---|---|---|
| R-001 | 5.1 Entity Operations | MUST | `get_entity(uid)` returns entity dict or null; includes tombstones |
| R-002 | 5.1 Entity Operations | MUST | `set_entity(uid, entity)` upserts entity dict without modification |
| R-003 | 5.1 Entity Operations | MUST | `delete_entity(uid)` removes record; no-op if absent |
| R-004 | 5.1 Entity Operations | MUST | `has_entity(uid)` returns bool; includes tombstones; O(1) for dict backends |
| R-005 | 5.1 Entity Operations | MUST | `iter_entities()` yields (uid, dict) for all records including retracted |
| R-006 | 5.1 Entity Operations | MUST | `entity_count()` returns total count including retracted |
| R-007 | 5.2 Relation Operations | MUST | `get_relation(uid)` returns relation dict or null |
| R-008 | 5.2 Relation Operations | MUST | `set_relation(uid, relation)` upserts relation dict |
| R-009 | 5.2 Relation Operations | MUST | `delete_relation(uid)` removes record; no-op if absent |
| R-010 | 5.2 Relation Operations | MUST | `has_relation(uid)` returns bool |
| R-011 | 5.2 Relation Operations | MUST | `iter_relations()` yields (uid, dict) for all records |
| R-012 | 5.2 Relation Operations | MUST | `relation_count()` returns total count |
| R-013 | 5.3 Bulk Operations | MUST | `bulk_load(entities, relations)` atomically replaces all content |
| R-014 | 5.3 Bulk Operations | MUST | `clear()` removes all records |
| R-015 | 5.4 Optional Optimization | MAY | `filter_relations()` returns filtered iterator or null for fallback |
| R-016 | 6 Backend Transparency | MUST | Backend MUST NOT interpret dict contents; retraction-unaware |
| R-017 | 7 In-Memory Backend | MUST | In-memory backend MUST be the default |
| R-018 | 7 In-Memory Backend | MUST | In-memory backend MUST use O(1) dict structures |
| R-019 | 7 In-Memory Backend | MUST | In-memory backend `filter_relations()` MUST return null |
| R-020 | 8 Graph Store Integration | MUST | Graph store MUST expose accessor methods delegating to backend |
| R-021 | 8 Graph Store Integration | MUST | `get_relations()` MUST try `filter_relations()` first, then fall back |
| R-022 | 8 Graph Store Integration | MUST | `to_dict()`/`load()` MUST use backend iteration/bulk_load |

---

## Appendix B: In-Memory Backend Reference

The following pseudocode illustrates a conforming in-memory backend. Language
syntax is illustrative — implementations MAY use language-idiomatic patterns.

```
class InMemoryBackend:
    entities = {}   // UID string → entity dict
    relations = {}  // UID string → relation dict

    get_entity(uid):      return entities.get(uid)
    set_entity(uid, e):   entities[uid] = e
    delete_entity(uid):   entities.remove(uid) if uid in entities
    has_entity(uid):      return uid in entities
    iter_entities():      return entities.entries()
    entity_count():       return entities.size()

    get_relation(uid):    return relations.get(uid)
    set_relation(uid, r): relations[uid] = r
    delete_relation(uid): relations.remove(uid) if uid in relations
    has_relation(uid):    return uid in relations
    iter_relations():     return relations.entries()
    relation_count():     return relations.size()

    bulk_load(e, r):      entities = copy(e); relations = copy(r)
    clear():              entities.clear(); relations.clear()

    filter_relations(source, target, property):
        return null  // signal fallback to graph store
```
