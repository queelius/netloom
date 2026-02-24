# CLAUDE.md

This file provides guidance to Claude Code when working in this repository.

## Project status

**This is a design-phase repository.** There is no implementation code yet. The DSL specification lives in `docs/spec.md` (implementation-ready, v1) and the examples in `docs/examples/`. All design questions have been resolved.

## What netloom is

A declarative YAML-based language for constructing complex networks (graphs) from structured data. The DSL describes:
- **Nodes**: units of the graph, extracted from source documents (potentially multiple types per document)
- **Links**: relationships between nodes (similarity, structural, attribute-based)
- **Network**: graph construction parameters (thresholds, community detection)

The output is a NetworkX `DiGraph` (Python API) or serialized graph file (CLI: GraphML, JSON, GEXF, edgelist).

## Key files

| File | Purpose |
|------|---------|
| `docs/spec.md` | Full DSL specification (implementation-ready, v1) |
| `docs/examples/conversations.yaml` | Heterogeneous graph: conversations + turns |
| `docs/examples/papers.yaml` | Homogeneous graph: academic papers with citations |
| `docs/examples/citations.yaml` | Citation network: reference links, multiple sources, named embeds |
| `pyproject.toml` | Python project config (no deps yet) |
| `README.md` | Project overview and motivation |

## Design principles

1. **DSL is configuration, not a programming language.** When you need if/then/else in YAML, you've outgrown YAML. The DSL covers ~90% of use cases (weighted averages with thresholds). The remaining 10% uses a Python escape hatch (custom provider classes).

2. **No shorthand syntax.** Always `type: text`, never `title: text`. Uniformity over brevity. Configs evolve (you add `default:`, `from:`, etc.) and shorthand makes evolution harder.

3. **Schema is opt-in.** Schema entries are useful when present but never required. Fields can be referenced directly by nodes and links without declaration.

4. **Extraction pipeline reads like English.** `from: turns, where: { role: user }, pluck: text, reduce: join` instead of JSONPath `$.turns[?(@.role=='user')].text`.

5. **Explicit over implicit.** The presence or absence of `chunking` and `aggregate` in an embed block tells you exactly what's happening. No hidden behavior.

6. **Plugins are first-class.** Built-in providers (tfidf, jaccard) and user-written providers use the same protocol and registry. "Internal = just the first set of plugins."

## Canonical test document

This JSON document was used to design the DSL. Use it for testing and examples:

```json
{
  "id": "conv-2024-0142",
  "title": "Debug authentication middleware",
  "created_at": "2024-11-15T09:23:00Z",
  "model": "claude-sonnet-4-20250514",
  "turns": [
    {
      "role": "user",
      "text": "The auth middleware is rejecting valid tokens after the Redis upgrade",
      "timestamp": "2024-11-15T09:23:00Z"
    },
    {
      "role": "assistant",
      "text": "Let me check the Redis connection config and token validation logic.",
      "timestamp": "2024-11-15T09:23:05Z",
      "tool_calls": ["read_file", "grep"]
    },
    {
      "role": "tool",
      "text": "Contents of auth/middleware.py...",
      "timestamp": "2024-11-15T09:23:06Z"
    },
    {
      "role": "assistant",
      "text": "Found it -- the Redis key prefix changed from 'session:' to 'sess:' in v7. The token lookup is using the old prefix.",
      "timestamp": "2024-11-15T09:23:15Z"
    },
    {
      "role": "user",
      "text": "Ah that makes sense, we upgraded Redis last week. Can you fix it?",
      "timestamp": "2024-11-15T09:23:30Z"
    }
  ],
  "tags": ["debugging", "auth", "redis"],
  "project": "backend-api",
  "outcome": "resolved",
  "tools_used": ["read_file", "grep", "edit"]
}
```

## Predecessor project

netloom's DSL evolved from [complex-network-rag](https://github.com/queelius/complex-network-rag), a topology-aware RAG system. Key changes from that project:

- **Renamed**: `embeddings` -> `nodes`, `similarities` -> `links`
- **Cut operators**: `gate`, `boost`, `when` (too much control flow for YAML)
- **Added**: heterogeneous node types, `between:` on links, `parent` method
- **Replaced**: JSONPath extraction with `from/where/pluck/reduce` pipeline
- **Replaced**: `gate` operator with `min:` on combine refs

## Working in this repo

When modifying the design:
- Update `docs/spec.md` as the source of truth
- Keep examples in `docs/examples/` consistent with the spec
- Update this file if design principles change
- The README should reflect the spec but is less detailed

When implementation begins:
- Follow the plugin architecture described in the spec
- Use the registry pattern for all pluggable components
- Keep zero heavy dependencies in the core package
