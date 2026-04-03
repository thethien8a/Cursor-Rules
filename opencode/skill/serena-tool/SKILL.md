---
name: serena-tool
description: "Primary code intelligence for LOCAL codebases. Use FIRST for: navigating projects, finding symbols, reading/editing code, managing project memory. LSP-powered semantic analysis. Only fallback to built-in tools if Serena unavailable."
mcp:
  serena:
    command: "cmd"
    args: [
        "/c",
        "uvx -p python3.13 --from git+https://github.com/oraios/serena serena start-mcp-server --context ide --project-from-cwd"
    ]
---

# Serena - Primary Code Intelligence (Token-Optimized)

**USE SERENA FIRST** for all code operations. Only fallback to built-in tools if Serena fails.

## Why Serena-First?
- Read ONE symbol instead of entire file → saves tokens
- Semantic search → more accurate than text grep
- Memory system → avoid re-exploring same code
- LSP refactoring → safer code modifications

## Startup Sequence

1. `check_onboarding_performed` → `onboarding` if not done
2. `list_memories` → `read_memory` — load saved context

## Name Path Syntax

- Simple: `my_function` — global search
- Relative: `MyClass/my_method` — method inside class
- Absolute: `/MyClass/my_method` — exact match
- Substring: `substring_matching=true` for partial matches

## Tools

### Project Setup

| Tool | Description |
|---|---|
| `check_onboarding_performed` | Check if onboarding done |
| `onboarding` | Analyze project structure (once per conversation) |
| `safe_delete_symbol` | Delete symbol if no references exist |

### Symbol Navigation (READ)

| Tool | Description | Key Parameters |
|---|---|---|
| `get_symbols_overview` | List symbols in file (token-efficient!) | `relative_path`, `depth` |
| `find_symbol` | Find & read symbol definition | `name_path_pattern`, `include_body=true` |
| `find_referencing_symbols` | Find all callers/references | `name_path`, `relative_path` |

### Symbol Editing (WRITE)

| Tool | Description | Key Parameters |
|---|---|---|
| N/A | Symbol editing tools not available | Use `replace_content` instead |

**Note:** Use `replace_content` for code modifications instead of symbol-level editing.

### File Operations

| Tool | Description | Key Parameters |
|---|---|---|
| `list_dir` | List directory | `relative_path`, `recursive` |
| `find_file` | Find files by pattern | `file_mask` |
| `search_for_pattern` | Regex search in codebase | `substring_pattern` |
| `replace_content` | Replace text in file | `needle`, `repl`, `mode` |

**Note:** Use built-in Read tool for reading file content.

### Memory (Persistent Knowledge)

| Tool | Description |
|---|---|
| `write_memory` | Store knowledge for future sessions |
| `read_memory` | Read stored memory |
| `list_memories` | List all memories |
| `edit_memory` | Edit memory content |
| `rename_memory` | Rename/move memory |
| `delete_memory` | Delete memory |

## Token Optimization Strategies

### Reading Code
```
BAD:  Read entire file (1000+ lines)
GOOD: get_symbols_overview → find_symbol(include_body=true)
      → Read only the function you need!
```

### Finding References
```
BAD:  Grep "function_name" → text matches, false positives
GOOD: find_referencing_symbols → semantic, accurate
```

### Editing Code
```
BAD:  Read file → Edit tool → replace entire section
GOOD: replace_content with regex mode → edit efficiently
      Use mode="regex" with wildcards to avoid quoting large sections
```

### Avoid Re-exploring
```
After understanding code → write_memory("architecture/overview")
Next session → read_memory → skip exploration
```

## Workflow

```
check_onboarding_performed → onboarding if needed
→ list_memories (check existing knowledge)
→ read_memory (if relevant)
→ get_symbols_overview (see structure)
→ find_symbol (read specific code)
→ find_referencing_symbols (trace dependencies)
→ replace_content (edit code)
→ write_memory (save learnings)
```

## Fallback to Built-in Tools

Use built-in tools for:
- Reading file content → Read tool (Serena's `read_file` not available)
- Creating files → Write tool (Serena's `create_text_file` not available)
- Shell commands → Bash tool
- Git operations → Bash tool
- Binary/image files → Read tool

## Supported Languages

40+ languages: Python, TypeScript, Java, C/C++, Go, Rust, Ruby, PHP, Kotlin, Swift, etc.
