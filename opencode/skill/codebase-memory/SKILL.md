---
name: codebase-memory
description: "Index and query codebase structure as a knowledge graph. Use when: understanding project architecture, finding functions/classes/modules, tracing call chains, detecting dead code, analyzing impact of changes, exploring HTTP routes, or when starting a new session and need full project context without reading all files. Replaces grep-based file exploration with graph queries — 99% fewer tokens."
mcp:
  codebase-memory-mcp:
    command: "C:\\Users\\Asus\\AppData\\Local\\Programs\\codebase-memory-mcp\\codebase-memory-mcp.exe"
    args: []
---

# Codebase Memory: Knowledge Graph for Code Intelligence

This skill indexes your codebase into a persistent knowledge graph (SQLite) using tree-sitter AST parsing.
Once indexed, LLM queries the graph instead of reading files — 99% fewer tokens, sub-ms responses.
All data stays local at `~/.cache/codebase-memory-mcp/`. Zero cloud, zero API keys.

## When to Use

- Starting a new session on an existing project — call `get_architecture` instead of reading the whole codebase
- Need to understand project structure, entry points, routes, hotspots
- Tracing "who calls this function?" or "what does this function call?"
- Finding dead/unused code
- Analyzing impact before making changes (what breaks if I change X?)
- Searching for specific functions, classes, or modules by pattern
- Recording architecture decisions that persist across sessions

## Workflow

### First Time: Index the Project
1. Call `mcp__codebase_memory_mcp__index_repository` with the project path
2. Wait for indexing to complete (milliseconds for most repos)
3. The graph is now persistent — survives restarts

### Every New Session: Get Full Context
1. Call `mcp__codebase_memory_mcp__get_architecture` — returns EVERYTHING in one call:
   - Languages, packages, entry points
   - HTTP routes, hotspots, boundaries
   - Functional clusters (Louvain community detection)
   - Architecture Decision Records
2. This single call replaces reading hundreds of files

### Querying the Graph
- `mcp__codebase_memory_mcp__search_graph` — Find functions/classes/modules by name pattern
- `mcp__codebase_memory_mcp__trace_call_path` — Who calls X? What does X call? (depth 1-5)
- `mcp__codebase_memory_mcp__get_code_snippet` — Read source of a specific function
- `mcp__codebase_memory_mcp__query_graph` — Cypher-like queries for complex questions
- `mcp__codebase_memory_mcp__detect_changes` — Git diff → affected symbols + risk

## Available Tools

### Indexing
- `mcp__codebase_memory_mcp__index_repository` — Index a repo into the graph (auto-syncs after)
- `mcp__codebase_memory_mcp__list_projects` — List all indexed projects with node/edge counts
- `mcp__codebase_memory_mcp__delete_project` — Remove a project from the graph
- `mcp__codebase_memory_mcp__index_status` — Check indexing progress

### Querying
- `mcp__codebase_memory_mcp__get_architecture` — Full project overview in ONE call
- `mcp__codebase_memory_mcp__search_graph` — Search by label, name pattern, file pattern
- `mcp__codebase_memory_mcp__trace_call_path` — BFS call graph traversal (inbound/outbound/both)
- `mcp__codebase_memory_mcp__get_code_snippet` — Read source of a specific function by qualified name
- `mcp__codebase_memory_mcp__detect_changes` — Map git diff to affected symbols + blast radius
- `mcp__codebase_memory_mcp__query_graph` — Cypher-like graph queries (read-only)
- `mcp__codebase_memory_mcp__get_graph_schema` — Node/edge counts and relationship patterns
- `mcp__codebase_memory_mcp__search_code` — Grep-like text search within indexed files
- `mcp__codebase_memory_mcp__manage_adr` — CRUD for Architecture Decision Records
- `mcp__codebase_memory_mcp__ingest_traces` — Ingest runtime traces to validate HTTP call edges

## Best Practices

1. **Always `get_architecture` first** in a new session — it gives you the full picture
2. **Use `search_graph` before `get_code_snippet`** — find the qualified name first, then read the code
3. **Use `trace_call_path` instead of grep** — graph traversal is faster and more accurate
4. **Record decisions with `manage_adr`** — they persist across sessions, unlike chat context
5. **Auto-sync is on** — graph updates automatically when you edit files, no re-indexing needed
6. **Use `detect_changes` before PRs** — see the blast radius of your changes
