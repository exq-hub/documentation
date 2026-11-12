---
title: Logging
description: Structured audit logging for search requests, item views, and client events.
icon: scroll
---

The engine keeps an audit log of what happens in each session, such as searches, item requests and events sent by the client. Entries are written in the background, so logging doesn't slow down the response.

## Log file

The audit log is written to the directory set in the [`[logging]` section](/configuration/#logging) of the config, which is `./logs/` by default. Each server start writes a new file, `audit_<YYYY-MM-DD_HH-MM-SS>.log`, and all collections log to that same file. The directory is created if it doesn't exist. Entries are only ever appended, so a log file is never truncated or overwritten.

## Format

Entries are encoded with [MessagePack](https://msgpack.org/), a compact binary format, and appended to the file one after another. Each entry is a map like this:

```python
{
  "timestamp": int,   # Unix time in milliseconds
  "session":   str,   # Session ID, or "system" for errors
  "action":    str,   # What happened, for example "CLIP Search"
  "data":      dict   # Details of the request or event
}
```

To read a log file in Python:

```python
import msgpack

with open("logs/audit_2026-01-01_12-00-00.log", "rb") as f:
    unpacker = msgpack.Unpacker(f, raw=False)
    for entry in unpacker:
        print(entry)
```

## What gets logged

Most `/exq/` endpoints add an entry, and the `action` names the endpoint in plain words. Besides the fields below, the `data` of most entries also holds the `session`, `modelId` and `collection` of the request.

| `action` | Endpoint | `data` |
|----------|----------|--------|
| `Initialize Exquisitor LSE Session` | `GET /exq/init/{session}` | The available collections |
| `Get Total Items` | `POST /exq/info/totalItems` | The collection and its number of items |
| `Get filter definitions for collection` | `GET /exq/info/filters/{session}/{collection}` | The returned filter definitions |
| `Get filter values for collection` | `GET /exq/info/filters/values/...` | `tagtypeId` and `tagsetId` |
| `Log Add Model`, `Log Remove Model` | `POST /exq/log/addModel`, `POST /exq/log/removeModel` | The request body as JSON |
| `Client Event: <action>` | [`POST /exq/log/clientEvent`](/api/admin/#post-%2Fexq%2Flog%2Fclientevent) | The client's `ts` in Unix milliseconds, `element_id`, `route` and the event body |
| `CLIP Search`, `Text Search` | `POST /exq/search/clip`, `POST /exq/search/text` | Query, index, filters, excluded IDs, number of seen items, `n_requested` and the suggestions |
| `RF Search` | `POST /exq/search/rf` | As above, plus the `pos` and `neg` IDs |
| `Temporal Search` | `POST /exq/search/temporal` | The hops and chain settings, filters, excluded IDs, number of seen items, `n_requested` and the suggestions |
| `Faceted Search` | `POST /exq/search/faceted` | Filters, index, `n_requested` and the suggestions |
| `Get Info For Item` | `POST /exq/item/info` | Media ID, name and media type |
| `Get Info For Items` | `POST /exq/item/info/batch` | Lists of media IDs and names, and the IDs that weren't found |
| `Get Detailed Info For Item` | `POST /exq/item/details` | Media ID |
| `Get Related Info For Item` | `POST /exq/item/related` | Media ID and name |
| `Check if item is in excluded group` | `POST /exq/item/excluded` | Media ID, name and whether it's excluded |
| `Error` | Any request that fails with an engine error | Error type, message and details, and the request path and method |

For search entries, `timestamp` is when the request arrived. They also have a `completion_time` next to it, which is how long the search took in milliseconds.

## How logging works

Route handlers don't write to the log themselves. They hand the logging call to FastAPI as a `BackgroundTask`, which runs after the response has been sent. `LoggingService` builds the entry and passes it to `AuditLogger`, which puts it on a queue. A worker task, started with the server, takes entries from the queue and appends them to the file under a file lock. This keeps responses fast even when disk writes are slow, for example on spinning-disk storage.

```python
# routes/search.py, simplified
background_tasks.add_task(
    logging_service.log_search_request,
    "CLIP Search",
    session, model_id, collection,
    query_data,
    result["suggestions"],
    result["request_timestamp"],
    result["completion_time"],
)
```
