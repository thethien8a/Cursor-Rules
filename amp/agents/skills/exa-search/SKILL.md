---
name: exa-search
description: "Search the web and find code examples using Exa AI. Use when needing to search for documentation, code snippets, programming solutions, current information, news, or any web content."
---

# Exa Search

Provides web search and code context search powered by Exa AI.

## Available Tools

### web_search_exa
Search the web for any topic and get clean, ready-to-use content.
- Use for: finding current information, news, facts, documentation
- Query tips: describe the ideal page, not just keywords
- Supports `freshness` filter: `24h`, `week`, `month`, `year`, `any`
- Supports `includeDomains` to scope results
- Use `numResults` to control result count (default: 8, max: 20)

### get_code_context_exa
Find code examples, documentation, and programming solutions.
- Use for: API usage, library examples, code snippets, debugging help
- Query tips: be specific (e.g., "Python requests library POST with JSON body")
- Use `numResults` to control result count (default: 8, max: 20)

## Tips
- If highlights from search are insufficient, follow up with `read_web_page` on the best URLs
- Prefer `get_code_context_exa` for programming questions, `web_search_exa` for general info
