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
├── swift/
│   ├── CLAUDE.md          # Swift/SwiftUI development guidelines
│   └── skills/            # Swift-specific skills (coming soon)
├── ts/
│   ├── CLAUDE.md          # TypeScript/React/Next.js guidelines
│   └── skills/            # Agentic development skills
│       ├── agent-loops/           # ReAct patterns, reasoning loops
│       ├── function-calling/      # LLM tool definitions, structured output
│       └── multi-agent-orchestration/  # Multi-agent coordination
├── agent-loop-skills/             # Agent loop skill set (derived from OpenClaw & OpenCode)
│   ├── agent-loop/                # Core agentic loop pattern
│   ├── agent-tools/               # Tool definition and execution
│   ├── memory-management/         # Agent memory and context management
│   ├── prompt-engineering/        # Prompt design patterns
│   ├── subagents/                 # Sub-agent orchestration
│   ├── session-management/        # Session state and persistence
│   ├── system-prompt/             # System prompt construction
│   ├── skills-system/             # Skills loading and management
│   ├── skill-writer/              # Authoring new skills
│   ├── travel-agent-apis/         # Travel domain API integration
│   └── google-travel-apis/        # Google Travel API patterns
└── social-algorithm-skills/       # Social algorithm skill set (derived from X algorithm)
    ├── x-algorithm-overview/      # X "For You" feed algorithm overview
    ├── social-media-algorithm/    # Social media ranking patterns
    ├── candidate-pipeline-framework/  # Candidate sourcing and pipeline design
    ├── engagement-scoring-weights/    # Engagement scoring and signal weighting
    ├── transformer-attention-masks/   # Transformer and attention mask patterns
    └── skill-writer/              # Authoring new skills
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

### Swift (`swift/CLAUDE.md`)

Guidelines for Swift and SwiftUI development including:

- Package-based architecture (CoreEnvironment, NetworkKit, ThemeKit, SharedModels)
- SwiftUI Environment pattern for state propagation
- Observable classes with strict encapsulation (private fields, public getters/setters)
- GraphQL and REST API integration patterns
- Theming system with protocol-based design
- Modern concurrency with async/await and actors

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

## Agent Loop Skills

The `agent-loop-skills/` directory contains a comprehensive skill set derived from the [OpenClaw](https://github.com/anthropics/claude-code) and [OpenCode](https://github.com/nichochar/open-code) projects. These skills capture the core patterns behind building interactive AI agents with tool use:

- **Agent Loop** -- The fundamental execute-reason-act cycle that drives agentic behavior
- **Agent Tools** -- Defining, registering, and executing tools within an agent context
- **Memory Management** -- Context window strategies, summarization, and long-term recall
- **Prompt Engineering** -- System prompt design, few-shot patterns, and instruction tuning
- **Subagents** -- Spawning and coordinating child agents for parallel or specialized work
- **Session Management** -- Persisting and resuming agent state across interactions
- **System Prompt** -- Constructing effective system prompts with context injection
- **Skills System** -- Dynamic skill loading and resolution at runtime
- **Skill Writer** -- Patterns for authoring new reusable skills
- **Travel Agent APIs / Google Travel APIs** -- Domain-specific API integration examples

## Social Algorithm Skills

The `social-algorithm-skills/` directory contains a skill set derived from the [X (Twitter) open-source recommendation algorithm](https://github.com/twitter/the-algorithm). These skills break down how large-scale social media feeds are ranked and served:

- **X Algorithm Overview** -- End-to-end walkthrough of the "For You" feed: Home Mixer orchestration, Thunder real-time post store, Phoenix ML ranker, and Candidate Pipeline framework
- **Social Media Algorithm** -- General social media ranking principles and feed construction
- **Candidate Pipeline Framework** -- Reusable Source → Filter → Scorer → Ranker pipeline design
- **Engagement Scoring Weights** -- Signal weighting (likes, retweets, replies, dwell time) and score aggregation
- **Transformer Attention Masks** -- Grok-based transformer architecture, attention mask strategies, and ML ranking internals
- **Skill Writer** -- Patterns for authoring new skills in this domain

## Usage

Copy the relevant `CLAUDE.md` file into your project root to establish AI coding standards. Skills can be referenced when building agentic applications.

## Philosophy

- Code readability over write speed
- Explicit over implicit
- Strong typing, no `any` or `dynamic`
- Test-driven development
- 95% of code should be self-documenting
