---
name: exa-tool
description: "Web search and code research using Exa AI. Use when: searching for up-to-date information, finding documentation, looking up API usage and code examples, researching libraries/frameworks, verifying facts, or answering questions that need current web data. Preferred over built-in web search for higher quality results."
mcp:
  exa:
    command: "npx"
    args: ["-y", "exa-mcp-server", "tools=web_search_exa,get_code_context_exa"]
    env:
      EXA_API_KEY: "x"
---

# Exa Search

This skill provides web search and code context tools via the Exa MCP server.

## Available Tools
- `web_search_exa` — Search the web for any topic
- `get_code_context_exa` — Find code examples, docs, and programming solutions
