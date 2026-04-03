---
name: exa-tool
description: "Web search and code research using Exa AI. Use when: searching for up-to-date information, finding documentation, looking up API usage and code examples, researching libraries/frameworks, verifying facts, or answering questions that need current web data."
mcp:
  exa:
    command: "npx"
    args: ["-y", "exa-mcp-server", "tools=web_search_exa,get_code_context_exa"]
    env:
      EXA_API_KEY: ""
---

# Exa Search

Web search and code context tools via Exa AI.

## When to Use
- Search for up-to-date information on any topic
- Find documentation and API references
- Look up code examples and usage patterns
- Research libraries and frameworks
- Verify facts with current web data

## Available Tools
- `web_search_exa` — Search the web for any topic
- `get_code_context_exa` — Find code examples, docs, and programming solutions

## Tips
- Use `web_search_exa` for general information
- Use `get_code_context_exa` for programming-specific queries
