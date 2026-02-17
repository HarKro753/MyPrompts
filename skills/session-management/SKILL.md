---
name: session-management
description: How agents maintain conversation continuity - history storage, context summarization, and session persistence across restarts. Use when implementing session handling, debugging context overflow, or building conversation memory.
---

# Session Management

This skill describes how an agent maintains conversation continuity - storing message history, summarizing when context grows too large, and persisting sessions across restarts.

## Core Concept

A **session** is a single conversation thread. It includes:
- All messages exchanged (user, assistant, tool calls, tool results)
- A summary of older messages (when history is truncated)
- Timestamps for creation and last update

Sessions are identified by a **session key** - typically `channel:chatID` (e.g., `telegram:123456`).

## Session vs Memory

These serve different purposes:

| Aspect | Session | Memory |
|--------|---------|--------|
| Scope | One conversation | All time |
| Content | Messages exchanged | Facts learned |
| Lifetime | Until summarized/cleared | Persistent |
| Purpose | Conversation flow | Long-term knowledge |

The agent uses BOTH:
- Session history for "what did we just talk about?"
- Memory for "what do I know about this user?"

## The Session Structure

```
Session {
  Key:      "telegram:123456"
  Messages: [
    {Role: "user", Content: "Hello"},
    {Role: "assistant", Content: "Hi there!"},
    {Role: "user", Content: "Read my config file"},
    {Role: "assistant", Content: "", ToolCalls: [read_file(...)]},
    {Role: "tool", Content: "file contents...", ToolCallID: "call_123"},
    {Role: "assistant", Content: "Here's your config..."}
  ]
  Summary:  "User greeted assistant and asked about config file..."
  Created:  2026-02-15T10:00:00Z
  Updated:  2026-02-15T10:05:00Z
}
```

## Message Types in History

### User Messages
```
{Role: "user", Content: "What's in my config?"}
```

### Assistant Messages (text only)
```
{Role: "assistant", Content: "Let me check that for you."}
```

### Assistant Messages (with tool calls)
```
{
  Role: "assistant",
  Content: "",
  ToolCalls: [
    {ID: "call_abc", Type: "function", Function: {Name: "read_file", Arguments: "{...}"}}
  ]
}
```

### Tool Results
```
{Role: "tool", Content: "file contents here", ToolCallID: "call_abc"}
```

The **ToolCallID** links each tool result to its corresponding tool call. This is critical for the LLM to understand the conversation flow.

## Session Operations

### Get or Create
When a message arrives, get the existing session or create a new one:
```
session = sessions.GetOrCreate("telegram:123456")
```

### Add Message
After each exchange, add messages to history:
```
sessions.AddMessage(key, "user", "Hello")
sessions.AddMessage(key, "assistant", "Hi there!")
```

### Add Full Message (with tool calls)
For messages containing tool calls or tool results, preserve the full structure:
```
sessions.AddFullMessage(key, {
  Role: "assistant",
  ToolCalls: [...]
})
```

### Get History
When building context, retrieve the conversation history:
```
history = sessions.GetHistory(key)
// Returns copy of messages array
```

### Save
Persist to disk after changes:
```
sessions.Save(key)
```

## Context Overflow Problem

LLMs have limited context windows. A long conversation will overflow:

```
System Prompt:     ~3000 tokens
Session History:   ??? tokens (grows unbounded)
Current Message:   ~100 tokens
                   ──────────
                   Must fit in context window
```

When history grows too large, the agent must **summarize**.

## The Summarization Strategy

### When to Summarize

Check after each response:
```
if len(history) > 20 OR estimateTokens(history) > (contextWindow * 0.75):
    triggerSummarization()
```

Thresholds:
- Message count: >20 messages
- Token estimate: >75% of context window

### How to Summarize

1. **Keep recent messages** (last 4) for immediate context
2. **Take older messages** for summarization
3. **Ask LLM** to create a concise summary
4. **Store summary** separately
5. **Truncate history** to recent messages only

```
Before:
  Messages: [m1, m2, m3, m4, m5, m6, m7, m8, m9, m10]
  Summary: ""

After:
  Messages: [m7, m8, m9, m10]  (kept last 4)
  Summary: "User asked about X, assistant helped with Y..."
```

### Multi-Part Summarization

For very long histories:
1. Split messages into two parts
2. Summarize each part separately
3. Merge the two summaries into one

This prevents the summarization prompt itself from overflowing.

### Oversized Message Guard

Skip messages larger than 50% of context window:
```
maxMessageTokens = contextWindow / 2
for each message:
    if estimateTokens(message) > maxMessageTokens:
        skip (don't include in summarization)
```

This prevents a single huge message (like a large file read) from breaking summarization.

## Using Summary in Context

When building messages for the LLM:

```
System Prompt
---
## Summary of Previous Conversation
[summary content here]
---
[recent history messages]
---
[current user message]
```

The summary provides context about what happened before, while recent messages provide the immediate conversation flow.

## Token Estimation

A simple heuristic for estimating tokens:
```
tokens = characterCount / 4  (for English)
tokens = runeCount / 3       (for CJK/mixed text)
```

This is approximate but good enough for threshold decisions.

## Session Persistence

Sessions are saved to disk as JSON files:
```
workspace/sessions/
  telegram_123456.json
  cli_direct.json
  whatsapp_789.json
```

### Filename Sanitization
Session keys like `telegram:123456` contain `:` which is invalid on Windows. The key is sanitized for filenames but preserved inside the JSON:
```
Key: "telegram:123456" → File: "telegram_123456.json"
```

### Atomic Writes
To prevent corruption on crash:
1. Write to temporary file
2. Sync to disk
3. Rename to final path

This ensures the session file is either complete or doesn't exist.

### Loading on Startup
When the agent starts, it loads all existing session files:
```
for each .json file in sessions/:
    parse JSON
    add to in-memory sessions map
```

Sessions survive restarts.

## Session Keys

The key format is `channel:chatID`:
- `telegram:123456` - Telegram user 123456
- `cli:direct` - CLI direct interaction
- `whatsapp:+1234567890` - WhatsApp number
- `system:heartbeat` - System processes

Different keys = different conversations, even for the same user across channels.

## Orphaned Tool Messages

A special edge case: if a session starts with a `tool` message (no preceding assistant message with tool calls), the LLM will error.

**Fix**: Strip leading tool messages when loading history:
```
while history[0].Role == "tool":
    history = history[1:]  // Remove orphaned tool message
```

This can happen if a session was saved mid-tool-execution.

## Concurrency

Session operations must be thread-safe:
- Multiple goroutines may access sessions simultaneously
- Use read/write locks
- Read lock for GetHistory, GetSummary
- Write lock for AddMessage, SetSummary, TruncateHistory

## Heartbeat Sessions

Some operations (like scheduled checks) don't need history:
```
ProcessHeartbeat(ctx, content, channel, chatID)
  → NoHistory: true
  → Don't load session history
  → Don't save to session
```

This keeps scheduled runs lightweight and independent.

## Best Practices

### 1. Save After Completion
Save the session after the agent finishes responding, not during tool execution.

### 2. Summary Quality
Good summaries preserve:
- Key decisions made
- Important facts learned
- Current task state

Bad summaries are just "User and assistant talked about stuff."

### 3. Don't Over-Summarize
Keep enough recent messages (4-6) for the LLM to understand immediate context.

### 4. Handle Failures Gracefully
If summarization fails, keep the full history rather than losing context.

### 5. Session Cleanup
Consider periodic cleanup of old/inactive sessions to prevent disk bloat.
