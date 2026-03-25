---
name: skill-creator
description: Create, modify, and improve Kilo Code skills. Use when the user asks to create a new skill, update an existing skill, scaffold a skill structure, or needs help writing SKILL.md files. Triggers on keywords like "create skill", "new skill", "build skill", "scaffold skill", "write skill", "skill template".
---

# Skill Creator

You are a Kilo Code Skill Creator — an expert at designing, scaffolding, and writing high-quality Agent Skills for Kilo Code.

## What is a Kilo Code Skill?

A skill is a folder containing a `SKILL.md` file (and optionally bundled resources) that teaches the agent how to perform a specific task. Skills are loaded on-demand when a task matches the skill's description.

## Skill Directory Locations

### Global Skills (apply to all projects)
- **Generic (all modes):** `~/.kilocode/skills/<skill-name>/SKILL.md`
- **Mode-specific:** `~/.kilocode/skills-<mode-slug>/<skill-name>/SKILL.md`

### Project Skills (apply to current project only)
- **Generic (all modes):** `.kilocode/skills/<skill-name>/SKILL.md`
- **Mode-specific:** `.kilocode/skills-<mode-slug>/<skill-name>/SKILL.md`

### Mode Slugs
Common mode slugs: `code`, `architect`, `debug`, `ask`, `orchestrator`

## SKILL.md Format

Every `SKILL.md` MUST follow this structure:

```markdown
---
name: <skill-name>
description: <Brief description of what the skill does and WHEN to use it. Include trigger keywords.>
---

# <Skill Title>

<Detailed instructions for the AI agent>
```

### Frontmatter Rules
- `name` (required): Must match the parent folder name exactly. Use kebab-case.
- `description` (required): Concise but descriptive. Include trigger keywords/phrases so the agent knows when to activate this skill. Max ~200 words.

## Skill Creation Workflow

When the user asks to create a skill, follow these steps:

### Step 1: Gather Requirements
Ask the user (if not already provided):
1. **What should the skill do?** — The core capability or workflow
2. **When should it activate?** — Trigger keywords and scenarios
3. **Which scope?** — Global (`~/.kilocode/skills/`) or project-level (`.kilocode/skills/`)
4. **Mode-specific?** — All modes or specific mode (code, architect, debug, etc.)
5. **Does it need bundled resources?** — Scripts, templates, references

### Step 2: Design the Skill
- Choose a clear, descriptive kebab-case name
- Write a precise `description` with trigger keywords
- Plan the instruction structure
- Determine if bundled resources (scripts, templates, references) are needed

### Step 3: Create the Skill Structure

#### Minimal skill (instructions only):
```
<skill-name>/
└── SKILL.md
```

#### Full skill (with bundled resources):
```
<skill-name>/
├── SKILL.md           # Required: instructions + metadata
├── scripts/           # Optional: executable code
│   └── main.py
├── references/        # Optional: documentation, examples
│   └── api-docs.md
└── templates/         # Optional: file templates
    └── component.tsx
```

### Step 4: Write the SKILL.md
Follow these best practices:

1. **Clear trigger description**: The `description` field determines when the agent loads this skill. Be specific about scenarios and include keywords users might say.

2. **Structured instructions**: Use Markdown headers, lists, and code blocks. Organize logically.

3. **Actionable steps**: Write instructions as concrete steps the agent should follow, not vague guidelines.

4. **Include examples**: Show input/output examples so the agent understands expected behavior.

5. **Reference bundled files**: If using scripts/references/templates, tell the agent exactly when and how to use them with relative paths like `./scripts/main.py`.

6. **Keep it focused**: One skill = one responsibility. Don't create a "do everything" skill.

7. **Add guardrails**: Include "DO NOT" rules if there are common mistakes to avoid.

### Step 5: Verify
- Confirm the `name` field matches the folder name
- Confirm the `description` includes relevant trigger keywords
- Confirm the skill is placed in the correct directory
- Test by asking the agent to perform the skill's task

## Writing Good Descriptions (Critical)

The `description` is the MOST important part — it determines if the skill gets loaded. Good descriptions:

✅ **Good:**
```
description: Generate unit tests for TypeScript/JavaScript code. Use when asked to "write tests", "add tests", "create test file", "test this function", or "generate test coverage".
```

❌ **Bad:**
```
description: Testing stuff
```

### Description Formula
```
<What it does>. Use when <scenarios/triggers>. Keywords: <comma-separated trigger words>.
```

## Modifying Existing Skills

When asked to modify a skill:
1. Read the existing `SKILL.md` first
2. Understand the current structure and purpose
3. Make targeted changes — don't rewrite unnecessarily
4. Preserve existing functionality unless explicitly asked to change it

## Common Skill Patterns

### Workflow Skill
Defines a multi-step process (e.g., "PR review checklist", "deployment workflow")

### Standards Skill
Enforces coding/design standards (e.g., "API design patterns", "naming conventions")

### Generator Skill
Creates files from templates (e.g., "React component generator", "API endpoint scaffolder")

### Integration Skill
Teaches the agent to work with specific tools/services (e.g., "Docker workflow", "AWS deployment")

## Priority Rules
When multiple skills share the same name:
1. **Project-level overrides global** — `.kilocode/skills/` > `~/.kilocode/skills/`
2. **Mode-specific overrides generic** — `skills-code/` > `skills/` when in Code mode

## Anti-Patterns to Avoid
- Don't create skills that duplicate built-in Kilo Code behavior
- Don't make skills too broad — split into focused skills instead
- Don't hardcode paths or environment-specific values
- Don't put sensitive data (API keys, passwords) in skills
- Don't create overly long skills — use `references/` for large docs
