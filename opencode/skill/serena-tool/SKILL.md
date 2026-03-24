---
name: serena-tool
description: "Explore and navigate LOCAL codebases using Serena. Use when: exploring project structure, finding symbols/functions/classes, tracing code flow and references, analyzing architecture, onboarding a new project, or saving/reading research findings to memory. This is for LOCAL projects only — not GitHub remote repos."
mcp:
  serena:
    command: "C:\\Users\\Asus\\.local\\bin\\uvx.exe"
    args: ["--from", "git+https://github.com/oraios/serena", "serena", "start-mcp-server", "--context", "ide-assistant", "--mode", "planning", "--mode", "interactive"]
---

# Codebase Explorer (Serena)

This skill provides codebase navigation and exploration via the Serena MCP server.

## Workflow
1. Call `serena_activate_project` first
2. Call `serena_check_onboarding_performed`
3. If not onboarded, call `serena_onboarding`
4. Use `serena_list_dir`, `serena_get_symbols_overview`, `serena_find_symbol`, etc.

## Available Tools

### Setup
- `serena_activate_project` — Activate a project for exploration
- `serena_check_onboarding_performed` — Check if project was onboarded
- `serena_onboarding` — Run onboarding for a new project

### Navigation
- `serena_list_dir` — List directory contents
- `serena_find_file` — Find files by name
- `serena_search_for_pattern` — Search for text patterns
- `serena_get_symbols_overview` — Get symbols in a file
- `serena_find_symbol` — Find a specific symbol
- `serena_find_referencing_symbols` — Find references to a symbol

### Memory
- `serena_write_memory` — Save findings for future reference
- `serena_read_memory` — Read saved findings
- `serena_list_memories` — List all saved memories
- `serena_edit_memory` — Edit an existing memory
- `serena_rename_memory` — Rename a memory
- `serena_delete_memory` — Delete a memory
