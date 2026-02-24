# netloom DSL Specification

**Status**: Implementation-ready (v1)

## 1. Overview

netloom is a declarative YAML-based language for constructing weighted directed graphs from structured data. A netloom config describes node types, their embeddings, and how nodes relate to each other. The system ingests source documents, extracts nodes and their attributes, computes similarity scores and structural relationships, and outputs a NetworkX `DiGraph`.

### Top-level sections

```yaml
defaults:     # Shared configuration (optional)
source:       # Where and how to load documents
schema:       # Data shape and extraction (optional)
nodes:        # Node types and embedding config
links:        # Relationship definitions between nodes
network:      # Graph construction parameters (optional)
```

All sections are optional, but a useful config needs at least `nodes` and `links`.

### Output

The primary interface is the Python API:

```python
import netloom

G = netloom.build("config.yaml")   # returns nx.DiGraph
```

The CLI serializes the graph to disk:

```bash
netloom build config.yaml                            # -> output.graphml (default)
netloom build config.yaml -o graph.json --format json
netloom build config.yaml --format gexf
netloom build config.yaml --format edgelist
```

Supported output formats: GraphML (default), JSON, GEXF, edgelist.

### Graph semantics

The graph is always a `DiGraph`. Symmetric methods (cosine, jaccard, dice, overlap, exact, numeric) produce **two** directed edges per pair (A->B and B->A) with identical weights. Asymmetric methods (parent, reference) produce **one** directed edge. Call `G.to_undirected()` for algorithms that require undirected graphs.

**Node attributes**: All declared `fields` become NetworkX node attributes. Every node also carries:

- `_type` -- the node type name from the config
- `_meta` -- provenance metadata from source ingestion

**Edge attributes**: Every edge carries:

- `weight` -- similarity score or structural weight (float)
- `link_type` -- the link name from the config
- `method` -- the comparison method used
- `source_type` -- the source node's type name
- `target_type` -- the target node's type name

### Config validation

Configs are validated using Pydantic models. YAML is parsed into typed models. Validation errors report the YAML path of the invalid value.

---

## 2. Defaults

The `defaults` block provides shared configuration that node types inherit. Currently supports embed defaults only.

```yaml
defaults:
  embed:
    model: tfidf
```

Node types inherit defaults unless they override:

```yaml
defaults:
  embed:
    model: tfidf

nodes:
  paper:
    from: .
    fields:
      abstract: { pluck: abstract }
    embed:
      field: abstract               # inherits model: tfidf from defaults

  review:
    from: .
    fields:
      text: { pluck: review_text }
    embed:
      field: text
      model: sentence-transformers  # overrides default
```

---

## 3. Source

The source section declares what to ingest and how to interpret it. Since nodes are decoupled from documents, the ingestion layer just needs to produce structured records that the extraction pipeline can work on.

### Configuration

Single source:

```yaml
source:
  path: data/conversations/       # file or directory
  format: jsonl                    # jsonl | json | yaml | markdown | text
```

Multiple sources:

```yaml
source:
  - path: data/conversations.jsonl
    format: jsonl
  - path: data/papers/
    format: json
```

When `format` is omitted, it is inferred from file extensions (`.jsonl`, `.json`, `.yaml`/`.yml`, `.md`, `.txt`). Unrecognized extensions are treated as plain text. A directory path ingests all matching files recursively.

### Source formats

| Format | Description | Record boundary |
|--------|-------------|-----------------|
| JSONL | One JSON object per line | Line |
| JSON array | Top-level array of objects | Array element |
| JSON files | Directory of `.json` files | File |
| YAML | Single document or multi-document (`---` separated) | Document |
| Markdown + frontmatter | YAML frontmatter + body | File |
| Markdown | Plain markdown, no frontmatter | File |
| Text | Plain text (`.txt`, `.log`, etc.) | File |

### Markdown handling

Markdown parsing is intentionally shallow: headings, sections, and body. Deeper structure (code blocks, links, lists, block quotes) is preprocessing.

**Markdown + frontmatter** (has `---` delimiters at top):

```markdown
---
title: Debug authentication middleware
tags: [debugging, auth, redis]
date: 2024-11-15
---

The auth middleware is rejecting valid tokens after the Redis upgrade...
```

Becomes:

```json
{
  "title": "Debug authentication middleware",
  "tags": ["debugging", "auth", "redis"],
  "date": "2024-11-15",
  "body": "The auth middleware is rejecting valid tokens after the Redis upgrade..."
}
```

The frontmatter fields are promoted to top-level keys. The markdown body becomes the `body` field.

**Plain markdown** (no frontmatter):

```markdown
# Debug authentication middleware

## Problem

The auth middleware is rejecting valid tokens after the Redis upgrade.

## Solution

The Redis key prefix changed from 'session:' to 'sess:' in v7.
```

Becomes:

```json
{
  "title": "Debug authentication middleware",
  "body": "## Problem\n\nThe auth middleware is rejecting...",
  "sections": [
    { "heading": "Problem", "level": 2, "body": "The auth middleware is rejecting valid tokens after the Redis upgrade." },
    { "heading": "Solution", "level": 2, "body": "The Redis key prefix changed from 'session:' to 'sess:' in v7." }
  ]
}
```

The parser extracts:
- **`title`**: first `#` heading (null if none)
- **`body`**: full content below the title heading
- **`sections`**: list of `{heading, level, body}` for each `##`+ heading

### Plain text handling

Plain text files (`.txt`, `.log`, or unrecognized extensions) produce a minimal record:

```json
{
  "body": "The full contents of the file...",
  "_meta": { ... }
}
```

The entire file content becomes `body`. The filename (without extension) is available via `_meta.source_path`. This is the simplest case -- a directory of `.txt` files can be turned into a network with just:

```yaml
source:
  path: corpus/
  format: text

nodes:
  document:
    from: .
    fields:
      text: { pluck: body }
    embed:
      field: text
      model: tfidf

links:
  similar:
    between: [document, document]
    method: cosine
    min: 0.3
```

Since `sections` is a list of objects, it works with the extraction pipeline just like JSON arrays:

```yaml
nodes:
  document:
    from: .
    fields:
      title: { pluck: title }

  section:
    from: sections
    fields:
      heading: { pluck: heading }
      text: { pluck: body }
    embed:
      field: text
      model: tfidf
```

This creates a heterogeneous graph from a directory of plain markdown files -- document nodes and section nodes -- without any frontmatter at all.

### Source metadata (provenance)

Every ingested record automatically gets a `_meta` object with full provenance. This is always available for extraction, regardless of format:

```json
{
  "_meta": {
    "source_path": "data/conversations/debug-auth.json",
    "source_line": null,
    "source_index": 0,
    "file_modified": "2024-11-15T09:30:00Z",
    "file_size": 1842,
    "file_hash": "sha256:a3f2b8c1d4e5...",
    "ingested_at": "2026-02-24T14:00:00Z"
  }
}
```

| Field | Description |
|-------|-------------|
| `source_path` | File path relative to source root |
| `source_line` | Line number for JSONL records (null for file-per-document) |
| `source_index` | Record index within file (0 for file-per-document) |
| `file_modified` | Last-modified timestamp of the source file |
| `file_size` | Source file size in bytes |
| `file_hash` | Content hash (SHA-256) of the source file |
| `ingested_at` | When this record was ingested |

Access provenance fields via the extraction pipeline:

```yaml
fields:
  filename: { from: _meta, pluck: source_path }
  modified: { from: _meta, pluck: file_modified }
```

This provides full data provenance -- you can always trace a node back to exactly which file, line, and time produced it.

### Preprocessing is out of scope

netloom ingests structured records. If your source data needs regex extraction, JSONPath queries, or other transformations to become structured, do that upstream and feed netloom clean JSONL. A 10-line Python script or `jq` pipeline is the right tool for ETL -- netloom is the declarative glue layer, not a data wrangling tool.

---

## 4. Schema

The schema section declares fields extracted from source documents. Entries are **optional** -- fields can be referenced directly by `nodes` and `links` without a schema entry. Schema is useful when you need to:

- Extract and reshape data (the extraction pipeline)
- Set default values
- Validate incoming documents
- Self-document the expected data shape

The more you declare, the more the system can check. But it never forces boilerplate for simple pass-through fields.

### Field declaration

Always use the long form. No shorthand syntax.

```yaml
schema:
  tags:
    type: list
    default: []

  title:
    type: text

  user_text:
    from: turns
    where: { role: user }
    pluck: text
    reduce: join
```

### Type system

Every field has a type, either declared or inferred:

| Type | Description |
|------|-------------|
| `text` | String value |
| `number` | Numeric value (int or float) |
| `list` | List of values |
| `bool` | Boolean |
| `auto` | Infer from extraction pipeline output or source data |

Three modes for every field:
1. **No schema entry** -- field referenced directly by nodes/links, system figures it out
2. **`type: auto`** -- "I want to declare this field, maybe set a default, but let the system infer the type"
3. **Explicit type** -- assertion and documentation; validation error if data doesn't match

### Missing field behavior

**Strict by default.** If a field referenced in the extraction pipeline or node `fields` block does not exist in the source document and has no `default:` value, it is a **build error**. This catches typos and data shape mismatches early.

### Extraction pipeline: `from -> where -> pluck -> reduce`

The pipeline extracts and reshapes data from source documents. Each step is optional; steps execute in order.

```yaml
user_text:
  from: turns           # navigate into the structure
  where: { role: user } # filter list items
  pluck: text           # grab one field from each item
  reduce: join          # collapse list to scalar
```

This reads like English: "From turns, where role is user, pluck text, join them."

#### `from`

Dot-path into the source document structure.

```yaml
from: turns              # top-level field
from: metadata.authors   # nested path
from: .                  # whole document (used in nodes section)
```

If the schema field name matches the source field name, `from` is not needed.

#### `where`

Simple key-value exact match filter. Only applies when `from` yields a list.

```yaml
where: { role: user }                    # single condition
where: { role: user, channel: support }  # multiple conditions (AND)
```

- AND semantics for multiple keys
- No OR, negation, or comparison operators
- Complex filtering should happen in preprocessing or via a Python plugin

#### `pluck`

Extract a single field from each item in the list.

```yaml
pluck: text        # grab the "text" field from each item
```

- Without `pluck`, you get whole objects
- Single field only (no multi-field plucking)

#### `reduce`

Collapse a list to a scalar value.

| Reduce | Input | Output |
|--------|-------|--------|
| `join` | list of text | text (space-separated) |
| `join_unique` | list of text | text (deduplicated, then space-separated) |
| `first` | list | first element |
| `last` | list | last element |
| `count` | list | number (length) |
| `count_unique` | list | number (count of unique elements) |
| `unique` | list | list (deduplicated) |
| `sum` | list of numbers | number |
| `mean` | list of numbers | number |
| `min` | list of numbers | number |
| `max` | list of numbers | number |

Without `reduce`, the pipeline output is a list.

**Configurable join separator**: `join` and `join_unique` use space as the default separator. Use the object form for a custom separator:

```yaml
reduce: join                   # space-separated (default)
reduce: { join: "\n\n" }      # paragraph-separated
reduce: { join_unique: ", " } # comma-separated, deduplicated
```

**No reduce chaining.** Use named composites (`count_unique`, `join_unique`) instead of chaining like `[unique, count]`. Additional composites may be added in v2.

#### Type inference from pipeline

The pipeline output determines the field type when `type: auto` or omitted:

| Pipeline | Inferred type |
|----------|--------------|
| `pluck: text` (no reduce) | list |
| `pluck: text` + `reduce: join` | text |
| `pluck: text` + `reduce: count` | number |
| `pluck: text` + `reduce: first` | text |

### Default values

```yaml
tags:
  type: list
  default: []

outcome:
  type: text
  default: "unknown"
```

---

## 5. Nodes

The nodes section defines **node types** in the graph. Each node type specifies how to extract items from source documents, what fields each node carries, and how to compute embedding vectors.

A single source document can produce multiple nodes of different types. This is the key to heterogeneous graphs.

### Structure

```yaml
nodes:
  <node_type_name>:
    from: <dot-path>          # where to find items (. = whole document)
    where: { key: value }     # filter (optional, for lists)
    fields:                   # data carried by each node
      <name>: { pluck: <field>, default: <value> }
    embed:                    # vector configuration
      field: <field_name>
      model: <provider_name>
      chunking: { ... }      # optional
      aggregate: <method>     # optional
```

### `from` and `where`

- **`from: .`** -- the whole document produces one node of this type
- **`from: turns`** -- each item in the `turns` list produces its own node
- **`from: turns` + `where: { role: user }`** -- only matching items become nodes

```yaml
nodes:
  conversation:
    from: .                    # one node per document
    fields:
      title: { pluck: title }

  user_turn:
    from: turns                # one node per matching turn
    where: { role: user }
    fields:
      text: { pluck: text }
```

### Fields

Fields declare what data each node carries. The `pluck` key extracts a value from the source item.

```yaml
fields:
  title: { pluck: title }
  tags: { pluck: tags }
  tools: { pluck: tool_calls, default: [] }
```

- `pluck:` grabs a field from the source item
- `default:` provides a fallback when the field is missing
- Fields can also use the full extraction pipeline (`from`, `where`, `pluck`, `reduce`)

### Embedding configuration

The `embed` block configures vectorization for similarity computation.

#### Single unnamed embed

When `embed` contains a `field` key (or `combine` key), it is a single unnamed embed:

```yaml
embed:
  field: title           # which node field to embed
  model: tfidf           # embedding provider (resolved via registry)
```

#### Multiple named embeds

When `embed` keys are not recognized embed properties, each key is a named embed block:

```yaml
embed:
  title_vec:
    field: title
    model: tfidf
  content_vec:
    field: abstract
    model: sentence-transformers
```

Auto-detection rule: if `embed` has any of `field`, `model`, `chunking`, `aggregate`, `combine` at the top level, it is a single unnamed embed. Otherwise, keys are named embed block names.

Named embed blocks are useful when:
- A node type needs multiple embeddings for different fields
- Links need to select which embedding to compare (see `embed` parameter on links)
- Combine blocks need to reference specific embeddings by name

#### Chunking

For long text fields, chunking splits the text into pieces before embedding:

```yaml
embed:
  field: text
  model: tfidf
  chunking:
    method: sentences     # sentences | paragraphs | fixed_tokens
    max_tokens: 256       # maximum tokens per chunk
    overlap: 50           # token overlap between chunks
  aggregate: mean         # how to combine chunk vectors
```

**Chunking on list fields**: When `chunking` is specified on a field that resolves to a list, each list item is chunked individually, then all resulting vectors from all items are aggregated together.

#### Aggregate

When embedding produces multiple vectors (from chunking a text field, or from embedding a list field), `aggregate` combines them into one:

| Aggregate | Description |
|-----------|-------------|
| `mean` | Average of all vectors (default) |
| `max` | Element-wise maximum |
| `first` | First vector only |
| `last` | Last vector only |
| `none` | Keep vectors separate (see `match` parameter on links) |

**`aggregate: none`** keeps per-chunk or per-item vectors separate rather than collapsing them. When nodes have separate vectors, the link must specify a `match` strategy (see Links section).

Three embedding patterns:
1. **Simple**: `field + model` -- one string produces one vector
2. **Chunked**: `field + model + chunking + aggregate` -- long text is split, embedded, combined
3. **List**: `field + model + aggregate` -- list of strings, each embedded, combined

The presence or absence of `chunking` and `aggregate` tells you which pattern is in use. No hidden behavior.

#### Combine (embedding-level)

A named embed block (or single unnamed embed) can blend vectors from other named embeds using weighted average:

```yaml
nodes:
  conversation:
    from: .
    fields:
      title: { pluck: title }
      user_text:
        from: turns
        where: { role: user }
        pluck: text
        reduce: join
    embed:
      title_vec:
        field: title
        model: tfidf
      user_vec:
        field: user_text
        model: tfidf
      blended:
        combine:
          - ref: title_vec
            weight: 0.3
          - ref: user_vec
            weight: 0.7
```

Combine refs point to named embed blocks by name. Names should be unique across the config when used in combine refs.

When a ref targets an embed from a different node type that produces multiple vectors per document (e.g., one vector per list item), add `aggregate:` on the ref to collapse them:

```yaml
blended:
  combine:
    - ref: title_vec
      weight: 0.3
    - ref: turn_text_vec
      weight: 0.7
      aggregate: mean      # collapse multiple turn vectors to one
```

Combine is scoped to the same source document -- it blends vectors extracted from the same input record.

---

## 6. Links

The links section defines **relationship types** between nodes. Each link specifies which node types it connects, how similarity is computed, and optional thresholds.

All link computations are **cross-document** (full corpus). All nodes of the specified types are compared pairwise, regardless of which source document they came from.

### Structure

```yaml
links:
  <link_name>:
    between: [<type_a>, <type_b>]   # which node types
    method: <comparison_method>      # how to compare
    field: <field_name>              # for attribute methods
    min: <threshold>                 # minimum score to create edge
```

### `between`

Explicit about what's connected. Always a two-element list.

```yaml
between: [user_turn, user_turn]         # within same type
between: [user_turn, assistant_turn]    # across types
between: [conversation, user_turn]      # parent-child
```

### Methods

| Method | Description | Requires | Edges |
|--------|-------------|----------|-------|
| `cosine` | Cosine similarity on embedding vectors | Both types have `embed` | Bidirectional |
| `jaccard` | Jaccard set similarity | `field:` (list field) | Bidirectional |
| `dice` | Dice coefficient | `field:` (list field) | Bidirectional |
| `overlap` | Overlap coefficient | `field:` (list field) | Bidirectional |
| `exact` | Boolean equality (1.0 or 0.0) | `field:` | Bidirectional |
| `numeric` | Gaussian kernel similarity | `field:` + `scale:` | Bidirectional |
| `parent` | Structural containment | Implicit from `from:` paths | Unidirectional (parent->child) |
| `reference` | Foreign-key lookup | `field:` + `target_field:` | Unidirectional (source->target) |

```yaml
links:
  text_similarity:
    between: [user_turn, user_turn]
    method: cosine
    min: 0.3

  tag_overlap:
    between: [conversation, conversation]
    method: jaccard
    field: tags

  same_project:
    between: [conversation, conversation]
    method: exact
    field: project

  contains:
    between: [conversation, user_turn]
    method: parent
```

### Numeric similarity

Uses a Gaussian kernel: `exp(-d^2 / 2*sigma^2)` where `d` is the absolute difference and `sigma` is the `scale` parameter.

```yaml
links:
  year_proximity:
    between: [paper, paper]
    method: numeric
    field: year
    scale: 5              # sigma -- required for numeric method
```

`scale` controls how quickly similarity decays with distance. A scale of 5 means papers 5 years apart have similarity ~0.61, papers 10 years apart have ~0.14.

### Reference links

Creates directed edges based on foreign-key relationships. The source node has a list field containing identifiers; the target node has a scalar field that those identifiers match against.

```yaml
links:
  cites:
    between: [paper, paper]
    method: reference
    field: references          # list field on source node (A)
    target_field: paper_id     # scalar field on target node (B)
    # Creates edge A->B when B.paper_id is in A.references
```

- `field:` -- list field on the first element of `between:` (source side)
- `target_field:` -- scalar field on the second element of `between:` (target side)
- `weight_field:` -- optional field for variable edge weights (default: 1.0)

```yaml
links:
  cites:
    between: [paper, paper]
    method: reference
    field: references
    target_field: paper_id
    weight_field: citation_strength   # if present on source, use as weight
```

### Parent links

Creates directed edges from parent to child based on structural containment. A node extracted from `from: .` (document-level) is the parent of nodes extracted from `from: turns` (list items within the same document).

```yaml
links:
  contains_turns:
    between: [conversation, user_turn]
    method: parent
    weight_field: importance    # optional, default 1.0
```

### `min` threshold

Edges with scores below `min` are not created.

```yaml
links:
  text_similarity:
    between: [user_turn, user_turn]
    method: cosine
    min: 0.3    # ignore similarities below 0.3
```

### `embed` parameter

For `cosine` links, the `embed` parameter selects which named embed block to use for comparison. This enables cross-type embedding comparisons.

```yaml
links:
  # Both sides use same named embed
  title_sim:
    between: [paper, paper]
    method: cosine
    embed: title_vec

  # Each side uses a different named embed
  cross_sim:
    between: [paper, review]
    method: cosine
    embed: [abstract_vec, text_vec]   # positional: paper uses abstract_vec, review uses text_vec

  # Omitted: uses the single unnamed embed from each node type
  basic_sim:
    between: [document, document]
    method: cosine
```

Rules:
- **String**: same named embed on both sides
- **List**: positional match to `between:` -- first element for first type, second for second type
- **Omitted**: uses the single unnamed embed from each node type

### `match` parameter

When at least one side of a link has `aggregate: none` (separate vectors per node), the `match` parameter controls how pairwise scores are combined.

```yaml
links:
  similar:
    between: [conversation, conversation]
    method: cosine
    match: max       # maximum pairwise similarity
    min: 0.3
```

In v1, only `max` is supported: the edge weight is the maximum similarity across all vector pairs between the two nodes.

### Combine (link-level)

Weighted combination of multiple link scores:

```yaml
links:
  overall:
    between: [conversation, conversation]
    combine:
      - ref: intent_sim
        weight: 0.5
        min: 0.2       # treat this component as 0 if below threshold
      - ref: title_sim
        weight: 0.15
      - ref: tag_sim
        weight: 0.15
      - ref: tool_sim
        weight: 0.1
      - ref: project_sim
        weight: 0.1
```

Combine computes a **weighted average** of the referenced link scores. The `min` on a ref means: "if this component's score is below the threshold, treat it as 0 in the weighted average."

In v1, weighted average is the only combine mode. Exotic modes (product, max, min) can be implemented via a custom `MetricProvider` plugin.

### What's NOT in the DSL

The following operators from the predecessor project (complex-network-rag) are deliberately excluded:

- **`gate`** -- replaced by `min:` on refs
- **`boost`** -- exotic composition; use a Python plugin
- **`when`** -- conditional branching; use a Python plugin

The DSL makes the common case trivial (weighted averages with thresholds cover ~90% of use cases). Exotic composition uses the Python escape hatch: write a class implementing the `MetricProvider` protocol, register it, reference it by name in the DSL.

---

## 7. Network

The network section controls graph construction parameters.

```yaml
network:
  min: 0.3                    # global minimum edge weight
  communities:
    algorithm: louvain         # community detection algorithm
```

### Global edge threshold

| Parameter | Description |
|-----------|-------------|
| `min` | Minimum edge weight to include in the graph. Applied after all link computations. |

### Community detection

```yaml
communities:
  algorithm: louvain            # louvain | label_propagation
```

---

## 8. Plugin Architecture

### Registry pattern

The DSL references components by name (`model: tfidf`, `method: jaccard`). The registry resolves names to provider classes at runtime. Built-in providers and user-written providers are structurally identical.

### Provider protocols

Each component type has a protocol:

```python
class EmbeddingProvider(Protocol):
    def embed(self, text: str) -> list[float]: ...

class MetricProvider(Protocol):
    def compute(self, a: Any, b: Any) -> float: ...

class ChunkingProvider(Protocol):
    def chunk(self, text: str, max_tokens: int, overlap: int) -> list[str]: ...
```

### Built-in providers

```
netloom/
  embeddings/
    tfidf.py                    # TF-IDF (no heavy deps)
    ollama.py                   # local Ollama models
    openai.py                   # OpenAI API
    sentence_transformers.py    # HuggingFace models
  metrics/
    cosine.py
    jaccard.py
    dice.py
    overlap.py
    exact.py
    numeric.py
  chunking/
    sentences.py
    paragraphs.py
    fixed_tokens.py
```

### Distribution

- **Optional deps via extras**: `pip install netloom[openai]`
- **Third-party plugins**: `pip install netloom-cohere` adds `model: cohere`
- **Local plugins**: drop a `.py` file in a configured plugins directory
- **Core package**: zero heavy dependencies

### Custom provider example

```python
from netloom import register_embedding

@register_embedding("my-model")
class MyEmbedding:
    def embed(self, text: str) -> list[float]:
        return call_my_api(text)
```

---

## 9. Node Identity

Node IDs are **auto-generated** and **deterministic**. The ID is derived from source provenance:

```
{source_path}:{node_type}:{index}
```

For example:
- `data/conversations/debug-auth.json:conversation:0` -- the conversation node from this document
- `data/conversations/debug-auth.json:user_turn:0` -- the first user turn
- `data/conversations/debug-auth.json:user_turn:1` -- the second user turn

Users never specify IDs in the config. If the source data has an `id` field (e.g., `"id": "conv-2024-0142"`), it can be attached as a regular node field for display:

```yaml
fields:
  id: { pluck: id }              # stored as node attribute, not used as graph ID
  title: { pluck: title }
```

Deterministic IDs mean the same source data always produces the same graph. This enables diffing graphs across config changes.

---

## 10. Deferred Features (v2)

The following features are explicitly deferred to v2. They are **not** part of the v1 specification.

### Density control

```yaml
network:
  max_per_node: 20              # maximum edges per node
  max_total: 10000              # maximum total edges
```

### Weight transforms

```yaml
network:
  weight_transform: log         # log | sqrt | none
```

### Additional match strategies

Beyond `max` for `aggregate: none` links. Candidates: `mean` (average pairwise similarity), `direct` (1-to-1 alignment).

### Embedding cache

Cache computed embeddings to avoid redundant recomputation. In v1, every build is a full rebuild.

### Reduce chaining

Compose reduces like `reduce: [unique, count]`. In v1, use named composites (`count_unique`) instead.

### Full markdown AST parsing

Extract code blocks, links, lists, block quotes from markdown. In v1, parsing is shallow (title, body, sections only).
