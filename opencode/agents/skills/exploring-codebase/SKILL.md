---
name: exploring-codebase
description: "Explores and understands codebases with minimal token consumption. Use when asked to explore, understand, onboard, or learn a codebase. Triggers on: explore codebase, understand project, codebase overview, onboard, walk through codebase."
---

# Exploring Codebase — Minimal Token, Maximum Understanding

Systematic codebase exploration that prioritizes structured metadata over raw file reads.
Every step is designed to minimize token usage while building comprehensive understanding.

## Core Principle

**Never read what you can summarize. Never summarize what you can list. Never list what you already know.**

Token cost hierarchy (cheapest → most expensive):
`list_dir` < `glob` < `get_symbols_overview` < `Grep` < `find_symbol` < `Read (with range)` < `finder` < `Read (full file)` < `oracle`

## Phase 1: Project Shape (≈200-500 tokens)

Goal: understand project type, language, framework, structure.

1. **List root directory** — non-recursive first:
   - `list_dir(".", recursive=false)` 
   - Identify: package manager files, config files, entry points, src layout

2. **Read ONLY config/manifest files** (small, high-signal):
   - `package.json`, `pyproject.toml`, `Cargo.toml`, `go.mod`, `pom.xml`, etc.
   - `README.md` (first 50 lines only via `read_range: [1, 50]`)
   - Extract: dependencies, scripts, project name, description

3. **List key directories** — one level deep:
   - `list_dir("src", recursive=false)` or equivalent main source dir
   - Skip: `node_modules`, `vendor`, `dist`, `build`, `.git`, `__pycache__`

**Output after Phase 1:** Project type, language, framework, rough module layout.

## Phase 2: Architecture Map (≈500-1500 tokens)

Goal: understand module boundaries, key abstractions, entry points.

4. **Symbols overview of entry points** — NO source code yet:
   - `get_symbols_overview("src/main.ts", depth=1)` (or equivalent entry file)
   - `get_symbols_overview("src/app.py", depth=1)`
   - This gives class/function names WITHOUT reading bodies

5. **Symbols overview of key modules** — depth=1 for top-level exports:
   - Pick 3-5 most important-looking directories from Phase 1
   - Run `get_symbols_overview` on their index/main files
   - Build a mental map: "module X exports classes A, B, C"

6. **Find core patterns with targeted Grep** (if needed):
   - `Grep("export class|export function", path="src")` — find public API surface
   - `Grep("@route|@app\\.|router\\.", path="src")` — find API endpoints
   - `Grep("class.*Model|class.*Schema", path="src")` — find data models

**Output after Phase 2:** Module map, key classes/functions, API surface, data models.

## Phase 3: Relationship Mapping (≈500-1000 tokens)

Goal: understand how modules connect, data flows, dependency graph.

7. **Trace key symbols** — find who calls what:
   - `find_referencing_symbols` on 3-5 core classes/functions
   - This reveals the dependency graph without reading code

8. **Selective deep-dive** — Read ONLY critical code:
   - Read entry point function body: `find_symbol("main", include_body=true)`
   - Read config/DI setup if exists
   - Use `read_range` to read specific line ranges, never full files

9. **Visualize with mermaid** — generate architecture diagram:
   - Draw module relationships based on what you found
   - Include file citations for clickable navigation

**Output after Phase 3:** Dependency graph, data flow, architecture diagram.

## Phase 4: Deep Understanding (on-demand, ≈1000-2000 tokens)

Only if user asks to go deeper on specific areas:

10. **Targeted symbol reading**:
    - `find_symbol("ClassName/methodName", include_body=true)` — read ONE method
    - Never read entire files when you only need one function

11. **Call chain tracing**:
    - `find_referencing_symbols` → follow the chain 2-3 levels deep
    - Map: entry point → service → repository → database

12. **Oracle for complex analysis** (most expensive, use last):
    - Only after you have concrete code context from steps above
    - Ask specific questions, not "explain this codebase"

## Anti-Patterns — NEVER Do These

| ❌ Token Waste | ✅ Do Instead |
|---|---|
| `Read` entire large files | `get_symbols_overview` → `Read` with `read_range` |
| `finder` with vague queries | `find_symbol` or `Grep` with exact terms |
| `Read` every file in a directory | `list_dir` → pick important ones → `get_symbols_overview` |
| Call `oracle` without context | Gather context with Serena first → then Oracle |
| Read test files to understand code | Read source files, tests are secondary |
| Read generated/compiled files | Skip `dist/`, `build/`, `.next/`, `__pycache__/` |
| Read lock files (`package-lock.json`) | Read manifest (`package.json`) only |
| Full recursive `list_dir` on large repos | Non-recursive first, drill into relevant dirs |

## Tool Selection Quick Reference

| Need | Best Tool | Token Cost |
|---|---|---|
| "What files exist?" | `list_dir` or `glob` | Very Low |
| "What's in this file?" (structure) | `get_symbols_overview` | Low |
| "Where is X defined?" | `find_symbol` | Low |
| "Who uses X?" | `find_referencing_symbols` | Low-Med |
| "Find pattern across files" | `Grep` (exact) or `search_for_pattern` | Low-Med |
| "Read specific code" | `Read` with `read_range` | Medium |
| "Find code by concept" | `finder` | Medium-High |
| "Analyze/review/debug" | `oracle` | High |
| "Understand external lib" | `librarian` or Exa | Medium |

## Output Format

After exploration, provide:
1. **One-liner**: what the project does
2. **Tech stack**: language, framework, key deps
3. **Architecture diagram**: mermaid with file citations
4. **Module map**: key modules and their responsibilities (3-5 bullet points)
5. **Entry points**: where execution starts
6. **Key patterns**: design patterns, conventions used

## Serena MCP Integration

If Serena is available (activated project with LSP):
- **Prefer Serena tools over Read/Grep** for code analysis
- `get_symbols_overview` > reading files for structure
- `find_symbol` > Grep for finding definitions
- `find_referencing_symbols` > Grep for finding usages
- Serena returns structured metadata = fewer tokens than raw code

If Serena is NOT available:
- Fall back to `glob` → `Read` (with ranges) → `Grep`
- Use `finder` for semantic search when exact terms unknown

## Token Budget Guidelines

| Exploration Depth | Estimated Tokens | Time |
|---|---|---|
| Quick overview (Phase 1) | 200-500 | 1-2 min |
| Architecture map (Phase 1-2) | 700-2000 | 3-5 min |
| Full understanding (Phase 1-3) | 1500-3500 | 5-10 min |
| Deep dive specific area (Phase 4) | +1000-2000 per area | +3-5 min |
