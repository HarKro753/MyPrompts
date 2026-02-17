---
name: memory-management
description: The workspace file system that defines an agent's identity, personality, user knowledge, and persistent memory. Use when setting up agent identity, configuring bootstrap files, or implementing persistent memory systems.
---

# Memory Management: The Workspace File System

This skill describes how an agent's identity, personality, and knowledge are stored in simple markdown files within a workspace directory. These files are loaded at startup and injected into every conversation, giving the agent its "self."

## Core Concept

An agent is not just an LLM with tools - it has an **identity** defined by files in its workspace. These files are:

- Human-readable and editable
- Loaded automatically into the system prompt
- The source of the agent's personality, knowledge, and behavior

```
workspace/
  SOUL.md         # Personality and values
  IDENTITY.md     # Who the agent is
  USER.md         # Information about the user
  AGENT.md        # Behavioral instructions
  memory/
    MEMORY.md     # Learned facts (long-term)
    YYYYMM/
      YYYYMMDD.md # Daily notes (episodic)
```

## The Bootstrap Files

These files are loaded at startup and become part of every system prompt.

### SOUL.md - The Personality

Defines **how** the agent behaves - its personality traits and values.

```markdown
# Soul

I am picoclaw, a lightweight AI assistant.

## Personality

- Helpful and friendly
- Concise and to the point
- Curious and eager to learn
- Honest and transparent

## Values

- Accuracy over speed
- User privacy and safety
- Transparency in actions
- Continuous improvement
```

**Purpose**: Shape the agent's tone, attitude, and ethical grounding. The agent reads this and adopts these traits.

**When to edit**: When you want to change how the agent "feels" - make it more formal, more playful, more cautious, etc.

### IDENTITY.md - The Self

Defines **who** the agent is - name, purpose, capabilities, philosophy.

```markdown
# Identity

## Name

PicoClaw

## Description

Ultra-lightweight personal AI assistant

## Purpose

- Provide intelligent AI assistance
- Support multiple LLM providers
- Enable easy customization through skills
- Run on minimal hardware

## Capabilities

- Web search and content fetching
- File system operations
- Shell command execution
- Multi-channel messaging

## Philosophy

- Simplicity over complexity
- Performance over features
- User control and privacy
```

**Purpose**: Give the agent a sense of self - what it is, what it can do, what it stands for.

**When to edit**: When deploying the agent for a specific purpose, changing its name, or adjusting its stated capabilities.

### USER.md - The Human

Stores information **about the user** - preferences, personal details, goals.

```markdown
# User

## Preferences

- Communication style: casual
- Timezone: UTC+8
- Language: English

## Personal Information

- Name: Alex
- Location: Singapore
- Occupation: Developer

## Learning Goals

- Wants help with coding tasks
- Prefers detailed explanations
- Interested in AI/ML topics
```

**Purpose**: Allow the agent to personalize interactions. It knows who it's talking to.

**When to edit**: Users can fill this in themselves, or the agent can update it when learning new facts about the user.

### AGENT.md - The Instructions

Defines **behavioral rules** - guidelines for how the agent should act.

```markdown
# Agent Instructions

You are a helpful AI assistant. Be concise, accurate, and friendly.

## Guidelines

- Always explain what you're doing before taking actions
- Ask for clarification when request is ambiguous
- Use tools to help accomplish tasks
- Remember important information in your memory files
- Be proactive and helpful
- Learn from user feedback
```

**Purpose**: Provide operational instructions. Unlike SOUL (personality) or IDENTITY (self), this is about **rules**.

**When to edit**: When you want to add safety rails, change behavior policies, or add specific instructions.

## The Memory System

Beyond bootstrap files, the agent has persistent memory for learned information.

### memory/MEMORY.md - Long-term Memory

Facts that persist indefinitely across all sessions.

```markdown
# Long-term Memory

## User Information

- User prefers dark mode
- User's project uses React + TypeScript

## Preferences

- Always explain code changes
- Use verbose git commits

## Important Notes

- User's deploy target is AWS Lambda
```

**Purpose**: Store learned facts that should always be available.

**When the agent writes here**: When it learns something important about the user or project that should persist.

### memory/YYYYMM/YYYYMMDD.md - Daily Notes

Episodic memory - what happened each day.

```markdown
# 2026-02-15

## 10:30 - Helped debug authentication issue

Found that JWT token was expiring too quickly.

## 14:00 - Updated deployment config

Changed Lambda memory from 512MB to 1024MB.
```

**Purpose**: Track what happens over time. Gives context about recent activities.

**Retention**: Recent notes (last 3 days) are loaded into context. Older notes remain on disk for reference.

## How Files Become Context

At every conversation, the agent builds its system prompt by assembling:

```
1. Core Identity (generated)
   - Current time
   - Runtime info
   - Workspace path
   - Available tools

2. Bootstrap Files (loaded from workspace)
   - SOUL.md content
   - USER.md content
   - IDENTITY.md content
   - AGENT.md content

3. Skills Summary (from skills directory)
   - List of available skills
   - How to invoke them

4. Memory Context (from memory/)
   - Long-term memory (MEMORY.md)
   - Recent daily notes (last 3 days)

5. Session Info (dynamic)
   - Current channel
   - Chat ID
```

All sections are joined with `---` separators into one system prompt.

## The Edit Cycle

These files are not static - they can be updated:

### By the User

Users can directly edit any file to:

- Change agent personality (SOUL.md)
- Update their preferences (USER.md)
- Modify agent behavior (AGENT.md)
- Correct learned facts (MEMORY.md)

### By the Agent

The agent can use file tools to update:

- USER.md when learning user preferences
- MEMORY.md when storing important facts
- Daily notes when logging activities

**Important**: The agent should ask before making significant changes to SOUL.md or IDENTITY.md - these define its core self.

## Design Principles

### 1. Plain Text

All files are markdown. No databases, no binary formats. Humans can read and edit them with any text editor.

### 2. Separation of Concerns

Each file has one purpose:

- SOUL = personality
- IDENTITY = self
- USER = human
- AGENT = rules
- MEMORY = learned facts

### 3. Graceful Defaults

If a file doesn't exist, the agent still works. Files enhance but are not required.

### 4. Transparency

The user can see exactly what the agent "knows" by reading these files. No hidden state.

### 5. Portability

Copy the workspace directory to move the agent's entire identity and memory to another machine.

## File Loading Order

Bootstrap files are loaded in this order:

1. AGENTS.md (if exists)
2. SOUL.md
3. USER.md
4. IDENTITY.md

This means later files can reference concepts from earlier files, and IDENTITY is loaded last (closest to the user message in context).

## Practical Examples

### Changing Agent Personality

Edit SOUL.md:

```markdown
## Personality

- Professional and formal
- Thorough and detailed
- Reserved but helpful
```

Next conversation, the agent adopts this new personality.

### Teaching Agent About User

User says: "I'm a backend developer working on microservices"
Agent updates USER.md:

```markdown
## Personal Information

- Occupation: Backend Developer
- Focus: Microservices architecture
```

### Storing Project Knowledge

Agent learns project structure, updates MEMORY.md:

```markdown
## Project Information

- Main language: Go
- Database: PostgreSQL
- Deployment: Kubernetes
```

## Anti-patterns

- **Overloading MEMORY.md**: Keep it focused on persistent facts, not conversation logs
- **Conflicting files**: Don't put personality in AGENT.md or rules in SOUL.md
- **Huge files**: Keep each file focused. If MEMORY.md grows too large, summarize or archive
- **Sensitive data**: Don't store passwords or secrets in these plain text files
