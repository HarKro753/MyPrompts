# MyPrompts

A personal collection of system prompts, coding guidelines, and agent skills for AI-assisted development. These prompts ensure AI assistants follow consistent coding standards, best practices, and architectural patterns across projects.

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
├── Rules/                     # Language-specific coding guidelines
├── agent-ai/                  # Agent & AI pattern skills (10)
├── travel-apis/               # Travel API skills (5)
├── social-ml/                 # Social media / ML algorithm skills (5)
├── obsidian/                  # Obsidian / knowledge management skills (7)
├── blockchain-payments/       # Blockchain, Web3 & payments skills (3)
├── dev-tools/                 # Developer tools & framework skills (7)
├── mobile/                    # Mobile development skills (2)
└── video-media/               # Video & media skills (1)
```

## Language Guidelines

### [TypeScript](https://github.com/HarKro753/MyPrompts/tree/main/Rules) — React / Next.js

- SOLID principles and clean code
- Server/client component patterns
- Security best practices (CSRF, CSP, HttpOnly cookies)
- File organization and naming conventions
- Performance optimizations

### [C#](https://github.com/HarKro753/MyPrompts/tree/main/Rules) — .NET

- Domain-driven design structure
- Entity Framework patterns
- Dependency injection practices
- Async/await patterns
- Security and validation

### [Swift](https://github.com/HarKro753/MyPrompts/tree/main/Rules) — SwiftUI

- Package-based architecture (CoreEnvironment, NetworkKit, ThemeKit, SharedModels)
- SwiftUI Environment pattern for state propagation
- Observable classes with strict encapsulation (private fields, public getters/setters)
- GraphQL and REST API integration patterns
- Theming system with protocol-based design
- Modern concurrency with async/await and actors

---

## Skills (by Domain)

**40 skills** across **8 domains**, each following the SKILL.md format.

### [Agent & AI Patterns](https://github.com/HarKro753/MyPrompts/tree/main/agent-ai) — 10 skills

Core patterns for building interactive AI agents with tool use, derived from [OpenClaw](https://github.com/anthropics/claude-code) and [OpenCode](https://github.com/nichochar/open-code):

| Skill | Description |
|-------|-------------|
| [`agent-loop`](https://github.com/HarKro753/MyPrompts/tree/main/agent-ai/agent-loop) | The core agentic loop — process messages, call tools, iterate until task completion |
| [`agent-tools`](https://github.com/HarKro753/MyPrompts/tree/main/agent-ai/agent-tools) | Designing and implementing tools — the interface between AI reasoning and actions |
| [`subagents`](https://github.com/HarKro753/MyPrompts/tree/main/agent-ai/subagents) | Spawning independent agent instances for parallel task execution and delegation |
| [`system-prompt`](https://github.com/HarKro753/MyPrompts/tree/main/agent-ai/system-prompt) | Designing effective system prompts for reliable agentic tool use |
| [`prompt-engineering`](https://github.com/HarKro753/MyPrompts/tree/main/agent-ai/prompt-engineering) | Crafting, optimizing, and debugging prompts for any LLM application |
| [`memory-management`](https://github.com/HarKro753/MyPrompts/tree/main/agent-ai/memory-management) | Context window strategies, summarization, and persistent memory |
| [`session-management`](https://github.com/HarKro753/MyPrompts/tree/main/agent-ai/session-management) | Conversation continuity — history storage, summarization, and session persistence |
| [`skills-system`](https://github.com/HarKro753/MyPrompts/tree/main/agent-ai/skills-system) | Dynamic skill loading and resolution at runtime |
| [`skill-writer`](https://github.com/HarKro753/MyPrompts/tree/main/agent-ai/skill-writer) | Authoring new reusable SKILL.md files for Claude Code |
| `find-skills` | Discover and install agent skills from the community |

### [Travel APIs](https://github.com/HarKro753/MyPrompts/tree/main/travel-apis) — 5 skills

APIs and integration patterns for building AI travel agents:

| Skill | Description |
|-------|-------------|
| [`travel-agent-apis`](https://github.com/HarKro753/MyPrompts/tree/main/travel-apis/travel-agent-apis) | Accommodation, routes, and places APIs for AI travel agents |
| [`google-travel-apis`](https://github.com/HarKro753/MyPrompts/tree/main/travel-apis/google-travel-apis) | Google Routes API and Places API for itinerary planning |
| [`hasdata-flights-api`](https://github.com/HarKro753/MyPrompts/tree/main/travel-apis/hasdata-flights-api) | Real-time Google Flights data via HasData API — one-way, round-trip, multi-city |
| [`lufthansa-api`](https://github.com/HarKro753/MyPrompts/tree/main/travel-apis/lufthansa-api) | Lufthansa Open API — flight status, schedules, seat maps, reference data |
| [`brightdata-google-hotels`](https://github.com/HarKro753/MyPrompts/tree/main/travel-apis/brightdata-google-hotels) | Scrape real-time hotel data from Google Hotels via Bright Data SERP API |

### [Social Media / ML / Algorithms](https://github.com/HarKro753/MyPrompts/tree/main/social-ml) — 5 skills

Derived from the [X (Twitter) open-source recommendation algorithm](https://github.com/twitter/the-algorithm):

| Skill | Description |
|-------|-------------|
| [`x-algorithm-overview`](https://github.com/HarKro753/MyPrompts/tree/main/social-ml/x-algorithm-overview) | End-to-end "For You" feed — Home Mixer, Thunder, Phoenix ML ranker |
| [`social-media-algorithm`](https://github.com/HarKro753/MyPrompts/tree/main/social-ml/social-media-algorithm) | Social media ranking principles and feed construction |
| [`candidate-pipeline-framework`](https://github.com/HarKro753/MyPrompts/tree/main/social-ml/candidate-pipeline-framework) | Reusable Source → Filter → Scorer → Ranker pipeline design |
| [`engagement-scoring-weights`](https://github.com/HarKro753/MyPrompts/tree/main/social-ml/engagement-scoring-weights) | Signal weighting (likes, retweets, replies, dwell time) and scoring |
| [`transformer-attention-masks`](https://github.com/HarKro753/MyPrompts/tree/main/social-ml/transformer-attention-masks) | Custom attention masks for transformer recommendation models |

### [Obsidian / Knowledge Management](https://github.com/HarKro753/MyPrompts/tree/main/obsidian) — 7 skills

Tools and patterns for working with Obsidian vaults and knowledge graphs:

| Skill | Description |
|-------|-------------|
| [`obsidian-bases`](https://github.com/HarKro753/MyPrompts/tree/main/obsidian/obsidian-bases) | Create and edit Obsidian Bases (.base files) with views, filters, and formulas |
| [`obsidian-cli`](https://github.com/HarKro753/MyPrompts/tree/main/obsidian/obsidian-cli) | Interact with Obsidian vaults via CLI — read, create, search, manage notes |
| [`obsidian-markdown`](https://github.com/HarKro753/MyPrompts/tree/main/obsidian/obsidian-markdown) | Obsidian Flavored Markdown — wikilinks, embeds, callouts, properties |
| [`obsidian-principles`](https://github.com/HarKro753/MyPrompts/tree/main/obsidian/obsidian-principles) | Personal vault design — structure, templates, property conventions, style rules |
| [`obsidian-retrieval`](https://github.com/HarKro753/MyPrompts/tree/main/obsidian/obsidian-retrieval) | Graph-based retrieval for navigating the vault as a knowledge graph |
| [`json-canvas`](https://github.com/HarKro753/MyPrompts/tree/main/obsidian/json-canvas) | Create and edit JSON Canvas files (.canvas) — nodes, edges, groups, mind maps |
| [`defuddle`](https://github.com/HarKro753/MyPrompts/tree/main/obsidian/defuddle) | Extract clean markdown from web pages, removing clutter to save tokens |

### [Blockchain / Web3 / Payments](https://github.com/HarKro753/MyPrompts/tree/main/blockchain-payments) — 3 skills

Smart contracts, escrow, and payment integration:

| Skill | Description |
|-------|-------------|
| [`blockchain-escrow`](https://github.com/HarKro753/MyPrompts/tree/main/blockchain-payments/blockchain-escrow) | Trustless escrow on EVM chains — OpenZeppelin patterns for marketplaces and betting |
| [`solidity-smart-contracts`](https://github.com/HarKro753/MyPrompts/tree/main/blockchain-payments/solidity-smart-contracts) | Write, deploy, and test Solidity contracts — Foundry, OpenZeppelin, Layer 2 |
| [`stripe-payments`](https://github.com/HarKro753/MyPrompts/tree/main/blockchain-payments/stripe-payments) | Stripe integration — payment processing, Connect multi-party payouts, webhooks |

### [Developer Tools & Frameworks](https://github.com/HarKro753/MyPrompts/tree/main/dev-tools) — 7 skills

Backend frameworks, testing, browser automation, and agent SDKs:

| Skill | Description |
|-------|-------------|
| [`python-fastapi`](https://github.com/HarKro753/MyPrompts/tree/main/dev-tools/python-fastapi) | Modern Python backend — FastAPI, Pydantic v2, uv, async SQLite, full type safety |
| [`adk-python`](https://github.com/HarKro753/MyPrompts/tree/main/dev-tools/adk-python) | Google Agent Development Kit for Python — Gemini models, tools, multi-agent systems |
| [`adk-google-search-grounding`](https://github.com/HarKro753/MyPrompts/tree/main/dev-tools/adk-google-search-grounding) | Real-time Google Search grounding for ADK agents — live web data and citations |
| [`opencode-plugins`](https://github.com/HarKro753/MyPrompts/tree/main/dev-tools/opencode-plugins) | OpenCode plugin system — event hooks, custom tools, environment injection |
| [`openclaw-integration`](https://github.com/HarKro753/MyPrompts/tree/main/dev-tools/openclaw-integration) | Integrate services with OpenClaw Gateway — inter-agent communication |
| [`browser-use`](https://github.com/HarKro753/MyPrompts/tree/main/dev-tools/browser-use) | Browser automation — web testing, form filling, screenshots, data extraction |
| [`test-coverage`](https://github.com/HarKro753/MyPrompts/tree/main/dev-tools/test-coverage) | Test coverage strategy for Python and TypeScript — statement, branch, mutation testing |

### [Mobile Development](https://github.com/HarKro753/MyPrompts/tree/main/mobile) — 2 skills

iOS/macOS app development with Swift and Xcode:

| Skill | Description |
|-------|-------------|
| [`swiftui-expert-skill`](https://github.com/HarKro753/MyPrompts/tree/main/mobile/swiftui-expert-skill) | SwiftUI code — state management, view composition, concurrency, Liquid Glass |
| [`xcode-build`](https://github.com/HarKro753/MyPrompts/tree/main/mobile/xcode-build) | Build and run iOS/macOS apps with xcodebuild — simulators, tests, UI automation |

### [Video & Media](https://github.com/HarKro753/MyPrompts/tree/main/video-media) — 1 skill

Programmatic video creation:

| Skill | Description |
|-------|-------------|
| [`remotion-best-practices`](https://github.com/HarKro753/MyPrompts/tree/main/video-media/remotion-best-practices) | Remotion best practices — video in React, captions, FFmpeg, audio viz, 3D |

---

## Usage

Copy the relevant `CLAUDE.md` file into your project root to establish AI coding standards. Skills can be loaded on demand when building agentic applications — use `find-skills` to discover available skills.

## Philosophy

- Code readability over write speed
- Explicit over implicit
- Strong typing, no `any` or `dynamic`
- Test-driven development
- 95% of code should be self-documenting
