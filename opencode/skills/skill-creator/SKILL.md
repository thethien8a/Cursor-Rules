---
name: skill-creator
description: Guide for creating effective OpenCode skills. Use when user wants to create, update, or improve a SKILL.md file. Covers frontmatter format, naming rules, content structure, progressive disclosure, token optimization, and validation.
---

# Skill Creator

Create effective, token-efficient OpenCode skills that follow official conventions.

## Skill File Structure

```
skill-name/
├── SKILL.md              (required — main instructions)
├── scripts/              (optional — executable code, not loaded to context)
├── references/           (optional — docs loaded on-demand when needed)
└── assets/               (optional — files used in output, never loaded)
```

Only create `SKILL.md`. Do NOT create README.md, CHANGELOG.md, INSTALLATION_GUIDE.md, or any auxiliary files.

## Placement Locations

| Scope | Path |
|-------|------|
| Project | `.opencode/skills/<name>/SKILL.md` |
| Global | `~/.config/opencode/skills/<name>/SKILL.md` |
| Claude-compat (project) | `.claude/skills/<name>/SKILL.md` |
| Claude-compat (global) | `~/.claude/skills/<name>/SKILL.md` |
| Agent-compat (project) | `.agents/skills/<name>/SKILL.md` |
| Agent-compat (global) | `~/.agents/skills/<name>/SKILL.md` |

## Naming Rules (Strict)

Skill `name` must:
- Be 1-64 characters
- Lowercase alphanumeric with single hyphen separators
- Not start/end with `-`, no consecutive `--`
- Match the containing directory name exactly
- Regex: `^[a-z0-9]+(-[a-z0-9]+)*$`

Valid: `clean-code`, `pr-review`, `tdd`, `api-generator`
Invalid: `Clean-Code`, `--review`, `my--skill`, `skill_name`

## Frontmatter Format

```yaml
---
name: skill-name
description: What the skill does AND when to trigger it. Be specific with trigger scenarios.
license: MIT                          # optional
compatibility: opencode               # optional
metadata:                             # optional, string-to-string map
  audience: developers
  language: python
---
```

### Required Fields

| Field | Rules |
|-------|-------|
| `name` | Must match directory name, follow naming rules above |
| `description` | 1-1024 chars. Must answer: "What does it do?" AND "When should agent load it?" |

### Writing Effective Descriptions

The description is the ONLY thing the agent reads before deciding to load a skill. All trigger context goes here — NOT in the body.

**Pattern:** `[What it does]. Use when [trigger scenarios]. Covers [key topics].`

Good:
```yaml
description: Clean code guidelines covering naming, functions, SOLID, and Python tooling. Use when writing, reviewing, or refactoring code.
```

```yaml
description: Review PRs with the rigor of a senior staff engineer - catches race conditions, API contract changes, security issues, and architectural problems others miss
```

Bad:
```yaml
description: A skill for code stuff    # too vague, agent can't decide when to trigger
description: Use this skill            # says nothing about what or when
```

### Anti-trigger tip
Add negative triggers when needed:
```yaml
description: "...Do NOT use for simple single-file edits."
```

## Body Structure

Write in imperative form. Challenge every paragraph: "Does this justify its token cost?"

### Recommended Sections

```markdown
# Skill Title

One-line summary of purpose.

## When to use me              (optional — reinforces triggers)
- Bullet list of scenarios

## Core Rules / Quick Rules     (the main content)
- Tables, bullet points, concise rules
- Code examples only when they add value the rule text can't

## Process / Workflow           (if skill is procedural)
1. Step-by-step numbered instructions
2. Include concrete commands where applicable

## Quick Reference              (optional — scannable summary)
- Checklists, decision tables, do/don't comparisons
```

### Content Guidelines

| Do | Don't |
|----|-------|
| Use tables for structured data | Write long prose paragraphs |
| Use bullet points for rules | Repeat what code already says |
| Show 1-2 concise examples per concept | Include 5+ examples per concept |
| Write for another AI instance | Write for human onboarding |
| Use imperative form ("Read the file") | Use passive ("The file should be read") |
| Include decision rules ("When X, do Y") | Leave decisions ambiguous |
| Add concrete commands for processes | Describe processes abstractly |

## Progressive Disclosure (3 Levels)

```
Level 1: Metadata (name + description)    → always in context (~100 words)
Level 2: SKILL.md body                    → loaded when skill triggers (<500 lines)
Level 3: references/ files                → loaded only when agent needs them
```

### Rules
- Keep SKILL.md body under 500 lines — split heavy content to `references/`
- Reference files should be one level deep (no deep nesting)
- For reference files >100 lines, include a table of contents
- Information lives in SKILL.md OR references — never both

### Splitting Pattern

```markdown
## Advanced Features
For detailed API reference, read `references/api-docs.md`
For schema definitions, read `references/schema.md`
```

## Degrees of Freedom

Match specificity to how critical the task is:

| Freedom | Format | When |
|---------|--------|------|
| **High** | Text instructions | Multiple valid approaches, context-dependent |
| **Medium** | Pseudocode / parameterized scripts | Preferred pattern exists, some variation OK |
| **Low** | Exact scripts, few params | Sequence is critical, errors are expensive |

## Token Optimization Checklist

- [ ] Body under 500 lines?
- [ ] No redundant examples? (1-2 per concept max)
- [ ] Tables instead of prose where possible?
- [ ] Heavy reference material in `references/` not SKILL.md?
- [ ] No content the agent already knows? (common language syntax, basic concepts)
- [ ] Imperative form throughout? (saves tokens vs passive voice)

## Creation Process

### Step 1: Gather Requirements
Ask the user:
- What should the skill do?
- What triggers should load it?
- What are 2-3 concrete usage examples?

### Step 2: Plan Content
For each example, determine:
- What instructions does the agent need?
- What scripts/references/assets would help?
- What does the agent already know (skip those)?

### Step 3: Create Skill
1. Create directory: `~/.config/opencode/skills/<name>/`
2. Create `SKILL.md` with frontmatter + body
3. Optionally create `scripts/`, `references/`, `assets/` subdirs

### Step 4: Validate
Check these before finishing:
- [ ] `SKILL.md` filename is uppercase
- [ ] Frontmatter has `name` and `description`
- [ ] `name` matches directory name
- [ ] `name` passes regex `^[a-z0-9]+(-[a-z0-9]+)*$`
- [ ] `description` is 1-1024 chars with clear triggers
- [ ] Body is under 500 lines
- [ ] No extraneous files (README.md, etc.)
- [ ] No TODO placeholders left

### Step 5: Iterate
After real-world use, refine based on:
- Where did the agent struggle?
- What instructions were unclear?
- What content was never used? (remove it)

## Template

Use this as starting point when creating a new skill:

```markdown
---
name: <skill-name>
description: <What it does>. Use when <trigger scenarios>. Covers <key topics>.
---

# <Skill Title>

<One-line purpose summary.>

## When to use me
- <Scenario 1>
- <Scenario 2>

## Core Rules

### <Topic 1>
| Column A | Column B |
|----------|----------|
| Rule 1   | Detail   |

### <Topic 2>
- Rule as bullet point
- Another rule

## Quick Reference
- [ ] Checklist item 1
- [ ] Checklist item 2
```

## Permissions (Optional)

Control skill access in `opencode.json`:

```json
{
  "permission": {
    "skill": {
      "*": "allow",
      "internal-*": "deny",
      "experimental-*": "ask"
    }
  }
}
```

| Permission | Behavior |
|------------|----------|
| `allow` | Loads immediately |
| `deny` | Hidden from agent |
| `ask` | User prompted before loading |
