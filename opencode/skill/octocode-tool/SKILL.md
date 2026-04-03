---
name: octocode-tool
description: "Semantic code research on GitHub/GitLab repositories. Use when: searching code across public/private repos, finding usage patterns, exploring PRs/issues, reading implementations from remote repositories, understanding how libraries are used in real projects, or researching code architecture. This is for REMOTE GitHub/GitLab repos — not local projects."
mcp:
  octocode:
    command: "npx"
    args: ["octocode-mcp@latest"]
---

# Octocode: Research Driven Development

Semantic code research and context generation for REMOTE GitHub/GitLab repositories.

## Prerequisites
- GitHub authentication required (run `npx octocode-cli` to set up)

## When to Use
- Search code across public & private GitHub/GitLab repos
- Find real implementations and usage patterns
- Explore PRs, issues, and code reviews
- Research how libraries/frameworks are used in production
- Generate AI-optimized context from any accessible codebase

**Note**: This is for REMOTE repositories. For LOCAL codebase navigation, use `serena-tool` instead.

## Workflow
1. Use search tools to find relevant code/repos
2. Read specific files or code sections
3. Explore PRs and issues for context
4. Generate summaries for AI-optimized context

## Available Tools

### GitHub/GitLab Search
- `search_repositories` — Search repos by name, topic, or description
- `search_code` — Search code with semantic understanding
- `search_issues` — Search issues and discussions
- `search_prs` — Search pull requests

### Code Reading
- `read_file` — Read file contents from a repository
- `read_directory` — Browse directory structure
- `get_file_summary` — Get AI-optimized summary of a file

### PR & Issues
- `get_pr_details` — Get detailed PR information
- `get_pr_diff` — Get PR diff/changes
- `get_issue_details` — Get issue details and comments

## Tips
- Combine search + read for deep research
- Check PRs for implementation context and discussions
- Search for usage patterns across multiple repos
