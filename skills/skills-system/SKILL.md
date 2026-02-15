---
name: skills-system
description: How skills extend agent capabilities through markdown files - the loader hierarchy, skill structure, and on-demand loading pattern
---

# The Skills System

This skill describes how agents gain new capabilities through skills - markdown files that teach the agent how to do specific things, loaded on-demand and organized in a hierarchical structure.

## Core Concept

A **skill** is a markdown file that contains instructions for accomplishing a specific type of task. Skills are:
- Written in natural language (not code)
- Loaded into context when needed
- Organized in directories by name
- Layered: workspace → global → builtin

Think of skills as "instruction manuals" the agent can read.

## Skill Structure

Each skill lives in its own directory with a `SKILL.md` file:

```
skills/
  github/
    SKILL.md
  docker/
    SKILL.md
  memory-management/
    SKILL.md
```

### The SKILL.md File

```markdown
---
name: github
description: How to work with GitHub repositories, issues, and pull requests
---

# GitHub Skill

Instructions for the agent on how to work with GitHub...

## Creating Issues
...

## Making Pull Requests
...
```

### Frontmatter

The YAML frontmatter is required:

```yaml
---
name: github
description: How to work with GitHub repositories, issues, and pull requests
---
```

- **name**: Unique identifier (alphanumeric with hyphens, max 64 chars)
- **description**: Brief explanation of what the skill covers (max 1024 chars)

### Content

After the frontmatter, write instructions in markdown. This is what the agent reads to learn the skill.

## The Skill Hierarchy

Skills are loaded from three locations, in priority order:

### 1. Workspace Skills (highest priority)
```
workspace/skills/{skill-name}/SKILL.md
```
Project-specific skills. Override global and builtin.

### 2. Global Skills
```
~/.picoclaw/skills/{skill-name}/SKILL.md
```
User's personal skills. Override builtin.

### 3. Builtin Skills (lowest priority)
```
{install-dir}/skills/{skill-name}/SKILL.md
```
Shipped with the agent. Default behaviors.

### Override Example
```
builtin: skills/git/SKILL.md    → "Use conventional commits"
global:  ~/.picoclaw/skills/git/SKILL.md → "Use emoji commits"
workspace: workspace/skills/git/SKILL.md → "Use ticket numbers"

Result: Workspace wins → "Use ticket numbers"
```

This allows customization at each level without modifying defaults.

## Skill Loading: On-Demand Pattern

Skills are NOT all loaded into the system prompt. Instead:

### 1. Summary in System Prompt
The agent sees a list of available skills:
```xml
<skills>
  <skill>
    <name>github</name>
    <description>How to work with GitHub repositories</description>
    <location>/workspace/skills/github/SKILL.md</location>
    <source>workspace</source>
  </skill>
  <skill>
    <name>docker</name>
    <description>Container management with Docker</description>
    <location>~/.picoclaw/skills/docker/SKILL.md</location>
    <source>global</source>
  </skill>
</skills>
```

### 2. Agent Reads When Needed
When the agent needs a skill, it uses `read_file`:
```
Agent: "I need to create a GitHub PR. Let me check my GitHub skill."
→ read_file("/workspace/skills/github/SKILL.md")
→ "# GitHub Skill\n\n## Creating Pull Requests\n..."
→ Agent follows the instructions
```

### Why On-Demand?

**Token efficiency**: 
- 10 skills × 500 tokens each = 5000 tokens always in context
- On-demand: ~200 token summary + only load what's needed

**Focus**:
- The agent only reads relevant skills
- Less noise in the context

**Scalability**:
- Can have dozens of skills without context overflow

## Skill Discovery

The loader scans each skill directory:

### Validation Rules
- Name: alphanumeric with hyphens only
- Name: max 64 characters
- Description: required
- Description: max 1024 characters
- Must have `SKILL.md` file

### Invalid Skills
Invalid skills are logged and skipped:
```
WARN: invalid skill from workspace: name exceeds 64 characters
```

### Listing Skills
```
ListSkills() → [
  {Name: "github", Path: "...", Source: "workspace", Description: "..."},
  {Name: "docker", Path: "...", Source: "global", Description: "..."},
]
```

## Writing Effective Skills

### 1. Clear Structure
Use headers to organize:
```markdown
# Skill Name

Brief introduction.

## When to Use This Skill

## Step-by-Step Process

## Examples

## Common Mistakes
```

### 2. Actionable Instructions
Tell the agent what TO DO:
```markdown
## Creating a PR

1. First, check you're on a feature branch
2. Run: git push -u origin HEAD
3. Use: gh pr create --title "..." --body "..."
```

### 3. Include Tool Usage
Reference the tools the agent should use:
```markdown
## Checking Logs

Use the `exec` tool to run:
exec("docker logs container_name --tail 100")
```

### 4. Provide Context
Explain WHY, not just WHAT:
```markdown
## Commit Messages

Use conventional commits because:
- Automated changelog generation
- Clear history
- Semantic versioning support

Format: type(scope): description
```

### 5. Handle Edge Cases
```markdown
## If PR Conflicts

1. Fetch latest: git fetch origin main
2. Rebase: git rebase origin/main
3. Resolve conflicts manually
4. Force push: git push --force-with-lease
```

## Skill Invocation Patterns

### Explicit Request
User: "Use the github skill to create a PR"
→ Agent reads github skill, follows instructions

### Implicit Recognition
User: "Create a pull request for this change"
→ Agent recognizes this needs GitHub skill
→ Reads skill, follows instructions

### Agent Initiative
Agent: "I should check my deployment skill before pushing to production"
→ Reads deployment skill proactively

## Skills vs System Prompt

| Aspect | System Prompt | Skills |
|--------|---------------|--------|
| Always loaded | Yes | No (on-demand) |
| Token cost | Constant | Only when used |
| Content type | Identity, rules | How-to instructions |
| Update | Requires restart | Live (just edit file) |
| Scope | Everything | Specific tasks |

## Skills vs Memory

| Aspect | Skills | Memory |
|--------|--------|--------|
| Content | Instructions | Facts |
| Source | Developer/user | Learned from interaction |
| Mutability | Edited by human | Updated by agent |
| Purpose | How to do things | What to remember |

## Organizing Skills

### By Domain
```
skills/
  git/
  github/
  docker/
  kubernetes/
```

### By Task Type
```
skills/
  code-review/
  debugging/
  deployment/
  documentation/
```

### By Project
```
skills/
  myapp-deploy/
  myapp-testing/
  myapp-monitoring/
```

## Skill Metadata Access

The loader extracts metadata from frontmatter:

### JSON Format (legacy)
```markdown
---
{"name": "github", "description": "..."}
---
```

### YAML Format (preferred)
```markdown
---
name: github
description: How to work with GitHub
---
```

Both are supported. YAML is cleaner.

## Best Practices

### 1. One Skill, One Focus
```
Good: skills/github-pr/      → Just PR workflows
Bad:  skills/github-everything/ → Too broad
```

### 2. Descriptive Names
```
Good: kubernetes-debugging
Bad:  k8s
```

### 3. Keep Skills Focused
500-1000 tokens is a good target. If longer, consider splitting.

### 4. Test Your Skills
Ask the agent to use the skill. Does it work? Iterate.

### 5. Version Control Skills
Skills are just files. Put them in git:
```
git add skills/new-skill/SKILL.md
git commit -m "Add new-skill for X"
```

### 6. Document Assumptions
```markdown
## Prerequisites

This skill assumes:
- Docker is installed
- User has access to container registry
- AWS CLI is configured
```

## Example Skill

```markdown
---
name: code-review
description: How to review code changes and provide constructive feedback
---

# Code Review Skill

## When to Use

Use this skill when asked to review code, PRs, or diffs.

## Review Process

1. **Understand the context**
   - Read the PR description
   - Understand the goal of the change

2. **Check the basics**
   - Does it compile/lint?
   - Are there tests?
   - Is documentation updated?

3. **Review the logic**
   - Is the approach sound?
   - Are there edge cases?
   - Is error handling appropriate?

4. **Provide feedback**
   - Be specific: line numbers, examples
   - Be constructive: suggest improvements
   - Be kind: assume good intent

## Feedback Format

Use this structure:
- **Must fix**: Bugs, security issues
- **Should fix**: Performance, maintainability
- **Consider**: Style, alternatives

## Common Issues to Watch For

- Hardcoded values that should be config
- Missing error handling
- N+1 query patterns
- Secrets in code
- Missing input validation
```
