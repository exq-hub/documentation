---
title: Logging
description: Structured audit logging for search requests, item views, and client events.
icon: scroll
---

The engine writes a structured audit log for every search request, item retrieval, and client event. Logs are written asynchronously as FastAPI background tasks so they never block request handling.

## Enabling logging

Set `LogDirectory` in the collection's config section:

```ini
[my_collection]
LogDirectory = ./logs/my_collection/
```

If `LogDirectory` is omitted, audit logging is disabled for that collection. The directory is created automatically if it does not exist.

## Log file

Each collection writes to a single file:

```
{LogDirectory}/audit.log
```

The file is opened in **append mode** and never truncated. Entries are written sequentially.

## Format

Log entries are serialised with [MessagePack](https://msgpack.org/), a compact binary format. Each entry is a MessagePack-encoded map (dictionary) appended to the file:

```python
{
  "action":    str,   # Event type (see below)
  "session":   str,   # Session ID
  "timestamp": int,   # Unix milliseconds
  "data":      dict   # Event-specific payload
}
```

To read the log in Python:

```python
import msgpack

with open("audit.log", "rb") as f:
    unpacker = msgpack.Unpacker(f, raw=False)
    for entry in unpacker:
        print(entry)
```

## Event types

| `action` | Trigger | `data` payload |
|----------|---------|----------------|
| `search_request` | Any search endpoint called | strategy, query/params, n, result IDs, timing |
| `item_request` | `/exq/item/base` or `/exq/item/details` | requested media IDs |
| `session_init` | `/exq/init/{session}` | available collections |
| `add_model` | `/exq/log/addModel` | modelId |
| `remove_model` | `/exq/log/removeModel` | modelId |
| `client_event` | `/exq/log/clientEvent` | batch of raw client event objects |

## Timing fields

Search entries include two timing fields:

| Field | Description |
|-------|-------------|
| `request_timestamp` | Unix ms when the request was received |
| `completion_time` | Unix ms when results were assembled |

The difference `completion_time − request_timestamp` gives the wall-clock search latency in milliseconds.

## LoggingService internals

`LoggingService` wraps `AuditLogger` and is initialised during application startup if `LogDirectory` is configured. It exposes async methods that schedule the actual disk write as a FastAPI `BackgroundTask`:

```python
# In route handler
background_tasks.add_task(
    logging_service.log_search_request,
    request=request,
    suggestions=result["suggestions"],
    timing=timing,
)
```

This pattern keeps request handlers fast even when writes are slow (e.g. on spinning-disk storage).
