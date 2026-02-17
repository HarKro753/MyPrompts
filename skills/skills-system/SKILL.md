---
name: skills-system
description: How skills extend agent capabilities through markdown files - the loader hierarchy, skill structure, and on-demand loading pattern. Use when creating skills, debugging skill loading, or understanding how agents learn new capabilities.
---

# The Skills System

Skills are markdown files that teach the agent how to accomplish specific tasks. They're loaded on-demand, organized hierarchically, and written in natural language rather than code.

## Quick start

Create a minimal skill:

```bash
mkdir -p skills/my-skill
cat > skills/my-skill/SKILL.md << 'EOF'
---
name: my-skill
description: Does X when the user asks about Y. Use when working with Y or Z.
---

# My Skill

Brief overview.

## Quick start
Simplest example here.

## Instructions
1. First step
2. Second step
3. Third step

## Examples
Concrete examples with code.

## Best practices
- Do this
- Don't do that
EOF
```

## Instructions

### Step 1: Choose skill location

Skills live in one of three locations (priority order):

| Location | Path | Use Case |
|----------|------|----------|
| Workspace | `workspace/skills/` | Project-specific |
| Global | `~/.agent/skills/` | Personal, cross-project |
| Builtin | `{install}/skills/` | Shipped defaults |

Higher priority overrides lower. Workspace skills override global and builtin.

### Step 2: Create the skill directory

```bash
mkdir -p skills/skill-name
```

Directory name must match the `name` in frontmatter.

### Step 3: Write SKILL.md with frontmatter

```yaml
---
name: skill-name
description: What it does. Use when X, Y, or Z.
---
```

**Name rules:**
- Lowercase letters, numbers, hyphens only
- Max 64 characters
- Must match directory name

**Description rules:**
- Max 1024 characters
- Include WHAT it does AND WHEN to use it
- Add trigger words users might say

### Step 4: Structure the content

```markdown
# Skill Name

Brief overview (1-2 sentences).

## Quick start
Simplest possible example to get started.

## Instructions
Step-by-step guidance:
1. First step
2. Second step
3. Third step

## Examples
Concrete usage with real code/commands.

## Best practices
- Key conventions
- Common pitfalls to avoid

## Requirements
- Dependencies
- Prerequisites
```

### Step 5: Test the skill

1. Restart agent (or reload skills)
2. Ask a relevant question
3. Verify skill is discovered and loaded
4. Check that instructions are followed

## Examples

### Example 1: Simple skill

```markdown
---
name: git-commits
description: How to write good git commits. Use when committing code or asking about commit messages.
---

# Git Commits

Write clear, conventional commit messages.

## Quick start

```bash
git commit -m "feat: add user authentication"
```

## Instructions

1. Use conventional commit format: `type(scope): description`
2. Types: feat, fix, docs, style, refactor, test, chore
3. Keep subject line under 50 characters
4. Use imperative mood ("add" not "added")

## Examples

```bash
feat(auth): add JWT token validation
fix(api): handle null response from server
docs: update README with setup instructions
```

## Best practices
- One logical change per commit
- Reference issue numbers when applicable
- Don't commit generated files
```

### Example 2: Skill with multiple files

```
skills/
  api-design/
    SKILL.md           # Main instructions
    reference.md       # Detailed API patterns
    examples/
      rest-api.md      # REST examples
      graphql.md       # GraphQL examples
```

Reference additional files from SKILL.md:
```markdown
For detailed patterns, see [reference.md](reference.md).
For REST examples, see [examples/rest-api.md](examples/rest-api.md).
```

### Example 3: Read-only skill with tool restrictions

```yaml
---
name: code-reviewer
description: Review code without making changes. Use for code review requests.
allowed-tools: Read, Grep, Glob
---
```

## Best practices

### Skill design

- **One skill, one focus** - Don't create mega-skills
- **Specific descriptions** - Include trigger words users will say
- **Clear instructions** - Write for the agent, not humans
- **Concrete examples** - Real code, not pseudocode
- **Progressive disclosure** - Basic info in SKILL.md, details in reference files

### Naming

- **Good**: `kubernetes-debugging`, `api-design`, `git-commits`
- **Bad**: `k8s`, `misc-utils`, `stuff`

### Content size

- **Target**: 300-800 tokens per skill
- **If larger**: Split into multiple files or separate skills

### Discovery

Skills are NOT loaded into context by default. The agent sees a summary:

```xml
<skills>
  <skill>
    <name>git-commits</name>
    <description>How to write good git commits...</description>
    <location>skills/git-commits/SKILL.md</location>
  </skill>
</skills>
```

When the agent needs a skill, it reads the file with `read_file`.

### Testing

- Ask questions that should trigger the skill
- Verify the agent reads and follows instructions
- Check edge cases are handled

## Requirements

### Directory structure

```
skills/
  skill-name/
    SKILL.md          # Required: main instructions
    reference.md      # Optional: detailed docs
    examples.md       # Optional: extended examples
    scripts/          # Optional: helper scripts
    templates/        # Optional: boilerplate
```

### Frontmatter format

```yaml
---
name: skill-name          # Required
description: What and when # Required
allowed-tools: Read, Glob  # Optional: restrict tools
---
```

### Validation rules

| Field | Rule |
|-------|------|
| name | Lowercase, hyphens, numbers only |
| name | Max 64 characters |
| name | Must match directory name |
| description | Max 1024 characters |
| description | Should include "Use when..." |

### Loading hierarchy

```
Workspace skills (highest priority)
    ↓ overrides
Global skills
    ↓ overrides
Builtin skills (lowest priority)
```
