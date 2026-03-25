# agents.md — Guide for AI Agents Working on cortex-embedded

You are working on **cortex-embedded**, the upstream cognitive engine for self-aware AI agents. This file tells you everything you need to know to be productive.

## What This Repo Is

cortex-embedded is a Rust crate that provides a complete graph-memory runtime. Everything — identity, knowledge, tool calls, LLM interactions, sub-agent work — is a node in a single SQLite-backed graph. The agent queries its own history the same way it queries any other knowledge.

**This is the engine.** It does NOT include omnichannel features (HTTP API, identity resolution, session management). Those live in downstream forks:
- **cede** — forkable starter kit (same code, rebranded)
- **omni-cede** — adds HTTP API, identity layer, session manager

## Repository Layout

```
src/
  lib.rs              # CortexEmbedded struct, background tasks, decay, consolidation
  types.rs            # All types: Node, Edge, NodeKind, EdgeKind, Message, LlmResponse, etc.
  error.rs            # CortexError enum, Result type alias
  config.rs           # Config struct with all tunable parameters
  agent/
    mod.rs            # Re-exports Agent
    orchestrator.rs   # Agent struct, run() and run_turn() methods, tool-call loop
    subagent.rs       # Sub-agent spawning and delegation
  db/
    mod.rs            # Db struct (Arc<Mutex<Connection>>), async call() wrapper
    schema.rs         # CREATE TABLE statements, migrations
    queries.rs        # All SQL queries as functions
  embed/
    mod.rs            # EmbedHandle — fastembed wrapper with LRU cache
  hnsw/
    mod.rs            # VectorIndex — 2-tier HNSW (built index + linear buffer)
  graph/
    mod.rs            # BFS traversal, graph walk scoring
  memory/
    mod.rs            # recall(), briefing(), briefing_with_kinds(), recency window
  tools/
    mod.rs            # ToolRegistry, builtin tools (remember, recall, forget, etc.)
  llm/
    mod.rs            # LlmClient trait, AnthropicClient, OllamaClient, MockLlm
  cli/
    mod.rs            # CLI commands: Chat, Ask, Memory, Soul, Sessions, Graph, etc.
    graph_tui.rs      # Interactive TUI graph explorer with chat panel
    graph_viz.rs      # ASCII graph visualization
  bin/
    cortex.rs         # Binary entry point — calls cli::run()
tests/
  integration.rs      # 22 integration tests covering all phases
```

## Key Architecture Concepts

### The Graph
Every piece of information is a `Node` with a `NodeKind`. Relationships are `Edge`s with an `EdgeKind`. The graph is stored in SQLite.

### Node Kinds (18 total)
- **Knowledge:** Fact, Entity, Concept, Decision
- **Identity:** Soul (decay: 0.0), Belief (decay: 0.0), Goal (decay: 0.0)
- **Conversational:** UserInput (decay: 0.02, importance: 0.4)
- **Operational:** Session, Turn, LlmCall, ToolCall, LoopIteration (decay: 0.05)
- **Sub-agents:** SubAgent, Delegation, Synthesis
- **Self-model:** Pattern, Limitation, Capability (decay: 0.005)

### Edge Kinds (6 total)
RelatesTo, Contradicts, Supports, DerivesFrom, PartOf, Supersedes

### Db Pattern
All database access goes through `Db::call(|conn| { ... })` which runs the closure on `spawn_blocking`. Never hold the mutex across an await.

### Graph-Native Chat Sessions
`Agent::run_turn(session_id, input)` does NOT maintain a growing message array. Instead:
1. Stores user input as a `UserInput` node (embedded, auto-linked)
2. Builds a FRESH briefing via HNSW semantic search
3. Merges a recency window (last 7 session nodes) for conversational continuity
4. Sends `[system(briefing), user(input)]` to the LLM

### Auto-Link Pipeline (3 tiers)
1. cosine >= 0.75 -> RelatesTo edge
2. cosine >= 0.85 + negation keywords -> candidate for Contradicts
3. If LLM available, adjudicate contradiction; else trust keyword heuristic

### Decay
Proportional to elapsed time: `importance * (1 - decay_rate)^steps`. Soul/Belief/Goal are immune (rate: 0.0).

## How to Build and Test

```bash
cargo build
cargo test -- --test-threads=1    # 28 tests (6 unit + 22 integration)
```

Tests use `MockLlm` and in-memory SQLite — no API keys needed.

## Important Conventions

- All async DB access: `db.call(move |conn| { ... }).await`
- Embeddings: 384-dimensional f32 vectors (BAAI/bge-small-en-v1.5)
- Node IDs: UUID v4 strings
- Timestamps: Unix seconds (i64)
- Error handling: `CortexError` enum, `Result<T>` type alias
- The `CortexEmbedded` struct owns db, embed, hnsw, config, background channels
- The `Agent` struct borrows shared resources from CortexEmbedded

## Branch Policy

- `master` is protected: no direct push, PRs required
- Work on `dev` branch, merge via PR
- All 28 tests must pass before merge