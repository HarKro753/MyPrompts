---
name: python-structlog
description: Structured logging for Python services using structlog. JSON lines in production, colored console in dev, request_id correlation via contextvars. Use when adding logging to a FastAPI/ADK service, debugging agent tool call traces, or setting up observability in any Python backend.
---

# Python Structured Logging — structlog

Production-grade logging for Python services: JSON in prod, human-readable in dev, zero config per file. Derived from TravelAgent backend.

## Why structlog over stdlib logging

| Feature | stdlib logging | structlog |
|---|---|---|
| Output format | Text | JSON (prod) or colored (dev) via renderer swap |
| Request correlation | Manual thread-local | `contextvars` — automatic per-request binding |
| Key-value context | String formatting | Native key=value pairs, searchable in any log aggregator |
| Dev experience | Ugly | Colored, aligned, readable |
| Performance | Fine | Fine (lazy evaluation) |

The killer feature: `structlog.contextvars`. Bind a `request_id` once at the HTTP layer — every log line in that request (tools, services, SSE) automatically includes it. No passing loggers around.

## Setup

```bash
uv add structlog
```

## `utils/logging.py` — the only file you need to touch

```python
import logging
import os
import sys
import structlog

def setup_logging() -> None:
    """Call once at app startup (main.py lifespan or top of script)."""
    log_level = os.getenv("LOG_LEVEL", "INFO").upper()
    is_prod = os.getenv("ENV", "development") == "production"

    shared_processors = [
        structlog.contextvars.merge_contextvars,        # inject request_id etc.
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.stdlib.add_logger_name,
    ]

    if is_prod:
        renderer = structlog.processors.JSONRenderer()  # JSON lines for log aggregators
    else:
        renderer = structlog.dev.ConsoleRenderer(colors=True)  # colored dev output

    structlog.configure(
        processors=shared_processors + [
            structlog.stdlib.ProcessorFormatter.wrap_for_formatter,
        ],
        wrapper_class=structlog.stdlib.BoundLogger,
        context_class=dict,
        logger_factory=structlog.stdlib.LoggerFactory(),
        cache_logger_on_first_use=True,
    )

    formatter = structlog.stdlib.ProcessorFormatter(
        processors=[
            structlog.stdlib.ProcessorFormatter.remove_processors_meta,
            renderer,
        ],
        foreign_pre_chain=shared_processors,
    )

    handler = logging.StreamHandler(sys.stdout)
    handler.setFormatter(formatter)

    root = logging.getLogger()
    root.handlers = [handler]
    root.setLevel(log_level)


def get_logger(name: str) -> structlog.stdlib.BoundLogger:
    """Use at module level: logger = get_logger(__name__)"""
    return structlog.get_logger(name)
```

## Per-file usage — 2 lines

```python
from utils.logging import get_logger
logger = get_logger(__name__)

# Anywhere in the file:
logger.info("search_flights.start", origin="FRA", destination="CPH", date="2026-03-15")
logger.info("search_flights.complete", duration_ms=3241, result_count=12)
logger.error("search_flights.failed", error=str(e), origin="FRA")
```

**Never use f-strings or % formatting in log calls.** Pass key=value pairs — structlog handles serialisation.

## Request correlation — FastAPI middleware

```python
# main.py
import uuid
import structlog
from fastapi import Request

@app.middleware("http")
async def request_id_middleware(request: Request, call_next):
    request_id = request.headers.get("X-Request-ID", str(uuid.uuid4()))
    structlog.contextvars.clear_contextvars()
    structlog.contextvars.bind_contextvars(request_id=request_id)
    response = await call_next(request)
    response.headers["X-Request-ID"] = request_id
    return response
```

Now every log line in the request — from HTTP handler to tool calls to SSE stream — includes `"request_id": "abc-123"` automatically.

## Tool logging pattern — input → time → output

```python
import time
from utils.logging import get_logger

logger = get_logger(__name__)

async def search_flights(origin: str, destination: str, date: str) -> dict:
    logger.info("search_flights.start", origin=origin, destination=destination, date=date)
    start = time.monotonic()
    try:
        result = await _call_api(origin, destination, date)
        duration_ms = int((time.monotonic() - start) * 1000)
        logger.info(
            "search_flights.complete",
            duration_ms=duration_ms,
            result_count=len(result.get("flights", [])),
            preview=str(result)[:300],  # truncate — tool results can be 5KB+
        )
        return result
    except Exception as e:
        duration_ms = int((time.monotonic() - start) * 1000)
        logger.error("search_flights.failed", error=str(e), duration_ms=duration_ms)
        raise
```

Pattern: **log input summary → time → log output summary + duration_ms**. Never log the full result blob — truncate to 300 chars and log the size separately.

## Agent SSE streaming — log every ADK event

```python
# services/chat.py — the SSE loop sees every ADK event; log here for full agent trace
async def stream_agent_response(session_id: str, message: str):
    logger = get_logger(__name__)
    async for event in runner.run_async(...):
        if event.is_final_response():
            logger.info("agent.response", session_id=session_id, length=len(event.text))
            yield format_sse("content", event.text)

        elif hasattr(event, "tool_calls"):
            for call in event.tool_calls:
                logger.info("agent_tool_call", tool=call.name, args=call.args)
                yield format_sse("tool_call_start", {"tool": call.name})

        elif hasattr(event, "tool_results"):
            for result in event.tool_results:
                logger.info(
                    "agent_tool_result",
                    tool=result.name,
                    size=len(str(result.output)),
                    preview=str(result.output)[:300],
                )
                yield format_sse("tool_call_complete", {"tool": result.name})
```

**Why here?** The SSE loop is the only place that sees every ADK event. Logging here gives a complete agent reasoning trace without monkey-patching ADK internals.

## Ruff config for logging

```toml
[tool.ruff.lint]
select = ["T20"]  # bans print() — forces use of logger
```

This makes `print()` a lint error, ensuring all output goes through structlog.

## Environment variables

```bash
LOG_LEVEL=DEBUG    # dev: see everything
LOG_LEVEL=INFO     # prod default
LOG_LEVEL=WARNING  # quiet mode
ENV=production     # switches to JSON renderer
```

## What you get in production (JSON)

```json
{"timestamp": "2026-03-01T14:23:41Z", "level": "info", "logger": "tools.search_flights",
 "event": "search_flights.complete", "request_id": "abc-123",
 "origin": "FRA", "destination": "CPH", "duration_ms": 3241, "result_count": 12}
```

Paste this into Datadog, Loki, CloudWatch, or any log aggregator — every field is searchable.
