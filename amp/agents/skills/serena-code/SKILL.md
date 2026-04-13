---
name: serena-code
description: "Language-server-powered code intelligence for reading, searching, navigating, and editing code symbols. Use when needing to understand code structure, find symbols/references, edit code by symbol, search patterns in codebase, or manage project memories."
---

# Serena Code Intelligence

Provides LSP-based code operations: symbol navigation, reference finding, pattern search, and intelligent code editing.

## Setup
- On first use in a new project, call `check_onboarding_performed`. If not done, call `onboarding`.

## Core Workflows

### Understanding Code
1. `get_symbols_overview` — get high-level view of a file's symbols
2. `find_symbol` — find symbols by name path pattern (supports depth, body inclusion)
3. `find_referencing_symbols` — find all references to a symbol
4. `list_dir` / `find_file` — navigate project structure

### Searching Code
- `search_for_pattern` — regex search across codebase with flexible file filtering
- Supports `restrict_search_to_code_files`, glob include/exclude, context lines

### Editing Code
- `replace_content` — replace content by literal or regex pattern (preferred for edits)
- `safe_delete_symbol` — safely delete unreferenced symbols

### Project Memory
- `write_memory` / `read_memory` / `list_memories` — persist notes across sessions
- `edit_memory` / `delete_memory` / `rename_memory` — manage existing memories
- Use `/` in names to organize by topic (e.g., `auth/login/logic`)

## Tips
- Prefer `get_symbols_overview` first when exploring a new file
- Use `find_symbol` with `include_body=true` to read implementation details
- Use `replace_content` with regex mode and wildcards for efficient multi-line edits
- Always use `safe_delete_symbol` before removing code to check for references
