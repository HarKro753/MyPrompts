# MyPrompts

A personal collection of system prompts and coding guidelines for AI-assisted development. These prompts ensure AI assistants follow consistent coding standards, best practices, and architectural patterns across projects.

## Purpose

When working with AI coding assistants (Claude, GPT, etc.), these prompts establish:

- Consistent code style and conventions
- Language-specific best practices
- Security requirements
- Project structure standards
- Test-driven development practices

## Structure

```
.
├── c#/
│   ├── CLAUDE.md          # C#/.NET development guidelines
│   └── skills/            # C#-specific skills (coming soon)
└── ts/
    ├── CLAUDE.md          # TypeScript/React/Next.js guidelines
    └── skills/            # Agentic development skills
        ├── agent-loops/           # ReAct patterns, reasoning loops
        ├── function-calling/      # LLM tool definitions, structured output
        └── multi-agent-orchestration/  # Multi-agent coordination
```

## Language Guidelines

### TypeScript (`ts/CLAUDE.md`)

Guidelines for React and Next.js development including:

- SOLID principles and clean code
- Server/client component patterns
- Security best practices (CSRF, CSP, HttpOnly cookies)
- File organization and naming conventions
- Performance optimizations

### C# (`c#/CLAUDE.md`)

Guidelines for .NET development including:

- Domain-driven design structure
- Entity Framework patterns
- Dependency injection practices
- Async/await patterns
- Security and validation

## Skills for Agentic Development

The `ts/skills/` directory contains patterns for building AI agents:

### Agent Loops

Patterns for autonomous LLM reasoning:

- ReAct (Reasoning + Acting) loops
- Plan-and-Execute workflows
- Self-correction mechanisms
- Memory management

### Function Calling

LLM tool integration patterns:

- Tool schema definitions (strict mode)
- Execution loops
- Structured output with Pydantic/Zod
- Parallel tool calls

### Multi-Agent Orchestration

Coordination patterns for multiple agents:

- Fan-out/Fan-in workflows
- Supervisor pattern
- Conflict resolution
- Agent communication bus

## Usage

Copy the relevant `CLAUDE.md` file into your project root to establish AI coding standards. Skills can be referenced when building agentic applications.

## Philosophy

- Code readability over write speed
- Explicit over implicit
- Strong typing, no `any` or `dynamic`
- Test-driven development
- 95% of code should be self-documenting
