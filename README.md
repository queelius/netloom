# netloom

Declarative language for constructing complex networks from structured data.

## What is netloom?

netloom is a YAML-based DSL that describes how to build a **heterogeneous similarity network** from structured documents. You declare **node types** (the units of your graph) and **link types** (how nodes relate), and netloom constructs the graph for analysis.

The output is a weighted graph you can analyze with standard tools: community detection, centrality measures, shortest paths, visualization.

## Why not just use a vector DB?

Vector databases are fast nearest-neighbor lookup engines. They answer "what's similar to X?" netloom answers a different question: "what is the *structure* of similarity across my corpus?"

|  | Vector DB | Custom Python | netloom |
|---|---|---|---|
| **Computes** | Single embedding, cosine/L2 | Anything you code | Multi-field similarity with composition |
| **Scaling** | ANN indexes, millions of docs | Depends | O(n^2) pairwise, practical to ~10K docs |
| **Retrieval** | Fast nearest-neighbor | Custom | Graph-aware: communities, hubs, bridges |
| **Metadata** | Filter only (WHERE clauses) | Custom | First-class similarity participant |
| **Configuration** | Code it | Code it | Declarative YAML |

**netloom wins when:**
- Metadata fields (tags, authors, categories) should *contribute to similarity*, not just filter results
- You care about graph structure: which documents are hubs, which bridge two communities
- You want declarative control over how similarity is composed from multiple signals
- You have multiple node types in the same graph (heterogeneous networks)
- You're running experiments or writing papers and need to iterate cheaply

**Sweet spot**: Small-to-medium corpora (<10K documents) where you want to understand structure, not just retrieve.

## Source formats

netloom ingests structured data from multiple formats:

| Format | Example |
|--------|---------|
| JSONL | `data/conversations.jsonl` |
| JSON files | `data/papers/*.json` |
| YAML | `data/config.yaml` (single or multi-document) |
| Markdown + frontmatter | `notes/*.md` (YAML frontmatter + body) |
| Plain markdown | `docs/*.md` (headings and sections extracted as structured data) |
| Plain text | `corpus/*.txt` (whole file becomes `body`) |

Markdown is treated as structured data: headings become `title`, `##` sections become a `sections` list, and the content becomes `body`. Plain text files are the simplest case — the entire content becomes `body`. Every record gets a `_meta` block with full provenance (source path, timestamps, content hash).

```yaml
source:
  path: data/conversations/
  format: jsonl
```

## Core abstractions

**Nodes** define the units of your graph. A single source document can produce multiple node types:

```yaml
nodes:
  conversation:
    from: .
    fields:
      title: { pluck: title }
      tags: { pluck: tags }
    embed:
      field: title
      model: tfidf

  user_turn:
    from: turns
    where: { role: user }
    fields:
      text: { pluck: text }
    embed:
      field: text
      model: tfidf
```

**Links** define relationships between nodes:

```yaml
links:
  intent_similarity:
    between: [user_turn, user_turn]
    method: cosine
    min: 0.3

  tag_overlap:
    between: [conversation, conversation]
    method: jaccard
    field: tags

  contains_turns:
    between: [conversation, user_turn]
    method: parent
```

**Network** controls graph construction:

```yaml
network:
  min: 0.3
  communities:
    algorithm: louvain
```

## Full example

Given a corpus of conversation JSON documents like this:

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
      "role": "assistant",
      "text": "Found it -- the Redis key prefix changed from 'session:' to 'sess:' in v7.",
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

This netloom config builds a heterogeneous graph where conversations and individual turns are separate node types, connected by semantic similarity, tag overlap, and structural containment:

```yaml
nodes:
  conversation:
    from: .
    fields:
      title: { pluck: title }
      tags: { pluck: tags }
      project: { pluck: project }
    embed:
      field: title
      model: tfidf

  user_turn:
    from: turns
    where: { role: user }
    fields:
      text: { pluck: text }
      timestamp: { pluck: timestamp }
    embed:
      field: text
      model: tfidf
      chunking:
        method: sentences
        max_tokens: 256

  assistant_turn:
    from: turns
    where: { role: assistant }
    fields:
      text: { pluck: text }
      tools: { pluck: tool_calls, default: [] }
    embed:
      field: text
      model: tfidf

links:
  user_intent_similarity:
    between: [user_turn, user_turn]
    method: cosine
    min: 0.3

  cross_role_similarity:
    between: [user_turn, assistant_turn]
    method: cosine
    min: 0.4

  tag_overlap:
    between: [conversation, conversation]
    method: jaccard
    field: tags

  same_project:
    between: [conversation, conversation]
    method: exact
    field: project

  contains_user_turns:
    between: [conversation, user_turn]
    method: parent

  contains_assistant_turns:
    between: [conversation, assistant_turn]
    method: parent

network:
  min: 0.3
  communities:
    algorithm: louvain
```

Each source document produces one `conversation` node and multiple `turn` nodes. The resulting graph has:
- **Semantic edges** between turns with similar content (cosine similarity)
- **Attribute edges** between conversations sharing tags (Jaccard) or the same project (exact match)
- **Structural edges** connecting conversations to their constituent turns (parent links)

## Plugin architecture

netloom uses a registry pattern for all pluggable components. Built-in providers and user-written providers are structurally identical:

```
netloom/
  embeddings/tfidf.py        # built-in, no heavy deps
  embeddings/ollama.py        # local Ollama models
  embeddings/openai.py        # OpenAI API
  metrics/jaccard.py          # built-in
  metrics/cosine.py           # built-in
  chunking/sentences.py       # built-in
```

Install optional providers via extras:

```bash
pip install netloom[openai]
pip install netloom[ollama]
```

Or write your own:

```python
from netloom import register_embedding

@register_embedding("my-model")
class MyEmbedding:
    def embed(self, text: str) -> list[float]:
        ...
```

Then reference it in the DSL:

```yaml
embed:
  field: text
  model: my-model
```

## Use cases

- **Conversation analysis**: Nodes are conversations and turns. Links are semantic similarity, topic overlap, temporal proximity, structural containment.
- **Paper citation networks**: Nodes are papers and authors. Links are citation, co-authorship, topic similarity.
- **Codebase analysis**: Nodes are files, functions, modules. Links are imports, call graphs, semantic similarity.
- **Multi-modal documents**: Nodes are text chunks, images, tables from the same document. Links are co-occurrence and cross-modal similarity.
- **E-commerce catalogs**: Nodes are products. Links are semantic description similarity, shared categories, price-range proximity.

## Status

**Design phase.** The DSL specification is in [`docs/spec.md`](docs/spec.md). No implementation code yet. We're refining the design before building.

## License

MIT
