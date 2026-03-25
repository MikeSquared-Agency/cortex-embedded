# cortex-embedded

**The cognitive engine. One crate. One SQLite file. A complete graph-memory runtime for self-aware AI agents.**

Everything — identity, knowledge, tool calls, LLM calls, sub-agent work, loop iterations, self-model — is a node in the graph. The agent queries its own history the same way it queries any other knowledge.

> **This is the upstream engine.** If you want to build your own agent, fork [**cede**](https://github.com/MikeSquared-Agency/cede) instead. If you want an omnichannel deployment with HTTP API, see [**omni-cede**](https://github.com/MikeSquared-Agency/omni-cede).

## Ecosystem

```
cortex-embedded          <-- you are here (the engine)
  |-- cede               <-- forkable starter kit for building self-aware agents
       |-- omni-cede     <-- omnichannel variant (HTTP API, identity, sessions)
```

## Features

- **Graph memory** — 18 node kinds, 6 edge kinds, full provenance tracking
- **Graph-native chat sessions** — each turn stores a `UserInput` node, builds a fresh HNSW-based briefing (no growing message history)
- **Hybrid recall** — HNSW ANN search + BFS graph traversal + trust scoring + recency decay + session recency window
- **Embeddings** — BAAI/bge-small-en-v1.5 via fastembed (384-dim, runs locally)
- **Auto-link** — background task creates `RelatesTo` and `Contradicts` edges automatically
- **Three-tier contradiction detection** — cosine similarity -> negation keyword heuristic -> LLM adjudication
- **Decay** — importance fades over time; Soul/Belief/Goal nodes are immune
- **Trust propagation** — `Supports` edges boost trust, `Contradicts` edges reduce it
- **Context compaction** — LLM extracts key facts from long conversations into the graph
- **LLM backends** — Anthropic Claude, Ollama (local), Mock (testing)
- **Tool registry** — tools write provenance-tracked results into the graph
- **Sub-agents** — spawn into the shared graph with scoped identity
- **TUI graph explorer** — interactive terminal UI with chat panel, node inspection, graph visualization
- **CLI** — chat, ask, memory search, identity management, consolidation, diagnostics

## Quick Start

```bash
# Build
cargo build --release

# Initialize database and download embedding model
cortex init

# Interactive chat (requires LLM)
ANTHROPIC_API_KEY=sk-ant-... cortex chat
# or with Ollama
cortex --ollama llama3 chat

# Single query
cortex ask "What do you know about Rust?"

# Interactive graph explorer with chat
ANTHROPIC_API_KEY=sk-ant-... cortex graph explore

# Graph overview
cortex graph overview

# Filter graph by node kind
cortex graph filter soul,belief,fact

# Memory stats
cortex memory stats

# Semantic search
cortex memory search "authentication"

# View identity
cortex soul show

# Check graph health
cortex doctor

# Run trust consolidation
cortex consolidate
```

## Architecture

```
+---------------------------------------------+
|              cortex-embedded                 |
+----------+----------+----------+------------+
|  recall  | briefing |  tools   |   agent    |
| (hybrid  | (context | (registry|  (loop +   |
|  search) |  doc)    |  + trust)| sub-agents)|
+----------+----------+----------+------------+
|              graph + memory                  |
|         (BFS walk, scoring, decay)           |
+----------+----------------------------------+
|   HNSW   |           SQLite                  |
| (2-tier) |  (WAL mode, bundled rusqlite)     |
+----------+----------------------------------+
|              fastembed                        |
|        (BAAI/bge-small-en-v1.5)              |
+---------------------------------------------+
```

### Graph-Native Chat Sessions

Unlike traditional chatbots that grow a message array, cortex-embedded stores each user input as a `UserInput` node in the graph. Every turn:

1. **Store** the user's text as a `UserInput` node (embedded, auto-linked)
2. **Build a fresh briefing** using the input as a semantic query against the full graph
3. **Include the recency window** — always surface the last 7 session nodes regardless of similarity
4. **Send** `[system(briefing), user(input)]` to the LLM — no growing history

This means the agent's "memory" is the graph itself. Relevant prior exchanges surface through HNSW similarity; meta-conversational messages ("stop using big words") surface through the recency window.

### Node Kinds

| Category | Kinds |
|----------|-------|
| Knowledge | `Fact`, `Entity`, `Concept`, `Decision` |
| Identity | `Soul`, `Belief`, `Goal` |
| Conversational | `UserInput` |
| Operational | `Session`, `Turn`, `LlmCall`, `ToolCall`, `LoopIteration` |
| Sub-agents | `SubAgent`, `Delegation`, `Synthesis` |
| Self-model | `Pattern`, `Capability`, `Limitation` |

### Edge Kinds

`RelatesTo` · `Contradicts` · `Supports` · `DerivesFrom` · `PartOf` · `Supersedes`

## How It Works

Every interaction creates a provenance chain:

```
UserInput -> Session
Fact -> ToolCall -> LoopIteration -> Session
```

The agent knows not just *what* it knows, but *how it came to know it*, *when*, *via which tool*, and *how much to trust it*.

**Recall pipeline:**
1. Embed query -> HNSW k-NN search
2. BFS graph walk from candidates
3. Score: `importance * trust * recency * proximity_bonus`
4. Merge recency window (last 7 session nodes)
5. Return ranked nodes with contradiction warnings

**Background tasks:**
- **Auto-link** — new nodes are compared against the graph; similar nodes get `RelatesTo` edges
- **Contradiction detection** — three-tier pipeline: cosine threshold -> negation keywords -> LLM adjudication
- **Decay** — every 60s, nodes lose importance proportional to elapsed time (floor: 0.01)

## Using as a Library

```rust
use cortex_embedded::{CortexEmbedded, types::*};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let cx = CortexEmbedded::open("my_agent.db").await?;

    // Store knowledge
    let node = Node::new(NodeKind::Fact, "Rust is fast")
        .with_body("Rust provides zero-cost abstractions and memory safety.");
    cx.remember(node).await?;

    // Recall
    let results = cx.recall("performance", RecallOptions::default()).await?;
    for r in &results {
        println!("[{}] {} - score: {:.3}", r.node.kind, r.node.title, r.score);
    }

    // Build briefing for LLM
    let briefing = cx.briefing("system design", 12).await?;
    println!("{}", briefing.context_doc);

    Ok(())
}
```

## Dependencies

| Crate | Purpose |
|-------|---------|
| `rusqlite` (bundled) | SQLite with WAL mode |
| `instant-distance` | HNSW approximate nearest neighbor search |
| `fastembed` | Local text embeddings (ONNX runtime) |
| `tokio` | Async runtime |
| `reqwest` | HTTP client for Anthropic API |
| `clap` | CLI argument parsing |
| `ratatui` + `crossterm` | TUI graph explorer |
| `async-channel` | Background task communication |

## Tests

```bash
# Run all 28 tests
cargo test -- --test-threads=1

# Unit tests only (HNSW)
cargo test --lib hnsw

# Integration tests only
cargo test --test integration -- --test-threads=1
```

## License

MIT