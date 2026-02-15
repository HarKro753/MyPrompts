---
name: agent-loop
description: The core agentic loop pattern - how an AI agent processes messages, calls tools, and iterates until task completion
---

# The Agent Loop

This skill describes the fundamental execution pattern of an AI agent: the iterative loop that transforms user requests into completed tasks through reasoning and tool use.

## Core Concept

An agent is not just an LLM - it's an **LLM in a loop with tools**. The agent loop is the heartbeat that drives autonomous behavior:

```
User Message -> Build Context -> LLM Call -> Tool Execution -> Loop or Respond
```

## The Basic Loop Structure

### Step 1: Receive Message

The agent receives input from some channel (CLI, chat, API, scheduled trigger).

### Step 2: Build Context

Assemble everything the LLM needs:

- **System prompt**: Identity, capabilities, rules
- **Memory**: Persistent facts and recent notes
- **Session history**: Previous messages in this conversation
- **Current message**: What the user just said
- **Tool definitions**: What actions are available

### Step 3: Call the LLM

Send the assembled context to the language model. The LLM can respond in two ways:

- **Direct answer**: Text response, no tools needed
- **Tool calls**: Requests to execute one or more tools

### Step 4: Handle Response

**If direct answer:**

- Save to session history
- Return response to user
- Loop ends

**If tool calls:**

- Execute each requested tool
- Collect results
- Add assistant message (with tool calls) to context
- Add tool results to context
- Go back to Step 3 (call LLM again)

### Step 5: Iterate Until Done

The loop continues until:

- LLM gives a direct answer (no tool calls)
- Maximum iterations reached (safety limit)
- Error occurs
- Context cancelled (shutdown)

## Message Flow Visualization

```
Iteration 1:
  [system prompt] + [memory] + [history] + [user message]
  -> LLM returns: tool_calls: [read_file("config.json")]

Iteration 2:
  [previous context] + [assistant: tool_calls] + [tool: file contents]
  -> LLM returns: tool_calls: [edit_file("config.json", ...)]

Iteration 3:
  [previous context] + [assistant: tool_calls] + [tool: "file edited"]
  -> LLM returns: "I've updated the configuration file."

Loop ends, response returned to user.
```

## Context Building

The system prompt is assembled from multiple sources:

1. **Identity section**: Who the agent is, current time, workspace location
2. **Bootstrap files**: SOUL.md, USER.md, IDENTITY.md, AGENT.md
3. **Skills summary**: Available skills and how to invoke them
4. **Memory context**: Long-term memory + recent daily notes
5. **Tools section**: List of available tools with descriptions
6. **Session info**: Current channel and chat ID

This layered approach allows customization at multiple levels without code changes.

## Session Management

Each conversation has a **session key** that identifies it. The session stores:

- Message history (user, assistant, tool messages)
- Conversation summary (when history gets too long)

### Summarization

When session history grows beyond thresholds:

1. Take older messages (keeping recent ones)
2. Ask LLM to summarize them
3. Store summary separately
4. Truncate history
5. Future contexts include: summary + recent history

This prevents context overflow while maintaining continuity.

## Iteration Safety

### Maximum Iterations

Always set a limit (e.g., 10-20 iterations) to prevent:

- Infinite loops from confused LLM
- Runaway costs from repeated API calls
- Stuck agents that never complete

### Handling Tool Errors

When a tool fails:

- Return error message as tool result
- Let LLM decide how to proceed
- LLM might retry, try alternative, or report failure to user

### Context Cancellation

Respect shutdown signals:

- Check for cancellation before each iteration
- Clean up gracefully when stopped
- Don't leave partial state

## Tool Call Execution

For each tool call in the LLM response:

1. **Parse arguments**: Extract tool name and parameters
2. **Find tool**: Look up in tool registry
3. **Set context**: Provide channel/chat info if needed
4. **Execute**: Run the tool with arguments
5. **Handle result**:
   - `ForLLM`: Content that goes back to the LLM
   - `ForUser`: Content shown directly to user (optional)
   - `Silent`: Don't show anything to user
   - `Async`: Tool continues in background

## The Message Types

| Role        | Purpose                            |
| ----------- | ---------------------------------- |
| `system`    | Agent identity and instructions    |
| `user`      | Messages from the human            |
| `assistant` | LLM responses (text or tool calls) |
| `tool`      | Results from tool execution        |

The conversation alternates: user -> assistant -> (tool -> assistant)\* -> user

## Subagents

The loop can spawn **subagents** - independent agent instances that:

- Run their own loop with their own tools
- Execute tasks asynchronously
- Report results back to main agent

This enables parallel task execution and delegation.

## Process Options

The loop can be configured per-invocation:

- **SessionKey**: Which session to use
- **EnableSummary**: Whether to trigger summarization
- **NoHistory**: Run without loading history (for heartbeats)
- **SendResponse**: Whether to push response to message bus

## The Heartbeat Pattern

Some agents run on schedules (cron). The heartbeat loop:

- Runs without session history
- Checks for triggers/conditions
- Takes actions if needed
- Doesn't accumulate context

This keeps scheduled runs lightweight and independent.

## Error Recovery

When the LLM call fails:

1. Log the error with context
2. Return error to caller
3. Don't corrupt session state
4. Allow retry from caller

The loop should be resilient - individual failures shouldn't break the agent.
