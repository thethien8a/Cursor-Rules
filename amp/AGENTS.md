# Global Rules

## CRITICAL — Skills First (MUST follow)
BEFORE doing ANY work, ALWAYS load the matching skill FIRST (Skill must be loaded if valid):
- ANY code reading/searching/editing task → load `serena-code` skill FIRST, then use ONLY serena MCP tools
- Web search needed → load `exa-search` skill FIRST
- Python coding task → load `python-clean-code` skill FIRST, read it, then code
- Docker task → load `docker-expert` skill FIRST

## Serena — MANDATORY for all code operations
**HARD RULE: After loading serena-code, NEVER call Read/Grep/glob/finder/edit_file for code files. Violations = broken rule.**

Startup sequence (NO EXCEPTIONS):
1. `check_onboarding_performed` → if not done → `onboarding` FIRST
2. Then use ONLY these for code:
   - Understand code → `mcp__serena__get_symbols_overview` + `mcp__serena__find_symbol`
   - Search code → `mcp__serena__find_referencing_symbols` / `mcp__serena__search_for_pattern`
   - Edit code → `mcp__serena__replace_content`
   - Find files → `mcp__serena__find_file` / `mcp__serena__list_dir`
   - Navigate → `mcp__serena__find_symbol` with name_path_pattern

Fallback to built-in tools ONLY if Serena MCP server fails to respond (error/timeout). 
Not using 'read-file' tool if serena tool is still can use and valid for this task (To optimize tokens)

## Response Format
1. Start with "YOOO!"
2. Mirror user's language
3. Classify: ASK (explain) vs EDIT (code changes)
4. If unsure → search first, never fabricate
5. If ambiguous/unclear → ask to clarify before acting

## Information Quality
1. Search for anything you're unsure about. Only say "I don't know" AFTER searching confirms no info exists
2. For problems with multiple solutions: list ALL with pros/cons, then recommend best suitable.
3. Include best practices when applicable or real-world solutions many people do if available.
4. Provide relevant source URLs
5. I need all the newest/up-to-date information, so please search for internet information after year 2025

## Coding Rules
1. Never use emoji in code
2. Comments: explain WHY, not WHAT — never restate code