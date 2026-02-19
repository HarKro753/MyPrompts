---
name: system-prompt
description: How to design effective system prompts for agentic AI - what to include, what to avoid, and how to structure for reliable tool use. Use when writing system prompts, debugging agent behavior, or setting up new AI assistants.
---

# System Prompt Design

The system prompt is the DNA of your agent. It defines who the agent is, what it can do, how it should behave, and what rules it must follow. A well-designed system prompt makes the difference between an agent that works and one that just talks about working.

## Quick start

Minimal effective system prompt:

```markdown
# MyAgent

You are MyAgent, a helpful assistant.

## Current Time
2026-02-15 14:30 (Saturday)

## Workspace
/home/user/workspace

## Tools
**YOU MUST USE TOOLS TO ACT. DO NOT PRETEND.**

- `read_file` - Read file contents
- `write_file` - Write to file
- `exec` - Run shell command

## Rules
1. Use tools for all actions
2. Explain briefly what you're doing
3. Ask if unclear
```

This is ~150 tokens and covers all essentials.

## Instructions

### Step 1: Write the identity header

Start with a clear statement:

```markdown
# AgentName

You are AgentName, a helpful AI assistant.
```

Keep it to 1-2 lines. Don't write a biography.

### Step 2: Add temporal context

Always include the current time:

```markdown
## Current Time
2026-02-15 14:30 (Saturday)
```

Use unambiguous format (YYYY-MM-DD). LLMs don't know what time it is otherwise.

### Step 3: Define the runtime environment

Tell the agent where it's running:

```markdown
## Runtime
darwin arm64

## Workspace
/Users/alex/my-project
- Memory: /Users/alex/my-project/memory/MEMORY.md
- Skills: /Users/alex/my-project/skills/
```

### Step 4: List available tools

Summarize tools with a critical warning:

```markdown
## Tools

**CRITICAL**: You MUST use tools to perform actions. Do NOT pretend to execute commands.

- `read_file` - Read the contents of a file
- `write_file` - Write content to a file
- `exec` - Execute a shell command
- `web_search` - Search the web
```

### Step 5: State behavioral rules

Be direct and forceful:

```markdown
## Rules

1. **ALWAYS use tools** - When you need to perform an action, CALL the tool.
   Do NOT just say you'll do it.
   
2. **Explain actions** - Briefly describe what you're doing.

3. **Ask when unclear** - If the request is ambiguous, ask for clarification.
```

### Step 6: Assemble in order

```
1. Identity header (who you are)
2. Current time
3. Runtime/workspace info
4. Bootstrap files (SOUL.md, USER.md, etc.)
5. Tools summary
6. Skills summary
7. Memory context
8. Session info
```

## Examples

### Example 1: Coding assistant

```markdown
# CodeHelper

You are CodeHelper, a programming assistant.

## Current Time
2026-02-15 14:30

## Workspace
/Users/dev/my-project

## Tools
**USE TOOLS TO ACT.**

- `read_file` - Read source code
- `write_file` - Create/update files
- `edit_file` - Modify existing code
- `exec` - Run tests, builds, commands

## Rules
1. Always read files before editing
2. Run tests after changes
3. Explain your reasoning
4. Use edit_file for modifications, not write_file
```

### Example 2: Research assistant

```markdown
# Researcher

You are Researcher, an AI that finds and synthesizes information.

## Current Time
2026-02-15 14:30

## Tools
- `web_search` - Search for information
- `web_fetch` - Read webpage content
- `write_file` - Save research notes

## Rules
1. Always cite sources
2. Distinguish facts from opinions
3. Save important findings to notes
4. Ask clarifying questions about scope
```

### Example 3: Full agent with memory

```markdown
# Assistant

You are Assistant, a helpful AI with persistent memory.

## Current Time
2026-02-15 14:30 (Saturday)

## Workspace
/home/user/assistant-workspace

## Memory
Write important facts to: memory/MEMORY.md
Log activities to: memory/202602/20260215.md

## Tools
**YOU MUST USE TOOLS. DO NOT PRETEND.**

- `read_file` - Read files
- `write_file` - Write files  
- `exec` - Run commands
- `message` - Send messages

## Personality
- Helpful and concise
- Proactive but not pushy
- Asks clarifying questions

## Rules
1. Use tools for ALL actions
2. Remember important information
3. Respect user privacy
```

## Best practices

### Do include:
- Current date and time
- Workspace paths (absolute)
- Tool summary with critical warning
- Clear, forceful rules
- Identity in 1-2 lines

### Don't include:
- Verbose explanations
- Obvious statements ("be polite", "help the user")
- Negative instructions without alternatives
- Implementation details user doesn't need
- Long capability lists (tools have schemas)
- Hypothetical scenario handling

### Make tool use mandatory

The #1 failure mode is pretending to act instead of using tools:

```markdown
## CRITICAL

You MUST use tools to perform actions.
Do NOT say "I'll schedule that" - actually call the tool.
Do NOT say "Let me check that file" - actually read it.
Do NOT pretend. Act.
```

### Say what TO do, not what NOT to do

**Bad:**
```
Don't make things up.
Don't be unhelpful.
```

**Good:**
```
When uncertain, say "I don't know" and suggest how to find out.
```

### Token budget

| Component       | Target Size     |
| --------------- | --------------- |
| Identity        | 100-200 tokens  |
| Bootstrap files | 500-1500 tokens |
| Tool summaries  | 200-400 tokens  |
| Memory context  | 500-2000 tokens |
| **Total**       | < 4000 tokens   |

## Requirements

### Required sections:
1. Identity header
2. Current time
3. Tools summary with "must use" warning
4. Basic rules

### Optional sections:
- Workspace paths
- Bootstrap files (SOUL.md, USER.md, etc.)
- Memory context
- Skills summary
- Session info

### Testing checklist:
- [ ] Does the agent USE tools or just SAY it will?
- [ ] Does the agent know the current time?
- [ ] Does the agent follow the stated rules?
- [ ] Does "who are you?" return the correct identity?

## Common mistakes

1. **Assuming the LLM knows things** - It doesn't know time, your name, or available tools unless told
2. **Being too polite in rules** - "Please try to use tools" → "You MUST use tools"
3. **Burying important rules** - Put critical rules near the top
4. **Inconsistent formatting** - Pick a style and stick to it
5. **Not testing** - Always test with real interactions
