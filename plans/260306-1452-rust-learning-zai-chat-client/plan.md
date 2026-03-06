---
title: "Rust Learning Path: Z.AI Chat Client"
description: "Learn Rust from scratch to expert while building a minimal AI chat client connecting to Z.AI provider"
status: pending
priority: P1
effort: 12 weeks
branch: master
tags: [rust, learning, ai, chat-client, zai, cli]
created: 2026-03-06
---

# Rust Learning Path: Z.AI Chat Client

## Overview

Build a minimal Z.AI chat client while learning Rust fundamentals. Target: .NET Backend developer transitioning to Rust.

**Reference Project:** ZeroClaw (https://github.com/zeroclaw-labs/zeroclaw) - Rust port with Z.AI provider in `src/providers/glm.rs`

## Progress Summary

| Phase | Name | Status | Effort | Key Deliverables |
|-------|------|--------|--------|------------------|
| 0 | Rust Fundamentals | pending | 2 weeks | Ownership, borrowing, traits mastery |
| 1 | Minimal Chat Client | pending | 2 weeks | Working Z.AI API integration |
| 2 | CLI Interface | pending | 2 weeks | Interactive REPL with commands |
| 3 | Memory & Persistence | pending | 2 weeks | SQLite-based conversation storage |
| 4 | Tool Execution | pending | 4 weeks | Shell commands, function calling |

## Architecture Preview

```
hello_rust/
├── Cargo.toml
├── .env.example
├── docs/
│   └── rust-learning-notes.md
├── src/
│   ├── main.rs
│   ├── lib.rs
│   ├── client/
│   │   ├── mod.rs
│   │   ├── http.rs
│   │   └── auth.rs
│   ├── config/
│   │   ├── mod.rs
│   │   └── settings.rs
│   ├── memory/
│   │   ├── mod.rs
│   │   ├── store.rs
│   │   └── types.rs
│   ├── tools/
│   │   ├── mod.rs
│   │   ├── traits.rs
│   │   └── shell.rs
│   └── cli/
│       ├── mod.rs
│       ├── repl.rs
│       └── commands.rs
└── tests/
```

## Phase Files

- [Phase 0: Rust Fundamentals](./phase-00-rust-fundamentals.md)
- [Phase 1: Minimal Chat Client](./phase-01-minimal-chat-client.md)
- [Phase 2: CLI Interface](./phase-02-cli-interface.md)
- [Phase 3: Memory & Persistence](./phase-03-memory-persistence.md)
- [Phase 4: Tool Execution](./phase-04-tool-execution.md)

## Key Dependencies

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
reqwest = { version = "0.12", features = ["json"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
anyhow = "1"
thiserror = "1"
dotenvy = "0.15"
clap = { version = "4", features = ["derive"] }
rustyline = "14"
rusqlite = { version = "0.31", features = ["bundled"] }
base64 = "0.22"
hmac = "0.12"
sha2 = "0.10"
```

## .NET to Rust Quick Reference

| .NET | Rust | Note |
|------|------|------|
| `class` | `struct` + `impl` | Data/behavior separated |
| `interface` | `trait` | No data in traits |
| `null` | `Option<T>` | Explicit null handling |
| `Exception` | `Result<T, E>` | Errors are values |
| `using` | `Drop` trait | RAII pattern |
| `async/await` | `async/await` | Same syntax, tokio runtime |
| `LINQ` | `iter()` | Functional style |

## Unresolved Questions

1. Streaming responses (SSE) - implement in Phase 1 or later?
2. Default model - glm-5 vs glm-4.7?
3. Rate limiting/retry strategy?

## Success Metrics

- [ ] Phase 0: Complete Rustlings exercises
- [ ] Phase 1: Successfully call Z.AI API and print response
- [ ] Phase 2: Interactive REPL with history navigation
- [ ] Phase 3: Conversations persist across restarts
- [ ] Phase 4: Shell commands execute safely