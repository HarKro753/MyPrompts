---
name: agent-tools
description: How to design and implement tools for agentic systems - the interface between AI reasoning and real-world actions. Use when creating new tools, designing tool interfaces, or troubleshooting tool execution.
---

# Agent Tools

This skill describes how tools work in an agentic system - the bridge between the LLM's reasoning and actual actions in the world.

## Core Concept

Tools are **functions the LLM can call** to interact with the world. Without tools, an LLM can only generate text. With tools, it becomes an agent that can:

- Read and write files
- Execute commands
- Search the web
- Send messages
- Control hardware
- And anything else you implement

## The Tool Interface

Every tool must define four things:

### 1. Name

A unique identifier for the tool. Keep it short, descriptive, snake_case.

- Good: `read_file`, `web_search`, `send_message`
- Bad: `doSomethingWithFile`, `tool1`, `utility`

### 2. Description

A clear explanation of what the tool does. This is what the LLM reads to decide when to use the tool.

- Good: "Read the contents of a file at the given path"
- Bad: "File reader"

### 3. Parameters

A schema defining what arguments the tool accepts:

- Parameter names and types
- Which are required vs optional
- Descriptions for each parameter

Example parameter schema:

```
{
  "type": "object",
  "properties": {
    "path": {
      "type": "string",
      "description": "The file path to read"
    },
    "limit": {
      "type": "integer",
      "description": "Maximum lines to return (optional)"
    }
  },
  "required": ["path"]
}
```

### 4. Execute

The function that actually performs the action when called.

## Tool Results

When a tool executes, it returns a structured result:

### ForLLM

Content sent back to the LLM as context. This is required - the LLM needs to know what happened.

### ForUser

Content shown directly to the user. Use this for outputs the user should see (command results, fetched content).

### Silent

When true, nothing is shown to the user. Use for background operations (file saves, internal state changes).

### IsError

Indicates the tool failed. The error message goes in ForLLM so the LLM can handle it.

### Async

Indicates the tool started a background operation. The result will come later via callback.

## Result Patterns

### Silent Result

For operations the user doesn't need to see:

```
"File saved successfully" -> ForLLM only, Silent=true
```

### User Result

For outputs the user should see:

```
"Command output: [...]" -> ForLLM and ForUser
```

### Error Result

For failures:

```
"File not found: config.json" -> ForLLM, IsError=true
```

### Async Result

For background tasks:

```
"Task started, will report when done" -> ForLLM, Async=true
```

## The Tool Registry

Tools are collected in a **registry** that:

- Stores all available tools by name
- Converts tools to LLM-compatible schemas
- Routes execution requests to the right tool
- Handles context injection and callbacks

### Registration

Tools are registered at startup:

```
registry.Register(ReadFileTool)
registry.Register(WriteFileTool)
registry.Register(WebSearchTool)
```

### Execution

When the LLM requests a tool:

```
result = registry.Execute("read_file", {"path": "config.json"})
```

### Schema Generation

The registry generates tool definitions for the LLM:

```
[
  {
    "type": "function",
    "function": {
      "name": "read_file",
      "description": "Read file contents",
      "parameters": {...}
    }
  }
]
```

## Tool Categories

### File System Tools

- `read_file`: Read file contents
- `write_file`: Write content to file
- `edit_file`: Replace text in file
- `append_file`: Add to end of file
- `list_dir`: List directory contents

### Execution Tools

- `exec` / `shell`: Run shell commands
- `subagent`: Spawn independent agent task

### Web Tools

- `web_search`: Search the internet
- `web_fetch`: Retrieve webpage content

### Communication Tools

- `message`: Send message to user/channel

### Hardware Tools (specialized)

- `i2c`: Communicate with I2C devices
- `spi`: Communicate with SPI devices

## Contextual Tools

Some tools need to know about the current context:

- Which channel the message came from
- Which chat/user to respond to

These tools implement an additional interface that receives context before execution:

```
SetContext(channel, chatID)
```

The agent loop calls this before executing tools that need routing information.

## Async Tools

Long-running operations shouldn't block the agent loop. Async tools:

1. Start the operation in the background
2. Return immediately with `Async=true`
3. Continue processing
4. Call a callback when done
5. The callback notifies the main agent

Example: Spawning a subagent

- Returns immediately: "Subagent started"
- Subagent runs independently
- When done, publishes result to message bus
- Main agent can handle completion

## Tool Design Principles

### 1. Single Responsibility

Each tool does ONE thing well. Don't combine read+write in one tool.

### 2. Clear Naming

The name should tell you what it does. `edit_file` not `file_op`.

### 3. Descriptive Parameters

Each parameter should be self-documenting. Include descriptions.

### 4. Graceful Errors

Return clear error messages. Don't crash - return ErrorResult.

### 5. Safe Defaults

Optional parameters should have sensible defaults.

### 6. Validation

Check inputs before acting. Return errors for invalid arguments.

### 7. Path Security

For file tools, validate paths are within allowed directories. Prevent directory traversal attacks.

## The Edit Pattern

File editing is a common operation. The pattern:

1. **Read current content**
2. **Find the text to replace** (must exist exactly once)
3. **Replace with new text**
4. **Write back**

Why require exact match?

- Prevents accidental changes to wrong location
- Ensures the LLM understands current file state
- Fails safely if file changed since last read

## Tool Summaries in System Prompt

The agent's system prompt includes a summary of available tools:

```
## Available Tools

- `read_file` - Read the contents of a file
- `write_file` - Write content to a file
- `web_search` - Search the web for information
- `message` - Send a message to the user
```

This helps the LLM understand its capabilities without the full schema.

## Adding New Tools

To extend an agent with new capabilities:

1. Define the tool with Name, Description, Parameters, Execute
2. Handle all error cases gracefully
3. Return appropriate result type
4. Register with the tool registry
5. The LLM automatically gains access

No prompt changes needed - the tool registry dynamically generates tool definitions.

## Anti-patterns to Avoid

- **God tools**: One tool that does everything
- **Unclear naming**: `util`, `helper`, `process`
- **Missing validation**: Accepting any input blindly
- **Silent failures**: Errors that don't get reported
- **Blocking operations**: Long tasks that freeze the agent
- **Unbounded output**: Returning huge amounts of data to LLM
