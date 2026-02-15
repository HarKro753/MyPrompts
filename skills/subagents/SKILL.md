---
name: subagents
description: How to spawn independent agent instances for parallel task execution - async vs sync patterns, task delegation, and result handling
---

# Subagents

This skill describes how a main agent can spawn independent agent instances to handle tasks in parallel, enabling delegation, background processing, and concurrent execution.

## Core Concept

A **subagent** is an independent agent instance that:
- Runs its own agent loop
- Has access to tools
- Completes a specific task
- Reports results back to the main agent

Think of it as the agent saying: "I'll handle this myself" vs "Let me delegate this to a helper."

## Why Subagents?

### Problem: Sequential Bottleneck
The main agent loop processes one thing at a time:
```
User: "Research topic A, then research topic B, then compare them"

Without subagents:
  Research A (30 seconds)
  → Research B (30 seconds)  
  → Compare (5 seconds)
  Total: 65 seconds
```

### Solution: Parallel Execution
```
With subagents:
  Spawn: Research A ──────┐
  Spawn: Research B ──────┼── (parallel, 30 seconds)
  Wait for both ──────────┘
  Compare (5 seconds)
  Total: 35 seconds
```

## Two Patterns: Async vs Sync

### Async (spawn)
**Fire and forget** - start the task, continue immediately.

```
Main Agent:
  "I'll spawn a subagent to research this in the background"
  → spawn(task: "Research topic X", label: "research-x")
  → Returns immediately: "Subagent spawned"
  → Main agent continues with other work
  
Later:
  Subagent completes
  → Result published to message bus
  → Main agent receives notification
```

**Use when**:
- Task is truly independent
- User doesn't need immediate result
- You want to do other things while waiting

### Sync (subagent)
**Wait for result** - block until task completes.

```
Main Agent:
  "Let me delegate this to a subagent"
  → subagent(task: "Research topic X", label: "research-x")
  → Waits for completion
  → Returns with result: "Research found: ..."
  → Main agent uses result immediately
```

**Use when**:
- You need the result to continue
- Task is a subtask of larger work
- Immediate response is required

## The Subagent Structure

### Task Definition
```
SubagentTask {
  ID:            "subagent-1"
  Task:          "Research the history of Go programming language"
  Label:         "go-research"
  OriginChannel: "telegram"
  OriginChatID:  "123456"
  Status:        "running" | "completed" | "failed" | "cancelled"
  Result:        "Go was created at Google in 2009..."
  Created:       1708012345678
}
```

### Task Lifecycle
```
created → running → completed
                 → failed
                 → cancelled
```

## Subagent Execution

When a subagent runs, it gets:

### 1. Minimal System Prompt
```
You are a subagent. Complete the given task independently and report the result.
You have access to tools - use them as needed to complete your task.
After completing the task, provide a clear summary of what was done.
```

No personality files, no memory - just focus on the task.

### 2. The Task as User Message
```
{Role: "user", Content: "Research the history of Go programming language"}
```

### 3. Access to Tools
Subagents can use the same tools as the main agent:
- File operations
- Web search
- Shell execution
- etc.

**Exception**: Subagents typically don't get `spawn` or `subagent` tools to prevent recursive spawning.

### 4. Its Own Tool Loop
The subagent runs the same LLM → Tool → LLM loop:
```
LLM: "I'll search for Go history"
→ web_search("Go programming language history")
→ "Results: Go was created..."
LLM: "Let me get more details"
→ web_fetch("https://go.dev/doc/...")
→ "Content: ..."
LLM: "Here's the summary: Go was created at Google in 2009..."
```

## Result Handling

### Async (spawn) Results

When async subagent completes:
1. Result is published to message bus as system message
2. Main agent receives it through normal message processing
3. Main agent can forward to user or take further action

```
Message Bus:
  Channel: "system"
  SenderID: "subagent:subagent-1"
  ChatID: "telegram:123456"  (origin info)
  Content: "Task 'go-research' completed.\n\nResult:\n..."
```

### Sync (subagent) Results

Sync subagent returns directly:
```
ToolResult {
  ForLLM:  "Subagent task completed:\nLabel: go-research\nIterations: 3\nResult: ..."
  ForUser: "Go was created at Google in 2009..."  (truncated if too long)
  Silent:  false
  IsError: false
}
```

The main agent can immediately use this in its reasoning.

## Context Preservation

Subagents need to know where results should go:

### Origin Channel & ChatID
```
OriginChannel: "telegram"
OriginChatID:  "123456"
```

When the subagent uses the `message` tool to communicate with the user, it routes to the original channel.

### Contextual Tools
Tools that need routing (like `message`) receive context:
```
tool.SetContext(originChannel, originChatID)
```

## The Subagent Manager

A central manager tracks all subagents:

### Spawn
```
manager.Spawn(ctx, task, label, channel, chatID, callback)
→ Creates task entry
→ Starts goroutine
→ Returns immediately
```

### Task Tracking
```
manager.GetTask("subagent-1")    → task details
manager.ListTasks()              → all active/completed tasks
```

### Resource Limits
```
MaxIterations: 10  (prevent runaway subagents)
```

## Async Callbacks

For async operations, callbacks notify completion:

```
callback := func(ctx, result) {
    // Handle completion
    // Log result
    // Maybe notify user
}

manager.Spawn(ctx, task, label, channel, chatID, callback)
```

The callback runs when the subagent finishes, regardless of success or failure.

## Error Handling

### Task Failure
```
if err != nil:
    task.Status = "failed"
    task.Result = "Error: connection timeout"
    result.IsError = true
```

The main agent receives the error and can decide how to handle it.

### Cancellation
```
if ctx.Cancelled():
    task.Status = "cancelled"
    task.Result = "Task cancelled during execution"
```

Subagents respect context cancellation for graceful shutdown.

### Timeout
Subagent execution should have timeouts:
```
ctx, cancel := context.WithTimeout(background, 120*time.Second)
defer cancel()
runSubagent(ctx, task)
```

## When to Use Subagents

### Good Use Cases
- **Parallel research**: "Look up X, Y, and Z simultaneously"
- **Background tasks**: "Compile this project and let me know when done"
- **Independent subtasks**: "Generate these 5 reports"
- **Long-running operations**: "Download and process this dataset"

### Bad Use Cases
- **Simple operations**: Don't spawn a subagent to read one file
- **Highly dependent tasks**: If B needs A's result, just do them sequentially
- **User-interactive tasks**: Subagents shouldn't ask questions
- **Critical path work**: If the user is waiting, sync is usually better

## Subagent vs Main Agent

| Aspect | Main Agent | Subagent |
|--------|------------|----------|
| System prompt | Full (identity, memory, skills) | Minimal (task focus) |
| Session history | Yes | No |
| Memory access | Yes | Can read, shouldn't write |
| Spawning | Can spawn subagents | Cannot spawn (prevents recursion) |
| User interaction | Primary | Via message tool only |
| Lifetime | Long-running | Task completion |

## Communication Patterns

### Subagent → User (via message tool)
```
Subagent uses message(channel, chatID, "Progress update...")
→ User receives update directly
```

### Subagent → Main Agent (via bus)
```
Subagent completes
→ PublishInbound to "system" channel
→ Main agent's processSystemMessage handles it
```

### Main Agent Decision
The main agent decides whether to:
- Forward result to user
- Process result further
- Just log and ignore

## Best Practices

### 1. Clear Task Descriptions
```
Good: "Search for Python async patterns and summarize the top 3 approaches"
Bad:  "Look into Python stuff"
```

### 2. Appropriate Labels
```
Good: "python-async-research"
Bad:  "task1"
```

### 3. Reasonable Scope
One subagent = one focused task. Don't overload.

### 4. Handle Failures
Always have a plan for when subagents fail:
```
if result.IsError:
    "I encountered an error with that research. Let me try a different approach..."
```

### 5. Don't Over-Parallelize
Spawning 10 subagents creates 10 concurrent LLM calls. Be mindful of rate limits and costs.
