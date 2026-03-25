# claude.md — Instructions for Claude Working on cortex-embedded

## Identity

You are working on **cortex-embedded** — the upstream cognitive engine for self-aware AI agents built by MikeSquared Agency. This is the core runtime that downstream forks (cede, omni-cede) depend on.

## Your Role

When working on this codebase you are an expert Rust systems programmer. You understand async runtimes, SQLite, embedding models, and graph data structures. You write idiomatic Rust with proper error handling.

## Critical Rules

1. **Never break the public API.** Downstream crates (cede, omni-cede) depend on `CortexEmbedded`, `Agent`, `Config`, `Node`, `Edge`, `NodeKind`, `EdgeKind`, and the `Db` type. Changing signatures here breaks the ecosystem.
2. **All DB access through `db.call()`** — never access the connection directly. The pattern is:
   ```rust
   db.call(move |conn| {
       // synchronous rusqlite code here
       Ok(result)
   }).await?
   ```
3. **Tests must pass.** Run `cargo test -- --test-threads=1` before any commit. There are 28 tests (6 unit + 22 integration). They use `MockLlm` and in-memory SQLite.
4. **UTF-8 only.** This codebase has had Windows-1252 encoding issues before. Always use `—` (UTF-8 em dash U+2014), never byte 0x97.
5. **No growing message arrays in sessions.** The `run_turn()` method builds a fresh briefing every turn. This is by design — the graph IS the memory.

## Architecture Quick Reference

| Struct | Location | Purpose |
|--------|----------|---------|
| CortexEmbedded | lib.rs | Top-level runtime, owns all resources |
| Agent | agent/orchestrator.rs | Runs queries and chat turns |
| Db | db/mod.rs | Arc<Mutex<Connection>> with async wrapper |
| VectorIndex | hnsw/mod.rs | 2-tier HNSW for semantic search |
| EmbedHandle | embed/mod.rs | fastembed with LRU cache |
| Config | config.rs | All tunable parameters |
| ToolRegistry | tools/mod.rs | Registered tools the agent can call |

## How to Add Things

### Adding a New Node Kind
1. Add variant to `NodeKind` enum in `types.rs`
2. Set decay_rate and default importance in the `impl NodeKind` block
3. Update `Display` and `FromStr` impls
4. Add test coverage in `tests/integration.rs`

### Adding a New Edge Kind
1. Add variant to `EdgeKind` enum in `types.rs`
2. Update `Display` and `FromStr` impls
3. If the edge has special semantics (like Contradicts), add logic in `memory/mod.rs`

### Adding a New Tool
1. Add to `ToolRegistry::builtins()` in `tools/mod.rs`
2. Provide name, description, parameter schema (JSON), and handler closure
3. The handler receives `(db, embed, hnsw, config, args)` and returns `Result<String>`

### Adding a New CLI Command
1. Add variant to the `Commands` enum in `cli/mod.rs`
2. Add match arm in the `run()` function
3. Keep it thin — delegate to library functions

## Style Guide

- Use `thiserror` derive macros for error variants
- Prefer `impl Into<String>` over `&str` in public APIs
- Use `tracing` for logging (not `println!` in library code)
- Keep functions under 50 lines when possible
- Document public items with `///` doc comments

## Config Defaults to Know

- `session_recency_window`: 7 (turns included in briefing)
- `auto_link_cosine_threshold`: 0.75
- `contradiction_cosine_threshold`: 0.85
- `max_iterations`: 10 (tool-call loop limit)
- `decay_interval_secs`: 60

## Common Pitfalls

- **CortexError::DbTask** is the variant for database errors, NOT `CortexError::Database`
- The HNSW index has a buffer that must be flushed (`build()`) for queries to see new vectors
- `fastembed` model download happens on first embed call — tests use `MockLlm` and pre-seeded embeddings
- SQLite in WAL mode — writes don't block reads, but only one writer at a time