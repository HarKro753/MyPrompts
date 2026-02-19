---
name: session-management
description: How agents maintain conversation continuity - history storage, context summarization, and session persistence across restarts. Use when implementing session handling, debugging context overflow, or building conversation memory.
---

# Session Management

A session is a single conversation thread that maintains continuity across multiple exchanges. It stores message history, manages context overflow through summarization, and persists across agent restarts.

## Quick start

Basic session handling:

```python
# Get or create session
session = sessions.get_or_create("telegram:123456")

# Add messages
session.add_message("user", "Hello")
session.add_message("assistant", "Hi there!")

# Get history for LLM context
history = session.get_history()

# Save to disk
session.save()
```

## Instructions

### Step 1: Create session storage

```python
class Session:
    key: str              # Unique identifier (e.g., "telegram:123456")
    messages: list        # Message history
    summary: str          # Summary of older messages
    created: datetime
    updated: datetime
```

### Step 2: Implement session operations

```python
def get_or_create(key: str) -> Session:
    if key in sessions:
        return sessions[key]
    session = Session(key=key, messages=[], summary="")
    sessions[key] = session
    return session

def add_message(session: Session, role: str, content: str):
    session.messages.append({"role": role, "content": content})
    session.updated = datetime.now()

def get_history(session: Session) -> list:
    return session.messages.copy()
```

### Step 3: Handle context overflow

When history grows too large, summarize:

```python
def check_and_summarize(session: Session, max_messages: int = 20):
    if len(session.messages) <= max_messages:
        return
    
    # Keep recent messages
    recent = session.messages[-4:]
    older = session.messages[:-4]
    
    # Summarize older messages
    summary = llm.summarize(older)
    
    # Update session
    session.summary = summary
    session.messages = recent
```

### Step 4: Include summary in context

```python
def build_context(session: Session, system_prompt: str) -> list:
    messages = [{"role": "system", "content": system_prompt}]
    
    # Add summary if exists
    if session.summary:
        messages.append({
            "role": "system", 
            "content": f"## Previous Conversation Summary\n{session.summary}"
        })
    
    # Add recent history
    messages.extend(session.get_history())
    
    return messages
```

### Step 5: Persist to disk

```python
def save(session: Session, path: str):
    filename = session.key.replace(":", "_") + ".json"
    filepath = os.path.join(path, filename)
    
    with open(filepath, 'w') as f:
        json.dump({
            "key": session.key,
            "messages": session.messages,
            "summary": session.summary,
            "created": session.created.isoformat(),
            "updated": session.updated.isoformat()
        }, f)
```

## Examples

### Example 1: Message types in history

```python
# User message
{"role": "user", "content": "What's in config.json?"}

# Assistant text response
{"role": "assistant", "content": "Let me check that for you."}

# Assistant tool call
{
    "role": "assistant",
    "content": "",
    "tool_calls": [{
        "id": "call_abc123",
        "type": "function",
        "function": {"name": "read_file", "arguments": '{"path": "config.json"}'}
    }]
}

# Tool result
{"role": "tool", "content": "{\"api_key\": \"...\"}", "tool_call_id": "call_abc123"}
```

### Example 2: Session key format

```python
# Channel:ChatID format
"telegram:123456"     # Telegram user
"cli:direct"          # CLI interaction
"whatsapp:+1234567"   # WhatsApp number
"slack:C04ABCD"       # Slack channel
"system:heartbeat"    # System processes
```

### Example 3: Summarization trigger

```python
# Check thresholds
if len(history) > 20 or estimate_tokens(history) > context_window * 0.75:
    trigger_summarization()

# Token estimation (rough)
def estimate_tokens(messages: list) -> int:
    total_chars = sum(len(m["content"]) for m in messages)
    return total_chars // 4  # ~4 chars per token for English
```

### Example 4: Handling orphaned tool messages

```python
# Problem: Session starts with tool message (no preceding assistant)
# This causes LLM errors

def sanitize_history(messages: list) -> list:
    # Strip leading tool messages
    while messages and messages[0]["role"] == "tool":
        messages = messages[1:]
    return messages
```

## Best practices

### Session hygiene

- **Save after completion** - Save after agent finishes responding, not mid-execution
- **Keep recent context** - Always keep 4-6 recent messages for immediate context
- **Quality summaries** - Preserve key decisions, facts learned, current task state

### Concurrency

- **Thread-safe operations** - Use locks for multi-threaded access
- **Read locks** - For get_history, get_summary
- **Write locks** - For add_message, set_summary, truncate

### Persistence

- **Atomic writes** - Write to temp file, sync, then rename
- **Sanitize filenames** - Replace `:` with `_` for Windows compatibility
- **Load on startup** - Restore all sessions from disk

### Error handling

- **Graceful degradation** - If summarization fails, keep full history
- **Don't lose context** - Better to overflow than lose messages
- **Handle corrupted files** - Skip and log, don't crash

### Cleanup

- **Periodic cleanup** - Remove old inactive sessions
- **Archive old sessions** - Compress and store for later reference

## Requirements

### Session structure

```python
Session {
    key: str           # "channel:chatID"
    messages: list     # Message history
    summary: str       # Summary of truncated history
    created: datetime  # Session creation time
    updated: datetime  # Last activity time
}
```

### Message structure

```python
Message {
    role: str          # "user" | "assistant" | "tool" | "system"
    content: str       # Message text
    tool_calls: list   # Optional: tool call requests
    tool_call_id: str  # Optional: links tool result to call
}
```

### Storage format

```
sessions/
  telegram_123456.json
  cli_direct.json
  slack_C04ABCD.json
```

### Dependencies

- JSON serialization
- File system access
- Threading/locking primitives
- LLM API for summarization
