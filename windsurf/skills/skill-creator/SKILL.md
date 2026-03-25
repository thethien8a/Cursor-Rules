---
name: skill-creator
description: Create, modify, and improve Windsurf skills. Use when the user asks to create a new skill, update an existing skill, scaffold a skill structure, or needs help writing SKILL.md files. Triggers on "create skill", "new skill", "build skill", "scaffold skill", "write skill", "skill template".
---

# Skill Creator

You are a Windsurf Skill Creator — an expert at designing, scaffolding, and writing high-quality Agent Skills for Windsurf's Cascade AI.

## What is a Windsurf Skill?

A skill is a folder containing a `SKILL.md` file (and optionally supporting resource files) that teaches Cascade how to perform a specific task. Cascade uses **progressive disclosure**: only the skill's `name` and `description` are shown to the model by default. The full `SKILL.md` content and supporting files are loaded only when Cascade decides to invoke the skill (or when the user `@mentions` it).

## Skill Directory Locations

### Workspace Skills (project-specific, committed with repo)
```
<project-root>/.windsurf/skills/<skill-name>/SKILL.md
```

### Global Skills (available in all workspaces, not committed)
```
~/.codeium/windsurf/skills/<skill-name>/SKILL.md
```

### Cross-Agent Compatible Locations (also discovered by Windsurf)
```
~/.agents/skills/<skill-name>/SKILL.md        # Cross-agent global
<project-root>/.agents/skills/<skill-name>/SKILL.md  # Cross-agent workspace
```

### System-Level Skills (Enterprise, read-only)
| OS | Path |
|---|---|
| macOS | `/Library/Application Support/Windsurf/skills/` |
| Linux/WSL | `/etc/windsurf/skills/` |
| Windows | `C:\ProgramData\Windsurf\skills\` |

## SKILL.md File Format

Every `SKILL.md` MUST follow this exact structure:

```markdown
---
name: <skill-name>
description: <Brief description — helps Cascade decide when to invoke this skill>
---

# <Skill Title>

<Detailed instructions for Cascade>
```

### Required Frontmatter Fields

| Field | Rules |
|---|---|
| `name` | Unique identifier. Lowercase letters, numbers, and hyphens only. Displayed in UI and used for `@mentions`. |
| `description` | Brief explanation shown to the model. This is the MOST critical field — it determines whether Cascade auto-invokes the skill. |

**No other frontmatter fields are supported.** Windsurf skills use only `name` and `description`.

## Skill Creation Workflow

When the user asks to create a skill, follow these steps:

### Step 1: Gather Requirements
Ask the user (if not already provided):
1. **What should the skill do?** — The core capability or workflow
2. **When should it activate?** — Trigger scenarios and keywords
3. **Which scope?** — Workspace (`.windsurf/skills/`) or Global (`~/.codeium/windsurf/skills/`)
4. **Does it need supporting files?** — Scripts, templates, checklists, configs

### Step 2: Design the Skill
- Choose a clear, descriptive name using only lowercase letters, numbers, and hyphens
- Write a precise `description` that helps Cascade understand when to invoke it
- Plan the instruction structure with clear steps
- Determine if supporting resource files are needed

### Step 3: Create the Skill Structure

#### Minimal skill (instructions only):
```
<skill-name>/
└── SKILL.md
```

#### Full skill (with supporting resources):
```
<skill-name>/
├── SKILL.md              # Required: metadata + instructions
├── checklist.md           # Optional: step-by-step checklist
├── template.tsx           # Optional: file template
├── config-template.yaml   # Optional: config template
└── reference-docs.md      # Optional: reference documentation
```

Supporting files are placed directly alongside `SKILL.md` in the same folder — no need for subdirectories like `scripts/` or `references/` (though you can use them if you prefer).

### Step 4: Write the SKILL.md

Follow these best practices:

1. **Clear trigger description**: The `description` field determines when Cascade auto-invokes this skill. Be specific about scenarios and include keywords users might say.

2. **Structured instructions**: Use Markdown headers, numbered lists, and code blocks. Organize logically into phases/steps.

3. **Actionable steps**: Write concrete steps Cascade should follow, not vague guidelines.

4. **Include examples**: Show input/output examples so Cascade understands expected behavior.

5. **Reference supporting files**: If using resource files, tell Cascade exactly when and how to use them. Example: "Read `./checklist.md` before starting the deployment."

6. **Keep it focused**: One skill = one responsibility. Don't create a "do everything" skill.

7. **Add guardrails**: Include "DO NOT" rules to prevent common mistakes.

### Step 5: Verify
- Confirm the `name` field uses only lowercase letters, numbers, and hyphens
- Confirm the `description` clearly explains when the skill should be invoked
- Confirm the skill is placed in the correct directory
- Test by typing `@skill-name` in Cascade or by describing the task naturally

## Writing Good Descriptions (Critical)

The `description` determines if Cascade auto-invokes the skill. This is the single most important thing to get right.

### Description Formula
```
<What the skill does>. <When to use it / trigger scenarios>.
```

### Examples

✅ **Good:**
```yaml
description: Guides the deployment process to production with safety checks. Use when deploying code, releasing to production, or running pre-deploy verification.
```

✅ **Good:**
```yaml
description: Generate unit tests for TypeScript and JavaScript code. Use when asked to write tests, add test coverage, or create test files.
```

❌ **Bad:**
```yaml
description: Testing stuff
```

❌ **Bad:**
```yaml
description: A skill for deployment
```

## Invocation Methods

Skills can be invoked in two ways:

### 1. Automatic Invocation (preferred)
When the user's request matches a skill's `description`, Cascade automatically invokes the skill. No action needed from the user.

### 2. Manual Invocation via @mention
User types `@skill-name` in the Cascade input to explicitly activate a skill. Useful when:
- You want to ensure a specific skill is used
- The skill might not be automatically triggered by the request

## Modifying Existing Skills

When asked to modify a skill:
1. Read the existing `SKILL.md` first
2. Understand the current structure and purpose
3. Make targeted changes — don't rewrite unnecessarily
4. Preserve existing functionality unless explicitly asked to change it
5. If supporting files exist, check them too before making changes

## Skills vs Rules vs Workflows

Before creating a skill, determine if the user actually needs a skill, rule, or workflow:

| | Skills | Rules | Workflows |
|---|---|---|---|
| **Purpose** | Multi-step procedures with supporting files | Behavioral guidelines | Prompt templates for repeatable tasks |
| **Structure** | Folder with `SKILL.md` + resource files | Single `.md` file with frontmatter | Single `.md` file |
| **Invocation** | Auto by Cascade or `@mention` | `always_on` / `glob` / `model_decision` / `manual` | Manual only via `/slash-command` |
| **Best for** | Deployments, code review, testing with scripts/templates | Coding style, project conventions | One-shot runbooks triggered explicitly |

**Rule of thumb:**
- Needs supporting files + auto-invocation → **Skill**
- Short behavioral constraint → **Rule** (in `.windsurf/rules/`)
- Always manually triggered → **Workflow** (in `.windsurf/workflows/`)

If the user asks to create something that fits better as a Rule or Workflow, suggest the appropriate alternative and explain why.

## Common Skill Patterns

### Deployment Skill
Multi-step deployment process with checklists, environment configs, rollback procedures.

### Code Review Skill
Style guides, security checklists, review templates bundled together.

### Testing Skill
Test templates, coverage requirements, CI/CD configs.

### Generator Skill
Creates files from templates (components, API endpoints, modules).

### Standards Skill
Enforces coding/design standards with reference documentation.

## Anti-Patterns to Avoid
- Don't create skills for simple behavioral rules — use Rules instead
- Don't create skills that are always manually triggered — use Workflows instead
- Don't make skills too broad — split into focused skills
- Don't hardcode absolute paths or environment-specific values
- Don't put sensitive data (API keys, passwords) in skills
- Don't exceed reasonable length — use supporting files for large docs
- Don't duplicate built-in Cascade behavior
