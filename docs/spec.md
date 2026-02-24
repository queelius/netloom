# netloom DSL Specification

**Status**: Draft (design phase, pre-implementation)

netloom configs have four top-level sections. All are optional, but a useful config needs at least `nodes` and either `links` or `network`.

```yaml
schema:       # Data shape and extraction (optional)
nodes:        # Node types and embedding config
links:        # Relationship definitions between nodes
network:      # Graph construction parameters
```

---

## Schema

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
| `join` | list of text | text (concatenated) |
| `first` | list | first element |
| `last` | list | last element |
| `count` | list | number (length) |
| `unique` | list | list (deduplicated) |
| `sum` | list of numbers | number |
| `mean` | list of numbers | number |
| `min` | list of numbers | number |
| `max` | list of numbers | number |

Without `reduce`, the pipeline output is a list.

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

## Nodes

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

### Embedding configuration

The `embed` block configures vectorization for similarity computation.

```yaml
embed:
  field: title           # which node field to embed
  model: tfidf           # embedding provider (resolved via registry)
```

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

#### Aggregate

When embedding produces multiple vectors (from chunking a text field, or from embedding a list field), `aggregate` combines them into one:

| Aggregate | Description |
|-----------|-------------|
| `mean` | Average of all vectors (default) |
| `max` | Element-wise maximum |
| `first` | First vector only |
| `last` | Last vector only |

Three embedding patterns:
1. **Simple**: `field + model` -- one string produces one vector
2. **Chunked**: `field + model + chunking + aggregate` -- long text is split, embedded, combined
3. **List**: `field + model + aggregate` -- list of strings, each embedded, combined

The presence or absence of `chunking` and `aggregate` tells you which pattern is in use. No hidden behavior.

### Combine (embedding-level)

Weighted merge of multiple node types' embeddings. Used when a single similarity computation should blend multiple signals at the vector level:

```yaml
nodes:
  # ... other node types with embed blocks ...

  intent_node:
    from: .
    fields:
      user_text:
        from: turns
        where: { role: user }
        pluck: text
        reduce: join
    embed:
      combine:
        - ref: user_vec
          weight: 0.67
        - ref: assistant_vec
          weight: 0.33
```

---

## Links

The links section defines **relationship types** between nodes. Each link specifies which node types it connects, how similarity is computed, and optional thresholds.

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

| Method | Description | Requires |
|--------|-------------|----------|
| `cosine` | Cosine similarity on embedding vectors | Both node types have `embed` |
| `jaccard` | Jaccard set similarity | `field:` (list field) |
| `dice` | Dice coefficient | `field:` (list field) |
| `overlap` | Overlap coefficient | `field:` (list field) |
| `exact` | Boolean equality (1.0 or 0.0) | `field:` |
| `numeric` | Numeric similarity | `field:` (number field) |
| `parent` | Structural containment | No computation; creates edges from source structure |

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

### `min` threshold

Replaces the old `gate` operator. Edges with scores below `min` are not created.

```yaml
links:
  text_similarity:
    between: [user_turn, user_turn]
    method: cosine
    min: 0.3    # ignore similarities below 0.3
```

### Combine (link-level)

Weighted combination of multiple link scores, with optional per-ref thresholds:

```yaml
links:
  overall:
    between: [conversation, conversation]
    combine:
      - ref: intent_sim
        weight: 0.5
        min: 0.2       # ignore intent_sim below 0.2
      - ref: title_sim
        weight: 0.15
      - ref: tag_sim
        weight: 0.15
      - ref: tool_sim
        weight: 0.1
      - ref: project_sim
        weight: 0.1
```

The `min` on a ref means: "if this component's score is below the threshold, treat it as 0 in the weighted average." This replaces the old `gate` operator with inline syntax -- no separate named intermediate, no indirection.

### What's NOT in the DSL

The following operators from the predecessor project (complex-network-rag) are deliberately excluded:

- **`gate`** -- replaced by `min:` on refs
- **`boost`** -- exotic composition; use a Python plugin
- **`when`** -- conditional branching; use a Python plugin

The DSL makes the common case trivial (weighted averages with thresholds cover ~90% of use cases). Exotic composition uses the Python escape hatch: write a class implementing the `SimilarityProvider` protocol, register it, reference it by name in the DSL.

---

## Network

The network section controls graph construction parameters.

```yaml
network:
  min: 0.3                    # global minimum edge weight
  strong: 0.6                 # threshold for "strong" edge designation
  communities:
    algorithm: louvain         # community detection algorithm
```

### Edge thresholds

| Parameter | Description |
|-----------|-------------|
| `min` | Minimum edge weight to include in the graph |
| `strong` | Threshold for marking edges as "strong" (for visualization/analysis) |

### Community detection

```yaml
communities:
  algorithm: louvain            # louvain | label_propagation
```

### Density control (planned)

```yaml
network:
  max_per_node: 20              # maximum edges per node
  max_total: 10000              # maximum total edges
```

### Weight transforms (planned)

```yaml
network:
  weight_transform: log         # log | sqrt | none
```

---

## Plugin architecture

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

## Open design questions

These are unresolved decisions that need investigation before implementation:

1. **Reduce chaining**: Should `reduce: [unique, count]` be supported for composing reduces? Alternative: named reduces like `count_unique`.

2. **Chunking on list fields**: What happens if `chunking` is specified on a field that resolves to a list? The intended behavior is "chunk within each list item" but this may be surprising.

3. **Embedding defaults**: Should there be a `defaults:` block to avoid repeating `model: tfidf` across every node type? Decided against (explicit > implicit) but the repetition is annoying.

4. **Aggregation operators**: Should `product` / `max` / `min` survive as `mode:` on `combine`, or are they cut entirely?

5. **Per-turn embeddings**: `aggregate: none` would keep per-chunk/per-item vectors separate, requiring a `match:` strategy (max, mean, direct) on the link side. Useful but second-tier.

6. **Duplicate YAML keys**: Two `contains` links with different `between` values can't share the same key name in YAML. Solution: use unique names (`contains_user_turns`, `contains_assistant_turns`).
