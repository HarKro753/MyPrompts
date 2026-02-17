---
name: system-prompt
description: How to design effective system prompts for agentic AI - what to include, what to avoid, and how to structure for reliable tool use. Use when writing system prompts, debugging agent behavior, or setting up new AI assistants.
---

# System Prompt Design

This skill describes how to craft system prompts that make AI agents actually work - reliably using tools, following instructions, and behaving predictably.

## Core Concept

The system prompt is the **DNA of your agent**. It defines:

- Who the agent is
- What it can do
- How it should behave
- What rules it must follow

A well-designed system prompt makes the difference between an agent that works and one that just talks about working.

## The Anatomy of an Effective System Prompt

### 1. Identity Header

Start with a clear statement of who the agent is:

```
# picoclaw

You are picoclaw, a helpful AI assistant.
```

**Why it matters**: The LLM needs grounding. A clear identity prevents confusion and provides a "character" to embody.

**Keep it short**: One or two lines. Don't write a biography.

### 2. Temporal Context

Always include the current time:

```
## Current Time
2026-02-15 14:30 (Saturday)
```

**Why it matters**:

- LLMs don't know what time it is
- Scheduling, reminders, and time-aware responses require this
- Date format should be unambiguous (YYYY-MM-DD)

### 3. Runtime Environment

Tell the agent where it's running:

```
## Runtime
darwin arm64

## Workspace
Your workspace is at: /Users/alex/picoclaw-workspace
- Memory: /Users/alex/picoclaw-workspace/memory/MEMORY.md
- Daily Notes: /Users/alex/picoclaw-workspace/memory/YYYYMM/YYYYMMDD.md
- Skills: /Users/alex/picoclaw-workspace/skills/{skill-name}/SKILL.md
```

**Why it matters**:

- File paths need to be absolute and correct
- OS-specific behavior (shell commands, paths) depends on this
- The agent knows where to find and store things

### 4. Available Tools

List tools with brief descriptions:

```
## Available Tools

**CRITICAL**: You MUST use tools to perform actions. Do NOT pretend to execute commands.

You have access to the following tools:

- `read_file` - Read the contents of a file at the given path
- `write_file` - Write content to a file, creating it if needed
- `edit_file` - Replace specific text in a file
- `exec` - Execute a shell command
- `web_search` - Search the web for information
- `message` - Send a message to the user
```

**Why it matters**:

- The LLM sees tool schemas separately, but a summary helps it reason about capabilities
- The **CRITICAL** warning prevents the common failure mode of pretending to act

### 5. Behavioral Rules

State the most important rules clearly and forcefully:

```
## Important Rules

1. **ALWAYS use tools** - When you need to perform an action (schedule reminders,
   send messages, execute commands, etc.), you MUST call the appropriate tool.
   Do NOT just say you'll do it or pretend to do it.

2. **Be helpful and accurate** - When using tools, briefly explain what you're doing.

3. **Memory** - When remembering something, write to memory/MEMORY.md
```

**Why it matters**: Rules that aren't stated clearly get ignored. Use bold, use caps, be direct.

## What TO Include

### Dynamic Information

- Current date and time
- Workspace paths
- Available tools (generated from registry)
- Session context (channel, chat ID)

### Static Identity

- Agent name and brief description
- Core personality traits (from SOUL.md)
- User information (from USER.md)
- Behavioral instructions (from AGENT.md)

### Contextual Memory

- Long-term memory contents
- Recent daily notes
- Skills summary

### Explicit Instructions

- Rules that MUST be followed
- How to use specific tools
- When to ask for clarification

## What NOT to Include

### 1. Verbose Explanations

**Bad:**

```
You are an AI assistant created to help users with various tasks.
You should always try to be helpful and provide accurate information.
When a user asks you to do something, you should consider whether
you have the capability to do it, and if so, proceed to do it in
the most efficient way possible while keeping the user informed...
```

**Good:**

```
You are picoclaw, a helpful AI assistant.
```

**Why**: Verbosity dilutes the important parts. The LLM doesn't need motivation.

### 2. Obvious Statements

**Bad:**

```
- Be polite
- Answer questions
- Help the user
- Don't be rude
```

**Why**: LLMs already do this by default. You're wasting tokens.

### 3. Negative Instructions (without positive alternative)

**Bad:**

```
Don't make things up.
Don't be unhelpful.
Don't ignore the user.
```

**Good:**

```
When uncertain, say "I don't know" and suggest how to find out.
```

**Why**: Negative instructions are weaker than positive ones. Say what TO do.

### 4. Implementation Details

**Bad:**

```
You are running on a server with 16GB RAM and an NVIDIA GPU.
The backend uses FastAPI with PostgreSQL.
API requests are rate-limited to 100/minute.
```

**Why**: Unless the agent needs to act on this, it's noise.

### 5. Long Lists of Capabilities

**Bad:**

```
You can:
- Read files
- Write files
- Edit files
- List directories
- Execute commands
- Search the web
- Fetch web pages
- Send messages
- Schedule tasks
- Manage reminders
- ...
(20 more items)
```

**Good:**

```
You have access to file, shell, web, and messaging tools.
Use them to accomplish tasks.
```

**Why**: The tool definitions already contain this. A summary is enough.

### 6. Hypothetical Scenarios

**Bad:**

```
If the user asks you to do something illegal, refuse politely.
If the user seems upset, be extra supportive.
If there's a system error, apologize and explain what happened.
```

**Why**: Handle these in behavioral guidelines, not exhaustive conditionals.

## Structuring for Tool Use

The #1 failure mode of agents is **pretending to act instead of using tools**. Combat this:

### Make Tool Use Mandatory

```
## CRITICAL RULE

You MUST use tools to perform actions.
Do NOT say "I'll schedule that for you" - actually call the tool.
Do NOT say "Let me check that file" - actually read it.
Do NOT pretend. Act.
```

### Explain Tool Locations

```
## Where Things Are

- To remember: write_file to memory/MEMORY.md
- To log: append_file to memory/YYYYMM/YYYYMMDD.md
- To learn skills: read_file from skills/{name}/SKILL.md
```

### Show the Consequences

```
If you don't use tools, nothing actually happens.
The user is relying on you to take real action.
```

## Assembly Order

The system prompt should be assembled in layers:

```
1. Core Identity (who you are, current time, runtime)
   ↓
2. Bootstrap Files (SOUL.md, USER.md, IDENTITY.md, AGENT.md)
   ↓
3. Tools Summary (dynamically generated from registry)
   ↓
4. Skills Summary (available skills and locations)
   ↓
5. Memory Context (MEMORY.md + recent daily notes)
   ↓
6. Session Context (channel, chat ID)
   ↓
7. Conversation Summary (if history was truncated)
```

Each layer is separated by `---` for clarity.

## Dynamic vs Static Content

### Generate Dynamically

- Current time (changes every request)
- Tool definitions (may vary by context)
- Memory contents (updated between sessions)
- Session info (channel, chat ID)

### Load from Files

- Personality (SOUL.md)
- Identity (IDENTITY.md)
- User info (USER.md)
- Rules (AGENT.md)

### Hardcode

- Core instructions that never change
- Fundamental rules
- Path patterns

## Token Budget

System prompts consume context. Be mindful:

| Component       | Typical Size    |
| --------------- | --------------- |
| Core identity   | 200-400 tokens  |
| Bootstrap files | 500-1500 tokens |
| Tool summaries  | 300-600 tokens  |
| Skills summary  | 200-500 tokens  |
| Memory context  | 500-2000 tokens |

**Total budget**: Aim for under 4000 tokens for the system prompt, leaving room for conversation.

### Strategies for Large Contexts

1. **Summarize memory** instead of including raw content
2. **List skills briefly** - agent can read full SKILL.md if needed
3. **Truncate daily notes** to most recent entries
4. **Exclude tools** that aren't relevant to the channel/context

## Testing Your System Prompt

### Test 1: Tool Use

Ask the agent to do something that requires a tool.

- Does it USE the tool, or just SAY it will?

### Test 2: Memory

Ask the agent to remember something, then start a new session.

- Did it actually write to memory?
- Does it recall in the new session?

### Test 3: Identity

Ask "who are you?"

- Does the response match your IDENTITY.md?

### Test 4: Rules

Try to get the agent to break a rule.

- Does it follow the rules you set?

## Common Mistakes

### 1. Assuming the LLM "knows" things

It doesn't know the time, your name, or what tools exist unless you tell it.

### 2. Being too polite in rules

"Please try to use tools when possible" → "You MUST use tools"

### 3. Burying important rules

Put critical rules near the top, not at the end.

### 4. Inconsistent formatting

Pick a format (headers, bullets, bold) and stick to it.

### 5. Not testing with real interactions

A prompt that looks good may fail in practice. Test it.

## Example: Minimal Effective Prompt

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

## Memory

Write important facts to: /home/user/workspace/memory/MEMORY.md
```

This is ~150 tokens and covers all essentials. Start minimal, add only what's needed.
