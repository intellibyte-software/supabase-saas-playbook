---
name: skill-creator
description: Guide for creating effective skills that extend AI agent capabilities. Use when users want to create a new skill or update an existing skill with specialized knowledge, workflows, or tool integrations. Covers skill anatomy, progressive disclosure, degrees of freedom, and the 6-step creation process.
---

# Skill Creator

Guidance for creating effective skills that extend AI agent capabilities.

## About Skills

Skills are modular, self-contained packages that provide specialized knowledge, workflows, and tools. They transform a general-purpose agent into a specialized one equipped with procedural knowledge.

### What Skills Provide

1. **Specialized workflows** — multi-step procedures for specific domains
2. **Tool integrations** — instructions for working with file formats or APIs
3. **Domain expertise** — company-specific knowledge, schemas, business logic
4. **Bundled resources** — scripts, references, and assets for repetitive tasks

## Core Principles

### Concise is Key

The context window is shared with system prompt, conversation, other skills, and the user request. **Default assumption: the AI is already very smart.** Only add context it doesn't already have.

Prefer concise examples over verbose explanations.

### Set Appropriate Degrees of Freedom

Match specificity to task fragility:

| Freedom Level | When to Use | Examples |
|--------------|-------------|---------|
| **High** (text instructions) | Multiple approaches valid, context-dependent | Architecture guidance, code review |
| **Medium** (pseudocode/params) | Preferred pattern exists, some variation OK | API integration, component patterns |
| **Low** (specific scripts) | Fragile/error-prone, consistency critical | Migration scripts, deployment steps |

### Anatomy of a Skill

```
skill-name/
├── SKILL.md (required)
│   ├── YAML frontmatter (name, description)
│   └── Markdown instructions
└── Bundled Resources (optional)
    ├── scripts/          # Executable code (Python/Bash)
    ├── references/       # Documentation loaded on demand
    └── assets/           # Files used in output (templates, images)
```

#### SKILL.md (required)

- **Frontmatter** (YAML): `name` and `description` only. Description is the primary trigger mechanism — be clear about what the skill does AND when to use it.
- **Body** (Markdown): Instructions loaded only after the skill triggers.

#### Scripts (`scripts/`)

Executable code for tasks needing deterministic reliability or that are repeatedly rewritten.

#### References (`references/`)

Documentation loaded as needed. Keeps SKILL.md lean. For files > 10k words, include grep patterns in SKILL.md.

#### Assets (`assets/`)

Files used in output, not loaded into context (templates, images, boilerplate).

### What NOT to Include

- README.md, INSTALLATION_GUIDE.md, CHANGELOG.md
- Setup/testing procedures, user-facing documentation
- Auxiliary context about the creation process

## Progressive Disclosure

Three-level loading system:

1. **Metadata** (name + description) — always in context (~100 words)
2. **SKILL.md body** — when skill triggers (<5k words)
3. **Bundled resources** — as needed (unlimited)

**Keep SKILL.md under 500 lines.** Split content into reference files when approaching this limit.

### Pattern 1: High-level guide with references

```markdown
# Processing Tool

## Quick start
[core code example]

## Advanced features
- **Feature A**: See [FEATURE_A.md](references/feature_a.md)
- **API reference**: See [API.md](references/api.md)
```

### Pattern 2: Domain-specific organization

```
my-skill/
├── SKILL.md (overview + navigation)
└── references/
    ├── domain-a.md
    ├── domain-b.md
    └── domain-c.md
```

When user asks about domain B, the agent only reads `domain-b.md`.

### Pattern 3: Conditional details

```markdown
## Basic operation
Simple inline instructions.

**For advanced case**: See [ADVANCED.md](references/advanced.md)
```

**Guidelines:**
- Keep references one level deep from SKILL.md
- For files > 100 lines, include a table of contents

## Skill Creation Process

### Step 1: Understand with Concrete Examples

Understand how the skill will be used. Ask:
- "What functionality should this skill support?"
- "Can you give examples of how it would be used?"
- "What would a user say that should trigger this skill?"

### Step 2: Plan Reusable Contents

For each example, identify:
1. What code/logic gets rewritten repeatedly → `scripts/`
2. What documentation is referenced repeatedly → `references/`
3. What templates/assets are copied repeatedly → `assets/`

### Step 3: Initialize the Skill

Create the directory structure:

```bash
mkdir -p skill-name/{scripts,references,assets}
```

Create SKILL.md with frontmatter template:

```yaml
---
name: skill-name
description: [What it does]. Use when [triggers/contexts].
---
```

### Step 4: Edit the Skill

1. Implement scripts, references, and assets identified in Step 2
2. Test scripts by running them
3. Delete unused example directories
4. Write SKILL.md body with instructions

**Frontmatter rules:**
- `name`: The skill name
- `description`: Primary trigger mechanism. Include BOTH what the skill does AND when to use it
- No other fields in frontmatter

### Step 5: Validate and Package

Verify the skill:
- [ ] YAML frontmatter has `name` and `description`
- [ ] Description explains what AND when to use
- [ ] SKILL.md is under 500 lines
- [ ] Reference files are one level deep
- [ ] No unnecessary auxiliary files
- [ ] Scripts (if any) actually work

### Step 6: Iterate

1. Use the skill on real tasks
2. Notice struggles or inefficiencies
3. Update SKILL.md or resources
4. Test again
