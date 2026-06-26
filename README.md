# Domain Context Graph (DCG) Protocol

A protocol for structuring and sharing domain knowledge so that AI systems and human developers can operate on verified context rather than statistical inference.

## Why DCG?

LLMs hallucinate domain relationships. Static analysis captures code structure but not business meaning. DCG bridges the gap: a compact, JSON-based knowledge graph that captures both structural facts (this function calls that function) and domain semantics (this function is part of the Payments domain and handles PCI compliance).

DCG graphs are:
- **Local-first** -- plain JSON files in your repo, no database required
- **Composable** -- stack multiple domain projects into a unified knowledge graph
- **AI-native** -- English property keys and compact attributes minimize tokens per entity

## Specification

### RFCs (Protocol Standards)

| Document | ID | Description |
|----------|-----|-------------|
| [Core Protocol](rfc/01-rfc.md) | DCG-001 | Wire format, data model, store protocol, on-disk format (83 requirements) |
| [Composition Extension](rfc/02-rfc-composition.md) | DCG-001-COMP | Stack composition: manifests, DAG, joins, cross-layer queries (36 requirements) |
| [Pack Extension](rfc/03-rfc-packs.md) | DCG-001-PACK | Pack system: declaration, loading, naming, built-in packs (15 requirements) |

### Specs (Implementation Guidance)

| Document | ID | Description |
|----------|-----|-------------|
| [Implementation & Reference Architecture](spec/01-spec-implementation.md) | DCG-001-IMP | Reference architecture and implementation guidance |
| [Population Workflow](spec/02-spec-population-workflow.md) | DCG-001-PW | How to populate and maintain domain graphs |

## Schemas

| Schema | Description |
|--------|-------------|
| [`graph_card.schema.json`](schema/graph_card.schema.json) | Project metadata manifest (`graph_card.json`) |
| [`graph_data.schema.json`](schema/graph_data.schema.json) | Graph data files (`graphs/*.json`) -- entities and relations |
| [`stack.schema.json`](schema/stack.schema.json) | Stack composition manifest (`dcg-stack.yml`) |

## Quick Start

A minimal DCG domain project is a directory with two files:

```
my-domain/
  graph_card.json          # metadata + ontology + graphs index
  graphs/
    default.json           # entity and relation data
```

See [DCG-001 Core Protocol](rfc/01-rfc.md) for the full entity model, property system, and composition rules.

## Reference Implementation

[domain-context-graph](https://github.com/agentkg/domain-context-graph) (`dcg` Python package) -- CLI, MCP server, and in-memory graph store.

## License

Apache-2.0
