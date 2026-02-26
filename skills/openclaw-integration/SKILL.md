# OpenClaw Integration Skill

How to integrate any service with OpenClaw Gateway — sending messages to agents, reading sessions, and routing inter-agent communication.

## Gateway Connection

OpenClaw Gateway exposes two integration paths:

### 1. HTTP Transport (recommended for services)

```
POST {GATEWAY_URL}/tools/invoke
Authorization: Bearer {GATEWAY_TOKEN}
Content-Type: application/json

{ "tool": "<tool_name>", "args": { ... } }
```

`GATEWAY_URL` defaults to `http://127.0.0.1:18789` for local installs.

### 2. CLI Transport (local only)

```bash
openclaw gateway call <method> --json --params '<json>'
openclaw agent --agent <id> --message "<text>"
```

---

## Key Gateway Methods

### Send a message to an agent session

```
tool: "chat.send"  (or: openclaw gateway call chat.send)
args: {
  "session_key": "agent:main",   # format: agent:<agentId> or custom key
  "message": "Hello",
  "role": "user"
}
```

Returns: `{ "ok": true }`

### Read chat history for a session

```
tool: "sessions.history"
args: {
  "session_key": "agent:main",
  "limit": 20
}
```

Returns: `{ "messages": [{ "role", "content", "timestamp" }] }`

### List all sessions

```
tool: "sessions_list"
args: {}
```

Returns: `{ "count": N, "sessions": [{ "key", "kind", "model", "totalTokens", "updatedAt" }] }`

### Gateway health

```
GET {GATEWAY_URL}/health
```
or
```
openclaw health --json
```

Returns: `{ "ok": true, "agent": {...}, "channels": [...] }`

### Create an agent

```
tool: "agents.create"
args: { "name": "my-agent" }
```

### Write a file to agent workspace

```
tool: "agents.files.set"
args: {
  "agent_id": "main",
  "path": "USER.md",
  "content": "# User\n..."
}
```

---

## Python Client Pattern

```python
import httpx
from typing import Any

class OpenClawClient:
    def __init__(self, gateway_url: str, token: str) -> None:
        self.base_url = gateway_url.rstrip("/")
        self.headers = {"Authorization": f"Bearer {token}", "Content-Type": "application/json"}

    async def invoke(self, tool: str, args: dict[str, Any]) -> Any:
        async with httpx.AsyncClient() as client:
            resp = await client.post(
                f"{self.base_url}/tools/invoke",
                json={"tool": tool, "args": args},
                headers=self.headers,
                timeout=30.0,
            )
            resp.raise_for_status()
            return resp.json()

    async def send_message(self, session_key: str, message: str) -> None:
        await self.invoke("chat.send", {"session_key": session_key, "message": message, "role": "user"})

    async def get_history(self, session_key: str, limit: int = 20) -> list[dict[str, Any]]:
        result = await self.invoke("sessions.history", {"session_key": session_key, "limit": limit})
        return result.get("messages", [])

    async def health(self) -> dict[str, Any]:
        async with httpx.AsyncClient() as client:
            resp = await client.get(f"{self.base_url}/health", headers=self.headers, timeout=10.0)
            resp.raise_for_status()
            return resp.json()
```

---

## Session Key Conventions

- `agent:main` — default single-user agent
- `agent:<agentId>` — specific agent by ID
- `closedclaw:user:<userId>` — multi-tenant platform pattern
- Custom keys are valid — gateway creates sessions lazily on first send

---

## Environment Variables

| Variable | Default | Description |
|---|---|---|
| `OPENCLAW_GATEWAY_URL` | `http://127.0.0.1:18789` | Gateway base URL |
| `OPENCLAW_GATEWAY_TOKEN` | — | Bearer token from gateway config |
| `OPENCLAW_TRANSPORT` | `cli` | `cli`, `http`, or `auto` |

---

## Notes

- Gateway is always single-user on the self-hosted install (Mission Control pattern)
- For multi-user (ClosedClaw pattern): each user gets their own gateway URL + token stored in DB
- Rate limit: no hard limit, but avoid polling faster than 1s intervals
- Sessions are persistent — chat history survives gateway restarts
