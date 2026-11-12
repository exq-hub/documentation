---
title: Architecture
description: Component overview, dependency injection, and request lifecycle for the Live Services Engine.
icon: layers
---

## Component map

```
     ┌──────────────────────────────────────────────────────────────┐
     │                      FastAPI (main.py)                       │
     │                                                              │
     │  /exq/search/...     ──>  SearchService  ──>  strategies     │
     │  /exq/item/...       ──>  ItemService                        │
     │  /exq/init         ─┐                                        │
     │  /exq/info/...     ─┼──>  admin routes                       │
     │  /exq/log/...      ─┘                                        │
     │  /health, /          ──>  main.py                            │
     │                                                              │
     │  most /exq/ routes   ──>  LoggingService  ──>  audit log     │
     └───────────────────────────────┬──────────────────────────────┘
                                     │
                            ApplicationContainer
                                     │
        ┌──────────────────┬─────────┴───────┬───────────────────┐
        ▼                  ▼                 ▼                   ▼
CLIPModelManager   TextModelManager   IndexRepository   DatabaseRepository
   (open_clip)        (sentence-      (FAISS / Zarr,     (one database
                     transformers)      per index)        per collection)
```

## Layers

### API layer (`app/api/`)

FastAPI route handlers. Each router module groups related endpoints:

| Module | Paths | Responsibility |
|--------|-------|----------------|
| `routes/search.py` | `/exq/search/...` | Searches, through `SearchService` |
| `routes/items.py` | `/exq/item/...` | Item information, through `ItemService` |
| `routes/admin.py` | `/exq/init`, `/exq/info/...`, `/exq/log/...` | Session start, collection and filter information, client logging |
| `main.py` | `/health`, `/` | Health check and service information |

Route handlers are kept thin. Once FastAPI has validated the request against its Pydantic schema, the handler calls a service, adds an audit log entry as a background task and returns the response. If something goes wrong, the error is turned into a status code with a `{"detail": "..."}` body, as described in [Exception handling](#exception-handling).

### Service layer (`app/services/`)

Business logic lives here, decoupled from HTTP concerns.

| Service | Responsibility |
|---------|----------------|
| `SearchService` | Builds the search strategies, picks one by name, and times the search |
| `ItemService` | Item information, including media URLs and metadata |
| `LoggingService` | Writes audit log entries in the background. See [Logging](/logging/). |

`SearchService` and `ItemService` are created for each request. They hold no state of their own, only the shared model managers and repositories. `LoggingService` is created once at startup.

### Strategy layer (`app/strategies/`)

Each type of search is a strategy class, and `SearchService` picks the right one by name for each request. How each search works is explained on its own page, for example [CLIP search](/search/clip/) or [Relevance feedback](/search/rf/).

Strategies that search embeddings share their common steps through `VectorSearchMixin` in `strategies/embedding_search.py`. These steps leave out any items the request marks as `seen` or `excluded`, and any that don't pass the [filters](/filters/). The remaining items are then either scored exactly or searched in the index, depending on [how many are left](/filters/#how-filters-are-applied). A new embedding search therefore only needs to define how it turns its query into a vector.

To add a type of search, write a strategy class based on the interfaces in `strategies/base.py` and register it in `SearchService`. Then add a request schema in `schemas/search.py` and a route in `routes/search.py`.

### Repository layer (`app/repositories/`)

Wrappers around the data on disk. They are created once and kept for the lifetime of the server.

| Module | Responsibility |
|--------|----------------|
| `IndexRepository` | Loads and searches the vector indexes, per collection and index. An index is loaded the first time it's used, unless it's preloaded at startup. |
| `DatabaseRepository` | Opens each collection's database at startup and runs its queries |
| `db_helper.py` | Turns a filter tree into one SQL query. See [Filters](/filters/#how-filters-are-applied). |

### Core layer (`app/core/`)

Infrastructure and cross-cutting concerns.

| Module | Responsibility |
|--------|----------------|
| `config.py` | Reads `data/config.toml` and validates it with Pydantic. See [Configuration](/configuration/). |
| `models.py` | `CLIPModelManager` (open_clip) and `TextModelManager` (sentence-transformers), which load the text encoders, and `ApplicationContainer` |
| `indexes.py` | The index interface `BaseIndex` and its implementations `FaissIndex` and `ZarrIndex` |
| `exceptions.py` | The exception classes, all based on `LSEException` |

## Dependency injection

`ApplicationContainer` holds the shared parts of the engine: the configuration, the two model managers and the two repositories. There is one container, `container` in `app/core/models.py`. Each part is created the first time it's needed.

At startup, the FastAPI lifespan in `main.py` loads everything through the container and starts the logging service:

```python
# main.py, simplified
@asynccontextmanager
async def lifespan(app: FastAPI):
    container.initialize()  # models, databases and preloaded indexes
    audit_logger = AuditLogger(f"{log_directory}/audit_{start_time}.log")
    app.state.logging_service = LoggingService(audit_logger)
    await app.state.logging_service.start()
    yield
    await app.state.logging_service.stop()
```

Routes get their services through `Depends()`, from the functions in `app/api/dependencies.py`:

```python
# routes/search.py, simplified
@router.post("/clip")
async def clip_search(
    request: TextSearchRequest,
    background_tasks: BackgroundTasks,
    search_service: SearchService = Depends(get_search_service),
    logging_service: LoggingService = Depends(get_logging_service),
):
    result = await search_service.search_text("clip", request)
    background_tasks.add_task(logging_service.log_search_request, ...)
    return {"suggestions": result["suggestions"]}
```

## Request lifecycle

For a CLIP search request, the flow is:

```
POST /exq/search/clip
  │
  ├─ FastAPI validates the TextSearchRequest
  │
  ├─ SearchService.search_text("clip", request)
  │     ├─ Note the start time
  │     ├─ CLIPSearchStrategy.search(...)
  │     │     ├─ Encode the text with the index's CLIP model, in a worker thread
  │     │     ├─ Skip seen items, excluded items and their groups
  │     │     ├─ With filters and at most 10,000 items left:
  │     │     │     score those items exactly (IndexRepository.search_subset)
  │     │     ├─ Otherwise: search the index, also skipping filtered items
  │     │     │     (IndexRepository.search)
  │     │     └─ Turn index positions into media IDs
  │     │           (DatabaseRepository.get_media_ids)
  │     └─ Note the completion time
  │
  ├─ Background task: LoggingService.log_search_request(...)
  │
  └─ Return {"suggestions": [...]}
```

## Exception handling

Errors return a JSON body with a message:

```json
{ "detail": "Text search failed with strategy clip: CLIP search failed: No index named 'my_index' configured for this collection" }
```

Routes handle the errors they expect. For example, a failed search returns `400`, and an unexpected error returns `500`. The status codes of each endpoint are listed in the [API reference](/api/search/#responses-and-errors).

An `LSEException` that a route doesn't handle is caught by a global handler in `main.py`. It returns `400` and writes the error to the audit log. The exception classes are in `app/core/exceptions.py`.

## Startup sequence

1. Read and validate `data/config.toml`.
2. Pick the device from `model_device`. With `"auto"`, that's CUDA if available, then MPS, then CPU.
3. Load the models of each collection's default index, and of all its indexes if `preload_all_indexes = true`. See [Model cache and running offline](/install/#model-cache-and-running-offline).
4. For each enabled collection, open its database and load its default index. With `preload_all_indexes = true`, load all its indexes.
5. Start the logging service, with a new `audit_<start time>.log` file in the `[logging]` directory.
6. Start accepting requests.

Models and indexes that aren't loaded at startup are loaded the first time a request uses them.

## Shutdown sequence

On shutdown, the logging service writes the audit entries that are still waiting, then stops. Everything else is released when the process exits.
